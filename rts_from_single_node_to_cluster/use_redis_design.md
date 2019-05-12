## 采用redis实现流信息状态管理

我们先用redis来做流信息状态管理。

### redis简介
Redis是一个开源的内存数据库，支持非常丰富的数据结构，比如字符串（strings）、
哈希表（hashes）、列表（lists）、集合（sets）、有序集合（sorted sets）、
位图（bitmaps）、hyperloglogs算法、地理空间索引（geospatial indexes）等等。
Redis内建支持数据多副本、LRU淘汰、持久化等功能。
另外Redis通过Redis Sentinel支持HA功能，通过Redis Cluster支持集群功能。

Redis支持丰富的数据结构的特性，让它非常适合在实时计算中记录各种各样的状态。
下面我们就来看看如何将这些数据结构用在我们在第四章中讨论的各种特征计算中。

### 计数
在第四章中，我们描述了计数算法的原理，并用如下的dsl来表示过去一周内在同一个设备上交易的次数：

```
COUNT(7d, transaction, device_id)
```

针对这种计数查询，非常适合用redis字符串指令中的INCR指令。INCR指令对存储在指定key的数值执行原子的加一操作。

#### 更新计算
这里我们将7天的时间窗口划分为7个小窗口，每个小窗口代表1天。
在每个小窗口内，分配一个key用来记录这个窗口的事件数。
key的格式如下：

```
$event_type.$device_id.$window_unit.$window_index
```

其中，`$event_type`表示事件类型，`$device_id`表示设备id，
`$window_unit`表示时间窗口单元，`$window_index`表示时间窗口索引。

比如，对于`device_id`为`d000001`的设备，如果在时间戳为`1532496076032`的时刻更新窗口，则计算如下：

```
$event_type = transaction
$device_id = d000001
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到时间窗口索引

$key = $event_type.$device_id.$window_unit.$window_index

redis.incr($key)
```

上面的伪代码描述了使用redis的INCR指令更新某个窗口的计数。
由于在我们的设计方案中，更新操作和查询操作是分开进行的，因此这里只需要更新一个小窗口的计数值，
而不需要更新整个窗口上所有小窗口的计数值。

#### 查询计算
我们已经将7天的时间窗口划分为了7个子时间窗口，每个子时间窗口是1天。
在更新计算中更新了子时间窗口的计算。因此在查询时，只需要对子时间窗口计数做查询并汇总即可。计算如下：

```
$event_type = transaction
$device_id = d000001
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到当前时间窗口索引

sum = 0
for $i in range(0, 7):
    $window_index = $window_index - $i
    $key = $event_type.$device_id.$window_unit.$window_index
    sum += redis.get($key)

return sum
```

在上面的伪代码中，用redis的GET指令，查询了过去7个子时间窗口，也就是是过去7天每天的计算，
然后将这些计数值汇总，就得到了最终我们想要统计的"过去一周内在同一个设备上交易的次数"。


### 求和
在第四章中，我们描述了计数算法的原理，并用如下的dsl来表示过去一天同一用户的总交易金额：

```
SUM(1d, transaction, amount, userid)
```

针对这种计数查询，可以用redis字符串指令中的INCRBY/INCRBYFLOAT指令。
INCRBY/INCRBYFLOAT指令对存储在指定key的数值执行原子的加法操作。

#### 更新计算
类似前面的COUNT计算，我们将1天的时间窗口划分为24个小窗口，每个小窗口代表1小时。
在每个小窗口内，分配一个key用来记录这个窗口的总和。
同样，key的格式如下：

```
$event_type.$userid.$window_unit.$window_index
```

其中，`$event_type`表示事件类型，`$device_id`表示设备id，
`$window_unit`表示时间窗口单元，`$window_index`表示时间窗口索引。
读者应该会发现，这里的key与COUNT实现时用的key完全一样。
实际实现时，可能这些状态都会记录在相同的redis集群中。
因此如果需要与前面COUNT的状态区分，可以再加个额外的命名空间字段，将不同操作的状态隔离开来。
但是这不影响我们对算法的描述，因此在此就不坐区分了。

比如，对于`userid`为`ud000001`的用户，交易金额`amount`为`166.6`，交易时间为`1532496076032`，则更新窗口的总和值如下：

```
$event_type = transaction
$userid = ud000001
$window_unit = 3600000  # 时间窗口单元为1小时，即3600000毫秒
$window_index = 1532496076032 / $window_unit = 425693    # 用时间戳除以时间窗口单元，得到时间窗口索引

$key = $event_type.$userid.$window_unit.$window_index
$amount = 166.6

redis.incrbyfloat($key)
```

上面的伪代码描述了使用redis的INCRBYFLOAT指令更新某个窗口总和值。

#### 查询计算
在更新计算中更新了子时间窗口的总和值，因此在查询时只需要各个子窗口的总和再做一次汇总即可。计算如下：

```
$event_type = transaction
$userid = ud000001
$window_unit = 3600000  # 时间窗口单元为1小时，即3600000毫秒
$window_index = 1532496076032 / $window_unit = 425693    # 用时间戳除以时间窗口单元，得到时间窗口索引

sum = 0.0
for $i in range(0, 24):
    $window_index = $window_index - $i
    $key = $event_type.$userid.$window_unit.$window_index
    sum += redis.get($key)

return sum
```

在上面的伪代码中，用redis的INCRBYFLOAT指令，查询了过去24个子窗口的求和值，也就是是过去一天各个小时的交易量总和，
然后将这些总和值再次汇总，最终得到"过去一天同一用户的总交易金额"。

### 关联图谱
除了时间序列上的集中度分析外，在风控场景中，我们可能还会在各种维度对目标对象进行集中度分析。
比如"同一IP段的不同设备数"、"同一设备上登录过的不同用户数"、"同一设备上登录过的不同用户，登录过的不同设备数"等等。
诸如此类的计算目标，可以使用redis的集合来实现。

比如对于统计"过去30天在同一设备上登录过的不同用户数"这个特征，我们的dsl如下：

```
COUNT_DISTINCT(30d, login, device_id, userid)
```

#### 更新计算
类似前面的COUNT计算，我们将30天的时间窗口划分为30个小窗口，每个小窗口代表1天。
在每个小窗口内，分配一个key用来记录这个窗口内同一设备上的不同用户数。
同样，key的格式如下：

```
$event_type.$device_id.$window_unit.$window_index
```

其中，`$event_type`表示事件类型，`$device_id`表示设备id，
`$window_unit`表示时间窗口单元，`$window_index`表示时间窗口索引。

比如，对于`device_id`为`d000001`， `userid`为`u000001`的用户，交易时间为`1532496076032`，则更新窗口内设备上不用用户的算法如下：

```
$event_type = login
$device_id = d000001
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到时间窗口索引

$key = $event_type.$device_id.$window_unit.$window_index
$userid = u000001

redis.sadd($key, $userid)
```

上面的伪代码描述了使用redis的SADD指令，将新到的用户`u000001`添加到了以`login.d000001.86400000.17737`为`key`的集合里面，
也就是将新到的用户添加到了时间窗口内记录设备不同登录用户的记录里。

#### 查询计算
在更新计算中更新了子时间窗口内设备上的不同用户数，因此在查询时只需要对各个子窗口的设备不同用户集合再做一次汇总即可。计算如下：

```
$event_type = login
$device_id = d000001
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到时间窗口索引

sum_set = [] # 创建一个用于记录不同用户的集合
for $i in range(0, 30):
    $window_index = $window_index - $i
    $key = $event_type.$device_id.$window_unit.$window_index
    sum_set += redis.smembers($key)   # 将返回的用户添加到集合里面

return sum_set.size()
```

在上面的伪代码中，用redis的SMEMBERS指令，查询了过去30个子窗口的设备不同用户，也就是是过去30天每天在设备上登录的不同用户，
然后将这些记录再次汇总，最终得到"过去30天同一设备上登录的不同用户数"。

这里使用了redis的SMEMBERS指令查询不同用户。实际上，还可以采用SUNION、SUNIONSTORE、SCARD等指令，将集合的合并计算交给redis自身完成，
这样也可以尽量地减小网络传输开销。


#### 近似计算
上一小节我们使用了redis集合来实现关联计算。但是在某些场景下，变量不同取值的数量非常多，
而我们只需要知道这些不同取值的数量是多少，并不需要知道它们的具体取值怎样，比如针对网页的UV访问统计。
在这类场景下，如果采用上面这种用集合记住所有不同取值的方案，就显得非常有局限了。
一方面，记录这些不同取值会占用过多的存储空间。另一方面，过大的数据量也会急剧增大计算量，使得计算不能实时完成。
特别是在反欺诈特征提取这种场景下，我们要分析和计算的事件属性组合非常多，比如用户关联的不同手机号码数、用户关联的不同银行卡数、
同一IP C段使用的设备数等等，这些都需要消耗计算资源。因此，我们需要寻求一种时间复杂度和空间复杂度都很简单的方案。

非常幸运的是，我们能够找到这样的算法。如果允许牺牲部分的结果精度，那么我们可以极大地降低时间复杂度和空间复杂度，从而得到极大的性能提升。
而很多场景下，我们也确实没有必要求得一个精确的结果。因此，这是一类近似计算的算法，但是它们也都有很好的理论支撑。
Hyperloglog就是这类近似算法中的一种。
而Redis非常体贴地为我们提供了Hyperloglog算法的支持。下面我们就用Hyperloglog算法来实现关联数量的计算。

#### 更新计算
Redis中与HyperLogLog算法相关的指令包括PFADD、PFCOUNT和PFMERGE。
其中，PFADD用于将元素添加到HyperLogLog寄存器中，PFCOUNT用于返回HyperLogLog的基数估算值，PFMERGE用于合并多个HyperLogLog寄存器。
在更新计算时，我们需要将不同值用PFADD指令添加到HyperLogLog寄存器中。

比如，对于`device_id`为`d000001`， `userid`为`u000001`的用户，交易时间为`1532496076032`，则更新窗口内设备上不用用户的算法如下：

```
$event_type = login
$device_id = d000001
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到时间窗口索引

$key = $event_type.$device_id.$window_unit.$window_index
$userid = u000001

redis.pfadd($key, $userid)
```

上面的伪代码描述了使用redis的PFADD指令，将新到的用户`u000001`添加到了以`login.d000001.86400000.17737`为`key`的HyperLogLog寄存器里。
通过这个寄存器的状态，我们可以估算出时间窗口内设备不同登录用户数。

#### 查询计算
在更新计算中更新了子时间窗口内设备上的不同用户数Hyperloglog寄存器，
因此在查询时只需要对各个子窗口的Hyperloglog寄存器做一次汇总即可。计算如下：

```
$event_type = login
$device_id = d000001
$window_unit = 86400000  # 时间窗口单元为1天，即86400000毫秒
$window_index = 1532496076032 / $window_unit = 17737    # 用时间戳除以时间窗口单元，得到时间窗口索引

$keys = [] # 创建一个用于记录不同用户列表
for $i in range(0, 30):
    $window_index = $window_index - $i
    $key = $event_type.$device_id.$window_unit.$window_index
    $keys += $key   # 将返回的用户添加到集合里面

$count_key = random_uuid() # 生成一个uuid用于临时存储Hyperloglog寄存器合并结果
redis.pfmerge($count_key, $keys)
$count = redis.pfcount($count_key)
redis.del($count_key)  # 删除临时寄存器

return $count
```
在上面的伪代码中，用redis的PFMERGE指令，将过去30个子窗口的设备不同用户数Hyperloglog寄存器值合并起来，
结果保存在临时寄存器`$count_key`内，然后用PFCOUNT指令根据临时寄存器的值，估计出整个窗口上不同值的个数，
也就是"过去30天同一设备上登录的不同用户数"了。当完成估计后，还需要删除临时寄存器，以防止内存泄漏。

## 总结
本节向读者展示了实时流上部分时间序列特征和关联图谱特征使用redis实现的方式。
通过本节，希望读者理解上一节中所说的"流信息状态"，大部分情况下，这些流信息状态是一些基于历史的分析和聚合结果。
另外，也希望通过部分redis指令的使用，对读者去阅读、了解并最终灵活使用redis中丰富的数据结构和指令起到抛砖引玉作用。
最后，在后续的章节中，我们还会再次讨论redis如何优化使用和集群的问题。届时，读者对分布式状态应该如何"分布"设计会有更好的理解。