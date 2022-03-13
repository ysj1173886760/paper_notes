就把这篇论文当作图计算的入门论文了

![20220312191047](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220312191047.png)

GNN中一个顶点的计算过程

要收集他的邻居的信息，然后aggregation，再传入到传统的DNN中做分类/回归

Roc用了一个linear regression model做partition

通过dp来最小化数据传输的代价