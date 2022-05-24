# Don't Hold My Data Hostage - A Case For Client Protocol Redesign

# Abstract

我感觉之后我还是把摘要整个写一下

从数据库传输大量的数据到客户端是一个相当昂贵的操作。这个传输时间可以很容易就占有整个语句执行时间的主导部分。这对于一些外部数据分析的工具来说是一个很大的影响。在这篇论文中，我们将分析并探索将结果集序列化的设计空间。通过实现表明现有的方法都会有性能不足的情况。然后我们提出了一种列式的序列化方法。

（我猜测是不是通过列式存储提高压缩率，从而减少网络负担）

# Introduction

Result Set Serialization（RSS)对整个系统的性能有比较大的影响

![20220524100034](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220524100034.png)

本地环回的传输时间

可以发现基本上所有的时间都是在做网络传输（我好奇linux不会对本地环回优化吗？比如引入zero-cost copy之类的）

现有的大多数分析工具都要求传入一个已经在内存中的数据表，并且即便是支持从数据库加载数据的工具，也会受到网络的限制。

有一些工作则是希望在数据库内部执行计算，从而避免数据的传输。然而这种大规模的系统级计算不容易实现以及优化

main contribution为：
1. benchmark了不同的序列化方法。并解释了他们为什么是这样，以及他们设计上的缺陷。
2. 探索了序列化方法的设计空间，调查并测评了一些可以用来优化RSS的技术，并且讨论了他们的优缺点
3. 提出了column-based序列化方法

# State Of The Art

![20220524102234](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220524102234.png)

客户和服务器一次交互的过程如图2

结果集的设计决定了每一步的执行时间。如果我们使用了heavy compression，那么结果集的序列化和反序列化就需要更多的时间，然而带给我们的是网络传输的优化。反之，我们可以更快的进行序列化和反序列化，但是代价就是更多的网络传输的时间。

## Overview

评测现有数据库的传输时间，隔离开每个部件分别计算。通过netcat（nc）传输一个相同的CSV文件来作为baseline

baseline的作用是为了包含需要传输这些数据的时间，而不引入额外的database-specific overhead

![20220524103331](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220524103331.png)

table1显示了隔离后的传输数据。没有系统可以接近baseline的速度

（我在想他这个测试是不是不太公平，毕竟loopback的传输开销很低。但是看table1的size貌似也不能说明数据库就是在压缩，可能是因为额外的overhead导致的？）

mongodb是文档模型，所以传输数据也传输了每一个field name，导致开销很大

文章中也说了loopback传输开销低，所以大多数的时间都是在做序列化和反序列化。但是即便如此这些文件的大小也没有小，甚至还大了

## Network Impact

netem是linux的工具，可以用来模拟有限带宽以及延迟下的网络连接。测试他们传了1百万行数据

限制延迟：
延迟增加将导致传输数据的固定开销增加，因为我们需要更长的时间去传确认包。

![20220524105428](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220524105428.png)

DB2和DBMS X貌似显式的发送确认包，从而导致了更多的开销。即便是没有显式发送确认，底层的TCP也会这样做，从而导致传输速度减慢（我在想如果我们可以显式的增大tcp缓冲区的长度是不是可以缓解这个问题，让他可以pipeline）

限制吞吐：
这里指的就是限制带宽

限制带宽意味着在socket上发送的字节越多，我们的性能就会越差。所以随着带宽的降低，数据压缩就越来越重要

![20220524105745](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220524105745.png)

