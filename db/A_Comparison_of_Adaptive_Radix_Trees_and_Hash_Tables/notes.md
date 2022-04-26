# A Comparison of Adaptive Radix Trees and Hash Tables

# Abstract

比较ART, Judy Array, 两种基于二次探测哈希的变体，三种Cuckoo Hashing的变体

结果发现ART和Judy都不能与哈希方法相比

# Introduction

这里提到了这里的比较只用于integer:
We only focus on keys from an integer domain. In this regard, we would like to point out that the story could change if keys were arbitrary strings of variable size

文章中提到了一个概念covering index/non-covering index（即索引内部是否存储了所有需要查询的key）

以及提到了Hashing approach中，hashing scheme和hash function都很重要（比如hash function可以决定我们的冲突等。而hash scheme决定怎么解决冲突，常用的链式方法就会导致比较差的局部性）

