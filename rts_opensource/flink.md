## Apache Flink
随着流计算领域的不断发展，关于流计算的理论和模型逐渐清晰和完善。Flink就是这些流计算领域最新理论和模型的优秀实践。
相比Spark在批处理领域的火爆流行，Apache Flink(后简称Flink)可以说是目前流计算领域最耀眼的新贵了。
Flink是一个分布式流处理和批处理平台。但是相比Spark偏向于批处理，Flink的核心是流计算引擎。

### 系统架构
Flink是一个主从（Master/Worker）架构的分布式系统。
主节点负责调度流计算作业，管理和监控任务执行。
当主节点从客户端接收到作业相关的JAR包和资源后，进行分析和优化，生成执行计划，也就是需要执行的任务，
然后将相关的任务分配给各个Worker，由Worker负责任务的具体执行。

Flink可以部署在诸如Yarn、Mesos和K8s等分布式资源管理器上，
其整体架构与Storm和Spark Streaming等分布式流计算框架也是类似的。

但是与这些流计算框架不同的是，Flink明确地把状态管理（尤其是流信息状态管理）纳入到了其系统架构中。

<div align="center">
<img src="../images/img7.2.SparkStreaming系统架构图.png" width="50%"/>
<div style="text-align: center; font-size:50%">img7.2.SparkStreaming系统架构图</div>
</div>

在Flink计算节点执行任务的过程中，可以将状态保存到本地。
然后通过checkpoint机制，再配合诸如HDFS、S3和NFS这样的分布式文件系统，
Flink在不降低性能的同时，实现了状态的分布式管理。


### 流的描述
在Flink中用DataStream来描述数据流。DataStream在Flink中扮演的角色犹如Spark中的RDD。
值得一提的是，Flink也支持批处理DataSet的概念，不过DataSet内部同样是由DataStream构成。
Flink中这种将批处理视为流处理特殊情况的做法，与Spark Streaming中将流处理视为连续批处理的做法截然相反。

Flink的数据输入（Source）、处理（Transformation）和输出（Sink）均与DataStream有关。

Source：用于描述Flink流数据的输入源，输入的流数据就表示为DataStream。
Flink的Source可以是消息中间件、数据库、文件系统或其它各种数据源。

Transformation：将一个或多个DataStream转化为一个新的DataStream，是Flink实施流处理逻辑的地方。
目前，Flink提供Map、FlatMap、Filter、KeyBy、Reduce、Fold、Aggregations、Window、Union、
Join、Split、Select和Iterate等类型的Transformation。

Sink：是Flink将DataStream输出到外部系统的地方，比如写入到控制台、数据库、文件系统或消息中间件等。


### 流的执行
我们从流的输入、流的处理、流的输出和反向压力四个方面来讨论Flink中流的执行过程。

#### 流的输入
Flink使用StreamExecutionEnvironment.addSource设置流的数据源Source。
为了使用方便，Flink在StreamExecutionEnvironment.addSource的基础上提供了一些内置的数据源实现。
StreamExecutionEnvironment提供的输入方式，主要包含了四类：

* 基于文件的输入。从文件中读入数据作为流数据源，比如readTextFile和readFile等。

* 基于套结字的输入。从TCP套接字中读入数据作为流数据源，比如socketTextStream等。

* 基于集合的输入。用集合作为流数据源，比如fromCollection、fromElements、fromParallelCollection和generateSequence等。

* 自定义输入。StreamExecutionEnvironment.addSource是最通用的流数据源生成方法，用户可以其为基础开发自己的流数据源。
一些第三方数据源，比如flink-connector-kafka中的FlinkKafkaConsumer08就是针对Kafka消息中间件开发的流数据源。

Flink将从数据源读出的数据流表示为DataStream。下面的示例演示了从TCP连接中构建文本数据输入流的过程。

```
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStream<String> lines = env.socketTextStream("localhost", 9999);
```

#### 流的处理
Flink对流的处理，是通过DataStream的各种转化操作完成的。
相比Spark中DStream的转化操作混淆了流数据状态管理和流信息状态管理，
Flink的设计思路更加清晰，明确地将流信息状态从流数据状态的管理分离出去。
DataStream转换操作只包含了两类操作，一类是常规的流式处理操作，比如map、filter、reduce、count、transform等。
另一类是流数据状态相关的操作，比如union、join、cogroup、window等。
这两类操作都是针对流本身的处理和管理。
从设计模式中单一职责原则的角度来看，Flink关于流的设计显然更胜一筹。

下面是一个对DataStream进行转化操作的例子。
```
DataStream<Tuple2<String, Integer>> words = lines.flatMap(new Splitter());
KeyedStream<Tuple2<String, Integer>, Tuple> keyedWords = words.keyBy(0);
WindowedStream<Tuple2<String, Integer>, Tuple, TimeWindow> windowedKeyedWords =
        keyedWords.timeWindow(Time.seconds(5));
DataStream<Tuple2<String, Integer>> wordCounts = windowedKeyedWords.sum(1);
```
在上面的例子中，先将从socket中读出的文本流`lines`，对每行文本分词后，用`flatMap`转化为单词计数元组流`pairs`。
然后用`keyBy`将计数元组流`pairs`以分组第一个元素（即word）进行分组，形成分组的计数元组流`keyedPairs`。
最后用`timeWindow`以5秒钟为时间窗口对分组后的流进行划分，并在窗口上进行sum聚合计算，最后得到`wordCounts`，
即每五秒各个单词出现的次数。


#### 流的输出
Flink使用DataStream.addSink设置数据流的输出方法。
同时Flink在DataStream.addSource的基础上提供了一些内置的数据流输出实现。
DataStream提供的输出API主要包含了四类：

* 输出到文件系统。将流数据输出到文件系统，比如writeAsText、writeAsCsv和writeUsingOutputFormat。

* 输出到控制台。将数据流打印到控制台，比如print和printToErr。

* 输出到套接字。将数据流输出到TCP套接字，比如writeToSocket。

* 自定义输出。DataStream.addSink是最通用的流数据输出方法，用户可以其为基础开发自己的流数据输出方法。
比如flink-connector-kafka中的FlinkKafkaProducer011就是针对Kafka消息中间件开发的流输出方法。

下面的示例演示了将DataStream表示的流数据打印到控制台。

```
dataStream.print();
```


#### 反向压力
Flink对反向压力的支持非常好，不仅实现了反向压力功能，还直接内置了反向压力的监控功能。
Flink采用有限容量的分布式阻塞队列来进行数据传递，当下游任务从消费队列中的消息过慢时，
就非常自然地减慢了上游任务往队列中写入消息的速度。
这种反向压力的实现思路，和我们在前面章节中使用JDK自带的BlockingQueue实现反向压力的方式完全一致。

值得一提的是，与Storm和Spark Streaming需要明确打开开关才能使用反向压力功能不一样的是，
Flink的反向压力功能是天然地包含在了其数据传送方案内的，不需特别再去实现，使用时也无需特别打开开关。
这与前面章节中我们做出的"没有反向压力功能的流计算框架根本不可用"之论断，是完全相符的。


### 流的状态
Flink是第一个明确地将流信息状态管理从流数据状态管理剥离开来的流计算框架。
大多数其它的流计算框架要么没有流信息状态管理，要么实现的流信息状态管理非常有限，
要么流信息状态管理混淆在了流数据状态管理中，使用起来并不方便和明晰。

#### 流数据状态
Flink有关流数据状态的管理，都集中在DataStream的转化操作上。
这是非常合理的，因为流数据状态管理本身是属于流转化和管理的一部分。
比如，流按窗口分块处理、多流的合并、事件乱序处理等功能的实现，
虽然也涉及数据缓存和有状态操作，但这些功能原本就应该由流计算引擎来处理。

DataStream上与窗口管理相关的API包括Window和WindowAll。
其中Window是针对KeyedStream，而WindowAll是针对非KeyedStream。
在窗口之，则提供了一系列窗口聚合计算的方法，比如Reduce、Fold、Sum、Min、Max和Apply等。

DataStream提供了一系列有关流与流之间计算的操作，比如Union、Join、CoGroup和Connect等。

另外，DataStream还提供了非常有特色的KeyedStream。所谓KeyedStream是指将流按照指定的键值，
在逻辑上分成多个独立的流。这些逻辑流在计算时，状态彼此独立、互不影响，
但是在物理上这些独立的流可能是合并在同一条物理的数据流中。
因此在KeyedStream具体实现时，Flink会在处理每个消息前，将当前运行时上下文切换到key值所指定流的上下文。
就像线程栈的切换那样，这样优雅地避免了不同逻辑流在运算时的相互干扰。

#### 流信息状态
Flink对流信息状态管理的支持，是其相比当前其它流计算框架更显优势的地方。
Flink在DataStream之外，提供了独立的状态管理接口。
可以说，实现流信息状态管理，并将其从流本身的管理中分离出来，是Flink在洞悉流计算本质后的明智之举。
因为，如果说DataStream是对数据在时间维度的管理，那么状态接口其实是对数据在空间维度的管理。
Flink之前的流数据框架对这两个概念的区分可以说并不是非常明确，这也导致它们关于状态的设计不是非常完善，甚至根本没有。

在Flink中有两种类型的状态接口：Keyed state和Operator state。
它们既可以用于流信息状态管理，也可以用于流数据状态管理。

* Keyed State
Keyed State与KeyedStream相关。前面提到，KeyedStream是对流按照key值做出的逻辑划分。
每个逻辑流都有自己的上下文，就像每个线程都有自己的线程栈一样。
当我们需要在逻辑流中记录一些状态信息时，就可以使用Keyed State。
比如"统计不同IP上出现的不同设备数"，就可以用将流按照IP分成KeyedStream，
这样来自不同IP的设备事件，会分发到不同IP独有的逻辑流中。然后在逻辑流处理过程中，
使用KeyedState来记录不同设备数。如此一来，就非常方便地实现了"统计不同IP上出现的不同设备数"的功能。


* Operator State
Operator State与算子有关。其实与Keyed State的设计思路非常一致，
Keyed State是按key划分状态空间，而Operator State是按照算子的并行度划分状态空间。
每个Operator State绑定到算子的一个并行实例上，因而这些并行实例在执行时可以维护各自的状态。
这有点像线程局部量，每个线程都维护自己的一个状态对象，在运行时互不影响。
比如当Kafka Consumer在消费同一个topic的不同partition时，可以用Operator State来维护各自消费partition的offset。

在Flink 1.6版本中，Flink引入了状态生存时间值（State Time-To-Live），这为实际开发中，淘汰过期的状态提供了极大的便利。
不过美中不足的是，Flink虽然针对状态存储提供了TTL机制，但是TTL本身实际是一种非常底层的功能。
如果Flink能够针对状态管理也提供诸如窗口管理这样的功能，会使Flink的流信息状态管理会更加完善和方便。


### 消息传达可靠性
Flink基于snapshot和checkpoint的故障恢复机制，在Flink内部提供了exactly-once的语义。
当然，得到这个保证的前提是，在Flink应用中保存状态时必须使用Flink内部的状态机制，比如Keyed State和Operator State。
因为这些Flink内部状态的保存和恢复都是包含在Flink的故障机制内，由系统保证了状态的一致性。
如果使用不包含在Flink故障恢复机制内的方案存储状态，比如用另外独立的Redis记录PV/UV统计状态，
那么就不能获得exactly-once级别的可靠性保证，而只能实现at-least-once级别的可靠性保证了。

要想在Flink中，实现从数据流输入到输出之间，端到端的exactly-once数据传送，
还必须有Flink connectors配合才行。不同的connectors提供了不同级别的可靠性保证。
比如在Source端，Apache Kafka提供了exactly once保证，Twitter Streaming API提供了at most once保证。
在Sink端，HDFS rolling sink提供了exactly once保证，Kafka producer则只提供了at least once的保证。


