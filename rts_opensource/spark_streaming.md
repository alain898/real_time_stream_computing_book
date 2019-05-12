## Spark Streaming
如今在大数据的世界里，Spark可谓是众所周知，风光无限了。
在批处理领域取得巨大成功后，Spark也开始向流计算领域进军，于是诞生了Spark Streaming。
Spark Streaming是建立在Spark框架上的实时流计算框架，提供了可扩展、高吞吐和错误容忍的实时数据流处理功能。

### 系统架构
Spark Streaming构建在Spark平台上，充分利用了Spark的核心处理引擎。
当Spark Streaming接收实时数据流时，将其分成一个个的RDD，然后由Spark引擎对RDD做各种处理。

<div align="center">
<img src="../images/img7.2.SparkStreaming系统架构图.png" width="50%"/>
<div style="text-align: center; font-size:50%">img7.2.SparkStreaming系统架构图</div>
</div>


### 流的描述
Sparking Streaming中对流计算过程的描述，包含以下核心概念。

RDD：Spark引擎的核心概念，代表了一个数据集合，是Spark进行数据处理的计算单元。

DStream：是Spark Streaming对流的抽象，代表了连续数据流。在系统内部，DStream是由一系列的RDD构成，每个RDD代表了一段间隔内的数据。

Transformation：代表了Spark Streaming对DStream的处理逻辑。目前，DStream提供了很多Transformation相关API，
包括map、flatMap、filter、reduce、union、join、transform和updateStateByKey等等。通过这些API，可以对DStream做各种转换，
从而将一个数据流变为另一个数据流。

Output Operations：是Spark Streaming将DStream输出到控制台、数据库或文件系统等外部系统中的操作。
目前，DStream支持的output operations包括print、saveAsTextFiles、saveAsObjectFiles、saveAsHadoopFiles和foreachRDD。
由于这些操作会触发外部系统访问，所以DStream各种转化的执行实际上是由这些操作触发。


### 流的执行
与Storm类似，我们从流的输入、流的处理、流的输出和反向压力四个方面来讨论Spark Streaming中的流执行过程。

#### 流的输入
Spark Streaming提供了三种创建输入数据流的方式。
* 基础数据源。通过StreamingContext的相关API，直接构建输入数据流。这类API通常是从socket、文件或内存中构建输入数据流。
比如socketTextStream、textFileStream、queueStream等。
* 高级数据源。通过外部工具类从Kafka、Flume、Kinesis等消息中间件或消息源构建输入数据流。
* 自定义数据源。当用户实现了org.apache.spark.streaming.receiver抽象类时，就可以实现一个自定义的数据源了。

Spark Streaming用DStream来表示数据流，所以输入数据流也表示为DStream。下面的示例演示了从TCP连接中构建文本数据输入流的过程。

```
SparkConf conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount");
JavaStreamingContext jssc = new JavaStreamingContext(conf, Durations.seconds(1));
JavaReceiverInputDStream<String> lines = jssc.socketTextStream("localhost", 9999);
```

#### 流的处理
Spark Streaming对流的处理，是通过DStream的各种转化操作API完成的。
DStream的转换操作大体上包含了三类操作，一类是常用的流式处理操作，比如map、filter、reduce、count、transform等。
另一类是流数据状态相关的操作，比如union、join、cogroup、window等。
还有一类是流信息状态相关的操作，目前有updateStateByKey和mapWithState。
下面是一个对DStream进行转化操作的例子。

```
// Count each word in each batch
JavaDStream<String> words = lines.flatMap(x -> Arrays.asList(x.split(" ")).iterator());
JavaPairDStream<String, Integer> pairs = words.mapToPair(s -> new Tuple2<>(s, 1));
JavaPairDStream<String, Integer> wordCounts = pairs.reduceByKey((i1, i2) -> i1 + i2);
```
在上面的例子中，先将从socket中读出的文本流`lines`，对每行文本分词后，用`flatMap`转化为单词流`words`。
然后用`mapToPair`将单词流`words`转化为计数元组流`pairs`。
最后，以单词为分组进行数量统计，通过`reduceByKey`转化为单词计数流`wordCounts`。


#### 流的输出
Spark Streaming允许DStream输出到外部系统，这是通过DStream的各种输出操作完成的。
DStream的输出操作可以将数据输出到控制台、文件系统或数据库等。
目前DStream的输出操作有print、saveAsTextFiles、saveAsHadoopFiles和foreachRDD等。
其中foreachRDD是一个通用的DStream输出接口，用户可以通过foreachRDD自定义各种Spark Streaming输出方式。
下面的例子演示了将单词计数流打印到控制台。

```
wordCounts.print();
```

#### 反向压力
早期版本Spark不支持反向压力，但从Spark 1.5版本开始，Spark Streaming也引入了反向压力功能。
默认情况下Spark Streaming的反向压力功能是关闭的。如需使用，要将`spark.streaming.backpressure.enabled`设置为`true`。
整体而言，Spark的反向压力借鉴了工业控制中PID控制器的思路，其工作原理如下：
1. 当Spark在处理完每批数据时，统计每批数据的处理结束时间、处理时延、等待时延、处理消息数等信息。
2. 根据统计信息估计处理速度，并将这个估计值通知给数据生产者。
3. 数据生产者根据估计出的处理速度，动态调整生产速度，最终使得生产速度与处理速度相匹配。


### 流的状态
Spark Streaming关于流的状态管理，也是在部分DStream提供的转化操作中实现的。

#### 流数据状态
由于DStream本身就是将数据流分成RDD做批处理，所以Spark Streaming天然就需要对数据进行缓存和状态管理。
换言之，组成DStream的一个个RDD，就是一种流数据状态。

在DStream上，提供了一些window相关的转化API，实现了对流数据的窗口管理。在窗口之上还提供了count和reduce两类聚合功能。

另外，DStream还提供了union、join和cogroup三种在多个流之间做关联操作的API。

#### 流信息状态
DStream的updateStateByKey和mapWithState操作提供了流信息状态管理的方法。
updateStateByKey和mapWithState都可以基于key来记录历史信息，并在新的数据到来时，对这些信息进行更新。
不同的是，updateStateByKey会返回记录的所有历史信息，而mapWithState只会返回处理当前一批数据时更新的信息。
就好像，前者是在返回一个完整的直方图，而后者则只是返回直方图中发生变化的柱条。
由此可见，mapWithState比updateStateByKey的性能会优越很多。
而且从功能上讲，如果不是用于报表生成的场景，大多数实时流计算应用中，使用mapWithState也会更合适。


### 消息传达可靠性保证
Spark streaming对消息可靠性的保证，是由数据接收、数据处理和数据输出共同决定的。

从1.2版本开始，Spark引入WAL（write ahead logs）机制，可以将接收的数据先保存到错误容忍的存储上。
当打开WAL机制后，再配合可靠的数据接收器（比如Kafka），Spark Streaming能够提供"至少一次"的消息接收。

从1.3版本开始，Spark又引入了Kafka Direct API，进而可以实现"精确一次"的消息接收。

由于Spark Streaming对数据的处理是基于RDD完成，而RDD提供了"精确一次"的消息处理。所以在数据处理部分，
Spark Streaming天然具备"精确一次"的消息可靠性保证。

但是，Spark Streaming的数据输出部分目前只具备"至少一次"的可靠性保证。
也就是说，经过处理后的数据，可能会被多次输出到外部系统。
在一些场景下，这个不会有什么问题。比如输出数据是保存到文件系统，重复发送的结果只是覆盖掉之前写过一遍的数据。
但是在另一些场景下，比如需要根据输出增量更新数据库，那就需要做一些额外的去重处理了。
一种可行的方法是，在各个RDD中新增一个唯一标志符来表示这批数据，然后在写入数据库时，使用这个唯一标志符来检查数据之前是否写入过。
当然，这时写入数据库的动作需要使用事务来保证一致性了。

