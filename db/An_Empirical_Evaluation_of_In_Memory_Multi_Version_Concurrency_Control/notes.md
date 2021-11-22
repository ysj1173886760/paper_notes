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

对于MVCC的read uncommitted，我们可以跟踪每个txn所读到的uncommitted data，并且只有读到的data全都commit以后才能提交当前事务

我们也可以让txn去更新一个被uncommitted txn读过的数据项。但是同样我们需要维护他们的依赖关系，并等所有依赖的txn提交之后，当前的txn才能提交

版本存储：

DBMS使用tuple中的pointer属性来创建一个latch-free的链表，就是version chain

添加新版本的方法有append-only，就是在后面追加

还有就是对每个tuple维护一个单独的表，里面存储了历史版本。主版本在主表中，历史版本单独存在一张表

好处就是遍历历史版本很快，我们有顺序扫描。同时我们可以直接断开历史版本的链接，达到快速回收的目的

还有一种就是优化，我们不存储每个tuple，而是存储他们的变化，也就是delta storage

好处就是空间占用小，但是我们需要额外的计算来重构tuple

对于存储来说，之前的论文有提到过，就是可以申请 thread local的空间，而不是去访问全局的数据结构。中心化的数据结构带来的冲突会影响到效率。这样对于每个worker划分空间的做法也就相当于对数据库进行了分区

垃圾回收：

我们需要回收那些被abort txn创建的tuple，以及不再能被现在的txn看到的tuple

通过一个中心化的数据结构来跟踪tuple是否能被当前活跃的txn看到。但是这样会因为访问中心化的数据结构所导致的冲突而产生瓶颈

主存数据库通过细粒度，epoch-based的方法来跟踪这些信息

每个epoch包含了一些txn，epoch中应该是维护了版本相关的信息，以及活跃的txn，当没有活跃的txn，并且这个epoch的版本不再能被新的txn看到的时候，就可以清除这个epoch

（这里我理解的不是很清楚，因为没有具体的实例）

GC有tuple level的以及txn level的

对于tuple level的，我们有background vacuuming，就是后台进程来回收

还有coop的方法，就是当一个新的txn读version chain的时候，他顺便把不会再被读到的版本标记并清除

txn level的就需要我们跟踪txn所更新的tuple，并且在这些tuple不再能被active txn看到的时候清除他们

索引管理：

我们有两种，逻辑指针和物理指针

对于物理指针来说，我们会在索引中指向版本链中所有的tuple。

当新的版本出现时，我们还需要将所有的辅助索引更新，添加这个新的key

对于逻辑指针来说，辅助索引指向的是主键或者一个对应tuple的唯一键，tuple-id之类的

逻辑指针指向的是版本链的表头，我们可能还需要一个额外的哈希表来映射tuple-id到具体的物理地址

但是更新的时候，我们就只需要修改这个哈希表即可，而不需要再去修改所有的辅助索引了

个人感觉就是没有实践的话，单独读论文的体验还是比较差的，我还是应该去找一些代码看，进而加深理解