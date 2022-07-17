# TicToc: Time Traveling Optimisitc Concurrency Control

The key contribution of TicToc is a technique that we call data-driven timestamp management: instead of assigning timestamps to each transaction independently of the data it accesses, TicToc embeds the necessary timestamp information in each tuple to enable each transaction to compute a valid commit timestamp after it has run, right before it commits.

This approach has two benefits. First, each transaction infers its timestamp from metadata associated to each tuple it reads or writes. No centralized timestamp allocator exists, and concurrent transactions accessing disjoint data do not communicate, eliminating the timestamp allocation bottleneck. 

Second, by determining timestamps lazily at commit time, TicToc finds a logical-time order that enforces serializability even among transactions that overlap in physical time and would cause aborts in other T/O-based protocols. In essence, TicToc allows commit timestamps to move forward in time to uncover more concurrency than existing schemes without violating serializability.

DTA-based OCC会分配一个区间的timestamp，然后在validation的时候进行改变，从而降低abort率。（可能是通过推迟某些事务的验证阶段？）

Silo通过epoch来分配粗粒度的时间戳。从而避免了ts分配的瓶颈。每40ms去分配一次时间戳，成为epoch。在epoch内部，通过txn id来标识版本并检查冲突。

# The TicToc Algorithm

TicToc也使用ts来标识txn的顺序。不同的是他不是分配一个ts给txn，而是在提交的时候，根据txn访问的tuple去计算他的ts

![20220716221102](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220716221102.png)

![20220716221046](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220716221046.png)

![20220716221222](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220716221222.png)

![20220716224854](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220716224854.png)

![20220717092151](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220717092151.png)

一个提交的例子，B虽然先提交，但是他计算得到了自己的ts为4。而A则得到了ts为3。

Note that when the DBMS validates transaction A, it is not even aware that tuple x has been modified by another transaction. This is different from existing OCC algorithms (including Hekaton and Silo) which always recheck the tuples in the read set.

验证阶段他不需要去重复验证读集。因为他的commit ts不一定是最新的。所以TicToc不是commit order-preserving的

tictoc的encoding方法，以及进行原子加载的方法

![20220717093636](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220717093636.png)

![20220717094200](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220717094200.png)

在延伸rts的时候，要注意防止overflow。要同时修改wts来防止delta overflow。修改wts的操作可以视为一个dummy write

（shift的选取是不是也是一个要考虑的点？）

确定了新的ts后，通过CAS把它换进去。

虽然修改了wts，但是不需要上锁，因为他不会引起正确性问题。但是其他的txn可能因为wts的不同而abort。即产生false positive

目前的paper中没有提出防止phantom的方法。

TicToc还可以利用parallel logging来加速。思路就是将log分为batch，同一个batch的txn的log可以是任意顺序的。而不同batch的log需要有序。我们可以计算出前一个batch的log的最大ts，然后当前batch的最小ts就设为前一个batch的最大ts即可。

（如果只有redo的log的话，是不是不需要有顺序？）

![20220717102342](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220717102342.png)

这个equal-delta表示的是something is equal to something by definition.

即A在串行化顺序中小于B，是等于A的ts小于B，或者A的ts等于B，且A的pt小于B。如果他们没有冲突的话，那么A的ts是可能等于B的，那么他们两个的串行化顺序是无所谓的，所以这里用物理时间来标识。

其实还有另一种情况就是读事务读到了写事务写的内容，这时候读事务可能拥有和写事务相同的logical ts，但是在serial order中读事务一定要在写事务之后。所以物理时间也是需要的。所以只有在相同的物理时间且相同的逻辑时间的情况下，他们可以是任意的。否则我们仍然需要物理时间来确定serial order。

# Optimizations

validation阶段拿锁的时候，我们可以根据primary key order去拿锁，从而防止死锁问题。但是会引入额外的等待。

![20220717105746](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220717105746.png)

这个例子中，ABCD就会变成串行执行。所以一个优化就是利用no-wait的策略，当获取锁失败的时候，就放锁，睡眠一小段时间，然后重启validation phase。

这样的话上面的C就会立刻abort，从而允许B继续执行。

支持SI。SI允许读写ts发生在不同的ts中。并且要求这个时间段中没有其他的人写和当前事务的写有交集。所以在TicToc中，我们可以使用两个ts来分离开读写。即commit rts和commit wts。验证读集的时候只需要验证commit rts，验证写集的时候只需要验证commit wts。

![20220717111059](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220717111059.png)

SI其实是将这个拆开，对于读集的元素，只要保证commit rts在他们之间，commit wts要大于写集的rts。当然还要保证commit wts大于等于commit rts。计算出这两个ts后，要验证一下write set中的wts不能存在于commit rts和commit wts之间。

具体的，可以先计算commit rts，然后设置commit wts最小值为commit rts，然后计算commit wts。最后验证write set的wts要小于commit rts。

