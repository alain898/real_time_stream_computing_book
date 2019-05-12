## 采用ignite实现流信息状态管理

上一章节中，我们使用了redis来做流信息状态管理。
在本节中，我们将使用另外一种不同的方案来实现流信息状态管理，这就是Apache Ignite。
这里"多此一举"地用使用两种方案来实现相同的功能，绝非为了"凑字数"。
读者会在后续的章节中理解这么做的原因。在本节中，我们就将重点放在Apache Ignite的讨论上。

## Apache Ignite简介
Apache Ignite（后文简称Ignite）是一个基于内存的数据网格解决方案。
作为数据网格的Ignite天生就是分布式的。
在数据格点之上，Ignite提供了符合JCache标准的数据访问接口。
Ignite也支持丰富的数据结构，虽然相比redis少了些，
但是Ignite却提供了兼容ANSI-99标准的SQL查询功能，这使得Ignite的使用变得非常灵活。
除了这些功能外，Ignite还提供了很多其它的功能，比如分布式文件系统、机器学习等等。
但是在本书中，只会将Ignite作为数据网格使用，并使用它的SQL查询接口。

下面我们就来看看如何使用Ignite来实现第四章中讨论的部分特征计算。

### 计数
在第四章中，我们描述了计数算法的原理，并用如下的dsl来表示过去一周内在同一个设备上交易的次数：

```
COUNT(7d, transaction, device_id)
```

由于Ignite支持JCache和SQL查询功能，我们就充分利用这两种查询方案来实现计数功能。

### 表设计

在使用Ignite SQL前，我们先要设计好用于信息状态存储的"表"。针对计数功能的表设计如下：

```
class CountTable implements Serializable {
    @QuerySqlField(index = true)
    private String name;
    @QuerySqlField(index = true)
    private long timestamp;
    @QuerySqlField
    private double amount;

    public CountTable(String name, long timestamp, double amount) {
        this.name = name;
        this.timestamp = timestamp;
        this.amount = amount;
    }

    // 必需重写equals方法，否则在经过序列化和反序列化后，Ignite会视为不同记录，实际上它们是同一条记录
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        CountTable that = (CountTable) o;

        if (timestamp != that.timestamp) return false;
        if (Double.compare(that.amount, amount) != 0) return false;
        return name != null ? name.equals(that.name) : that.name == null;
    }

    // 因为重写了equals方法，所以hashCode()方法也跟着一起重写
    @Override
    public int hashCode() {
        int result;
        long temp;
        result = name != null ? name.hashCode() : 0;
        result = 31 * result + (int) (timestamp ^ (timestamp >>> 32));
        temp = Double.doubleToLongBits(amount);
        result = 31 * result + (int) (temp ^ (temp >>> 32));
        return result;
    }
}
```

由于我们需要在特征提取过程中，根据具体需要提取的特征动态创建表。
因此，这里我们使用了Ignite提供的表定义注解功能。
在这个用来存储计数状态的表中，各个字段的含义如下：
name：字符串类型， 用于记录状态的关键字，
timestamp：长整型，用于记录事件处理时的时间戳，
amount：双精度浮点型，用于记录状态发生的次数。

另外，我们还重写了表`CountTable`类的`equals`方法和`hashCode`方法，
这是因为Ignite在查询计算（比如replace方法）时，可能会需要进行对象的比较，
而由于Ignite为分布式系统，在查询过程中会涉及到序列化和反序列化的过程，
这个时候如果不重写`equals`方法，而默认使用Object类的`equal`方法，
会导致原本字段完全一样的记录会被视为不同记录，使得程序错误运行。

#### 更新计算
与redis的实现思路完全一致，我们也将7天的时间窗口划分为7个小窗口，每个小窗口代表1天。
在每个小窗口内，分配一个用来记录这个窗口事件数的关键字，也就是`CountTable`表定义中的`name`字段。
`name`取值的格式如下：

```
$event_type.$device_id.$window_unit.$window_index
```

其中，`$event_type`表示事件类型，`$device_id`表示设备id，
`$window_unit`表示时间窗口单元，`$window_index`表示时间窗口索引。

比如，对于`device_id`为`d000001`的设备，如果在时间戳为`1532496076032`的时刻更新窗口，则计算如下：

```
$event_type = "transaction"
$device_id = "d000001"
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到时间窗口索引

$atTime ＝ ($window_index + 1) * $window_unit
$name = "$event_type.$device_id.$window_unit.$window_index"

$cache = ignite.getOrCreateCache()

$id = md5($name);
$newRecord = new CountTable($name, $atTime, 1);
do {
    $oldRecord = $cache.get($id);
    if ($oldRecord != null) {
        $newRecord.amount = $oldRecord.amount + 1;
    } else {
        $oldRecord = $newRecord;
        $cache.putIfAbsent(id, oldRecord);
    }
    $succeed = $cache.replace($id, $oldRecord, $newRecord);
} while (!succeed);

$cache.incr($key)
```

上面的伪代码描述了使用Ignite的JCache接口更新某个窗口的计数的一种方法，要实现的功能与redis并无二致。
但是需要注意的是，由于Ignite并没提供类似于Redis中INCR指令那样的原子加操作。
因此需要自行实现并发安全的累加功能。
这里笔者并没有采用锁的方案，而是采用了CAS（Compare And Swap）的方案。
CAS是一种无锁机制（也可使视为是一种乐观锁），在高并发场景下通常比传统的锁拥有更好的性能表现。
在上面为代码的`do while`循环部分，就是CAS的实现部分，
其中Ignite cache的`replace`是原子操作，从而保证了CAS的并发安全。

#### 查询计算
在查询时，只需要对子时间窗口计数做查询并汇总。计算如下：

```
$event_type = "transaction"
$device_id = "d000001"
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到时间窗口索引

$atTime ＝ ($window_index + 1) * $window_unit
$startTime = $atTime - $window_unit * 7;    # 窗口为7天
$name = "$event_type.$device_id.$window_unit.$window_index"

$cache = ignite.getOrCreateCache()

$sumQuery = "SELECT sum(amount) FROM CountTable " + 
                "WHERE name = $name and timestamp > $startTime and timestamp <= $atTime";
sum = $cache.query($sumQuery)

return sum
```

在上面的伪代码中，用Ignite的SQL查询接口，结合`sum`函数，非常方便地获得了过去7天交易的总次数，
也就是"过去一周内在同一个设备上交易的次数"。

求和的实现过程，与计数的算法十分相识，只需要在更新计算时，
将计数器加一改成计数器加上新增的增量即可，这里就不再详细展开。


### 关联图谱
下面我们用Ignite实现关联图谱的计算。同样是统计"过去30天在同一设备上登录过的不同用户数"这个特征，其dsl定义如下：

```
COUNT_DISTINCT(30d, login, device_id, userid)
```

### 表设计
用于存储关联信息状态的表设计如下：

```
class CountDistinctTable implements Serializable {
        @QuerySqlField(index = true)
        private String name;
        @QuerySqlField(index = true)
        private long timestamp;
        @QuerySqlField
        private String value;

        public CountDistinctTable(String name, long timestamp, String value) {
            this.name = name;
            this.timestamp = timestamp;
            this.value = value;
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;

            CountDistinctTable that = (CountDistinctTable) o;

            if (timestamp != that.timestamp) return false;
            if (name != null ? !name.equals(that.name) : that.name != null) return false;
            return value != null ? value.equals(that.value) : that.value == null;
        }

        @Override
        public int hashCode() {
            int result = name != null ? name.hashCode() : 0;
            result = 31 * result + (int) (timestamp ^ (timestamp >>> 32));
            result = 31 * result + (value != null ? value.hashCode() : 0);
            return result;
        }
    }
```

在这个用来存储关联信息状态的表中，各个字段的含义如下：
name：字符串类型， 状态的关键字，
timestamp：长整型，处理事件时的时间戳，
value：双精度浮点型，状态的取值。

#### 更新计算
将30天的时间窗口划分为30个小窗口，每个小窗口代表1天。
与redis的实现方案有所不同的是，由于Ignite提供了灵活的SQL功能，
我们可以不必在单独的每个子窗口记录各时间段的不同用户，而只需要更新用户在设备上的登录时间。
在每个小窗口内，用`name`来记录设备，用`value`来记录用户，用`timestamp`记录登录发生时所在的子时间窗口。

比如，对于`device_id`为`d000001`， `userid`为`u000001`的用户，交易时间为`1532496076032`，则更新窗口内设备上不同用户的算法如下：

```
$event_type = "login"
$device_id = "d000001"
$userid = "u000001"
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到时间窗口索引

$name = $device_id
$value = $userid

$atTime ＝ ($window_index + 1) * $window_unit

$cache = ignite.getOrCreateCache()

$id = md5($name, $value);
$record = $cache.get($id);
if ($record == null) {
    $record = new CountDistinctTable($name, $atTime, $value);
} else {
    $record.timestamp = $atTime
}

$cache.put($id, $record);
```

上面的伪代码描述了使用Ignite的JCache接口更新用户在设备上登录时间的过程。
如果是相同设备上的相同用户在相同子时间窗口内多次登录，那么它们在Ignite中记录的状态完全相同，因而也不存在并发安全的问题。
所以相比计数算法在实现时使用CAS技术，这里的更新计算会简单很多。

#### 查询计算
由于每次都是更新用户在设备上的登录时间，因而不同设备上的不同用户在状态表里只存在一条记录。

```
$event_type = "login"
$device_id = "d000001"
$userid = "u000001"
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到时间窗口索引

$name = $device_id
$value = $userid

$atTime ＝ ($window_index + 1) * $window_unit
$startTime = $atTime - $window_unit * 30;    # 窗口为30天

$cache = ignite.getOrCreateCache()

$countQuery = "SELECT count(value) FROM CountDistinctTable " +
                "WHERE name = $name and timestamp > $atTime and timestamp <= $atTime";
count = $cache.query($countQuery)

return count
```
在上面的伪代码中，使用Ignite的SQL函数`count`，统计由`$atTime`和`$atTime`限定的时间窗口内，`name`为指定设备的`value`个数。
查询结果就是"过去30天设备上登录的不同用户数"。


## 总结
本节向读者展示了实时流上部分时间序列特征和关联图谱特征使用Ignite实现的方式。
对于单节点上的实现而言，Ignite和Redis的实现所体现出来的设计思路区别不是非常明显。
在本节中我们也并没有将Ignite作为数据格点的特性体现出来。
但是在下一节中，我们将详细地比较当将Redis和Ignite分别扩展为集群后的区别。
通过这种比较，我们将更加充分地认识到分布式系统的不同设计思路。
