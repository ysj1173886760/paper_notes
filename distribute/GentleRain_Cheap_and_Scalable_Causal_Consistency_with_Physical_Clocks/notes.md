# GentleRain: Cheap and Scalable Causal Consistency with Physical Clocks

# Abstract

GentleRain是geo-replicated data store，提供因果一致性

GentleRain用periodic aggregation protocol来决定更新是否能被其他人看到

GentleRain通过一个标量的时间戳来实现因果一致性。时间戳是从loosely synchronized physical clocks导出的，时钟漂移不会影响因果一致性，但是可能导致推迟一个更新被他人可见的时间。

# Introduction

一个在Geo-replication data store的关键决策就是一致性模型。选择强一致性模型可以让我们有更简单的语义，但是延迟会增加，并且不能容忍网络分区。而最终一致性模型可以提供很好的性能，并且可以容忍网络分区，缺点就是编程模型会变的更加复杂。

大型的数据中心会将数据分区用来存储非常大的数据集。（这里有一个比较有意思的地方，他说：Recent papers have shown how to implement causal consistency in a replicated partitioned data store without incurring the serialization bottleneck of going thought a single log. 他的意思应该是大型数据中心有一个接受者负责将接受到的数据写入log）

更新是异步复制的，casual dependencies也会随着更新一起发送。在远端的数据中心上，接受者会发送依赖检查消息给其他分区的方式来确认所有的依赖是否已经满足，然后安装新的版本。然而这个方法的缺点就是依赖检查消息的交换可能导致吞吐量的下降。

GentleRain提供了不同的方案：
* GentleRain eliminates dependency check messages for updates
* GentleRain uses only a single physical timestamp to track dependencies

GentleRain相较于其他系统而言会导致更长的延迟（在远端的数据中心中，这个更新可见的所需要的延迟更长）

# Motivation

比较causal consistency和eventual consistency的性能

![20220526101412](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220526101412.png)

这个workload中，用户会读所有的partition，然后更新一个partition。从而导致跟踪因果一致性的开销变大。

他做的实验是移除掉causal consistency所需要的分片间依赖检查，为no remote dep check，以及不去掉检查，但是去掉计算逻辑，为fake remote dep check。

根据试验结果得到的motivation就是要想提高causal consistency的吞吐量，我们不能引入dependency check message

# Definition and Model

## Causality

应该Causality都是从lamport的那个文章中来的。也就是happens-before的关系

当以下三种情况之一出现的时候，则出现了因果关系：
* Thread-of-execution. a and b are operations in a single thread of execution, and a happens before b.
* Reads-from. a is a write operation, b is a read operation, and b reads the value written by a.
* Transitivity. There is some other operation c such that `a happens-before c` and `c happens-before b`

A store is causally consistent if, when a certain version of a data item is visible to a client, then all of its causal dependencies are also visible.

## Architecture and Interface

![20220526103348](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220526103348.png)

接口：
* PUT(key, val)
* val <- GET(key)
* {vals} <- GET-SNAPSHOT{keys}: 返回一个因果一致的快照。即对于任何一个数据，他依赖的其他数据也会存在于这个快照中。有一个特殊的点就是: Datastore is free is return a snapshot from any point in the past. It can therefore exclude values that have been already ready bt the client before executing the snapshot read.（如果用户曾经读到了一些值的话，说明这些值所依赖的所有数据都已经在这个store中了，我不太明白为什么他会去除掉这些值）
* {vals} <- GET-ROTS{keys}: 和上面的因果一致的快照类似，但是他保证单调读。

# GentleRain Protocol

GentleRain的timestamp是物理时间加上partition和replica标识符。这个timestamp可以为所有的更新操作提供全序关系

## States

![20220526105718](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220526105718.png)

Client States：
客户端会为其会话维护一个依赖时间DT。这个值是client读到的所有数据的最大的timestamp。
同时客户端还会维护GST。

Server States：
每一个服务器会维护一个version vector。VV[i]表示当前服务器已经从副本i处接受了直到VV[i]之前所有的更新

