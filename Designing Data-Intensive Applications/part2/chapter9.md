# 第九章 一致性与共识

*活着但是是错的与正确但是死掉了，哪个更好？*

杰·克莱普斯，*关于Kafka与Jepsen的几个注意事项*（2013）

---

正如第8章所讨论的，分布式系统中许多事情会出错。处理这些故障最简单方法就是直接让整个服务失效，并向用户显示错误消息。如果这个解决方案不可接受，我们需要找到容错的方法——也就是即使某些内部组件出现故障，也可以保持服务正常运行。

在这一章里，我们将讨论构建可容错分布式系统算法与协议的一些例子。我们将假设第8章中的所有问题都会发生：网络中的数据包会丢失，会被重新排序，会重复或任意地延迟; 时钟最多是近似的; 并且节点可以暂停（例如，由于垃圾收集）或者随时崩溃。

构建容错系统的最佳方式是找到一些具有有用保证的通用的抽象概念，实现它们，然后让应用程序依赖这些保证。 这与我们在第7章中用于事务的方法相同：通过使用事务，应用程序可以假装没有崩溃（原子性），没有其他人同时访问数据库（隔离性），并且存储设备是完全可以依赖的（耐用性）。即使发生崩溃，竞态条件和磁盘故障确实会发生，但是事务抽象隐藏了这些问题所以应用程序不需要担心它们。

我们现在将继续沿着同样的路线前进，尝试寻找能够让应用程序忽略分布式系统部分问题的抽象概念。例如，分布式系统最重要的抽象之一就是共识：也就是说，让所有的节点都能达成一致。我们将在本章中看到，尽管存在网络故障和进程故障，可靠地达成一致是个令人惊讶的棘手问题。

一旦实现了协商一致，应用程序可以将其用于各种目的。例如，假设您有一个单主机复制的数据库。如果主机失效而您需要故障迁移到另一个节点，那么其余的数据库节点可以使用协商一致来选出新的主机。正如在“处理节点中断”中所讨论的，重要的是只有一个主机，并且所有节点都同意谁是主机。如果两个节点都认为自己是主机，那么这种情况就叫做裂脑，并且常常导致数据丢失。正确实现协商一致才可以避免此类问题。

在本章的后面，我们将在“分布式事务与协商一致”一节中研究解决协商一致以及相关问题的算法。但是首先，我们需要探索在分布式系统中可以提供的各种保证和抽象概念。

我们需要了解能做什么不能做什么的范围：在某些情况下，系统是可以容忍错误并继续工作的；在其他情况下，这就不可能了。在理论证明和实际实现中已经深入探讨了什么可能什么不可能的极限。我们将在这一章里概述这些基本限制。

分布式系统领域的研究人员几十年来一直在研究这些问题，因此有大量的材料——我们只会接触一些表面问题。在这本书中，我们没有空间去深入到正式模型与证明的细节，所以我们将坚持非正式的直觉。如果你感兴趣的话，这些参考文献提供了足够多的深度讨论。

## 一致性保证

在“复制延迟问题”一节中，我们研究了复制了的数据库中出现的一些计时问题。如果同时查看两个数据库节点，你很可能会在两个节点上看到不同的数据，因为写入请求在不同的时间到达不同的节点。无论数据库使用哪种复制方法（单主机、多主机或无主机复制），都会发生这些不一致的情况。

大多数复制了的数据库至少提供了最终一致性，这意味着如果停止对数据库的写入并等待一段时间，那么最终所有读取请求都会返回相同的值。换句话说，不一致是暂时的，最终是可以自己解决的（假设任何的网络故障也最终被修复了）。最终一致性的一个更好的名称可能是收敛，因为我们期望所有的副本最终收敛到相同的值。

然而，这是一个很弱的保证——它没有提到副本什么时候会收敛。在收敛之前，读取可以返回任何内容或什么都不返回。例如，如果写入一个值然后马上读取它，则无法保证你会看到刚才写入的值，因为读取请求可能被路由到另一个副本（见“读取你自己的写入”一节）。

对于应用程序开发人员来说最终一致性是很难，因为它与普通单线程程序中变量的行为非常不同。如果给变量赋值然后立即读取它，你不会期望读到旧值，或者读取失败。数据库表面上看起来像一个可以读写的变量，但实际上它有更复杂的语义。

当使用只提供弱保证的数据库时，您需要一直意识到它的局限性，而不是意外地假设太多。Bug通常很微妙，很难通过测试找到，因为大部分时间应用程序都能正常工作。最终一致性的边缘情况只有在系统中出现故障（例如网络中断）或高并发时才会变得明显。

在这一章里我们将探讨数据系统会选择提供的更强的一致性模型。它们并不是免费的：与有较弱保证的系统相比，具有更强保证的系统性能会更差，容错性会更低。尽管如此，因为更方便正确地使用，更有力的担保是有吸引力的。一旦您了解了几个不同的一致性模型，您就可以更好地决定哪一个最适合您的需要。

分布式一致性模型与我们前面讨论过的事务隔离级别的层次结构有一些相似之处（见“弱隔离级别”一节）。但是虽然有一些重叠之处，但它们绝大部分是独立的：事务隔离主要是为了避免由于并发执行事务而产生的竞争条件，而分布式一致性则主要是在遇到延迟和错误时协调副本的状态。

这一章涵盖了广泛的主题，但正如我们将看到的，这些领域之间实际上有着深刻的联系：

* 我们会首先研究其中一个最常用的最强一致性模型，线性化模式，并研究其利与弊。

* 然后我们将研究分布式系统中的事件排序问题（“排序保证”），特别是围绕因果关系和完全排序的问题。

* 在第三部分（“分布式事务和协商一致”）中，我们将探讨如何原子性地提交分布式事务，这将最终为协商一致问题找到解决方案。

## 线性化

在最终一致的数据库中，如果同时问两个不同的副本相同的问题，你可能会得到两个不同的答案。这太让人困惑了。如果数据库能给人一种错觉，以为只有一个副本（即只有一个数据副本），那岂不是简单得多吗？这样，每个客户端都将拥有相同的数据视图，而不必担心复制滞后。

这就是线性化（也称为*原子一致性*[7]、*强一致性*、*即时一致性*或*外部一致性*）背后的思想。线性化的确切定义是非常微妙的，我们会在这一节剩下的部分中对其进行探讨。但是基本的思想是让系统看起来好像只有一个数据的副本，并且对它的所有操作都是原子的。有了这个保证，即使现实中会有多个副本，应用程序也不需要担心它们。

在一个可线性化的系统中，一旦一个客户端成功地完成了一次写入，所有正在从数据库读取的客户端都必须能够看到刚刚写入的值。保持单个数据副本的错觉意味着确保读取的值是最近更新的值，而不是来自陈旧的缓存或副本。换句话说，线性化是新近性的保证。为了澄清这个想法，让我们看一个非线性化的系统的例子。

*图9-1 系统是非线性化的，这让球迷非常的困惑*

图9-1展示了一个非线性体育网站的例子。爱丽丝和鲍勃坐在同一个房间里，两人都在手机上查看2014年世界杯决赛的结果。最后的比分刚刚公布以后，爱丽丝刷新了页面，看到宣布了获胜者，兴奋地告诉了鲍勃。鲍勃疑惑地点击了手机上的重新加载按钮，然而他的请求转到了一个滞后的数据库副本，于是他的手机显示比赛仍在进行。

如果爱丽丝和鲍勃同时点击重新加载按钮，得到两个不同的查询结果就不会那么令人惊讶了，因为他们不知道服务器处理各自请求的确切时间。然而，鲍勃知道他在听到爱丽丝惊呼最后的分数后按了重新加载按钮（启动了他的查询），因此他希望他的查询结果至少和爱丽丝的一样新。他的查询返回了一个旧结果的事实违反了线性化。

### 是什么让系统线性化的？

线性化背后的基本思想很简单：使系统看起来好像只有一个数据副本。然而，准确地确定这意味着什么，实际上要很小心。为了更好地理解线性化，让我们看更多的例子。

图9-2展示了三个客户端同时在线性化数据库中读写相同的键*x*。在分布式系统文献中，*x*被称为*寄存器*——在实践中，它可以是键值对存储中的一个键，关系型数据库中的一行，或者文档型数据库中的一个文档。

*图9-2 如果读取请求与写入请求并发，它要么返回新值，要么是旧值。*

为了简单起见，图9-2只展示了客户端角度的请求，而不是数据库的内部。每个条形框是一个客户端发出的请求，条形框的起始位置是发送请求的时间，终止位置是客户端接收响应的时间。由于可变的网络延迟，客户端不知道数据库什么时候会处理它的请求——它只知道发生的时间一定在客户端发送请求之后与接收响应之间。

在本例中，寄存器有两种类型的操作：

* *read(x) ⇒ v*表示客户端请求读取寄存器*x*的值，数据库返回了值*v*。

* *write(x, v) ⇒ r*表示客户端请求将寄存器*x*设置为值*v*，数据库返回响应*r*（可以是*正常*或*错误*）。

在图9-2中，x的值初始为0，客户端C执行写入请求将其设置为1。当这种情况发生时，客户端A和B在反复轮询数据库以读取最新值。A和B在它们的读取请求中可能得到什么响应？

* 客户端A的第一个读取操作在写入开始之前完成，因此它必然返回旧值0。

* 客户端A的最后一次读取是在写入完成后开始的，因此，如果数据库是可线性的，则必须返回新的值1：我们知道，数据库写入必然是在写入操作开始和结束之间的某个时间进行的，读取数据库必然是在读取操作开始和结束之间的某个时间进行的。如果读取是在写入结束后开始的，那么读取的处理必然是在写入之后的，因此它必然看到写入的新值。

* 任何与写入操作重叠的读取操作都可能返回0或1，因为我们不知道在处理读取操作时写入是否已经生效。这些操作与写入是并行的。

但是，这还不足以充分地描述线性化：如果与写入并发的读取操作可以返回旧值也可以返回新值，那么读者在写入过程中可以多次看到旧值和新值之间的来回翻转。这不是我们所期望的、可以模拟“单一数据副本”的系统。

为了使系统可线性化，我们需要添加另一个约束，如图9-3所示。

*图9-3 在任何一个读取操作返回新值之后，之后所有（在相同或其他客户端上的）读取操作也必须返回新值。*

在线性化系统中我们设想必然有某个时间点（在写入操作的开始和结束之间）*x*的值从0原子翻转为1。因此，如果一个客户端的读取操作返回了新的值1，那么所有后续读取操作也必须返回新的值，即使写入操作还没有完成。

图9-3中的箭头展示了这种时间依赖性。客户端A是第一个读到新值的，1。在A的读取请求返回之后，B开始了新的读取操作。由于B的读取操作严格地发生在A的读取操作之后，所以它也必须返回1，即使C的写入仍在进行之中。（这与图9-1中的爱丽丝和鲍勃的情况相同：爱丽丝读取了新值之后，鲍勃也希望读取到新值。）

我们可以进一步细化这个时间图，从而可视化每个发生在某个时间点的，原子性生效的操作。图9-4显示了一个更复杂的例子。

在图9-4中我们添加了读取与写入之外的第三种操作：

* *cas(x, v<sub>old</sub>, v<sub>new</sub>） ⇒ r*意味着客户端请求了原子性的比较后设置操作（见“比较后设置”一节）。如果寄存器x的当前值等于*v<sub>old</sub>*，则应该原子地将其设置为*v<sub>new</sub>*。如果*x ≠ v<sub>old</sub>*，那么操作应该保持寄存器不变，并返回一个错误。*r*是数据库的响应（*正常*或*错误*）。

在我们认为操作被执行的时侯，图9-4中的每个操作都用垂直线（每个操作的条形框内部）标记。这些标记按顺序连接，结果对于寄存器来说必须是有效的读写顺序（每个读取操作必须返回由最近一次写入设置的值）。

线性化的要求是连接操作标记的线总是在时间上向前移动（从左到右），而不是向后移动。这个要求确保了我们前面讨论过的新近性保证：一旦一个新的值被写入或读取，所有后续的读取都会看到被写入的值，直到它再次被覆写。

*图9-4 可视化读写生效的时间点。B的最终读取操作不是线性化的。*

图9-4中一些有趣的细节需要指出：

* 首先客户端B发送读取*x*的请求，然后客户端D发送将*x*设置为0的请求，然后客户端A发送将x设置为1的请求。然而，返回给B的读值是1（由A写的值）。这是正常的：这意味着数据库首先处理了D的写入请求，然后是A的写入请求，而最后是B的读取请求。虽然这不是请求发送的顺序，但是这是一个可接受的顺序，因为这三个请求是并发的。也许B的读取请求在网络中稍微延迟了一些，所以在两次写入完成之后才到达数据库。

* 客户端B的读取请求在客户端A从数据库接收到响应之前返回1，说明值1的写入是成功的。这也是正常的：这并不意味着值是在写入之前读取的，它只是意味着从数据库到客户端A的*正常*响应在网络中稍微延迟了。

* 此模型不假定任何事务隔离：另一个客户端会随时更改值。例如，C首先读取1，然后读取2，因为B在两个读取请求之间更改了值。原子性的比较后设置（CAS）操作可以用于检查没有被另一个客户端更改的值：B和C的*比较后设置*请求成功，但D的*比较后设置*请求失败了（当数据库处理它时，*x*的值不再为0）

* （在阴影条形栏中的）客户端B的最后一次读取操作不是线性化的。这个操作与C的*比较后设置*写入并行，它把*x*从2更新到4。在没有其他请求的情况下，B的读取返回2是正常的。然而，客户端A在B的读取开始之前已经读取了新值4，所以B不允许读取比A更老的值。这与图9-1中的爱丽丝和鲍勃的情况还是一样的。

这就是线性化背后直觉的知识；正式的定义更精确地描述了它。通过记录所有请求和响应的时间，并检查它们是否可以排列成有效的顺序[11]，可以测试系统的行为是不是线性化的（尽管计算的成本很高）。

> 线性化 vs 串行化
>
> 线性化很容易与串行化混淆（参见“串行化”），因为这两个词似乎都意味着“可以按顺序排列”。然而，它们是两种完全不同的保证，必须加以区分：
>
> *串行化*
>
> 串行化是事务的隔离属性，每个事务都可以读写多个对象（行、文档、记录）——见“单对象与多对象操作”一节。它保证事务的行为与它们按照某种串行顺序执行的行为相同（每个事务在下一个事务启动之前执行完毕）。这个串行顺序与实际执行事务的顺序不同是可以的。
>
> *线性化*
>
> 线性化是寄存器（单个对象）读写的新近性保证。它不会将操作分组到事务中，因此不会防止诸如写偏（见“写偏和幻影”一节）等问题，除非采取了其他措施，例如物化冲突（参见“物化冲突”）。
>
> 数据库可以同时提供可串行化和线性化，这种组合被称为*严格串行化*或*强单拷贝串行化*（strong-1SR）[4，13]。基于两阶段锁定（见“两阶段锁定（2PL）”一节）或实际串行执行（参见“实际串行执行”）的串行化实现通常是线性化的。
>
> 然而，串行化快照隔离（见“串行化快照隔离（SSI）”一节）不是线性化的：根据设计，它从一致快照进行读取，以避免读取器和写入器之间的锁定争用。一致快照的全部要点是，它不包括比快照更近期的写入，因此从快照中读取的内容不是线性化的。

### Relying on Linearizability

In what circumstances is linearizability useful? Viewing the final score of a sporting match is perhaps a frivolous example: a result that is outdated by a few seconds is unlikely to cause any real harm in this situation. However, there a few areas in which linearizability is an important requirement for making a system work correctly.

#### Locking and leader election

A system that uses single-leader replication needs to ensure that there is indeed only one leader, not several (split brain). One way of electing a leader is to use a lock: every node that starts up tries to acquire the lock, and the one that succeeds becomes the leader [14]. No matter how this lock is implemented, it must be linearizable: all nodes must agree which node owns the lock; otherwise it is useless. 

Coordination services like Apache ZooKeeper [15] and etcd [16] are often used to implement distributed locks and leader election. They use consensus algorithms to implement linearizable operations in a fault-tolerant way (we discuss such algorithms later in this chapter, in “Fault-Tolerant Consensus”). iii There are still many subtle details to implementing locks and leader election correctly (see for example the fencing issue in “The leader and the lock”), and libraries like Apache Curator [17] help by providing higher-level recipes on top of ZooKeeper. However, a linearizable storage service is the basic foundation for these coordination tasks.

Distributed locking is also used at a much more granular level in some distributed databases, such as Oracle Real Application Clusters (RAC) [18]. RAC uses a lock per disk page, with multiple nodes sharing access to the same disk storage system. Since these linearizable locks are on the critical path of transaction execution, RAC deployments usually have a dedicated cluster interconnect network for communication between database nodes.

#### Constraints and uniqueness guarantees

Uniqueness constraints are common in databases: for example, a username or email address must uniquely identify one user, and in a file storage service there cannot be two files with the same path and filename. If you want to enforce this constraint as the data is written (such that if two people try to concurrently create a user or a file with the same name, one of them will be returned an error), you need linearizability. 

This situation is actually similar to a lock: when a user registers for your service, you can think of them acquiring a “lock” on their chosen username. The operation is also very similar to an atomic compare-and-set, setting the username to the ID of the user who claimed it, provided that the username is not already taken.

Similar issues arise if you want to ensure that a bank account balance never goes negative, or that you don’t sell more items than you have in stock in the warehouse, or that two people don’t concurrently book the same seat on a flight or in a theater. These constraints all require there to be a single up-to-date value (the account balance, the stock level, the seat occupancy) that all nodes agree on.

In real applications, it is sometimes acceptable to treat such constraints loosely (for example, if a flight is overbooked, you can move customers to a different flight and offer them compensation for the inconvenience). In such cases, linearizability may not be needed, and we will discuss such loosely interpreted constraints in “Timeliness and Integrity”.

However, a hard uniqueness constraint, such as the one you typically find in relational databases, requires linearizability. Other kinds of constraints, such as foreign key or attribute constraints, can be implemented without requiring linearizability [19].

#### Cross-channel timing dependencies

Notice a detail in Figure   9-1: if Alice hadn’t exclaimed the score, Bob wouldn’t have known that the result of his query was stale. He would have just refreshed the page again a few seconds later, and eventually seen the final score. The linearizability violation was only noticed because there was an additional communication channel in the system (Alice’s voice to Bob’s ears).

Similar situations can arise in computer systems. For example, say you have a website where users can upload a photo, and a background process resizes the photos to lower resolution for faster download (thumbnails). The architecture and dataflow of this system is illustrated in Figure   9-5.

The image resizer needs to be explicitly instructed to perform a resizing job, and this instruction is sent from the web server to the resizer via a message queue (see Chapter   11). The web server doesn’t place the entire photo on the queue, since most message brokers are designed for small messages, and a photo may be several megabytes in size. Instead, the photo is first written to a file storage service, and once the write is complete, the instruction to the resizer is placed on the queue.

*Figure 9-5. The web server and image resizer communicate both through file storage and a message queue, opening the potential for race conditions.*

If the file storage service is linearizable, then this system should work fine. If it is not linearizable, there is the risk of a race condition: the message queue (steps 3 and 4 in Figure   9-5) might be faster than the internal replication inside the storage service. In this case, when the resizer fetches the image (step 5), it might see an old version of the image, or nothing at all. If it processes an old version of the image, the full-size and resized images in the file storage become permanently inconsistent.

This problem arises because there are two different communication channels between the web server and the resizer: the file storage and the message queue. Without the recency guarantee of linearizability, race conditions between these two channels are possible. This situation is analogous to Figure   9-1, where there was also a race condition between two communication channels: the database replication and the real-life audio channel between Alice’s mouth and Bob’s ears. 

Linearizability is not the only way of avoiding this race condition, but it’s the simplest to understand. If you control the additional communication channel (like in the case of the message queue, but not in the case of Alice and Bob), you can use alternative approaches similar to what we discussed in “Reading Your Own Writes”, at the cost of additional complexity.