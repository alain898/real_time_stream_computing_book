## 采用CompletableFuture实现单节点流处理

在理解了CompletableFuture的工作原理后，我们就可以开始对前面的流计算过程做改造了。
事实上，在CompletableFuture框架的支持下，新的流计算过程非常简单。

### 采用CompletableFuture的流计算实现

闲话少叙，咱们先直接上代码：

```
byte[] event = receiver.receive();
CompletableFuture
        .supplyAsync(() -> decoder.decode(event), decoderExecutor)
        .thenComposeAsync(extractor::extract, extractExecutor)
        .thenAcceptAsync(sender::send, senderExecutor)
        .exceptionally(e -> {
            logger.error("unexpected exception", e);
            return null;
        });
```

看，是不是非常简单！
在上面的代码中，我们先从receiver中读取消息，然后通过supplyAsync方法将这条消息放到了"流水线"上。
流水线的第一道工序是解码（decode），负责这道工序的工作小组是decoderExecutor。
消息在解码完成后再进行特征提取（extract），负责特征提取的工作小组是extractExecutor。
由于特征提取这道工序内部又有自己的"小流水线"（即实现特征并行计算时使用的fork/join结构），
故采用了thenComposeAsync将这个小流水线嵌入到整体的流水线中来。
流水线的最后一道工序是发送（send）到消息中间件kafka中去，
因为不需要后续处理了，所以是使用thenAcceptAsync来吞掉这条消息。

每个小组的人员数（即计算资源）均可以根据工序的繁忙程度和资源类型进行合理安排（即设置executor的各种参数）。

相比之前我们自己造的流计算轮子，使用CompletableFuture的流计算实现具有以下优点：
1. 在构建DAG拓扑时，仅需选择合适的CompletableFuture方法。相比在前面的实现方案中，像是在逐点逐线地画拓扑，这种方法要简洁和方便很多。
2. 在实现DAG节点时，仅需将相关逻辑实现为一个回调函数。而我们之前实现的方案中在实现回调时或多或少还牵涉了框架的内部逻辑，对回调实现者并不直观。
3. 能够静态或动态地控制DAG节点并发度和资源使用量，要实现这点只需要设置相应的executor即可，参数设置也更加灵活。
4. 更方便地实现流的优雅关闭（graceful shutdown）。

### 需要注意的点

我们已经用CompletableFuture框架（注意笔者在本书把CompletableFuture视为一个框架，而不仅仅是一个工具类或库）
非常方便地实现了一个流计算的过程。但是很多时候，这个世界上并没有"简单"的事情。
使用CompletableFuture框架也需要注意以下问题。

#### 反向压力
反向压力的问题我们在第二章中已经讲解过，但是这里还是得再次强调下。
因为这是流计算系统和异步系统的阿喀琉斯之踵。在实际生产环境，不考虑反向压力的流计算或异步系统毫无用处。
考虑我们前面的演示代码片段，如果特征提取（extract）得较慢，而数据接收和解码很快，这样会出现什么情况？
毫无疑问，如果没有反向压力，数据就会不断地在JVM内积累起来，直到JVM抛出OOM灾难性错误，最终JVM崩溃退出。
而且实际上这种上下游之间速度不一致的情况随处可见，所以不处理好反向压力的问题，系统时刻都有着OOM的隐患。
那怎样在CompletableFuture框架中加入反向压力的能力呢？
其实也很简单，只需在使用CompletableFuture的各种异步（以Async结尾）API时，使用的executor具备反向压力即可。
也就是说，executor的execute方法能够在资源不足时，阻塞执行直到资源可用为止。

在第二章中我们实现的MultiQueueExecutorService类就是实现了反向压力的executor。
具体实现读者可以参见第二章，这里不再详述。

而在本节的CompletableFuture流计算实现中，使用的executor就是MultiQueueExecutorService类。

```
private final ExecutorService decoderExecutor = new MultiQueueExecutorService(
        "decoderExecutor", 1, 2, 1024, 1024, 1);
private final ExecutorService extractExecutor = new MultiQueueExecutorService(
        "extractExecutor", 1, 4, 1024, 1024, 1);
private final ExecutorService senderExecutor = new MultiQueueExecutorService(
        "senderExecutor", 1, 2, 1024, 1024, 1);
private final ExecutorService extractService = new MultiQueueExecutorService(
        "extractService", 1, 16, 1024, 1024, 1);
```

如此，我们即实现了流计算的反向压力功能。

#### 死锁
从某种意义上来说，流（或异步）这种方式是最好的并发方式，因为采用这种方式编写程序，会自然的避免掉"锁"的使用。
当使用流时，被处理的对象依次从上游流到下游。
当对象在某个步骤被处理时，它是被这个步骤的线程池中的某个线程唯一持有，因此不存在对象竞争的问题。
但是，这是不是就说不会出现死锁的问题呢？不是的。
考虑某个流依次有A和B这两个步骤，并且具备反向压力能力。
如果A的输出已经将B的输入队列占满，而B的某些输出又需要重新流向B的输入队列，
那么由于反向压力的存在，B会一直等待其输入队列有空间可用，而B的输入队列又因为B在等待，永远也不会有空间被释放。
这样，就形成了一个死锁的问题，程序会一直卡死下去。
当然，真实的场景下不大会出现这种B的输出重新作为B的输入的问题，但是会有另外一种类似的情况，
就是给多个不同的步骤，分配同一个executor，这样同样会出现死锁问题。
不过话说回来，只要我们避免输出重新流回输入和不同步骤使用同一个executor的问题，
就可以用流这种方式开开心心地编写程序，而不用考虑锁的问题了。
这一方面简化了我们的程序设计，无需考虑竞态（race condition），另一方面也给程序带来了性能的提升。


### 再论流与异步的关系
在第二章的结尾处，我们简单地提及到流与异步的相似性。
而在本节，我们就完全使用CompletableFuture这个异步编程框架，实现了一个流计算的过程。

异步和流本就是相通的。异步是流的本质，流是异步的一种表现形式。

在上一节中我们自行实现的流计算框架，主要工作机制是service线程从其输入队列中读取数据进行处理，然后输出到下游service线程的输入队列。
而在本节中我们使用CompletableFuture异步框架，选择带BlockingQueue做任务队列的ExecutorService做任务处理。
这两种实现的工作原理在本质上不是完全一致吗？它们都是一组线程从其输入队列中读取消息进行处理，然后输出给下游的输入队列。
而这正是"流计算"的主要模式。

当然，CompletableFuture异步框架也可以选择其它类型的executor。
比如使用栈管理线程，每次execute方法被调用时，就从栈中取出一个线程来执行任务的executor。
当使用这种不带任务队列的executor时，就和我们的流计算模式相差较远了。
这也是笔者为什么说流是异步的一种表现形式的原因之一。

不过异步和流在表现层却有着非常重要的区别，有时候这些区别会直接影响到我们的设计思路和方案选择。
异步的着重点在回调。当业务逻辑较多时，不可避免地会使回调嵌套层次较深，使得程序设计和实现都会变得很复杂。
而流的着重点在业务流程。即使业务流程再多，也无非是在"流水线"上多加几个步骤而已。
异步和流是两种不同的思考问题的方法。大多数情况下，面对复杂的业务流程做设计时，使用流这种思路会使得设计和实现都会更加清晰易懂。
总而言之，异步会让事情变得复杂，而流会让事情变得简单。
当再次碰到程序设计的问题时，读者不妨试着按照流这种方式来考量一翻，说不定就会有意外的惊喜。


