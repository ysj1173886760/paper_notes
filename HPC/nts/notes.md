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