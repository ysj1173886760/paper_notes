# Efficiently Compiling Efficient Query Plans for Modern Hardware

# Abstract

这个Abstract写的很清楚。现有的iterator model在执行的时候对locality，以及instruction prediction利用率很差。导致执行性能比不上hand-written的代码。即便是有vectorized tuple processing，只能缓解这个问题，但是性能还是不够。

本文提出了一种基于LLVM框架的将query翻译成机器码的方法。通过对于locality和instruction prediction的关注，从而获得可以和hand written code竞争的性能。

（如果他没有说LLVM可能就要先去看看LLVM相关的东西）

![20220617085059](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220617085059.png)

这个是llvm框架。具体的代码通过前端编译成统一的IR，然后优化，再根据后端从IR转化成对应的机器码。

可以大概猜测我们做的就是类似JIT，根据query生成llvm ir，然后llvm ir优化后生成为对应的机器码再执行。（或者可能执行是数据库模拟的虚拟机）

# Introduction

传统的iterator模型就不介绍了，核心就是若干个实现了next的算子，不断递归调用

iterator model是从以前的I/O dominated的时候发展出来的，那个时候CPU开销不大。因为每次iterator的next调用都是通过virtual call实现的，从而降低了分支预测的性能。并且每一个tuple都需要一次函数调用，从而加剧了上面virtual call的performance downgrade。同时，这种模型要求维护很多信息，并且局部性不好。比如我们需要记录上一次扫描到字节流的位置等。

所以现代的系统中使用的是block-oriented，也就是一次处理一个block，多个tuple。从而均谈iterator model的开销。然而这种模型也削弱了iterator model的主要特点，也就是去pipeline data，即tuple流动的过程中不需要额外的拷贝。
（这里我的理解就是由于我们需要一个block一个block的传输数据，当operator要更改数据的时候，比如用过select筛掉一些tuple，或者join得到一些新的tuple的时候，我们需要重新构建这个block，也就是需要额外的拷贝来将tuple组装成新的block传给下一个operator）
然而block的好处还有就是我们可以利用向量指令来加速执行，也就是减少instruction count

![20220617091457](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220617091457.png)

figure1中可以发现，hand written code很明显性能会好很多。关系代数算子模型对于多个查询来说很有用（因为比较generialize），但是对于单个查询内部，则表现的不好。（因为这种表示法可以简便的表示任意种查询，但是对于单个查询的性能，则没有好的保证。我个人感觉他说的是iterator model，或者说是这种算子表示法的问题）

他的核心点：
1. Processing is data centric and not operator centric. Data is processed such that we can keep it in CPU registers as long as possible. Operator boundaries are blurred to achieve this goal.
2. Data is not pulled by operators but pushed towards the operators. This results in much better code and data locality
3. Queries are compiled into native machine code using the optimizing LLVM compiler framework.

（核心在于是data centric，我们操纵的是data，而非实例化很多operator去pull data）

（他是用的现有的llvm framework，而非集成编译技术到query engine中）

# Related Work

MonetDB会将所有的结果都物化，从而不需要重复调用operator function。对于OLTP system比较适用

MonetDB/X100会传一批数据。也就是block oriented

一些系统会将query编译成java bytecode，从而通过jvm执行。然而他们仍然在使用iterator model（我猜测可能是通过编译来避免虚函数调用）

HIQUE系统将query编译成C代码。他们为每个operator编写code template，然后生成C代码。并且他们通过物化结果来消除掉iterator model的影响，但是iterator的边界仍然十分清晰。（我猜测应该是利用编译技术来避免虚函数调用，然后通过物化结果才减少function call，以及去除其他的book keeping）。并且还有一个缺点就是生成C代码的开销比较高。

还有一些技术可以用来加速query processing。一个比较关键的就是去结合连接词谓词，比如AND，OR，来减少branching的数量。还有比如通过SIMD指令来加速评估谓词。

# The Query Compiler

## Query Processing Architecture

首先定义pipeline breaker:
An algebraic operator is a pipeline breaker for a given input side if it takes an incoming tuple out of the CPU registers. It is a full pipeline breaker if it materializes all incoming tuples from this side before continuing processing.

他这个定义是从寄存器角度来定义的。更加严格，也和他前面说的对应，从bottom up向上push数据可以拥有对寄存器的控制。

核心点就是我们会将数据传到内存的操作视为pipeline breaker。从而尽可能的让数据停留在寄存器中

iterator model每次的function call都会导致寄存器的内容被存到栈中。而block-oriented的方法虽然减少了function call，但是一个block肯定也不能存在register中。文章中提到：any iterator-style processing paradigm that pulls data up from the input operators risks breaking the pipeline

由于在提供iterator-based view的时候，我们需要提供一个*linearized access interface*。（这里我理解就是在为operator提供这种lineraized access interface的时候，这种pull data的操作需要额外的book keeping（比如栈，迭代器指针等），从而会导致breaking pipeline）

所以我们反转data flow control的方向。我们会持续push data到consumer operator中，直到遇到一个pipeline breaker。所以data is *always pushed from one pipeline-breaker into another pipeline-breaker*

![20220617104448](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220617104448.png)

那个长的很像F的我后面就用F，是group by

从R2扫描元组，然后通过z去group by，再和R3的c去join。得到的元组再让R3.b和R1.a去join

去看一下这个过程中的data flow，我们可以发现tuple总是从一个materialized point到另一个。最上面的join，tuple会从一个materializaed state（scan of R1）移动到上面join的hash table中。

在figure3中是这些物化点

![20220617111817](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220617111817.png)

由于我们无论如何也需要这些物化点。所以我们将query编译成执行流在物化点之间的移动。在物化点内部则是pure CPU operation。

可以看到核心在于是以full pipeline breaker为粒度的执行。在代码段内部是full pipelined。并且是tight loop，对于数据以及代码的局部性利用的都很好。（因为如果让我们自己写代码，写出来的也是类似的样式）

（在iterator模型中这种物化也是必须的。但是他的思路是不根据operator来划分，而是根据materialized point来划分。根据operator划分的时候，我们的视角是tuple经由一个一个的operator流动，operator是控制的核心点。而根据materialized point划分的时候，则是整体的数据从一个物化点移动到另一个物化点，我们不需要关心这个过程中经过了多少的operator。这个思路很棒要多加体会）

下面一个问题就是我们要怎么将关系代数的执行计划转化成这样的代码段了

