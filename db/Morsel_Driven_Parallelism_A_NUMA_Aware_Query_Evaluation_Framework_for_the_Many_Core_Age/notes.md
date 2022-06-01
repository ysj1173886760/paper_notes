# Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age

# Abstract

随着计算机架构的发展，在parallel query execution中出现了两个问题：
* 为了利用好多核的优势，每个查询需要被均匀的分布到每个线程中
* 即便是我们拥有相当准确的统计数据，也很难将负载均匀的划分开

这两个问题导致了目前的plan-driven parallelism陷入了负载均衡和上下文切换的瓶颈中

第三个问题是在many-core的架构中，内存控制器是去中心化的（可能是若干个核对应一个内存控制器），从而导致了NUMA

这篇文章提出了morsel-driven query execution framework。调度将变成细粒度NUMA-aware的运行时调度。并行度不再在plan中详细规划，而是可以在执行时动态的变化。调度器是NUMA-aware的，所以绝大多数的查询将会在本地内存中进行

# Introduction

之前的火山模型中的并行执行是通过exchange算子来实现的。exchange会将元组流转发给多个线程，每个线程都会执行相同的pipeline segments。这样的实现被称为plan-driven：optimizer statically determines at query compile-time how many threads should run, instantiates one query operator plan for each thread, and connects these with exchange operators.

而morsel-driven的idea如下

![20220601091048](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220601091048.png)

一个查询会被划分为若干个segments，每个segment取一小部分的数据并执行，并最终将他们物化到pipeline breaker中。

图中的颜色则代表了NUMA-aware的特性：A thread operates on NUMA-local input and writes its result into a NUMA-local storage area.

文章中提到了他的dispatcher是pin在固定的核上的，所以不会出现由于OS移动线程导致的局部性缺失的问题（我的感觉是dispatcher的位置是无所谓的，因为他与数据无关，然而工作线程应该是pin在核上才对）

morsel-wise的框架中，所有的算子在任何阶段都是可以被并行化的，这相对于火山模型来说不仅在输入输出数据是并行化的，在中间的状态也是并行化的。比如用于做join的hash table，就需要被多个核共享访问。

在火山模型的框架中，并行会被算子隐藏起来，并且我们会避免共享的状态（比如要执行hash join的时候就需要先物化哈希表，变成一个算子的工作）。并且由于隐藏了并行的细节，我们就需要在exchange operator中进行on-the-fly的数据分区，（比如round-robin），而这个工作可以通过我们的locality-aware的dispatcher来进行。对于算子并行来说，有的系统提倡使用per-operator的并行从而实现执行的灵活性，然而这需要引入额外的算子间的同步。我们认为morsel-wise的框架可以被集成到现有的系统中。我们可以修改exchange operator的实现来达到morsel-wise scheduling，并且引入数据结构的共享。

（我感觉核心的思路就是将operator-thread变成了morsel-thread，这样我们的线程就不需要绑定到具体的operator，从而引发额外的同步或者灵活性较差的问题。现在每个线程只需要执行自己的morsel就可以。直观的感觉就是之前的模型就是提前搭建好了管道，然后我们往里喂数据。现在则是一堆工人在数据堆里一块一块的取。有点data-oriented的感觉，因为每一个morsel我们不需要关注他的worker是谁（当然优先local worker），只要关注执行的逻辑即可）

# Morsel-Driven Execution

通过这个例子来演示：

![20220601101622](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220601101622.png)

假设R是filter之后最大的表，所以我们会用R作为probe input，并在另外两个表上构建哈希表

![20220601101742](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220601101742.png)

我们得到了3个pipeline：
1. scan，filter，并构建哈希表HT(T)
2. scan，filter，并构建哈希表HT(S)
3. scan，filter，并用结果从HT(S)中探测，然后再从HT(T)中探测，最后存储结果

morsel-driven execution会被QEPobject来控制。他会把executable pipeline传给dispatcher执行。QEPobject负责跟踪数据的依赖性。比如上面的例子中，只有当前两个pipeline都执行完成的时候，最后一个pipeline才能开始。QEPobject会为每个pipeline分配临时的空间用于存储他们的结果。当pipeline结束后，这个临时的空间会被划分为等大小的morsels。这样后续的pipeline就会从新的morsel开始执行，而不会保留之前的morsel的边界（因为可能之前相同的morsel的结果不是等大的，从而可能导致负载偏斜）

![20220601105641](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220601105641.png)

对应生成的pipeline

![20220601110541](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220601110541.png)

构建哈希表的pipeline

每一个线程会操作一个morsel。基本表T存在了morsel-wise的NUMA-organized memory中。

每当一个线程处理完一个morsel的时候，他要么会被分配到另一个任务上（被dispatcher），要么会获取相同颜色的另一个morsel来作为下一个任务。

而构建哈希表的这个pipeline是由两个物理的pipeline组成的。逻辑上讲，我们是读取一个tuple，做filter，并插入到哈希表中。实际上在figure3中则是划分为两个阶段。

第一阶段主要是filter tuple，并插入到NUMA-local的存储区。

当所有的morsel都被scan并filter了以后，我们会重新扫描这些结果，并将指针插入到哈希表中。（我好奇难道不能直接插入吗）

这里说到是因为我们可以知道有多少元素会被插入到哈希表中，从而可以得到一个具体的哈希表的大小（这里我感觉是因为动态变化大小的哈希表不容易处理，并且可以均匀的把哈希表划分到每个NUMA-area中）。并且由于会有很多的线程同时插入，所以lock-free的实现是必要的

![20220601112458](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220601112458.png)

当所有的哈希表都构建完毕的时候，我们就可以开始执行probing pipeline

和之前一样，dispatcher会让本地的核去获取本地的数据，并将结果存到本地的内存中。（这里他没提到的一点，我猜测hash table的访问肯定会引入remote memory access，但是由于我们是给全局共享了哈希表，这也是必要的）

和火山模型不同的是morsel-driven的执行中，pipeline不是独立的（pipeline segment不同并且有依赖）。morsel-driven的pipeline会共享数据结构，从而需要一些同步手段。还有就是不同的pipeline segment的线程数不同，并且pipeline segment内部的线程也可以变化。

（这个感觉有点像数据并行和模型并行，数据并行不需要考虑依赖和同步，数据与数据之间是独立的。而模型并行则需要考虑依赖性，并且内部也会有数据的并行。但是这篇文章的重点不在这里，而是在细粒度的执行，NUMA-aware的scheduling，以及共享的数据结构）