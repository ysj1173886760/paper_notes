# Serializable Snapshot Isolation in PostgreSQL

# Abstract

就是SSI在Postgres中的实现

# Overview

Postgres之前只有snapshot isolation。9.1版本提供了SSI的实现。

Postgres的SSI实现必须要和现有的特性结合，而不能像research prototype一样忽略很多细节。比如要支持replication，two phase commit，subtransaction。还有控制memory useage。

和之前的系统，比如MySQL不同的是，mysql之前提供的是基于2PL的serializable，在port到SSI的时候，他可以利用已有的predicate locking infrastructure来处理冲突。对于Postgres，他们构建了一个新的LockManager来处理冲突。

# Snapshot Isolation Versus Serializability

主要是提了一下SI发生的异常。一个是最简单的写偏斜。还有一个则是和read only transaction有关的

![20220709125526](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220709125526.png)

这里的report希望读到所有batch号为x - 1的receipts。但是在SI的隔离级别中，在figure2的执行顺序下，他没有达成目的。因为他没有读到T2插入的receipt。

和write skew不同的是，除掉t1后，t2 t3本身却是可串行化的。只有在加入t1这个只读事务后异常才会出现。

在[这篇paper](https://www.cs.umb.edu/~poneil/ROAnom.pdf)中作者说出了他的本质原因。`The fact that SI allows commit order different than serial order is what causes the anomaly`

在上面的例子中，serial order为 T2 < T3，然而由于t3提前提交，导致后来的txn看到了t3，却没看到t2。虽然看到了所有事务的更改，并且没有看到intermediate result，但是仍然发生了异常。