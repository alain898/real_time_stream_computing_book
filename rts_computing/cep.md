## 事件序列

针对流数据，有个专门的领域来研究和处理事件序列问题，即复杂事件处理（Complex Event Processing，简称为CEP)。
复杂事件处理通过分析事件流上事件之间的关系，比如时间关系、空间关系、聚合关系、依赖关系等等，产生一个具有高级意义的复合事件。
根据这个复合事件内含的高级意义，可以作出一些有用的推断和决策。这就是复杂事件处理的价值所在。

下面我们来看几种事件间的关系，从而更好地理解CEP的用途和使用场景。

#### 时间关系
事件之间的时间关系分为两类，即事件发生的先后顺序和事件发生的时间间隔。
我们经常会关注事件发生的先后顺序。比如
事件发生的时间间隔是另一种我们经常关注的模式。比如

#### 空间关系
事件发生时的空间关系也是一种常见的分析模式。
狭义的"空间"是指事件发生的地点。
而广义的"空间"就像数学上定义的"空间"那样，可以是对事件在任意维度属性的描述。比如衣服的颜色、环境的温度。

#### 聚合关系
如果一个不常发生的事件在短期内接二连三地发生，可能就需要引起我们的注意了。

#### 依赖关系
A发生且B发生或A发生但B没发生。
比如在商店，取商品、付款、离开构成一次

所有这些事件之间的关系，都可以视为事件发生某种模式（Pattern）。
因此，针对CEP的实现技术，一般都是先定义一个模式，然后在事件流上监控这种模式的发生。

### CEP的实现
CEP的实现方式有多种，比较常见的有自动机、匹配树、Petri网、有向图等等。
这里我们不会具体讨论CEP的实现方式，因为这超出了本书范围。我们把重点放在CEP技术的使用上。

提供CEP功能的产品也比较丰富，WSO2 CEP（Siddhi）、Drools、Pulsar、Esper、Flink CEP等等。
下面我们就以Flink CEP来说明CEP如何使用。对于其它CEP产品，读者可以自行查询相关质料。
应该说这些产品各有特色且都名声在外，相关资料和文档也比较丰富，读者不妨了解一下。

#### Flink CEP
在Flink CEP的实现中，将事件间的各种各样的关系，抽象为模式（Pattern）。
在定义好模式之后，当数据流过时，如果匹配到这些模式，就会触发一个复合事件。
这个复合事件包含了所有参与这次模式匹配的事件。

Flink CEP为了方便用户定义事件间的关系，也就是模式，提供了丰富的API。下面我们列举些常用的API。

1. 新的开始

```
begin(#name)
```
表示一个CEP模式的开始。

2. 紧接着下一个是谁

```
next(#name)
```
指定后一个事件匹配的模式。必需是紧接着前一个事件，中间不能有其它事件存在。

3. 跟在后面就行

```
followedBy(#name)
```
指定后面的事件匹配的模式。中间可以有其它事件存在。

4. 在一段时间内

```
within(time)
```
指定匹配的事件必需是在一段时间内发生，过期不候。

5. 发生一次或多次

```
timesOrMore(#times)
```
指定的模式匹配一次或多次事件。

6. 匹配的条件

```
where(condition)
```
指定事件复合模式的条件

注意上面只是列举了部分Flink CEP的API，用于让读者对CEP开发有个初步的影响。
实际上Flink CEP还有很多其它API，非常丰富。读者可以自行参考Flink官方文档，这里就不再赘述。

#### CEP例子
下面我们以仓库环境温度监控的例子来演示Flink CEP在实际场景中的运用。
考虑我们需要监控仓库的环境温度，以及时发现和避免仓库发生火灾。
我们设定了告警规则，当15秒内连续两次监控温度超过阈值时发出预警，
当30秒内产生连续两次预警，且第二次预警温度高于第一次预警温度时，发出严重告警。
采用Flink CEP的实现如下。

1. 定义`15秒内连续两次监控温度超过阈值`的模式

```
DataStream<JSONObject> temperatureStream = env
        .addSource(new PeriodicSourceFunction())
        .assignTimestampsAndWatermarks(new EventTimestampPeriodicWatermarks())
        .setParallelism(1);

Pattern<JSONObject, JSONObject> alarmPattern = Pattern.<JSONObject>begin("alarm")
        .where(new SimpleCondition<JSONObject>() {
            @Override
            public boolean filter(JSONObject value) throws Exception {
                return value.getDouble("temperature") > 100.0d;
            }
        })
        .times(2)
        .within(Time.seconds(15));
```

在上面的代码中，我们用`begin`开始定义一个模式`alarm`，再用`where`指定了我们关注的是温度高于100摄氏度的事件。
然后用`times`配合`within`，指定高温事件在15秒内发生两次才发出预警。

2. 将预警模式安装到温度事件流上
```
DataStream<JSONObject> alarmStream = CEP.pattern(temperatureStream, alarmPattern)
        .select(new PatternSelectFunction<JSONObject, JSONObject>() {
            @Override
            public JSONObject select(Map<String, List<JSONObject>> pattern) throws Exception {
                return pattern.get("alarm").stream()
                        .max(Comparator.comparingDouble(o -> o.getLongValue("temperature")))
                        .orElseThrow(() -> new IllegalStateException("should contains 2 events, but none"));
            }
        }).setParallelism(1);
```

当定义好预警模式后，我们就将这个预警模式`alarmPattern`安装到温度事件流`temperatureStream`上。
当温度事件流上，有匹配到预警模式的事件时，就发出一个预警事件，这是用`select`函数完成的。
在`select`函数中，我们指定了发出的预警事件，是两个高温事件中，温度更高的那个事件。


3. 定义严重告警模式

```
Pattern<JSONObject, JSONObject> criticalPattern = Pattern.<JSONObject>begin("critical")
        .times(2)
        .within(Time.seconds(30));
```

与预警模式的定义类似，在上面的代码中，我们定义了严重告警模式，即"在30秒内发生两次"。

4. 将告警模式安装在告警事件流上

```
DataStream<JSONObject> criticalStream = CEP.pattern(alarmStream, criticalPattern)
        .flatSelect(new PatternFlatSelectFunction<JSONObject, JSONObject>() {
            @Override
            public void flatSelect(Map<String, List<JSONObject>> pattern,
                                   Collector<JSONObject> out) throws Exception {
                List<JSONObject> critical = pattern.get("critical");
                JSONObject first = critical.get(0);
                JSONObject second = critical.get(1);
                if (first.getLongValue("temperature") <
                        second.getLongValue("temperature")) {
                    JSONObject jsonObject = new JSONObject();
                    jsonObject.putAll(second);
                    out.collect(jsonObject);
                }
            }
        }).setParallelism(1);
```

这一次，我们的告警模式不再是安装在温度事件流上，而是安装在步骤2中的预警事件流上。
当预警事件流中，有事件匹配上告警模式，也就是在30秒内发生两次预警时，就触发告警。
不过我们还有个要求没有达到，即第二次预警温度比第一次预警温度高。
这个是通过`flatSelect`来实现的，在`flatSelect`中，
我们设定只有第二次预警温度比第一次预警温度高时，才将告警事件输出`out.collect`。

至此，一个关于仓库环境温度监控的CEP应用就实现了。

