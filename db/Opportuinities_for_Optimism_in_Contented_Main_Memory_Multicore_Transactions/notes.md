# Opportunities for Optimism in Contended Main-Memory Multicore Transactions

看这篇文章主要是看看各种并发控制的protocol

# Abstract

他提到那些与concurrency control无关的implementation choices是导致性能下降原因。

# Intrudoction

partially-pessimistic concurrency control, dynamic transaction reordering以及MVCC修改了CC protocol来支持高冲突下的txn。

他们比较了Silo, DBx1000, Cicada, ERMIA和MOCC（之后可以仔细去看看），并发现了engineering choices会极大程度的影响这些系统。

# Background

![20220710143528](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220710143528.png)

架构的Overview。索引上存的是RecordPtr。Record中存的是VersionChain。有些version可能被inline

## OSTO

Silo OCC protocol

执行的时候，生成读写集。在提交的时候，有三个阶段。第一阶段会锁住写集中的所有的元组。如果出现死锁就abort。获取一个commit timestamp。第二阶段会验证读集中的元组没有被其他的txn修改或者锁住。如果有就会abort。第三阶段会install new version，更新他们的timestamp，然后释放锁

（这不就是标准的OCC么？Silo没有用什么变种吗）

OSTO目的是减少memory contention。他们用了RCU来减少读写锁。（是本文，而非Silo）

![20220710151124](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220710151124.png)

维护这些变量用来回收空间。要删除一个对象的话，txn会将它存入到一个链表中，并附带上freeing timestamp为wts_th。根据图上的解释可以知道。wts_th用来标识删除的对象。当所有人都读不到这个对象的时候，他就可以被安全的删除掉。

思路是这样的，global write不断递增。一个线程可以把他要删除的对象注册到write timestamp上。当所有人都不会再读到这个对象的时候，我们就可以把它安全的删除掉。那怎么表示一个线程不会读到这个对象呢？

![20220710153419](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220710153419.png)

所有并发的线程可能读到我们要删除的对象。而这些并发的线程的范围则是[min(wts_th), max(wts_th)]。比如T2，他所修改的对象必须要在T4结束后才能安全的删除。但是要注意我们不可能通过统计max(wts_th)去删除一个对象。因为他是不断递增的。而上面的范围说的是这一时刻的max(wts_th)。

这里做的是维护每个事务可以读到的最早的事务timestamp。每个事务开始的时候，为他分配read timestamp，即min(wts_th)。表示这个事务可以看到read timestamp及之后的事务的修改。那么小于min(read_timestamp)的对象就可以被安全的删除。

本质上是维护了一个low watermark，即每个事务的并发事务的最小时间戳。

## MSTO

MSTO是一个MVCC的变体。基于Cicada。每个version上有两个值。分别是write timestamp和read timestamp。write timestamp就是创建的时间。read timestamp则是最近提交的读取过这个版本的事务。

对于version chain来说。我们有rtsi >= wtsi, wts(i+1) >= rtsi, wts(i+1) > wtsi

第一个属性说的是version要被提交才能被读到。第二个属性说的是如果存在更新的版本则要读更新的版本。保证串行化。加入当前版本的rts大于下一个版本的wts，说明这个rts对应的txn没有读到正确的版本。而最后一个属性则是版本链的wts递增。

txn开始的时候会获取一个ts用来读取版本。对于只读事务来说用的是rtsg（即保证他读到的所有事务都是提交的，保证无冲突）。而对于rw事务，则是wtsg。

对于读来说，版本和元组都会存到read set中。对于写来说则存储元组。

在提交的时候。MSTO首先从wtsg中取一个commit timestamp。`ts := wtsg++`

然后在第一阶段他会原子的在write set的version chain中插入一个pending version。（并发的Pending会spin wait（死锁怎么办））。第二阶段会检查读集。他会根据commit ts来重新读取，如果和之前读到的版本不同。则abort。否则的话更新read timestamp。第三阶段，将Pending版本变成Committed。并将早版本的数据根据之前的方法删除，并等待回收。

（怎么感觉唯一不同的就是多了个垃圾回收的机制，维护读取的low watermark，然后删除掉旧版本。而且也没有看到read timestamp的作用）

# Basis Factors

## Contention regulation

over eager retry会导致contention collapse。即冲突过大导致性能下降。

over delayed retry会导致core idle

推荐方法是randomized exponential backoff

## Memory allocation

推荐是fast general purpose scalable memory allocator。比如rpmalloc

因为memory pool会引起争用。还会导致很多其他的overhead。

preallocation会极大程度的影响性能。（他的意思应该是测试情况下）

## Abort mechanism

C++ exceptions会获取一个全局的锁来保护exception handling data structures（来防止dynamic linker修改他）

推荐使用explicitly-checked return values

## Index types

哈希表更快

## Contention-aware indexes

contention aware index说的就是不相交的range不会引起contention。

![20220710172016](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220710172016.png)

## Other factors

维护read set和write set的实现。通过哈希表来将RID映射到物理指针上。

deadlock avoidance or detection strategy。
某些OCC会sort写集。比如通过memory address写入。
或者bounded spin，虽然会出现false positive，但是开销低。