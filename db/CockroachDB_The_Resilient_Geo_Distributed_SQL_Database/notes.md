# CockroachDB: The Resilient Geo-Distributed SQL Database

![20220717140845](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220717140845.png)

一个真实的例子。

It has the following requirements: to comply with the EU’s General Data Protection Regulation (GDPR), personal data for its European users must be domiciled within the EU

Features:
* Fault tolerance and high availability To provide fault tolerance, CRDB maintains at least three replicas of every partition in the database across diverse geographic zones. It maintains high availability through automatic recovery mechanisms whenever a node fails.
* Geo-distributed partitioning and replica placement CRDB is horizontally scalable, automatically increasing capacity and migrating data as nodes are added. By default it uses a set of heuristics for data placement (see Section 2.2.3), but it also allows users to control, at a fine granularity, how data is partitioned across nodes and where replicas should be located. We will describe how users can use this feature for performance optimization or as part of a data domiciling strategy 
* High-performance transactions CRDB’s novel transaction protocol supports performant geo-distributed transactions that can span multiple partitions. It provides serializable isolation using no specialized hardware; a standard clock synchronization mechanism such as NTP is sufficient. As a result, CRDB can be run on off-the-shelf servers, including those of public and private clouds

分层
SQL：做解析，optimize，并转化成读写请求
Transactional KV：负责处理事务相关逻辑，保证事务的隔离性以及原子性。
Distribution：为上层提供一个logical key space（类比address space）。数据可以在这个key space中被定位到。数据通过一个two-level index来定位，并且Range的位置会被缓存起来。（这里和BigTable是一样的）
Replication：通过consensus-based replication实现持久性
Storage：单机的KV

The unit of replication in CRDB is a command, which represents a sequence of low-level edits to be made to the storage engine

为什么不固定leaseholder为raft group leader？看起来CRDB可能使用follower做leaseholder，然后在变更的时候要提交一个log来保证只有一个人可以获取lease。（目的是为了把consensus和leaseholder的逻辑解偶开吗？）

![20220717152253](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220717152253.png)

这个的算法主要是做write pipeline。跟踪每个op的依赖。非commit操作的依赖就是之前相同key上的操作。

paralle commit说的就是并行的写入txn record和writes。引入了额外的一个状态叫Staging。当都成功的时候，我们可以直接给用户返回成功，然后异步的去将txn标志为committed。（1PC优化）

![20220717161902](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220717161902.png)

如果op是写操作的话，我们需要push他的ts，来解决RW-Conflict。同时我们需要保证读到的内容在推进的这段ts中没有改变。如果改变了就要提前abort。

回复给coordinator的时候只要计算了ts就可以直接回复，然后异步的复制。如果是commit的话要保证commit被持久化了再返回。

读或写的时候要拿一个latch，来防止并发的操作相互影响。比如更新ts等

写的时候先写write intent，write intent指向txn record。里面会标识txn的状态。pending, staging, committed, aborted.

read遇到write intent的时候会尝试resolve。如果是pending的话就会阻塞。而对于staging的话，the reader attempts to abort the transaction by preventing one of its writes from being replicated.

WR-Conflict通过等待write intent来解决。如果是一个更大的ts的write intent，我们就直接跳过

RW-Conflict通过提高WriteTs来解决。

WW-Conflict则和WR-Conflict类似，如果是write intent，则等待。如果是一个比较大的CommitedWrite则提高WriteTs。