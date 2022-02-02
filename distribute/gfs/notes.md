##### First pass

The design of GFS has been driven by key observations of google application workloads and technological environment

在分布式系统的环境中，component failures are the norm rather than the exception.

当数据快速增长，去管理billions级别的kb大小的文件是unwise的（因为我们需要管理大量的inode，并且读写文件以及对应的block size也要重新考虑）

accessing pattern， 大多数文件都是通过append来添加数据的，同时读取也是顺序的。这个特性成为优化性能和保证原子性的重点

通过co-design applications and file system API来增加系统的灵活性。通过relaxed GFS`s consistency model来简化文件系统。引入atomic append operation来避免额外的同步手段

通过constant monitoring, replicating curcial data, and fast automatic recovery来提供fault tolerance. 用checksumming来检测data corruption

分离file system的control和data transfer。通过租借chunk以及提高chunk size来减少master server的操作，从而让simple centralized master不成为瓶颈