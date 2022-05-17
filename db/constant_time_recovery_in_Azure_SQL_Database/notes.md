# Constant Time Recovery in Azure SQL Database

# Abstract

这个恢复机制结合了ARIES和MVCC，从而实现了常数时间的恢复。

允许连续的log trucation，从而减少了日志空间的使用量，即使是有长事务的存在（对比Innodb，如果Undo table用完了就不能开新的事务了）

对于云数据库（Cloud database，应该是DBaaS）来说，这个能力是相当重要的，因为：
1. 数据库大小是不断增加的
2. 对于commodity hardware来说，failure很常见
3. 高可用的要求
4. 云平台负责管理和升级软件，所以可能导致对于用户不可预见的failure（这里指的是服务软件的升级由云平台决定，而非用户。如果是用户控制的话他们可能选择一个流量小的时间进行维护）

# Introduction

ARIES使用WAL，并且定义了不同的恢复阶段（Scan，Redo，Undo）来避免专用的恢复手段（ad-hoc recovery techniques，这里应该说的是针对不同种类的transaction有不同的恢复技术，而ARIES把这个过程泛化了)

恢复的一个很主要的点就是Undo，我们需要把那些未提交的事务全部Undo，对于长事务来说，这个过程可能持续很久。一个例子就是一个用户尝试在一个事务中加载亿级的数据，然而当数据库在这个过程中崩溃的话，我们需要把加载的这些数据整个undo

在SQL Server中，他有一些针对恢复阶段的优化，比如利用并行来加速恢复。尽管这些优化对于内部部署的数据库是足够的（指的应该是企业内部的服务器部署）。但是当数据库迁移到云端的时候，情况就不一样了：
* 数据库大小的增加通常导致了更长的事务
* 云平台所使用的commodity hardware失效的频率越高，就会导致在此之上的服务崩溃的概率也变高
* 上面提到过的云平台负责管理和升级服务，导致用户不可预见的崩溃

我猜测应该是利用MVCC的版本机制来加速Undo。

有两个要点，一个就是恢复机制和MVCC怎么结合来实现常数时间的恢复，还有一个就是他是怎么允许aggressively truncating log的，对于长事务来说他的log应该只能在提交的时候被截断，否则我们无法保证原子性

# Background On SQL Server

第一部分就是正常的ARIES算法

![20220517105420](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220517105420.png)

可以看到这里的日志就是物理逻辑的，即日志标识了数据的位置，并且记录了页内的逻辑操作。减少了日志空间的占用。并且相对于逻辑日志来说允许了更高的并发性，并且逻辑更简单，因为逻辑日志依赖数据库的元数据。

但是SQL server在Redo阶段有个变化

![20220517105336](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220517105336.png)

他为了提高可用性，他会记录最小的活跃事务，并获取对应的锁。这样就可以在Redo阶段把数据库恢复到失效前的状态，从而提高可用性（Undo阶段则是启动后再进行）。但是这会导致Redo阶段和最长的活跃事务大小相关（因为我们需要重新获得上面的锁）

Undo阶段会扫描日志，并undo未提交的日志。这个阶段则和未提交的事务的大小相关（或者说是活跃事务的大小）。为了保证Undo的幂等性，我们还需要用CLR来记录undo操作。

MVCC，SQL Server更新会原地更新，然后把老的版本放到version store中。

![20220517110814](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220517110814.png)

version locator是（PageId， SlotId）

SQLServer会在重启的时候释放掉Version store，因为新的事务看不到老版本的数据