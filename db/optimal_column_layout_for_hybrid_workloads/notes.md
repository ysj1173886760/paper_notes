# Optimal Column Layout for Hybrid Workloads

# Abstract

现代的analytical system是基于列存储。然后通过delta store来进行插入和更新

我们通过确定分区的数量，他们的大小和范围，以及缓冲区大小以及他们是如何分配的来组织数据的分布。

给出workload knowledge以及performance requirements，给出一个优化的物理布局

# Introduction

目前的系统对于数据的布局都是固定的，这意味着他们会被局限在某个地方，而不能根据workload来达到比较好的效果。

而我们的insight则是这些设计决策不能被限制在一个先前的决定中，即在设计系统的时候就固定了这些决策。我们可以学习这些决策并且调整他们从而支持HTAP的workload

在这个paper中，我们关注三个比较关键的决策：
1. 数据的物理布局
2. 列存储是否是密集的
3. 怎么为更新操作分配buffer（delta store还是ghost values）

![20220428092752](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428092752.png)

Channenge：
1. Fast Layout Discovery。可以看到我们有很大的提升，但是这不是没有代价的。找到这个优化的布局是一个比较昂贵的操作。最坏情况下有指数级别的数据布局需要我们去枚举以及评估。我们将这个问题转化成binary optimization problem，并且用现成的solver来求解。并且利用数据是存储在column chunk的特性，我们为每个chunk单独的进行优化，从而降低复杂度
2. Workload Tailoring。我们需要有一个比较有代表性的workload的sample。然后分析数据的访问频率以及访问模式。
3. Robustness。当去优化layout的时候我们可能有过拟合的风险

我们针对的是analytical application，有着比较稳定的workload。我们的工具会分析workload，并离线的准备好data layout。类似现代系统的index advisor

他说支持service-level agreements，这个的意思应该是说保证给用户提供什么样的performance（应该和前面对应的performance requirement相关）

# Column Layout Design Space

Casper（就是本文的系统）有很大的设计空间：
1. 使用range partitioning作为额外的模式（现有的是根据key排序，或者根据插入顺序排列）
2. 支持原地的，非原地的，以及混合的更新
3. 支持无缓冲，全局缓冲，以及per-partition的缓冲

![20220428102234](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428102234.png)

这里提到存储的scheme和buffer是正交的。还提到了水平分区和垂直分区，我感觉应该就是切分行以及切分列。并且他还支持tiles（不清楚是什么）和projection。对于projection来说，意思应该是读负载可以被projection分摊开，并且不同的projection还可以用不同的layout来得到更好的性能

分区数量也是一个比较关键的因素。

可以看这个图

![20220428105209](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428105209.png)

比如说对于一个column chunk，里面有Mc个元素。我们有k个不相交的分区。那么读的开销大概就是Mc/k，而插入（删除）的开销则平均是k/2（因为我们可能需要跨分区移动一些数据）

所以对于读负载，他喜欢更大的分区数量（因为只要读一个分区），而写负载则更喜欢小的分区数量（因为不需要跨越很多分区去修改）

并且对于只读的负载，如果不同的区域有着不同的访问模式，那么等宽的分区可能就会导致不必要的读。对于不常访问的地方，粗粒度的分区就足够了

所以理想情况下，局部优化的分区策略可以让一些有skewed access的workload达到理想的性能（对比则是均衡access的workload）

Ghost Values。当更新数据的时候，如果我们放松对于整个column都有序的这个要求，那么就可以减少数据的移动。对于一个删除操作来说，他只需要找到对应的partition，然后把对应的数据标记为删除即可，从而引入了empty-slot。为了更好的利用，我们可以把empty slot移动到partition的最后面，从而直接处理到来的插入请求。

这些empty slot就叫做ghost value，需要额外的维护，但是可以使更新操作更容易。Ghost value减少了更新的代价，但是带来的是额外的内存使用。trading space amplification for update performance

![20220428140304](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428140304.png)

这个是通过增加buffer space来减少写的开销，代价是读变慢（如果每次都是把empty slot放到最后，为什么读会变慢呢？因为缓存吗？）

我们通过考虑数据区域的访问模式分布来达成细粒度的决策（it takes into account the access pattern distribution with respect to the data domain)。casper收集每一种操作访问分布的直方图，并传给我们的cost model

# Accessing Partitioned Columns

下面，假设我们有k个可变大小的分区，以及一个轻量级的k叉树作为索引

Point Queries。点查询就是在索引上先确定我们要找的分区，然后分区内部顺序扫描一下。一旦找到我们就可以返回这个值，或者元素的位置（那要是后面变了怎么办？），作为后面算子的输入

Range Queries。首先通过索引找到区间的起始的分区以及结束的分区，然后通过范围过滤一下。中间的分区则可以直接复制给后面的算子

一个演示
![20220428143246](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428143246.png)

Inserts。通过ripple-insert algorithm，来达成O(k)的插入一个元素到分区的操作。

![20220428144155](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428144155.png)

ripple-insert把一个empty slot从column的最后移动到对应的partition中。我们插入到这个empty slot中，并且修正index中刚才移动的partition的边界。

Deletes。对于删除来说，首先就是通过index来确定对应的partition，然后删除对应的元素，并得到一个empty slot。我们根据上面的步骤，再一步一步把empty slot移动到column的末尾。当然也要修正index

![20220428144754](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428144754.png)

update通过delete加insert来完成

# Modeling Column Layouts

总共是5个操作，不同的决策对于不同的操作影响都是不一样的。我们首先考虑没有ghost value的情况

![20220428151838](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428151838.png)

要优化的问题考虑了dataset以及workload。我们需要考虑数据的分布，然后在此之上覆盖访问模式，从而得到effective access distribution

## Representing a Partitioning Scheme

通过用位向量来标记分区的边界，从而表示一个分区的模式

一个列由Nb个大小相同的块组成。块的大小是和column chunk的大小一起决定的。我们通过Nb个布尔变量表示分区模式。当一个块的终点作为一个分区的终点的时候，我们把对应的变量设为1

![20220428152757](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428152757.png)

一个示例，所以分区的大小要和block大小对齐？

block的大小是cache line大小的整数倍，最小就是一个cache line，从而可以影响我们分区的粒度。

## The Frequency Model

基于上面的Partition Scheme上来介绍访问模式的表示

访问的数据会以logical block的形式整理起来，每一个block中的每一个操作的访问模式都被记录下来。logical block的大小是可变的

我们记录那些block都被那些operation使用了，从而构成若干个直方图，我们称这个直方图为Frequency Model。因为他存储的是每一个区域数据访问的频率

FM利用了10个直方图：
![20220428155549](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428155549.png)

每一个block都有这样的10个counter。最后的直方图的每一块都对应了一个block。

在sample workload上生成直方图的时候，我们不会真正的去计算结果，而只是去捕捉访问模式（更新counter）

![20220428160158](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220428160158.png)

这里的更新，当从3到16的时候，empty slot是向前，所以更新的是udf和utf。否则就是udb和utb

虽然示例是一个列，但是我们可以把它放入多个列（但是只能根据一个排序）。因为FM不会在意他内部具体存储的数据（也就是不在乎一个数据项内存的都是什么，只关注block的访问模式）