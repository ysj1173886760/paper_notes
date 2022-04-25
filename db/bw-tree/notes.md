# Building a Bw-Tree Takes More Than Just Buzz Words

两个贡献，一个是Bw-Tree的实现教程，并且提出了新的优化策略。第二个则是发现BwTree并不如其他使用锁的并发数据结构更快

# Introduction

Lock-free的数据结构实现的难点：
1. 需要明白所有的race conditions
2. 并发线程的同步点通常不会放到算法中，导致人们实现出错，最后变成了busy-waiting loop
3. 需要保证所有的读者全部离开后才能回收内存（在mit os中提到过linux在使用RCU的时候，会等待一个时间上限再去清理）
4. 原子操作会成为performance bottleneck

BwTree的思想就是通过间接层，将逻辑标识符映射到物理地址中，从而避免锁。每个线程通过追加delta record到modification log中来实现修改。后续的操作就必须重放这些delta record来获得当前的状态

间接层和delta record有两个优势：
1. 通过将global state变成atomic steps来避免锁的开销
2. 由于是追加delta record而不是修改，会减少cache invalidation的次数，从而获得更小的同步开销

# BwTree Essentials

BwTree和其他的基于B+Tree的区别就是BwTree避免了直接修改树上的节点（因为会导致cache invalidation）。他为每个节点都维护了一个delta chain，从而可以让BwTree通过CAS来更新。indirection layer的作用就是可以让我们原子的更新对树节点的引用（比如孩子节点和祖父节点都指向父亲节点，我们可以通过间接层原子的让他们都指向新的父亲节点）

![20220425102129](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220425102129.png)

每一个节点都有一个逻辑ID，节点在指向其他节点的时候会用这个逻辑ID。当一个线程需要他的物理地址的时候，就会去MappingTable中查。MappingTable允许我们通过CAS来更新节点的地址

## Base Nodes and Delta Chains

一个逻辑节点在BwTree中有两个部分，一个base node，以及一个delta chain

这里有两种类型的base node：
1. inner base node，存储的是有序的（key， NodeID）的数组
2. leaf based node，存储的是有序的（key， Value）的数组
（和b树一样）

最开始的时候，BwTree有两个节点，一个空的叶子节点，以及一个inner节点，指向了叶子节点。

Base Node是不可变的

![20220425103530](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220425103530.png)

delta chain是一个单向链表，存储的是按序排的针对base node的修改历史

base node和delta record都存储了可以表示那个时刻状态的元信息

![20220425103952](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220425103952.png)

这样我们就不需要一步一步重放操作来获得最新状态。（但是当需要数据的时候应该还是需要重放，这里只是保存了状态而已）

线程在修改数据的时候，会追加delta record，再修改mapping table让他指向新的delta record

## Mapping Table

BwTree借用了BlinkTree的设计，每个节点有两个入指针，一个是从父亲来的，一个是从左兄弟来的。在更新的时候我们需要进行原子的修改这些指针，所以要么使用transactional memory，或者multi-wrod CAS

而BwTree通过用间接层来避免这些问题。从而允许一次CAS来完成原子修改

当CAS失败的时候，这次操作就会重启。每次重启都会从树根开始重新遍历。虽然更复杂的重启协议是可能的，但是我们认为从树根开始遍历简化了实现。并且我们要遍历的节点很可能会在cache中

他提了一点就是这里的Mapping Table只关注了BwTree的实现。实际上他也可以支持log-structured updates。（这里我不太明白，可能需要再去研究一下log-structured相关的细节）

