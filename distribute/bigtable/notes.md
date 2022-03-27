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

每一个SSTable都存储了一系列的块。我们通过block index去索引这些块。当SSTable被打开的时候，这些索引是存储在主存中的。当我们想要查找某些数据的时候，就会根据主存中的index来进行二分查找。然后再从磁盘中读取对应的块

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

在metadata table中还额外存储了一些log，比如什么时候一个服务器开始服务对应的tablet。从而实现debugging和performance analysis