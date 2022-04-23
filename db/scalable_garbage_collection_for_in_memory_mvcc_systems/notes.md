# Abstract

他首先提出HTAP workload中，GC通常会成为bottleneck

现有的GC技术过于粗粒度。并且不能很好的处理sudden spike的workload

# Introduction

MVCC的一个问题就是如果workload中有很多的long-running transactions，那么活跃的版本就会增加的非常快，并且我们不能删除掉这些版本因为他们可能要被活跃事务使用

![20220423093805](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220423093805.png)

所以这些long-running transaction就会导致一个恶性循环

因为他们持续的越久，那么活跃的版本就越多，导致事务读的速度会更慢，从而导致更多的long-running transaction

![20220423094142](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220423094142.png)

一个比较直观的图，显示了version的数量对于performance的影响

# Versioning In MVCC

还是MVOCC可见性核心的一点，我们只能看到commitTimestamp小于我们的事务开始时间的元组

![20220423101512](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220423101512.png)

这里给出了一个示例，中间有很多无用的元组。其实从这里我们也可以有一定的感觉，他所谓的细粒度的GC就是不去维护high water mark，而是具体的可见性。

## Identifying Obsolete Versions

可见的版本与并发运行的事务有关。即一个版本的生命周期取决于目前的活跃事务。

这里提到了，传统的垃圾回收器只会跟踪活跃事务中最老的那个timestamp。所以会得到一个比较粗粒度的估计，导致很多其实无用的版本不能被删除

## Practical Impacts of GC

这里其实是一个对于figure1的解释

当出现长事务的时候，version record就会堆积。当reader结束的时候，writer才能开始清理这些元组，并且在GC的时候新的事务不能进行写操作。(这里的意思应该是后台的GC线程需要锁住版本链才能就检查是否需要进行GC，版本链越长进入临界区的时间也就越久，导致其他的写入会被阻塞)

并且随着版本的增多，读事务会越来越慢，导致更多的长事务

总结起来就是传统的垃圾回收器有三个缺点：
1. scalability due to global synchronization
2. vulnerability to long-living transactions
3. inaccuracy in garbage identification

（我感觉23其实是一个问题，细粒度的去检查就可以了，1他所谓的global synchronizatio我不太清楚，感觉可能是lock-free的GC方法，从而防止GC线程阻塞worker线程）