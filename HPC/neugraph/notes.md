# NeuGraph Programming Abstraction

GCN:

初始情况下，每个vertex都有一个feature vector

每一个顶点都收集他邻居的特征向量，然后根据边上的权重进行加和。

然后一个全连接的NN来计算新一层的特征向量

比如在推荐系统中，如果用户对某一个item进行评分，就可以在用户顶点和item顶点之间连边，评分即作为边值。然后GCN可以从graph以及用户和item的特征中学习用户和item的embeddings。最后通过这些embedding来预测缺失的user-item评分

GAT和GCD主要的不同点就是GAT对于每个边都计算一个attention value，用来在通过边传输features的时候使用

![20220317202535](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220317202535.png)

## Running Example

G-GCN的例子

![20220317204556](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220317204556.png)

等式2代表了EdgeNN，通过两个点的feature来计算边权

等式1代表了VertexNN，他从邻居中聚合信息，并在外层做NN操作

## SAGA-NN Model

![20220317205000](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220317205000.png)

SAGA-NN模型有两个user-defined functions。ApplyEdge和ApplyVertex。用来让用户去定义在edge和vertex上的NN计算操作

ApplyEdge定义了在edge上的计算，输入为`edge`和`p`，其中`p`是可学习的参数。`edge`是一个tensor的三元组[src, dest, data]。这个函数可以用来在边上应用NN模型，输出的就是中间结果。对应图中的edge output

ApplyVertex函数定义了在vertex上的计算，输入是`vertex data`，聚合后的数据`accum`，以及可学习参数`p`。返回的则是应用完NN之后的新的vertex data

另外两个state，Scatter和Gather，则是用来做数据传输，并将数据准备好提供给ApplyEdge和ApplyVertex的。

同时，NeuGraph也避免将聚合操作暴露给user，而是提供了一些默认的操作。我们通过设置Gather.accumulator来改变聚合的方式

* Scatter将顶点数据传输给他的出边，从而构建edge
* 然后ApplyEdge阶段会根据user-defined function会开始并行的计算，并为每条边生成结果
* Gather阶段会将这些结果沿着边在目标点处进行聚合
* 最后ApplyVertex会执行user-defined function，根据我们聚合的结果，这个点原始的数据最后得到下一层的数据

![20220317211412](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220317211412.png)

这个则是G-GCN在SAGA-NN模型上的实现

数据流抽象使我们表达神经网络结构以及自动微分更加容易。通过well-defined stages来建模GNN，可以让我们在graph computation以及dataflow scheduling进行优化

# NeuGraph System

NeuGraph包含了

1. 翻译引擎，用来将SAGA-NN表示的GNN翻译成chunk-granularity的dataflow graph
2. streaming scheduler来减少在GPU以及主机之间的data movement，并且最大化communication和computation之间的overlap
3. graph propagation engine来提供各种算子
4. dataflow execution runtime（这是用来干啥的）

## Graph-Aware Dataflow Translation

由于图数据无法放入到GPU中，所以目前的DL framework不能在GPU上直接使用

2D graph partitioning

将顶点数据划分成p等份。然后将邻接矩阵放到P×P的矩阵中

![20220317220409](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220317220409.png)

其中Eij中包含了Vi到Vj的边

这样划分的时候，我们可以对每个chunk单独进行处理，同时只会涉及到原点和目标点。这样就可以放到GPU中进行计算。

![20220317221050](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220317221050.png)

figure5是计算v0的过程，前向传播

对于反向传播来说，由于ApplyEdge和ApplyVertex都是由tensor组成的dataflow computations。所以我们可以通过自动微分来的到反向传播的梯度。同时还提供了`backward-Scatter`和`backward-Gather`，一个用来收集梯度，一个用来分发梯度

计算的过程不需要全局的barrier，根据依赖关系不断的调度即可

在计算的时候，我们可以使用row-oriented或者column-oriented。对于前向传播来说，如果使用row-oriented的方法，那么要存储accum的结果的节点就会很多，导致我们不能很好的复用accum（虽然我们可以复用当前的节点），而对于column-oriented的情况下，我们可以直接计算出某一个chunk的accum的值，而且好处是这个accum的值还可以继续复用，在下面的ApplyVertex stage中我们可以用这个值来更新对应的节点。所以对于前向传播来说，column-oriented的方法更好一些

而对于反向传播的情况，梯度会从终点传向起点。所以在row-oriented的情况下，我们可以复用一个chunk下起点的梯度，并且后续可以直接更新他

决定vertex chunk的数量P也是比较关键的。比如在前向传播的时候，我们使用column-oriented，那么就需要进行P次IO来加载源点的chunk。所以希望有一个更小的P。在NeuGraph中，他们选择的就是能保证每个chunk存到GPU中最小的P