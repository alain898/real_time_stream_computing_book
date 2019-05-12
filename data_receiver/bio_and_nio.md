## BIO与NIO
使用Spring Boot开发数据接收服务器的过程轻松愉快，测试上线一帆风顺，似乎一切都很美好。
但上线生产环境后，晴朗天空悠悠然飘来两朵"乌云"。

1. 随着用户越来越多，接收服务器连接数逐渐增加，甚至在高峰时出现成千上万并发连接的情况。
每个连接的服务质量急剧下降，不时返回408或503错误。
2. doExtractCleanTransform比较耗时，可能是计算比较复杂，也可能是有较多的IO操作，还可能是有较多的外部服务。
接收服务器性能表现很差，但是看系统监控又发现CPU和IO的使用效率并不高。

我们先看接收服务器连接的问题。
当使用Spring Boot做Web服务开发时，默认情况下Spring Boot使用Tomcat容器。早期版本的Tomcat默认使用BIO连接器。
虽然在现在的版本中已经去掉BIO连接器，并默认采用NIO连接器，但是我们还是来比较下BIO和NIO连接器的区别，
这样对理解BIO和NIO、同步和异步的原理，以及编写高性能程序都有很大的帮助。

### BIO
当Tomcat使用BIO连接器时，初始化过程中创建一批worker线程，并用栈的方式管理这些worker线程。
在acceptor接收到一个socket后，从栈中取出一个worker，并将socket交给该worker由其全权处理后续事宜。
然后worker从socket中读取数据，并进行相关处理，再将结果写回socket，最终将socket关闭。至此完成一次完整的请求处理。

<div align="center">
<div style="text-align: center; font-size:50%">图2.2 tomcat-bio原理</div>
<img src="../images/img2.2.tomcat-bio原理.png" width="80%"/>
</div>

不过这个过程在一些情况下会存在问题。

考虑worker处理较慢的情况，比如计算逻辑较复杂或外部IO较多。
那么当所有worker都在干活时，可用worker耗尽，这时acceptor将阻塞在等待worker可用，而不能接收新的socket。
当worker完成socket处理后，由于没有立即可用的新socket作处理，它必须等到acceptor接收新的socket之后，才能继续工作。

经过以上分析就会发现，这种处理方案的性能表现会比较低下。
一方面acceptor和worker都很忙碌，而另一方面acceptor和worker却要时不时的相互等待。
这就导致CPU和网络IO很多时候处于空闲状态，资源大量空闲浪费，性能却还不如人意。

#### BIO提升性能的方法
为了在使用BIO连接器时提高资源的使用效率，一种行之有效的方法是增加worker数量。
理想情况下，如果有成千上万甚至上百万个worker来处理socket，
那么acceptor再也不用担心worker不够用，因为任何时候总会有worker可用。
这样，数据接收服务器的并发连接数也能够达到成千上万（至于百万并发连接，需要特别配置操作系统，这里就不讨论了）。

虽然理想很丰满，现实却很骨感。当前大多数操作系统，在处理上万个甚至只需大几千个线程时，性能就会变得低下。
操作系统在做线程调度和上下文切换时，占用大量CPU时间，使得真正用于任务计算的有效CPU时间变少。
每个线程拥有自己独立的线程栈，当线程过多时，还需要大量内存。
不过多启动一些线程还是有好处的，大量线程触发IO任务，使IO资源的利用更充分。

既然不能在一台机器上运行太多线程，我们很自然地想到可以用多台机器来分担计算任务。
不错，这是个很好的办法。在多个对等的服务节点之前，架设一个负载均衡器比如Nginx，可以有效地将请求分发到多台服务器。
但我们不能立刻这样做。作为有极客精神的程序员，同时也是为了节约成本着想，
在将一台机器的资源充分榨干前，不能简单地寄希望于通过扩充机器的方式来提高处理性能。

进一步思考，如果acceptor无阻塞地接收socket，将新接收的socket暂存到缓冲区。
当worker在处理完一个socket后，从缓冲区取出新的socket进行处理。
这样acceptor可以不停地接收新socket，而worker们的任务也被安排得满满当当。

因此，BIO连接器的本质缺陷是acceptor和worker执行步调耦合太紧。
如果将acceptor和worker通过缓存区隔离开来，让它们互不干涉地独立运行，
那么CPU和IO资源的使用效率都会得到提升，进而提高了程序性能。

接下来我们将看到，Tomcat的NIO连接器正是这样做的。

### NIO
在本书写作时，最新版本的Tomcat已经将NIO作为默认连接器。
当Tomcat在使用NIO连接器时，初始化过程启动acceptor线程和poller线程。
在acceptor接收新socket后，将socket放入poller的输入队列。
之后poller从其输入队列取出socket，并用一个selector对象管理这些socket，
当有socket可读取时，就将其挑选出来交由worker处理。
新的worker管理机制不再使用栈，而是采用executor这种队列加线程池的方式。

<div align="center">
<div style="text-align: center; font-size:50%">图2.3 tomcat-nio原理</div>
<img src="../images/img2.3.tomcat-nio.png" width="80%"/>
</div>

从上面的过程可以看出，Tomcat的NIO连接器和BIO连接器，
在acceptor接收socket以及worker处理socket这两步时，本质是完全相同的，都采用了阻塞IO。
但是NIO连接器通过中间的poller将acceptor和worker这两者的执行隔离开来，不让它们彼此之间因为对方阻塞而影响自己的连续运行。
这样acceptor和worker都能尽其所能地工作，从而更加充分地使用CPU和IO资源。
同时，因为有了poller缓存待处理的socket，NIO连接器能够保持的并发连接数也就不再受限于worker数量，而只受限于poller能够缓存的socket数量。
这样，无须分配大量线程，数据接收服务器就能支持大量并发连接。

### 总结
本节对比了BIO连接器和NIO连接器的工作原理。
NIO连接器将socket接收和处理的过程分开，使服务器使用少量线程就可以支撑大量并发连接。
至此也就解决了两朵"乌云"中的第一朵问题。
在下节中，我们将结合NIO和异步编程，解决另一朵"乌云"问题。
