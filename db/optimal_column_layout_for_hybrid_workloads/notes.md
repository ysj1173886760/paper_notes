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