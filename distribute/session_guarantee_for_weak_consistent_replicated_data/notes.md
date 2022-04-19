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