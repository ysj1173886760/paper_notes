# Integrating Compress and Execution in Column-Oriented Database Systems

# Abstract

列存储的数据库可以提高相邻数据项之间的相似度，从而带来更好的压缩效果。

压缩的最好方式不仅取决于数据的属性，还取决于查询的workload（sql workload）

# Introduction

列存压缩的一个优势就是我们可以在压缩的数据上直接应用算子，从而提高CPU的性能。对于RLE（游程编码）来说，比如他把数据压缩成（42,1000），表示有1000个值为42的数据项。那么我们可以快速计算出他的SUM，而从避免了后续的扫描。

主要点：
* 总揽数据库的压缩算法，并说明他们是怎么被应用到列存的系统中的。并且对比专门用于列存的压缩算法
* 通过实验说明这些算法的tradeoff，并且构建了一个决策树来帮助决策要怎么进行压缩
* 介绍一个可以将算子直接应用在压缩数据上的执行器的架构
* 说明列存中order的重要性

# Related Work

前人的工作指出了将压缩算法的具体实现和数据库实现代码隔离开的关键性。通常来说我们可以通过在数据到达算子之前将他们解压缩来实现。但是某些情况下我们可以通过将算子直接应用在压缩的数据上来提高性能。本文的工作就是提出了在提供隔离性的同时，还能从这些优化中获得性能提升的一种方案。

（即算子和压缩模式的解耦）

（其实我比较好奇的是行存的数据库是怎么压缩的呢？RLE的可能性比较小，感觉要么是根据attribute来做dictionary compression，要么是写盘前来把数据看成未解释的字节，然后应用一些压缩算法，基于滑动窗口什么的）

# C-Store Architecture

C-store中一个table会被表示为若干个projections。每个projection都由若干个列组成，存储为有序的列存格式。每一列至少在一个projection中，并且一列数据可以在多个projection中存储（这么看起来他应该不会关注跨越projection之间的关系？或者是存储对应的ID来重组整个元组）

他这里说了不同的projection之间是通过join indices来关联的。join indices就是一个排列，可以把一个projection的元组映射到另一个上。（这就带来了两个问题，排列所带来的映射应该是单向的，而我们可能需要双向的映射，还有就是他只能用于两个对象，所以对于多个projection的时候，带来的复杂度是n^2的，但是好处就是我们可以很快速的找到一个projection对应另一个projection的元组，而不需要根据index来进行整个的扫描，即space/computation的tradeoff。不过这样想貌似插入的开销也很大，因为需要重构permutation）

C-store实现了大多数的列版本的关系算子。他和传统的关系算子不同的有：
* selection算子生成的是bitmap，然后一个特殊的mask算子可以根据bitmap和列来进行物化数据得到结果
* 特殊的permute算子，通过之前说到的join index来重排序一个列（从而让他们对应）
* 投影是免费的，因为直接输出列就可以，不需要改变数据（对比row-store，我们得到元组后需要根据output schema重构这个元组）
* join得到的是position，而非value，后面会说为什么（我猜得到的可能是index pair，这样我们可以在推迟物化，从而根据后面的投影来进行物化？）

# Compression Schemes
