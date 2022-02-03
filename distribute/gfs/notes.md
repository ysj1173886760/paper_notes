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

