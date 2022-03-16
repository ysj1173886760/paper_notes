# Motivation

虽然最先进的共享内存处理系统可以高效的处理图。但是缺乏可拓展性使得他们无法处理那些单台机器无法承载的图。而分布式解决方案虽然可以将图拓展到更大的规模。但是他们的性能和成本效率往往不是很好

一个对于前沿系统的比较
![20220316193642](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220316193642.png)

可以发现分布式系统的网络没有饱和。限制他的主要因素是计算而非通信

与共享内存系统相比，他们执行了更多的额指令，更多的内存引用，更差的局部性以及多核利用率低。这种低效性有多个来源：（1）通过hashmap来在全局和局部状态间转换vertexID，（2）维护顶点的副本，（3）在GAS中的communication-bound apply phase，（4）动态调度

# Gemini Graph Processing Abstraction

graph processing problem updates information stores in vertices, while edges are viewed as immutable objects. 

undirected edges could be replaced with a pair of directed edges.

For a common graph hprocessing application, the processing is done by propagating vertex updates along the edges, until the graph state converges or a given number of iterations are completed.

正在更新的顶点叫活跃顶点，他们的出边构成了活跃边集

## Dual Update Propagation Model

图处理的时候，活动边集的可能是密集的或者稀疏的。通常由活动顶点传出边的总数决定。

比如CC（connected components）在最开始的时候活动边集是密集的，但是会随着顶点接收到其最终标签变得越来越稀疏。SSSP（单元最短路）从一个稀疏的边集开始的，当更多的顶点被其邻居激活的时候，他会变的更加密集，当算法收敛时他会再次变得稀疏

稀疏情况下，更适合用push模型（每个点通过其出边来更新对应的点），因为我们只需要遍历活动顶点传出来的边

密集的情况下，则更适合用pull模型（每个顶点的更新通过入边来收集相邻顶点的信息），因为这样可以减少通过锁或者原子操作更新顶点时的争用

在gemini系统中，graph会被partition在多个不同的节点上，信息的传递和更新是通过显示的message passing。

gemini使用master mirror的概念，每一个节点被分配给一个分区，在该分区中他是master vertex，作为维护顶点状态数据的主副本。同一个顶点会有多个副本，称为mirror，每一个mirror对应的分区上都至少拥有一个他的邻居。每个master-mirror pair之间有一条双向边，但是在任意一个传播模式下只使用一条。同时gemini的mirror不存储真实数据，只是占位符。

![20220316204536](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220316204536.png)

signals和slots用来表示user-defined vertex-centric function，用来描述发送和接受信息的行为

figure1中是两种模式下的操作

![20220316213633](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220316213633.png)

论文中这块有点怪怪的，我没太看懂为什么复杂度降低到了vertex数量

