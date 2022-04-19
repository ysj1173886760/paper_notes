这个paper提出了session guarantee从而可以避免弱一致性级别带来的问题，同时还可以保持弱隔离级别的优势

A session is an abstraction for the sequence of read and write operations performed during the execution of an application

提出session的目的不是为了和事务对应（事务是用来保证ACID的），session的目的则是为了给用户提供一个一致性的视角。

贴一下原文：Sessions are not intended to correspond to atomic transactions that ensure atomicity and serializability. Instead, the intent is to present individual applications with a view of the database that is consistent with their own actions, even if they read and write from various, potentially inconsistent servers.

我们希望在一个session中的操作可以像是在一个服务器上进行的一样

在session的基础上，提供了四个保证：
* Read Your Writes - read operations reflect previous writes
* Monotonic Reas - successive reads reflect a non-decreasing set of writes
* Writes Follow Reads - writes are propagated after reads on which they depend
* Monotonic Writes - writes are propagated after writes that logically precede them

我们可以通过在一些弱一致性的系统上构建一个层来提供上面的这些保证。我们通过让应用操作的这些服务器是足够up-to-date的，来提供上面的这些保证

# Data storage model and terminology

假设下层的系统: Such a system consists of a number of servers that each hold a full copy of some replicated database and clients that run applications desiring access to the database.

session guarantees适用于那些客户和服务器处于不同的机器上，并且客户的访问会访问到不同的机器中的场景

比如一个移动客户端可能会根据他的地区来选择server

每一个写操作都有一个全局独一的标识符，称为WID

定义DB(S, t)为服务器S在t时间及以前收到的写操作的有序序列。概念上，服务器S从一个空的数据库开始，然后根据顺序apply接受到的写操作，与此同时处理读请求。现实中，服务器可以按照不同的顺序应用写操作，只要他们的效果是不变的就行（我猜测，可能是类似tomas写规则那样的？）。并且DB（S)中写操作的序列并不代表是他们接受到写操作的顺序（也就是说不是先接受就先应用，应该是根据WID来的）

我们假设底层的系统提供了最终一致性，那么也就提供了能够保证最终一致性的机制，即: total propagation和 consistent ordering

写操作通过反熵(anti-entropy)来在服务器之间进行传播，这个操作有的时候也叫做rumor mongering, lazy propagation, update dissemination。反熵保证了每一个写操作最终都会被每一个服务器接收到。（反熵带来了total propagation）

对于不可交换的操作（比如覆盖写），所有服务器必须有相同的应用顺序。形式化的说：定义WriteOrder(W1, W2)为一个布尔谓词，代表W1是否应该在W2之前。系统保证了如果有WriteOrder(W1, W2)，那么对于任何的接收到W1和W2的服务器来说，W1被排序到W2之前。这篇文章对于系统如何保证Order没有任何的假设，他只假设了在每台服务器上写操作都有一个一致的顺序。（比如想要写操作match real time，我们可以用一个TSO来做order，如果没有要求的话可以通过logical lock来做order）

最后，弱一致性系统通常允许写冲突，比如两个并发的写同一个数据项。在某些系统中，Write order会决定那个写胜利了，也有的系统会让人来解决这些冲突。系统怎么解决这些冲突对于用户很关键，但是对我们的session guarantees不重要。（我感觉是因为我们的session guarantee说的是一个人的一系列操作能看到的事情，即看到一个一致的数据库，而上面的冲突解决说的是怎么处理并发的操作）

# Read/Write guarantees

这一节就是定义一下，然后给了几个例子

## Read Your Writes

RYW-guarantee: If Read R follows Write W in a session and R is performed at server S at time t, then W is included in DB(S, t)

## Monotonic Reads

We say a Write set WS is `complete` for Read R and DB(S, t) if and only if WS is a subset of DB(S, t) and for any set WS2 that contains WS and is also a subset of DB(S, t), the result of R applied to WS2 is the same as the result of R applied to DB(S, t)

即对于R来说，WS是包含R中所有元素的最新的写操作的集合

Let RelevantWrites(S, t, R) denote the function that returns the smallest set of Writes that is complete for Read R and DB(S, t)

MR-guarantee: If Read R1 occurs before R2 in a session and R1 accesses server S1 at time t1 and R2 accesses server S2 at time t2, then RelevantWrites(S1, t1, R1) is a subset of DB(S2, t2)

也就是说t2时刻的S2至少要包含t1时刻S1中关于R1的写操作

## Writes Follow Reads

这个我的理解就是因果一致性了（类似因果一致性但是不是，后面会说）

WFR-guarantee: If Read R1 precedes Write W2 in a session and R1 is performed at server S1 at time t1, then for any server S2, if W2 is in DB(S2), then any W1 in RelevantWrites(S1, t1, R1) is also in DB(S2) and WriteOrder(W1, W2)

解释一下就是如果读后写的时候，对于新的写操作应用在服务器上的时候，要保证之前读到的那些值也已经被应用了，并且之前的写要排序在新的写操作之前。

这个guarantee和之前的区别在于他是在说session之外的事情，即其他的client应该也能看到相同的写操作的顺序

对于这个保证，其实我们是保证了两件事。一个是写操作的顺序，我们保证有相关性的写操作是有序的。第二个则是传播的顺序，我们保证只有当看到了依赖的写操作后，才能看到之后的写操作。

WFRO(Order)-guarantee: If Read R1 precedes Write W2 in a session and R1 is performed at server S1 at time t1, then WriteOrder(W1, W2) for any W1 in RelevantWrites(S1, t1, R1)

WFRP(propagate)-guarantee: If Read R1 precedes Write W2 in a session and R1 is performed at server S1 at time t1, then for any server S2, if W2 is in DB(S2) then any W1 in RelevantWrites(S1, t1, R1) is also in DB(S2)

其实就是把上面的这个拆开了，一个保证的是WriteOrder(W1, W2)，另一个保证的则是if W2 is in DB(S2), then W1 is also in DB(S2)

## Monotonic Writes

MW-guarantee: If Write W1 precedes Write W2 in a session, then, for any server S2, if W2 in DB(S2) then W1 is also in DB(S2) and WriteOrder(W1, W2)

就是写操作要有序并且按序传播

防止immortal write

在某些情况下，MW可以通过WFR和RYW推出。但是有的时候不能，比如操作W1 R W2，对于MW来说，W1应该在W2前面，但是对于WFR和RYW来说，R有可能读到了其他的写操作Wx，那么因果就变成了Wx happen before W2了，而W1和W2就没有了关系

通过jepsen的这个图来想一下

![20220419151234](https://picsheep.oss-cn-beijing.aliyuncs.com/pic/20220419151234.png)

PRAM是pipelined random access memory，指的是读写操作是向流水线一样产生效果的，所以我们天然的就有了MW，MR以及RYW，因为读写操作不会沿着流水线往回走，所以他们只会越来越新

PRAM没有对跨越session的操作做任何保证，所以我们加上WFR，来保证跨session的操作的因果一致性，就得到了Causal consistency。那么WFR和Causal的区别是什么呢？就是对于单个session的操作的顺序的保证。因为WFR只保证了读写依赖的情况下的顺序，但是没有保证单个session情况下的读写顺序，而causal consistency是保证了的。更准确的来说，应该是happen before的关系

而对于sequential来说，则没有什么更多额外的保证，只是在不可比较的事件之间提供了一个全序关系，让我们可以排序没有因果的事件。那么有个问题就是，sequential是不是就没什么作用了，毕竟他也没提供real time的保证，只是排序了不相关的东西。其实不是，因为当我们无法排序不相关的事件的时候，那么我们就无法保证某一时刻数据库的一致性，比如两个写操作同时写入同一个数据项，怎么确保谁先谁后呢？就算是我们有确定的手段确保这个顺序，还是会有异常。比如我们某一时刻的读可能在两个不同的服务器上读到的是不同的状态，那么我们之后所做的抉择就取决于去那个服务器上读了，也是一个比较奇怪的现象。所以目前的系统如果可以都会提供这样的全序关系。

# Providing the guarantees