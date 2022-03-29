首先明确，bigtable是一个用来管理结构化数据的分布式存储系统

# Introduciton

bigtable的目标就是去scale到大数据的规模。并且应用范围比较广，从面向吞吐量的批处理任务到延迟敏感的用户服务。

bigtable并不提供完整的关系数据模型。而是提供了一个较为简单的数据模型，并且支持动态的控制数据的布局以及格式。从而让客户去推断数据的位置属性。

数据通过行和列名来进行索引。

bigtable将数据看作是未解析的字符串。

用户可以通过他们定义的schema来控制他们数据的位置。并且bigtable还允许客户控制他们是通过磁盘还是主存来提供数据

# Data Model

bigtable是一个稀疏的，分布式的，持久化的多维有序的map

map通过行键和列键以及时间戳来索引。得到的值就是未解释的字节数组。

![20220327185827](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220327185827.png)

一个具体的例子就是存储网页和相关的信息。

用URL作为行键，各种其他的信息作为列键，然后把网页内容存储在content中。content是一个带有时间戳的列

![20220327191450](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220327191450.png)

这个图等下解释

## Rows

行键也是一个任意的字符串。每次对于一行数据的读写是原子的。

bigtable中的行按照字典序排序。并会自动进行分区。每一个行的范围称为一个tablet，是分布式以及负载均衡的单位。

对于一个小范围的行的读，通常只需要涉及到很少一部分机器的通信，所以会比较高效。用户可以通过利用这个属性来提高数据访问的局部性。

## Column Families

列键会被分组成为若干个集合，成为列族。列族是访问控制的基本单位。同一个列族中的数据通常是相同类型的。

磁盘和内存的accounting都是在列族级别的。这里的意思应该就是我们可以制定某一个列族是通过磁盘还是内存来服务。

## Timestamp

bigtable中的每个单元都可以存储多个版本的数据。我们通过时间戳去索引这些版本的数据。

bigtable允许我们通过一些列族级别的设置来进行垃圾回收。比如我们只保存最新的n个版本的数据，或者只保存最近几天的数据

这时看到上面的例子就可以理解一些

我们有两个列族，分别是content和anchor

content的值中有三个版本。

anchor这个列族中有两个列，分别是anchor:cnnsi.com和anchor:my.look.ca

不过数据模型这块并不是bigtable的关键，简单的理解成一个kv-storage就好

# API

![20220327210048](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220327210048.png)

一个修改bigtable的例子

打开对应的table，然后用RowMutation找到对应的行

为anchor列族添加一个元素，并删除另一个anchor

Apply会执行一次对Webtable的原子修改

我猜测应该是对同一个列族的操作是原子的

![20220327210926](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220327210926.png)

用scanner去扫描一个row中所有的anchor

我们也可以修改条件，去限制anchor，timestamp，row等，从而得到想要的内容

bigtable提供了单行事务。

所以我们就可以把bigtable看成是一个简易的数据库，或者高级一点的kv-storage。因为row-key是主键，而其他的则相当于是kv中存储了额外的schema来帮我们处理数据。

# Building Blocks

bigtable用GFS来存储log和数据文件。同时依赖一个集群管理系统来做任务调度，管理资源，处理机器失效以及监控机器状态。（这个应该说的是borg，也就是现在的kubernetes）

用SSTable file来存储bigtable的额数据。SSTable可以提供一个持久化的不可变的map。

每一个SSTable都存储了一系列的块（64kb可配置）。我们通过block index去索引这些块。当SSTable被打开的时候，这些索引是存储在主存中的。当我们想要查找某些数据的时候，就会根据主存中的index来进行二分查找。然后再从磁盘中读取对应的块

这样读取的时候就不需要读取整个的SSTable，而是根据块索引来进行读取。后面还会提到我们会对块进行分别的压缩，从而独立出块读取的操作。

bigtable还依赖一个高可用的分布式锁服务Chubby。Chubby通过Paxos来保证副本的一致性。Chubby提供了一个namespace，里面的每个文件都可以用作一个lock，并且对这个文件的读写是原子的。

每一个Chubby的client都维护了和Chubby的一个session。当client不能够去刷新他的session租约的时候，这个session就会过期。同时他会失去掉他获得的所有锁

bigtable通过chubby来做很多任务：
1. 确保同一时间只有一个master
2. 发现tablet server，以及处理tablet server的下线
3. 存储bigtable的schema（每个table的列族信息），以及访问控制信息

# Implementation

bigtable有三个组成部分
1. 用于链接客户端的库
2. 一个master server
3. 若干个tablet server

master负责将tablet分配给tablet server。检查tablet server的添加或者删除，对tablet server进行负载均衡，垃圾回收GFS中的文件（可能是被删除掉的tablet，在GFS中进行回收），以及处理schema的改变，比如table的创建或者列族的修改

每一个tablet server都会负责若干个tablet。每一个tablet server负责处理对这个tablet的读写操作，并且负责分裂过大的tablet。

和GFS类似，client的数据不需要过master。client直接和tablet server进行交互。并且bigtable的client不想要master来告知tablet的位置，所以大多数的client永远不会和master交互。

## Tablet Location

使用类似B+树的层级结构来存储tablet的位置信息

![20220327214800](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220327214800.png)

Chubby中存储了root tablet的位置

在metadata table中存储了所有的tablet的位置。每一个metadata tablet都存储了一部分的tablet的位置。而root tablet则是metadata table的第一个tablet。但是root tablet永远不会分裂，从而保证我们的层级结构不会超过三层。

在图中可以看到，root tablet存储了其他metadata tablets的位置，然后这些metadata tablet存储是data tablet的位置

metadata table中的行也是kv的，其中value就是tablet的位置，而key则是对应tablet最后一行以及tablet的identifier的编码

用户库会缓存这些tablet的位置。

当用户没有缓存的时候，我们需要三次RTT来查询到tablet的位置

但是当用户的缓存是过期的时候，我们需要六次。因为前三次每次都需要一个缓存不命中来去向上查询。

并且这些tablet的数据都是存储在主存中的，所以不需要去额外的访问GFS。我们也可以通过去预取tablet location来减少查询时候的成本。

在metadata table中还额外存储了一些log，用来存储一些事件，比如什么时候一个服务器开始服务对应的tablet。从而实现debugging和performance analysis

## Tablet Assignment

master跟踪当前有那些tablet server。以及tablet的分配信息。如果有tablet未分配给tablet server的时候，master就会找一个有足够空间的tablet server，发送一个tablet load request来将该tablet分配给他。

bigtable通过chubby来跟踪tablet server。当一个tablet server启动了，他就会在chubby的namespace中的一个特定目录下创建一个唯一的文件，并获得上面的锁。

master通过监视这个目录来追踪tablet server。当tablet server失效的时候，他就会失去对应chubby上的锁。由于网络分区的情况下，一个活着的服务器可能会丢失他与chubby的会话，他会重新尝试获得这个文件上的锁。如果文件被删除了，tablet server就会自杀，因为他不可能重新获得锁了。

当一个tablet server不再服务的时候，master负责将他之前负责的tablet重新分配。master会定期的向tablet server询问他们的状态（而不是向chubby）。当tablet server报告说他的锁失效的时候，或者master和tablet server出现分区的时候。master就会去chubby上自己尝试获得这个锁。如果master成功了，说明chubby没问题。那么master就会删除掉这个server的文件，从而保证他无法再服务。然后将这个tablet server之前的tablet放到未分配的表中。

当master与chubby的session过期的时候，master就会自杀。（我猜测是要开一个新的master继续服务，否则当一个tablet server失效的时候，master同时也失效了，那么这些对应的tablet就会是失效的状态）

当master启动的时候，他需要知道那些tablet被分配了，那些没有。master顺序执行以下步骤
1. master首先获取chubby中的master lock
2. master扫描chubby中的server directory。找到所有的在线的服务器
3. master与这些服务器通信，获知他们已经被分配了那些tablet
4. master去扫描metadata table，当遇到没有被分配的tablet的时候，他就会记录这些tablet

注意上面的步骤中，如果metadata table没有被assign的话，我们是无法去查找的。所以master会额外记录root table是否被assign了。

还有一点就是tablet的分裂，是由tablet server开始的。tablet server通过在metadata表中记录新的tablet的信息来提交这次split。当split被提交，tablet server就会告诉master。但是如果master或者tablet server失效的话，这个通知就会丢失。比如原本的tablet被分配到另一个服务器的时候，master会让新的tablet server去load这个tablet。新的tablet server就会发现数据已经被split了（比如通过范围等），这时他就可以告诉master这个数据已经被split了。

## Tablet Serving

![20220328100330](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220328100330.png)

对于修改操作来说，我们通过提交log来实现。最近提交的修改会被存储到主存中，叫做memtable。而老一些的修改则会被存储到SSTable中

对于重构一个tablet来说，metadata table中存储了一个SSTable的列表，以及一系列的redo point。服务器会把SSTable的索引读入到主存中（用来后续查找），并且根据redo point之后的日志重构memtable

当一个写操作到达tablet server的时候，server首先检查写操作是不是完好的，以及发送者的权限。（权限是通过读取Chubby中的一个存储了有权限写的writer列表的文件来实现的）

一个有效的更改操作会被写到日志中。组提交用来提高很多小型更改的吞吐量。当log提交了以后，这个更改就会被应用到memtable中

对于读操作来说，他也会首先检查操作的格式，以及权限。读操作会在memtable以及一系列的SSTable上进行操作。因为SStable以及memtable是有序的，所以我们可以实现高效的查找

由于我们的SSTable都是有序的，所以split以及merge的操作不会影响到我们的正常读写操作。

## Compactions

memtable会不断变大，当我们不断执行写操作。当memtable的大小到达了一个阈值的时候，现有的memtable会被冻结。我们会创建一个新的memtable。同时现有的memtable会被转化成一个SSTable并写入到GFS中。

这个过程叫做minor compaction这样做有两点好处
1. 减少memory useage
2. 减少了恢复时所需要重放的redo log

但是如果SSTable不断的增长，我们的读请求会需要查很多的SStable。通过周期性的执行一个后台的merging compaction。merging compaction会读取一些SSTable以及memtable，并输出一个新的SSTable。那些老的SSTable和memtable就可以安全的被删除掉

将所有的SSTable都合并为一个SSTable的叫做major compaction。对于某些删除的数据来说，他们会在老的SSTable中存在，而在新的SSTable中被记录为删除。当执行了major compaction以后，我们就只会保存一个版本的数据，不需要去记录那些数据被删除了。bigtable周期性的对tablet进行major compaction。从而允许我们进行垃圾回收。这也保证了数据的删除是一个周期性的操作，为一些敏感数据提供了很好的保证。

所以注意一共是三种，minor compaction， merging compaction以及major compaction

merging compaction和major compaction的不同？merging compaction只会涉及到最近的SSTable，通常比较快。并且他会保留tombstone标记，因为他不包含的数据可能在更老的SSTable中。而major compaction不涉及到memtable，并且压缩了所有的SSTable，所以他不需要包含tombstone

很关键的一点就是这些操作都可以在后台进行并且是天然并发的，因为他们都是不可变的。而一些database则不行，因为他们要处理locking的问题

# Refinements

这一节描述了对前面实现的一些调整，从而达到高性能，高可用

## Locality groups

用户可以将若干个列族分组到一起成为一个locality group。每个locality group对应了一个SSTable。通过分离不常一起访问的列族可以带来更加高效的读取。因为可以涉及到更少的GFS读

用户还可以指定一个locality group的读取方式，比如加载到主存。这样对应的SSTable在读取的时候就会被加载到tablet server中的主存中，后续的读取就会变快。这个技术对于频繁的小块的数据读取很有用，比如在metadata table中的location column family就是这样设置的。

## Compression

用户可以控制一个locality group的SSTable是否被压缩。我们可以单独的压缩每个SSTable块，从而可以让我们读取一小块数据，而不需要去解压缩整个SSTable

论文中提到他们使用的两阶段压缩主要是为了快速，但是他们的空间压缩比也很高。因为在SSTable中是按序存储的，这时候相似的数据都会存储到一起。从而让一些基于滑动窗口的算法的压缩比很高。并且在SSTable中存储多个版本的文件的时候压缩比会更高。

这里个人想法就是因为workload了。网页中相同的host的网页内容的样板也很类似。但是核心还是在于locality group的隔离性，让我们可以分开压缩分开读取。从而在减少空间使用的同时快速的读取。当然这也需要用户来指定访问的pattern。

## Caching for read performance

bigtable有两层cache
1. Scan Cache用来存储kv对，所以对于重复的数据请求我们就可以直接返回结果
2. Block Cache用来缓存从GFS中读到的SSTable块，这样我们就有一定的访问局部性

## Bloom filters

在上面描述读操作的时候，我们需要读取所有的SSTable直到找到最新的row。我们可以对每个SSTable创建一个bloom filter，这样可以快速确定SSTable中是否含有某个特定的行。通过在主存中放置bloom filter可以极大程度的减少我们访问GFS的频率。

## Commit-log implementation

如果我们对于每个tablet的log都独自存储一个文件，这样的话大量的文件写就会并发的去请求GFS。所以为了解决这个问题。使用的方法是一个tablet server一个commit log。

虽然一个log可以提高我们的性能，但是缺点就是当一个tablet server失效了，他负责的tablet可能分发到其他几个不同的tablet server中。

这样如果tablet分发到100个机器中，我们就会有100次重复读取，每个机器都只重放他对应的tablet的日志。

通过排序log来防止这种重复的读。这样一个tablet中的log就会放到一起

![20220328140256](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220328140256.png)

论文里提到的两个写入线程有点奇怪，这里没搞懂

受到胡哥的指导，这里的意思是两个日志线程分别写不同的GFS文件，这样当一个GFS文件坏掉或者网路出现问题的时候，我们就可以写另一个GFS文件。从而减缓了GFS波动带来的影响。

## Speeding up tablet recovery

如果master将一个tablet从一个服务器移动到另一个服务器。那么源服务器会首先做一次minor compaction。这样可以减少在那个tablet上的恢复时间。然后源服务器停止服务，在他卸载掉这个tablet之前，他会再做一次minor compaction。因为我们之前的压缩不会停止服务，所以在这个过程中可能又有新的log。所以在新的tablet server加载tablet的时候，他不需要任何的恢复。（目的应该是把log的恢复移动到SSTable中，因为我们的log都是写在一起的，再去排序比较耗时）

## Exploiting immutability

由于我们的SSTable都是不可变的，所以并发的操作不需要任何的同步手段。所以只有memtable是可变的，我们只需要在memtable上面实现简单的并发控制，就可以很容易的得到单行事务的支持。

为了不阻塞读，在memtable上实现COW，从而可以让读写并发的操作（MVCC，同时bigtable也有记录多版本的需求，所以很自然的就用多版本并发控制就好）

SSTable是不可变的，所以删除数据的操作就会转化为对SSTable的垃圾回收。我们通过在metadata table中移除掉对应的SSTable作为垃圾回收的标志。

同时由于SSTable是不可变的，我们在split的时候不需要去为每一个child生成一个新的SSTable，而是直接让child共享父SSTable

（这个共享的思路很棒呀，这样我们只需要重定向新的请求到对应的child中，他们新的memtable和SSTable就是对应范围的表，而老的表就直接查父亲的就行。之后就可以通过compaction去回收父亲的表，但是要额外加一个refcnt，两个人都用完才能真的删除这个SSTable file）

有一点要注意就是上面说的两级cache是在SSTable上的，而不是tablet。并且由于SSTable的不可变性，只要SSTable还在，缓存就有效

极客时间这个图比较好

![20220328145227](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220328145227.png)

# Performance

![20220328151004](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220328151004.png)

看这个图后面的scale的问题

没有线性拓展的一个主要因素就是负载均衡的问题。因为负载均衡需要我们移动tablet，从而导致一段时间的不可用。

随机读是性能最差的一个，因为每次读都需要我们从GFS中传输一个整块的文件。

对于顺序读写论文中没写，但是我觉得是受到了GFS的限制

# Lessons

大型分布式系统对于很多类型的错误都是非常脆弱的。而不是只是network partition以及失效等问题。

比如内存以及网络的错误导致的问题。时钟的偏斜。挂起的机器。非对称的网络分区。以及其他系统中的bug

另一个问题就是去延迟添加新的feature，直到我们清楚的理解到新的feature是怎么使用的。

最关键的一点是simple designs。最开始的时候tablet server的管理是通过master去发送lease来进行的。当一个tablet server的lease过期了，他们就会自杀。这个方法减少了我们系统的可用性，并且严重依赖于master的恢复时间。可能这也是为什么后来GFS被替换掉。

所以分离了这个职责给chubby，而master只负责管理tablet的分配，不负责处理tablet server的成员管理。

最后列一下我感觉比较不错的一个[文章](https://www.read.seas.harvard.edu/~kohler/class/cs261-f11/bigtable.html)，通过re-build的方式来讲解bigtable的设计

bigtable使用列族的主要目的：假设bigtable只是简单的kv-storage，由于我们只能保证单行的事务。那么我们能做的就只是原子的对一个值进行更改，这对于大多数的用户来说都是不够的。所以bigtable拓展了行的概念，让我们在一行里可以存储若干个kv对，并且不同的行还可以有不同的键。这样我们原本的atomic value operation就可以拓展成atomic structure operation。

所以虽然不支持整体的事务，但还是提供了一定程度的简便的编程模型。最直观的体现就是我们之前只能原子改一个值，现在可以改多个了。

![20220328191343](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220328191343.png)

和db的对比