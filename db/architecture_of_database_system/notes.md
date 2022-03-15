# 2

## 2.4 准入控制

DBMS无法保证缓冲区的工作空间，导致出现不断的页替换的情况。比如在使用排序或者哈系join的时候就会消耗大量的内存

事务处理发生死锁后也会导致事务的重启，从而导致系统挂起

所以当系统拥有一个好的准入控制器的时候，在资源不够的情况下，新的请求将不会被接受

DBMS的准入控制有两个层面

1是保证客户端连接数不会超过一个值，可以避免类似网络这样的基础资源被消耗。这种机制有的时候也会被上层应用提供

2是在查询处理器上实现的。

准入控制器在查询语句转换和优化完成后执行这一步操作，由该操作来决定，是否要推迟执行一个查询，是否要使用更少的资源来执行查询，以及是否需要额外的限制条件来执行查询。准入控制器依靠查询优化器的信息来执行，如查询所需的资源以及系统并发资源等信息。特殊情况下，优化器的查询计划可以：
（1）确定查询所需要的磁盘设备以及对磁盘的 I/O 请求数；
（2）根据查询中的操作以及要求的元组数目判断 CPU 负载；
（3）评估查询数据结构的内存使用情况，包括在连接和其他操作期间的排序和哈希所消耗的内存

第三点最关键，因为内存通常是出现抖动的主要原因（不断的换页？）

# 3

并行架构

1.共享内存，单机中，多个处理器核共享一个内存

2.无共享。由多个独立的计算机通过网络进行互联

每个表格会通过水平数据分区传播到集群中的多个系统中

典型的数据分区方案包括：元组属性基于哈希的分区，元组属性基于范围的分区，round-robin，以及混合型分区方案（前两种方案的混合）

数据库元组采用基于值的分区导致的一个很自然的结果就是，在这些系统中，只需要最少的协调工作。

然而，为了得到很好的性能，就需要很好地数据分区。这就给数据库管理员一个重大的负担――合理明智地确定表的分布。同时给查询优化器一个重大的负担――需要在负载分区方面做得很好。

在共享内存的系统中，当某一个处理器发生故障的时候通常会导致整个系统停止运行

而在无共享系统中，一个节点发生故障不一定会影响到其他的节点。

如果发生故障的时候，我们停止整个系统，其实本质上就是在模拟一个共享内存系统

第二种方法是跳过故障节点的执行。在数据的可用性比完整性更重要的情况下很有用

第三种方法是冗余，元组副本会分布在集群中的多个节点，当某一个节点故障的时候，系统可以将负载分配到剩下的节点

3.共享磁盘

所有的处理器可以访问大致性能相同的磁盘，并且当一个节点故障的时候，不会影响到其他节点的工作

但是单个磁盘也会导致单点故障的问题

并且，由于不会共享内存，我们不能很自然的去协调内存中的数据。所以我们需要显式的进行数据的共享协调。

共享磁盘系统依赖于一个分布式锁管理设备和一个管理分布式缓冲池的高速缓存一致性协议。(协调多个节点的数据)
