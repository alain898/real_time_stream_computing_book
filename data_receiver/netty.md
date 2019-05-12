## Netty
Netty是一个基于NIO的异步事件驱动网络应用框架，用于快速开发具有可维护性的高性能协议服务器和客户端。
为了更好的讲解如何结合NIO和异步编程以实现支持高并发、高性能的数据采集服务器，本节采用Netty框架重构数据采集服务器。

### 用Netty实现数据采集服务器
下面是Netty实现数据采集服务器的主要逻辑。

```
public static void main(String[] args) {
    final int port = 8081;
    final EventLoopGroup bossGroup = new NioEventLoopGroup();
    final EventLoopGroup workerGroup = new NioEventLoopGroup();
    try {
        final ServerBootstrap bootstrap = new ServerBootstrap()
                .group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ServerInitializer())
                .option(ChannelOption.SO_BACKLOG, 1024);

        final ChannelFuture f = bootstrap.bind(port).sync();
        logger.info(String.format("NettyDataCollector: running on port[%d]", port));

        f.channel().closeFuture().sync();
    } catch (final InterruptedException e) {
        logger.error("NettyDataCollector: an error occurred while running", e);
    } finally {
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}

public class ServerInitializer extends ChannelInitializer<SocketChannel> {

    private static final int MAX_CONTENT_LENGTH = 1024 * 1024;

    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast("http-codec", new HttpServerCodec());
        ch.pipeline().addLast(new HttpObjectAggregator(MAX_CONTENT_LENGTH));
        ch.pipeline().addLast("handler", new ServerHandler());
    }
}

public class ServerHandler extends
        SimpleChannelInboundHandler<HttpRequest> {
    private static final Logger logger = LoggerFactory.getLogger(NettyDataCollector.class);

    private final String kafkaBroker = "127.0.0.1:9092";
    private final String topic = "collector_event";
    private final KafkaSender kafkaSender = new KafkaSender(kafkaBroker);

    private JSONObject doExtractCleanTransform(JSONObject event) {
        // TODO: 实现抽取、清洗、转化具体逻辑
        return event;
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, HttpRequest req)
            throws Exception {

        byte[] body = readRequestBodyAsString((HttpContent) req);
        // step1: 对消息进行解码
        JSONObject bodyJson = JSONObject.parseObject(new String(body, Charsets.UTF_8));

        // step2: 对消息进行抽取、清洗、转化
        JSONObject normEvent = doExtractCleanTransform(bodyJson);

        // step3: 将格式规整化的消息发到消息中间件kafka
        kafkaSender.send(topic, normEvent.toJSONString().getBytes(Charsets.UTF_8));

        // 通知客户端数据采集成功
        sendResponse(ctx, OK, RestHelper.genResponse(200, "ok").toJSONString());
    }
}
```

在上面的代码中，使用HttpServerCodec将接收的数据按照HTTP协议格式解码，解码后的数据再交由ServerHandler处理。
针对消息的解码、ECT、发送至消息中间件等过程和Spring Boot实现相似。


### 使用异步编程
Netty实现的数据接收服务器内置使用了NIO，但是不是这样就让CPU和IO的能力彻底释放出来了呢？这可不一定。
仔细查看前面的实现过程会发现，虽然我们采用NIO使请求的处理和请求的接收分离开来，但是在处理请求的时候依然使用的是同步方式。
也就是说，对消息解码、ECT、发送至消息中间件以及最终返回客户端结果这几个步骤，都是在同一个线程里依次顺序执行完成。
通过上一节有关CPU和IO密集性问题的讨论，我们知道在CPU和IO都很密集的情况下，结合NIO和异步编程才能尽可能同时提高CPU和IO的使用效率。

下面就来看看如何使用异步编程的方式，改进前面用Netty实现的数据接收服务器。
在Netty中可以很方便地将worker执行请求从同步方式改成异步方式，具体如下。

```
private static class RefController {
    private final ChannelHandlerContext ctx;
    private final HttpRequest req;

    public RefController(ChannelHandlerContext ctx, HttpRequest req) {
        this.ctx = ctx;
        this.req = req;
    }

    public void retain() {
        ReferenceCountUtil.retain(ctx);
        ReferenceCountUtil.retain(req);
    }

    public void release() {
        ReferenceCountUtil.release(req);
        ReferenceCountUtil.release(ctx);
    }
}

protected void channelRead0(ChannelHandlerContext ctx, HttpRequest req)
        throws Exception {
    logger.info(String.format("current thread[%s]", Thread.currentThread().toString()));
    final RefController refController = new RefController(ctx, req);
    refController.retain();
    CompletableFuture
            .supplyAsync(() -> this.decode(ctx, req), this.decoderExecutor)
            .thenApplyAsync(e -> this.doExtractCleanTransform(ctx, req, e), this.ectExecutor)
            .thenApplyAsync(e -> this.send(ctx, req, e), this.senderExecutor)
            .thenAccept(v -> refController.release())
            .exceptionally(e -> {
                try {
                    logger.error("exception caught", e);
                    sendResponse(ctx, INTERNAL_SERVER_ERROR,
                            RestHelper.genResponseString(500, "服务器内部错误"));
                    return null;
                } finally {
                    refController.release();
                }
            });
}
```

在上面的代码中，`channelRead0()`函数的输入参数`ChannelHandlerContext`实例和`HttpRequest`实例，
是在每次接收socket连接请求时新建的对象。这些对象不是全局的，可以在不同线程间自由地传递和使用它们，并且不用担心并发安全的问题。
接下来，我们将处理步骤细分成解码、ECT和发送到消息队列三个步骤，然后使用`CompletableFuture`让这三个步骤形成异步执行链。
由于Netty会对其使用的部分对象进行分配和回收管理，在`channelRead0`方法返回时，Netty框架会立刻释放`HttpRequest`对象。
而`channelRead0`方法将请求提交异步处理后立刻返回，此时请求处理可能尚未结束。
因此，在将请求用提交异步处理之前，必须先调用`refController.retain()`来保持对象，
而在请求处理完后，再调用`refController.release()`来释放`HttpRequest`对象。


### 流量控制和反向压力
上面的改造已经将worker的执行过程彻彻底底异步化。至此CPU和IO都可以毫无阻碍地尽情干活，它们的生产力得到充分解放。
但是，有关异步的问题还没有彻底解决。

通过将worker执行异步化后，socket接收线程可以不停接收新的socket请求，并将其交给worker处理。
由于worker使用了异步执行方案，它对socket的处理其实是进一步交给了各个步骤的executor。
在这整个过程中没有任何阻塞的地方，只不过各个步骤待处理的任务都被隐式地存放在了各个executor的任务队列中。
如果各executor处理得足够快，那么它们的任务队列都能被及时消费，这样不会存在问题。
但是，一旦有某个步骤的处理速度比不上socket接收线程接收新socket的速度，那么必定有部分executor任务队列中的任务会不停增长。
由于executor任务队列默认是非阻塞且不限容量的，这样当任务队列里积压的任务越来越多时，
终有一刻，JVM的内存会被耗尽，抛出OOM系统错误后程序异常退出。

实际上，这也是所有异步系统非常普遍、而且必须重视的问题。
在纤程里，可以通过指定最大纤程数量来限制内存的使用量，非常自然地控制了内存和流量。
但是在一般的异步系统里，如果不对执行的各个环节做流量控制，那么就很容易出现前面所说的OOM问题。
因为当每个环节都不管其下游环节处理速度是否跟得上，不停将其输出塞给下游的任务队列时，
只要上游输出速度超过下游处理速度的状况持续一段时间，
必然会导致内存不断被占用，直至最终耗尽，抛出OOM灾难性系统错误。

为了避免OOM问题，我们必须对上游输出给下游的速度做流量控制。
一种方式是严格控制上游的发送速度，比如每秒控制其只能发1000条消息。
但是这种粗糙的处理方案会非常低效。
比如如果实际下游能够每秒处理2000条消息，那上游每秒1000条消息的速度就使得下游一半的性能没发挥出来。
再比如如果下游因为某种原因性能降级为每秒只能处理500条，那在一段时间后同样会发生OOM问题。

更优雅的一种解决方法是被称为反向压力的方案，即上游能够根据下游的处理能力动态调整输出速度。
当下游处理不过来时，上游就减慢发送速度；当下游处理能力提高时，上游就加快发送速度。
反向压力的思想，实际上正逐渐成为流计算领域的共识，
比如与反向压力相关的标准Reactive Streams正在形成过程中。

### 实现反向压力
回到Netty数据接收服务器的实现中来，那该怎样加上反向压力功能呢？

由于socket接收线程接收的新socket及其触发的各项任务，
被隐式地存放在负责各步骤执行的executor的任务队列中，并且executor默认任务队列是非阻塞和不限容量的。
因此要加上反向压力的功能，只需要从两个方面来控制：
1. executor任务队列容量必须有限；
2. 当executor任务队列中的任务已满时，就阻塞上游继续向其提交新的任务，直到任务队列重新有空间可用为止。

按照上面这种思路，我们可以很容易地实现反向压力。
下面正是一个具备多任务队列和多executor，并具备反向压力能力的ExecutorService实现。

```
private final List<ExecutorService> executors;
private final Partitioner partitioner;
private Long rejectSleepMills = 1L;

public MultiQueueExecutorService(String name, int executorNumber, int coreSize, int maxSize,
                            int capacity, long rejectSleepMills) {
    this.rejectSleepMills = rejectSleepMills;
    this.executors = new ArrayList<>(executorNumber);
    for (int i = 0; i < executorNumber; i++) {
        ArrayBlockingQueue<Runnable> queue = new ArrayBlockingQueue<>(capacity);
        this.executors.add(new ThreadPoolExecutor(
                coreSize, maxSize, 0L, TimeUnit.MILLISECONDS,
                queue,
                new ThreadFactoryBuilder().setNameFormat(name + "-" + i + "-%d").build(),
                new ThreadPoolExecutor.AbortPolicy()));
    }
    this.partitioner = new RoundRobinPartitionSelector(executorNumber);
}

@Override
public void execute(Runnable command) {
    boolean rejected;
    do {
        try {
            rejected = false;
            executors.get(partitioner.getPartition()).execute(command);
        } catch (RejectedExecutionException e) {
            rejected = true;
            try {
                TimeUnit.MILLISECONDS.sleep(rejectSleepMills);
            } catch (InterruptedException e1) {
                logger.warn("Reject sleep has been interrupted.", e1);
            }
        }
    } while (rejected);
}

@Override
public Future<?> submit(Runnable task) {
    boolean rejected;
    Future<?> future = null;
    do {
        try {
            rejected = false;
            future = executors.get(partitioner.getPartition()).submit(task);
        } catch (RejectedExecutionException e) {
            rejected = true;
            try {
                TimeUnit.MILLISECONDS.sleep(rejectSleepMills);
            } catch (InterruptedException e1) {
                logger.warn("Reject sleep has been interrupted.", e1);
            }
        }
    } while (rejected);
    return future;
}
```

在上面的代码中，`MultiQueueExecutorService`类在初始化时新建`ThreadPoolExecutor`对象作为实际执行任务的executor。
创建`ThreadPoolExecutor`对象时采用`ArrayBlockingQueue`，这是实现反向压力的关键之一。
将`ThreadPoolExecutor`拒绝任务时采取的策略设置为`AbortPolicy`，
这样在任务队列已满再执行`execute`或`submit`方法时，会抛出`RejectedExecutionException`异常。
在`execute`和`submit`方法中，通过一个`do while`循环，循环体内捕获表示任务队列已满的`RejectedExecutionException`异常，
直到新任务提交成功才退出，这是实现反向压力的关键之二。

接下来就可以在Netty数据接收服务器中，使用这个带有反向压力功能的`MultiQueueExecutorService`了。

```
final private Executor decoderExecutor = new MultiQueueExecutorService("decoderExecutor",
        1, 2, 1024, 1024, 1);
final private Executor ectExecutor = new MultiQueueExecutorService("ectExecutor",
        1, 8, 1024, 1024, 1);
final private Executor senderExecutor = new MultiQueueExecutorService("senderExecutor",
        1, 2, 1024, 1024, 1);


@Override
protected void channelRead0(ChannelHandlerContext ctx, HttpRequest req)
        throws Exception {
    logger.info(String.format("current thread[%s]", Thread.currentThread().toString()));
    final RefController refController = new RefController(ctx, req);
    refController.retain();
    CompletableFuture
            .supplyAsync(() -> this.decode(ctx, req), this.decoderExecutor)
            .thenApplyAsync(e -> this.doExtractCleanTransform(ctx, req, e), this.ectExecutor)
            .thenApplyAsync(e -> this.send(ctx, req, e), this.senderExecutor)
            .thenAccept(v -> refController.release())
            .exceptionally(e -> {
                try {
                    logger.error("exception caught", e);
                    if (RequestException.class.isInstance(e.getCause())) {
                        RequestException re = (RequestException) e.getCause();
                        sendResponse(ctx, HttpResponseStatus.valueOf(re.getCode()), re.getResponse());
                    } else {
                        sendResponse(ctx, INTERNAL_SERVER_ERROR,
                                RestHelper.genResponseString(500, "服务器内部错误"));
                    }
                    return null;
                } finally {
                    refController.release();
                }
            });

}
```

从上面的代码可以看出，我们只需把`taskExecutor`用到的`executor`和`uploadEvent`中各个步骤用到的`executor`，
替换成`MultiQueueExecutorService`，就实现了反向压力功能，其它部分的代码不需要做任何改变！

通过以上改造，当上游步骤往下游步骤提交新任务时，
如果下游处理较慢，上游会停下来等待，直到下游开始接收新任务，上游才能继续提交新任务。
如此一来，上游自动匹配下游的处理速度，最终实现了反向压力功能。

在`MultiQueueExecutorService`的实现中，之所以采用封装多个executor的方式，
是考虑到要使用`M * N`个线程，有下面三种不同的使用场景：
1. 每个executor使用`1`个线程，使用`M * N`个executor
2. 每个executor使用`M * N`个线程，使用`1`个executor
3. 每个executor使用`M`个线程，使用`N`个executor
在不同场景下，三种使用方式的性能表现也会稍有不同。读者如果需要使用这个类，请根据实际场景作出合理设置和必要测试。


### 异步的不足之处
在前面有关异步的讨论中，我们总在"鼓吹"异步比同步更好，能够更有效地使用CPU和IO资源，提高程序性能。
那是不是同步就一无是处，而异步毫无缺点呢？其实不然。
从一开始我们就说过，理论上讲纤程是最完美的线程。
虽然纤程在内部使用了异步机制实现，但基于纤程开发程序只需采用同步的方式即可，完全不需要考虑异步问题。
这说明我们在程序开发的时候，并不是为了异步而异步，而是为了提高资源使用效率、提升程序性能才使用异步的。
如果有纤程这种提供同步编程方式，而且保留非阻塞IO优势的方案，我们大可不必选择异步编程方式。
毕竟通常情况下，异步编程相比同步编程复杂太多，稍有不慎就会出现各种问题，比如资源泄漏和前面提到的反向压力等等。

除了编程更复杂外，异步方式相比同步方式，对于同一个请求的处理也会有更多的额外开销。
这些开销包括，任务在不同步骤（也就是不同线程）之间的辗转，在任务队列中的排队和等待等等。
所以，对于一次请求的完整处理过程，异步方式相比同步方式通常会花费更多的时间。

还有些时候，系统会更加强调请求处理的时延。
这时一定要注意，应该先保证处理的时延能够达到性能指标要求，再在满足时延要求的情况下，尽可能提升每秒处理请求数。
这是因为，在CPU和IO等资源有限的情况下，为了提升每秒处理请求数，会让CPU和IO都尽可能处理忙碌状态。
这就需要使用到类似于异步编程中，用任务队列缓存未完成任务的方法。
这样做的结果是，系统的每秒处理请求数可能通过拼命压榨CPU和IO得到了提升，
但也同时导致了各个环节任务队列中的任务排队过长，增加了请求处理的时延。
因此，如果系统强调的是请求处理的时延，那么异步几乎不会对降低请求处理时延带来任何好处。
这时只能先通过优化算法和IO操作来降低请求处理时延，然后提高并行度以提升系统每秒处理请求数。
提高并行度既可以在JVM内实现，也可以在JVM外实现。
在JVM内增加线程数，直到再增加线程时，处理时延就满足不了时延指标要求为止；
在JVM外则是在多个主机上部署多个JVM进程，直到整个系统的每秒请求处理数，满足TPS指标要求为止。
需要注意的时，在提高并行度的整个过程中，任何时候都必须保证请求处理的时延是满足时延指标要求的。


### 小结
本节在Netty框架基础上，结合NIO和异步编程，实现了完全异步执行的数据接收服务器，并且着重分析了异步编程中反向压力的问题。
虽然异步能够改善程序性能，但也增加了系统的复杂性。在下一小结中，我们将从一个新的角度来审视异步问题。
我们将发现，换一个角度后，原本复杂的异步系统也会变得如此清晰和自然。
