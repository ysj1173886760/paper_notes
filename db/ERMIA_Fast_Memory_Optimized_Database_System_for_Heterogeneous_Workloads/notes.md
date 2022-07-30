# ERMIA: Fast Memory-Optimized Database System for Heterogeneous Workloads

Desired properties:
* To provide robust and balanced concurrency control for the logical interactions over heterogeneous transactions.
* To address the physical interactions between threads in a scalable way and have a lightweight recovery methodology.

provide snapshot-isolation

通过SSN来保证serializability

* Highlights the mismatch between existing OCC mechanisms and heterogeneous workloads, and revisits snapshot isolation with cheap serializability guarantee as a solution
* Presents a system architecture to efficiently support CC schemes in a scalable way with latch-free indirection arrays, scalable centralized logging, and epoch-based resource managers.
* Presents a comprehensive performance evaluation that studies the impact of CC and physical layer in various workloads, from the traditional OLTP benchmarks to heterogeneous workloads.

我们主要关注的就是第二点，他的latch-free indirection arrays, scalable centralized logging和epoch-based resource managers

# Design Directions

## Scalable Centralized Logging

单点的logging容易成为bottle neck。

H-Store放弃使用logging，而是通过replication来避免这个问题

Silo放弃了txn的total order来避免logging bottleneck，但是缺点就是不容易实现弱隔离级别，比如SI

ERMIA选择了一个在fully coordinated logging和fully uncoordinated logging的折中点

## Latch-free indirection arrays

这里的indirection是类似TupleID这样的间接层，从而避免在插入新版本的时候要更改所有对旧版本的引用

从而减少了对索引的更新。（间接的减少logging压力）

## Append-only storage

因为append only storage逻辑简单，所以可以简化代码中对corner case的处理。并且简化了IO pattern

## Epoch-based resource management

这个就是和MVCC配合的EBR

# ERMIA

![20220730164635](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220730164635.png)

![20220730155913](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220730155913.png)

### Initialization

初始化阶段先将txn加入到三个Epoch-based resource manager中。分别是Log Manager，TID Manager，以及Garbage Collector

获取log buffer，TID，以及begin ts

### Forward Processing

txn通过索引访问到indirection array，然后再去遍历版本链

安装新版本的时候，将自己的TID写入到版本的begin ts field中。

读者读到了具有TID的version就会根据tid查询txn context，来判断这个版本是否可见。

日志则是会写入到private log buffer中避免冲突。

（和Hekaton没啥区别，如果是SI的话就从read view中查询状态就可以）

### Pre-commit

拿到一个commit ts。然后根据CC protocol来执行验证等操作

将Log buffer放到全局的log buffer中。并将状态设置为Committed

### Post-commit

遍历write set，将TID替换为Commit Ts。

最后将自己从epoch manager中拿出来。这样在并发访问的线程离开当前事务的context后，epoch manager就可以把它释放

## Log Manager

txn的ts是由lsn决定的，所以其实ERMIA是把LSN的推进和ts的获取放到了一起。这样每个txn只需要修改一次lsn，就可以获得log buffer中的空间，以及commit ts

ERMIA的log将物理偏移量和LSN解偶开。将日志划分为若干个segment。感觉主要的目的是为了回收日志buffer

## Epoch-based resource management

Epoch manager在切换epoch的时候，那些没有切换的线程中可能有的是没有任务的线程。所以他不会声明quiscent状态，从而阻止某个Epoch关闭。

ERMIA通过3种状态的Epoch来减少这种问题。当一个新的Epoch开始的时候，老的Epoch会被变成Closing状态。而再次出现新的Epoch的时候，Closing的Epoch才会被变成Closed的状态。这时候仍然停留的线程可能就是idle的线程。这时候就需要一些别的手段去处理他们。

## SSN

![20220730173638](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220730173638.png)

pstamp就是读到的时间。对于写集中的元素，我们为了避免RW-conflict，就要保证提交时间在pstamp之后。

而sstamp应该是serial stamp的意思，代表了真正的串行顺序。即如果txn读到了某个版本，那么这个txn的ts就一定要小于这个版本的sstamp，否则就会导致读集被改变。

所以最终计算出来t.sstamp才是真正的serial stamp。然后验证他要大于pstamp，保证不会影响其他人的读。

这样的话其实CommitTs并不能保证是按照serial order来的。

（感觉和tictoc的思路很类似，只不过tictoc会移动这里的commit ts，而非去验证他在这个范围内）