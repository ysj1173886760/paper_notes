##### First pass

The design of GFS has been driven by key observations of google application workloads and technological environment

在分布式系统的环境中，component failures are the norm rather than the exception.

当数据快速增长，去管理billions级别的kb大小的文件是unwise的（因为我们需要管理大量的inode，并且读写文件以及对应的block size也要重新考虑）

accessing pattern， 大多数文件都是通过append来添加数据的，同时读取也是顺序的。这个特性成为优化性能和保证原子性的重点

通过co-design applications and file system API来增加系统的灵活性。通过relaxed GFS`s consistency model来简化文件系统。引入atomic append operation来避免额外的同步手段

通过constant monitoring, replicating curcial data, and fast automatic recovery来提供fault tolerance. 用checksumming来检测data corruption

分离file system的control和data transfer。通过租借chunk以及提高chunk size来减少master server的操作，从而让simple centralized master不成为瓶颈

##### Second pass

2.1 assumptions.

1. system is built from inexpensive commodity
2. system stores a modest number of large files
3. workloads consist of large streaming reads and small random reads
4. workloads have many large sequential writes that append data to files. Once written, files are seldom modified again
5. well-defined semantics for concurrent append operations. Atomicity with minimal synchronization overhead is essential
6. focus on high bandwidth(throughput) than low latency.

2.2 interface

GFS support usual file operations. Moreover, GFS has `snapshot` and `record append` operations. Snapshot creates a copy of a file or a directory tree at low cost. Record append allows concurrently appending the files while guarantee the atomicity.

2.3 architecture

single master and multiple chunkservers

![20220203113457](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220203113457.png)

files are divided into fixed-size chunks. Each chunk is identified by an immutable and globally unique 64bit chunk handle assigned by the master at the time of chunk creation.

For reliability, each chunk is replicated on multiple chunkservers. By default, we store three replicas. User can designate different replication level for different regions for the file namespace.

Master maintains all file system metadata. This includes the namespace, access control information, the mapping from files to chunks, and the current location of chunks. It also controls system-wide activities such as chunk lease management, garbage collection of orphaned chunks, and chunk migration between chunkservers. Mast periodically communicates with each chunkserver in HeartBeat messages to give it instructions and collect its state.

Clients interact with the master for metadata operations, but all data-bearing communication goes directly to the chunkservers.

这里原文有一句：We do not provide the POSIX API and therefore need not hook into the Linux vnode layer. Linux中的vnode是对不同文件系统的一个抽象，比如NFS中不存在inode，这里通过vnode的抽象层来为上层提供一个更加简洁的封装。而GFS没有提供POSIX的API，所以他也就不能作为一个单独的文件系统用vnode来抽象。

Neither the client nor the chunkserver caches file data. 因为cache这样的大文件对并不会有太高的收益，并且文件太大也不容易去cache。不使用cache还可以让我们不去考虑cache coherence问题来简化系统。(Client还是会cache metadata的)。对于chunkserver来说，因为chunk本身存储在linux之上，所以linux的buffer cache就会帮我们进行cache

2.4 single master

single master的好处, simplifies design, enable master to make sophisticated chunk placement and replication decisions using global knowledge.

client never read and write file data through master. It asks the master which chunkserver it should contact. It caches this information for a limited time and interacts with the chunkservers directly for many subsequent operations.

通过上面Figure 1的一个解释

1. 通过fixed chunk size， client可以将filename和byte offset转化成一个chunk index
2. 发送file name和chunk index
3. master会把对应的chunk handle和replicas的位置发回给client
4. 然后client就会以filename和chunk index为键来缓存master返回的信息
5. client向replicas发送request（most likely the closest one)，这个request包含了chunk handle和byte range

之后读取相同位置的操作就不需要在让client和master交互了，直到cache的信息过期，或者file被重新打开了（这个我不太清楚是为什么）

client在向master请求的时候可以请求多个块，master回复的时候也可以返回多个信息。从而更多的减少client与master的交互（将request进行batch）

2.5 chunk size

选择一个大的chunk size（64mb）有几个优点

1. 减少client-master的通信
2. 在大数据块上，client可能要执行许多操作，通过与chunkserver保持持久的tcp连接来避免网络开销（比如chunk size小，那么我们就需要更多次的去查找新的chunk，从而需要更多次的tcp连接）
3. 减少metadata的大小，允许我们将metadata放到memory中

大块的缺点就是对于小型的文件，可能只有一个chunk，如果许多client都去访问相同的文件，就会导致文件成为hot spot。但是对GFS的workload来说，这不是一个问题

然后是一个hot spot的例子和解决办法，这个我感觉还挺有意思的，所以记录下来

However, hot spots did develop when GFS was first used by a batch-queue system: an executable was written to GFS as a single-chunk file and then started on hundreds of machines at the same time. The few chunkservers storing this executable were overloaded by hundreds of simultaneous requests. We fixed this problem by storing such executables with a higher replication factor and by making the batchqueue system stagger application start times. A potential long-term solution is to allow clients to read data from other clients in such situations.

通过增加replication factor来分散读取压力，然后错开batch-queue system启动这个application的时间。更好的解决方法是允许client read from other client（P2P？）

2.6 metadata

metadata存储namespaces，file to chunk的映射以及每个replicas的位置

metadata都存储在master的memory中

对于namespaces和mapping通过`operation log`来记录他们的变化到disk中，并且复制到remote machines，从而保证他们的持久性

master不会存储chunk replicas的位置，而是当master启动的时候或者chunkserver加入cluster的时候去询问chunkserver他存储有关chunk的信息

将metadata存到master的memory中有两个好处，一是加速了master operation，二是方便master在后台周期性的去扫描master的状态，从而用来实现chunk的GC，re-replication（当chunkserver fail的时候），以及chunk migration（用于负载均衡）

master通过HeartBeat messages来监测chunkserver的状态，并且控制着chunk的placement。通过让master去监测chunkserver，而不是让master去存储chunk的位置信息，我们可以不用考虑有关chunkserver和master信息的同步

论文中有提到，要认识到是chunkserver对其拥有那些块有最终话语权，而不是master。所以尝试在master上去维护一个对chunk location的consistent view是没意义的

`operation log`记录了对GFS中的metadata的改变历史。他不仅是对metadata的一个persistent record，同时还定义了并发operation的顺序

对于用户的操作，我们会把对应的operation log存储到本地和远端来防止master失效。master会batch这些log的flush和replication操作，从而提高整体的系统吞吐量（和log based的file system一样）

同样，为了防止log的数量过多导致的重启时间过长，master在log超过一定数量的时候就会进行checkpoint

论文中提到checkpoint是一种B-tree的形式，可以直接映射到内存中，从而加速恢复。这里还不太清楚，看后面会不会细说一下

和fs一样，checkpoint不会阻塞后续的operation。当checkpoint的时候，master会切换到新的log file并且在新的thread中创建checkpoint（个人猜测应该会shadow一些页来保证操作的原子性，或者保证operation的幂等性）

2.7 consistency model

file namespace的修改是atomic的

A file region is consistent if all clients will always see the same data, regardless of which replicas they read from. 
A region is defined after a file data mutation if it is consistent and clients will see what the mutation writes in its entirety
When a mutation succeeds without interference from concurrent writers, the affected region is defined (and by implication consistent): all clients will always see what the mutation has written

当并发的同时写一个文件的时候，就有可能导致数据出现混合的情况，但是所有的writer都看到的是相同的数据。所以这是consistent，但是非defined的，因为我们不确定到底能看到是什么

GFS通过（a）在所有的replicas上以相同的顺序应用mutation，（b）使用version number检测stale的replica，来保证file是defined
