# Introduction

一层GNN layer的例子

![20220323091027](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220323091027.png)

![20220323090737](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220323090737.png)

dependencies cached方法

对于一个k层的GNN模型，每个结点的k跳邻居的featrue都会被提前放到一个worker中，作为cache

比较明显的例子就是图中的partition0，可以看到虽然我们会在第二层中把节点3的信息传给1，但是在第一层中3还是聚合了2的信息，而不是从另一个worker哪里拿。

也就是说对于有关联的节点，会让他们在自己的worker中进行重复的计算

有一个直观的体会就是对于k来说，k越大我们要缓存的节点也就越多，从而导致了更多的重复计算。

另一个方法则是dependencies communicated方法

![20220323091614](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220323091614.png)

每次计算的时候不是重复的在本地进行计算，而是从远端的worker哪里拉取数据。

缺点就是comminucation overhead会很大，可能导致性能的下降

比如如果依赖少的情况下（以及较大的隐藏层，这个我不太明白），使用DepCache效果会好一些。因为冗余的计算不会影响性能。

而在high network bandwidth的情况下，对于高依赖的GNN数据，DepComm的方法会更好一些，因为communication cost不会影响太多。

后来问了学长明白了，这个hidden layer指的是中间层生成的顶点的feature，小的时候通信开销就比较小

![20220323093602](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220323093602.png)

论文中主要提到的就是hybrid的方法

NTS结合了DepCache和DepComm来提高性能。NTS通过给出的图数据以及硬件环境，去估计DepCache和DepComm的开销，然后选择对于顶点最优的方式进行计算

通过vertex-cur master-mirror的方式来解耦跨越worker之间的图操作（gemini）

# Execution Patterns of GNN

## GNN Training

![20220323094723](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220323094723.png)

一个Epoch下的GNN训练过程

输入是图数据，layer数量，顶点featrue，顶点的标签，以及对于每个layer的模型

输出的结果则是每个layer训练出来的模型

和之前的SAGA-NN类似，对于每个边，我们根据边的权值，以及边上两个顶点的feature来输入到NN中计算这个边的feature。然后对于每个顶点，聚合他的入边的feature，和当前层他自己的feature输入到NN中得到下一层的feature

然后通过feature来预测，得到loss。反向传播的时候则完全相反。首先每个顶点把他自己的gradient传给他的入边，然后每个边再把梯度传给边上的两个点

在算法中可以看到，loss会传给最上层的点，第l层的点的梯度会传给l层的边，然后l层的边会传给l - 1层的点

## Distributes GNN Training Approaches

![20220323103148](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220323103148.png)

DepCache的训练过程

首先将V分割成若干个子集，分发给若干个worker

因为我们有l层GNN，所以对于每个worker，获取他们l跳邻居的信息。包括edge以及顶点的feature

然后执行algorithm1

这里需要同步的原因是我们每一个节点在同一层中使用的参数是相同的，所以有一个隐含的共享点就是需要共享模型

更新这个共享模型我感觉会需要很大的开销

![20220323210348](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220323210348.png)

相比与DepCache，DepComm就会显得复杂一些

最开始还是先分区

然后对于每一层来说，首先拿到remote worker上的数据，然后跑algorithm1

反向传播则是会首先计算顶点的梯度。然后把这些部分梯度发送给对应的remote worker

最后同步的更新参数

### Existing GNN Systems Review

对于DepCache的系统来说，为了减少重复的计算，他们只采样一部分多跳邻居的信息来进行训练

sampling通常和mini-batch梯度下降训练方法相结合。虽然会牺牲一定的准确性，但是我们可以处理大规模的图数据了

对于DepComm的系统来说，他们不会有精度下降因为他们没有冗余的计算。所以他们的目的主要是去优化worker之间的通信以及主机和GPU之间的通信。

![20220323211440](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220323211440.png)

## Performance of the Two Approaches

![20220323211813](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220323211813.png)

通过vary一些参数去观察两种方法的性能

第一个图则代表了不同的图输入的情况，比如不同的依赖情况的影响

第二个图代表了不同的隐藏层大小，影响了DepComm传输数据时的大小

第三个图则vary了集群的环境，对应不同的网络带宽以及不同的计算能力

结论就是DepCache适合少依赖，较大的hidden layer size，以及高算力的集群

DepComm适合多依赖，较小的hidden layer size，以及高网络带宽的集群

# Hybrid Dependency Management

通过一个cost model来量化冗余计算开销以及通信开销。

![20220324200105](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220324200105.png)

冗余计算的公式，其中V(u)表示的就是u的入点，E(u)表示的就是入边

比如第一层的1号点，他的V就是0,3,5。因为他在上一层的入点就是0,3,5

而他的E就是(0,1),(3,1),(5,1)

![20220324201729](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220324201729.png)

其中d表示的是第k层的维度。Tv和Te表示每一维的vertex tensor和edge tensor计算的代价。注意本地的顶点和边集被去除掉了

而对于DepComm来说计算开销就容易一些

![20220324205941](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220324205941.png)

u节点在第l层的通信开销其实就是把l-1层的u节点传过来。Tc表示的就是每一维传输的开销。d表示的就是维度

Hybrid的方法则是根据cost model，选择部分节点做DepCache，选择部分节点做DepComm

这里我截图一下原文

![20220324210359](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220324210359.png)

D表示远端的依赖，R表示DepCache的集合，C表示DepComm的集合。由R和C一同构成远端的依赖

结合起来考虑GNN的计算代价就是

![20220324210517](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220324210517.png)

其中miu是一个因子，因为DepCache的情况下很多的计算依赖都是有重复的。我们应该只计算一次

我们的任务就是找到最优的R和C的集合。这个问题可以被转化成一个经典的NP hard问题，叫做0/1 linear planning problem

![20220324211047](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220324211047.png)

Algorithm4是一种贪心的方法来解决这个问题。

第一行是通过在一个小型的测试图上去探测前面需要的factor Tv，Te和Tc

然后预计算出每一个节点在每一层的依赖

然后对于每一层，首先把每个节点的tr值放到优先队列里。然后不断取出最小的tr值的依赖点u。如果这个点的tr小于tc，说明用Cache的方法更好，我们把它加入到R中，并且把这个点的依赖点加入到我们已有的依赖集合Vrep中

最后剩下的没加入R的就是用DepComm的

依赖管理和图分区是正交的，所以可以使用不同的图分区策略来配合hybrid的依赖管理

# NTS

![20220325085310](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220325085310.png)

nts的架构，根据那些子模块也可以看到nts应用的优化

lock-free的队列，任务的重叠，Ring-based通信以及基于source的sub-chunking

![20220325090236](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220325090236.png)

其他的和NeuGraph类似，主要就是GetFromDepNbr这块。这里会根据hybrid的策略来选择是从远端拉取数据还是说从本地拿缓存

![20220325090340](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220325090340.png)

这里则是一层GNN计算的过程。可以看到最后的参数更新是通过All-reduce完成的

![20220325091312](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220325091312.png)

figure7中演示的是nts的DepComm的实现，通过master-mirror来实现双向通信

partition0的1和0先发给partition1的结点2。反向传播的时候则是2结点发送给0和1,然后再发送给远端的master节点

![20220325093040](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220325093040.png)

ring-based的scheduling

后面应该和gemini就类似了

最后这个lock-free的队列，貌似不是普通的lock-free，而是说通过目的地直接指定写入的位置，从而避免写冲突。