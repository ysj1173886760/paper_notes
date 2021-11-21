感觉这篇自己理解的也不是很通透，之后还需要再看。这里记一些读到的要注意的点

在多核机器上拓展MVCC不是一个容易的事情，当线程的数量变的很多的时候，同步所造成的开销会大于并发带来的加速

MVCC的四个决策点：并发控制协议，版本存储，垃圾回收，索引管理

MVCC主要的好处就是writer don't block reader, 以及 Read-Only txns can read a consistent snapshot without acquiring lock

MVCC有个额外的好处就是如果不做垃圾回收的话，我们天然的可以得到time travel operation

在主存数据库中，MVCC的实现将一些meta-data内联到了tuple中。tuple header中有一项就是txn-id，他代表了这个tuple的latch

两个属性begin-ts和end-ts代表了这个tuple的生命期。还有一个pointer指向版本链中的上一个版本或者下一个版本

并发控制协议：

Timestamp Ordering(MVTO)

在header中维护一个read-ts，表示读到这个tuple的最大的txn的id，和Timestamp Ordering的方法一样，通过timestamp来确定执行顺序

对于同样想修改同一个tuple的不同txn，先修改的人胜利，后到的txn则会abort

MVTO允许txn读到一个还未提交的条目，所以为了保证可串行化，我们需要额外的数据结构来维护abort对tuple的影响。

多版本对于TO的改进其实是提供了一个consitent snapshot

OCC （MVOCC）

这个就是在原本OCC的基础上加上了多版本

好处在于我们不用担心写的冲突问题，因为多版本会帮我们解决。所以validation阶段我们只需要去检验read是否有效即可。即是否读到了不该读的，或者没读到该读的tuple

2PL （MV2PL）

在tuple中内嵌了一个read-cnt，表示读锁，原本的txn-id表示写锁。这样可以通过CAS操作来实现latch-free的操作

让txn提交的时候，我们释放latch，并更新tuple的版本

但是说白了还是2PL，我们还是会有等待，所也也会出现死锁等情况。

在这里，因为我们内嵌了锁，所以如果要进行DL_DETECT的话需要额外的数据结构。因此我们使用死锁预防，比如no-wait，或者wait-die

个人的理解，MV2PL还是有等待的，所以MV的添加其实是缓解了2PL的冲突。原本所有的txn都要等待一个tuple，现在有了多个版本，冲突也就相应的减少了。