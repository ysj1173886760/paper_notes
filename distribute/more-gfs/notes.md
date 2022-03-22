对GFS的一些补充，主要是来自mit pdos

### 为什么atomic record append是至少一次，而不是exactly-once？

如果一次写操作失败的话（有可能只是一个从副本失败了），客户端就会重试这次写操作。这会导致在没有失败的地方会出现重复的数据。

其实可以去修改设计让服务器检测到重复的请求，但是这样会影响performance，以及影响复杂性

### Application是怎么知道一个chunk中的数据是padding或者重复数据呢？

对于padding来说，用户可以在有效的record之前放上一个magic number，或者放上校验和，并且只有record有效的时候校验和才有用。

而对于重复元组可以为元组添加unique ID，这样在读的时候就可以知道这个元组是曾经读到过的。（大量的元组会不会导致去重困难？或者排序后去重？）

GFS提供了一个库来处理这些情况

### 用户怎么找到他们的数据当atomic record append会在一个不确定的offset中写入？

Append的应用主要是为了后续读全部文件，他们希望获得的是元组的集合，而不是元组的位置。所以offset不重要

### checksum 是什么？

checksum就是输入一堆字节，然后返回他们聚合的值，比如是一个和。GFS在chunk内部存储了chunk的checksum。当GFS尝试写入一个chunk的时候，他会首先计算这个chunk的checksum，并和chunk一同写入磁盘。当读取的时候，他们就会重新计算校验和，并和已有的校验和进行检查。如果校验和不匹配，则说明数据损坏了。那么我们可以去别的chunk server读取数据。有的GFS应用会存储他们自己的校验和。从而区分padding和有效的元组。

### reference counts是什么？

在GFS中，他们是为了snapshot服务的。当GFS创建snapshot的时候，他不会去复制chunk，而是增加对应chunk的reference count。当写入的时候，master会发现这个reference count大于1。那么master会首先去复制这个chunk，从而让client来更新这个副本。就是COW技术的应用

### GFS怎么确定最近的副本在哪里？

paper中说到是通过IP地址。Google可能特意的做了这样的操作，如果机器在同一个机房或者机架，那么他们的地址会相近

### 租约是什么？

租约就是在某一段时间中，一个chunk server会成为一个chunk的primary。从而防止我们需要重复的询问谁是primary

### GFS中，如果一个primary出现了partition，此时master分配了第二个primary，会不会出现两个primary？

不会，因为GFS保证了他只有在前一个人超过租约时间以后才会分配第二个primary。所以当出现第二个primary的时候，第一个primary的lease已经过期了。

### 64MB貌似太大了？

GFS分割和存储文件的粒度就是64MB。客户端可以进行更小的操作，而不是每次必须进行64MB的数据处理。使用大块的目的主要是减少元数据的存储，并且减少需要大量数据传输的client的限制。第二点则是如果文件太小则我们无法获得太多的并行性。（overhead会更大）

### GFS是怎么处理数据的正确性和performance以及simplicity？

强一致性需要复杂的协议，同时需要机器之间的通信。通过利用特定程序可以容忍宽松一致性的方式，可以设计出性能良好的系统。比如GFS针对MapReduce进行优化，从而获取高文件读取性能，并且这些程序对于重复数据，文件中有hole，以及不一致的读取是可以容忍的。然而GFS则不适合去存储银行数据

### Master failed怎么办？

需要存在具有主状态完整副本的replica。比如可以通过Raft去切换到备份

### single master是一个好主意吗？

这个想法简化了最初的部署。但是长远来看，当文件数量足够多的时候，我们无法将所有文件的元数据存储在一个RAM中。client足够多以至于一个master没有足够的能力来为他们服务。并且GFS中从故障master切换到备机需要人来操作，使得恢复变慢。

### GFS通过弱一致性获得了什么？

通过思考我们怎么让GFS实现强一致性来思考这个问题

在写入数据的时候，我们需要一个2PC的过程来保证原子的写入（exactly-once）

我们还需要保证secondaries具有最新的primary数据，从而在primary宕掉的时候成为最新的primary

处理client重新发送的消息

client会缓存chunk的位置，但是这个位置有可能是过时的。我们还需要保证这个操作不会成功

感觉这些问题都可以通过chunk副本之间的共识来完成。但是需要消耗的代价就太多了。