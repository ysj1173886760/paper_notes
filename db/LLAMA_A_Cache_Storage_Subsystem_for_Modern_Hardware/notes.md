# LLAMA: A Cache/Storage Subsystem for Modern Hardware

LLAMA is a subsystem designed for new hardware environments that supports an API for page-oriented access methods, providing both cache and storage management

CL(Cache Layer) support data updates and managements updates via latch-free CAS atomic state changes on its mapping table.

SL(Storage Layer) uses the same mapping table to cope with page location changes produced by log structuring on every page flush.

适应现代的硬件：
1. Good processor utilization and scaling with multi-core processors via latch-free techniques
2. Good performance with multi-level cache based memory systems via delta updating that reduces cache invalidations
3. Write limited storage in two senses: (1) limited performance or random writes; (2) flash write limits; via log structuring

![20220731155918](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220731155918.png)

![20220731160242](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220731160242.png)

LLAMA向上提供Page的抽象

通过PID来确认Page的状态（在Secondary Storage中，还是在缓存中）

读取的时候将Page从Secondary Storage拿到缓存中

intrudoction部分还说了很多让人看不太懂的东西，所以先看看后面

# LLAMA Interface

LLAMA提供两种形式的更新。而对于常用的CRUD来说，都通过更新来实现。

## Page Data Operations

data operation用来提供对数据的修改

比如bwtree会通过Update-D来添加delta。然后在某个时间通过Update-R做consolidate

1. Update-D(PID, in-ptr, out-ptr, data)
   delta-update会添加一段delta，用来描述对当前page的修改。其中in-ptr表示page之前的状态。out-ptr表示page之后的状态
2. Update-R(PID, in-ptr, out-ptr, data)
   replacement-update会为page生成一个全新的状态。其中data参数必须是page整体的状态。
3. Read(PID, out-ptr)
   读取这个page，out-ptr则是指向目标page的物理地址

## Page Management Operations

1. Flush(PID, in-ptr, out-ptr, annotation)
   Flush会将page拷贝到LSS（log structured store）IO buffer中。他会为page添加一个带有annotation的delta，这个delta带有Flush的标记
2. Mk-Stable(LSS address)
   保证到LSS address之前的所有buffer都已经被刷入到secondary storage中
3. Hi-Stable(out-LSS address)
   返回secondary storage中最大的LSS address
4. Allocate(out-PID)
   申请一个新的page。返回PID。保证这次申请是被持久化的。（通过system transaction）
5. Free(PID)
   释放一个page。操作仍然是被持久化的。（通过system transaction）

## System Transaction Operations

system transaction用来提供持久化的原子操作的。（比如提供SMO，类似mini txn）

1. TBegin(out-TID)
   开启一个事务，会将这个事务插入到ATT(active transaction table)
2. TCommit(TID)
   将事务从ATT中移除。并将page状态的变化安装到mapping table中，然后刷入到LSS buffer中
3. TAbort(TID)
   将page重置到事务开始的时候

