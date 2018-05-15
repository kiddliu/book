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

在“复制延迟问题”一节中，我们研究了复制了的数据库中出现的一些计时问题。如果同时查看两个数据库节点，你很可能会在两个节点上看到不同的数据，因为写入请求在不同的时间到达不同的节点。无论数据库使用哪种复制方法(单主机、多主机或无主机复制)，都会发生这些不一致的情况。

大多数复制了的数据库至少提供了最终一致性，这意味着如果停止对数据库的写入并等待一段时间，那么最终所有读取请求都会返回相同的值。换句话说，不一致是暂时的，最终是可以自己解决的(假设任何的网络故障也最终被修复了)。最终一致性的一个更好的名称可能是收敛，因为我们期望所有的副本最终收敛到相同的值。

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

在最终一致的数据库中，如果同时问两个不同的副本相同的问题，你可能会得到两个不同的答案。这太让人困惑了。如果数据库能给人一种错觉，以为只有一个副本(即只有一个数据副本)，那岂不是简单得多吗？这样，每个客户端都将拥有相同的数据视图，而不必担心复制滞后。

这就是线性化(也称为*原子一致性*[7]、*强一致性*、*即时一致性*或*外部一致性*)背后的思想。线性化的确切定义是非常微妙的，我们会在这一节剩下的部分中对其进行探讨。但是基本的思想是让系统看起来好像只有一个数据的副本，并且对它的所有操作都是原子的。有了这个保证，即使现实中会有多个副本，应用程序也不需要担心它们。

在一个可线性化的系统中，一旦一个客户端成功地完成了一次写入，所有正在从数据库读取的客户端都必须能够看到刚刚写入的值。保持单个数据副本的错觉意味着确保读取的值是最近更新的值，而不是来自陈旧的缓存或副本。换句话说，线性化是新近性的保证。为了澄清这个想法，让我们看一个非线性化的系统的例子。

*图9-1 系统是非线性化的，这让球迷非常的困惑*

图9-1展示了一个非线性体育网站的例子。爱丽丝和鲍勃坐在同一个房间里，两人都在手机上查看2014年世界杯决赛的结果。最后的比分刚刚公布以后，爱丽丝刷新了页面，看到宣布了获胜者，兴奋地告诉了鲍勃。鲍勃疑惑地点击了手机上的重新加载按钮，然而他的请求转到了一个滞后的数据库副本，于是他的手机显示比赛仍在进行。

如果爱丽丝和鲍勃同时点击重新加载按钮，得到两个不同的查询结果就不会那么令人惊讶了，因为他们不知道服务器处理各自请求的确切时间。然而，鲍勃知道他在听到爱丽丝惊呼最后的分数后按了重新加载按钮(启动了他的查询)，因此他希望他的查询结果至少和爱丽丝的一样新。他的查询返回了一个旧结果的事实违反了线性化。

### 是什么让系统线性化的？

线性化背后的基本思想很简单：使系统看起来好像只有一个数据副本。然而，准确地确定这意味着什么，实际上要很小心。为了更好地理解线性化，让我们看更多的例子。

图9-2展示了三个客户端同时在线性化数据库中读写相同的键*x*。在分布式系统文献中，*x*被称为*寄存器*——在实践中，它可以是键值对存储中的一个键，关系型数据库中的一行，或者文档型数据库中的一个文档。

*图9-2 如果读取请求与写入请求并发，它要么返回新值，要么是旧值。*

为了简单起见，图9-2只展示了客户端角度的请求，而不是数据库的内部。每个条形框是一个客户端发出的请求，条形框的起始位置是发送请求的时间，终止位置是客户端接收响应的时间。由于可变的网络延迟，客户端不知道数据库什么时候会处理它的请求——它只知道发生的时间一定在客户端发送请求之后与接收响应之间。

在本例中，寄存器有两种类型的操作：

* *read(x) ⇒ v*表示客户端请求读取寄存器*x*的值，数据库返回了值*v*。

* *write(x, v) ⇒ r*表示客户端请求将寄存器*x*设置为值*v*，数据库返回响应*r*(可以是*正常*或*错误*)。

在图9-2中，x的值初始为0，客户端C执行写入请求将其设置为1。当这种情况发生时，客户端A和B在反复轮询数据库以读取最新值。A和B在它们的读取请求中可能得到什么响应？

* 客户端A的第一个读取操作在写入开始之前完成，因此它必然返回旧值0。

* 客户端A的最后一次读取是在写入完成后开始的，因此，如果数据库是可线性的，则必须返回新的值1：我们知道，数据库写入必然是在写入操作开始和结束之间的某个时间进行的，读取数据库必然是在读取操作开始和结束之间的某个时间进行的。如果读取是在写入结束后开始的，那么读取的处理必然是在写入之后的，因此它必然看到写入的新值。

* 任何与写入操作重叠的读取操作都可能返回0或1，因为我们不知道在处理读取操作时写入是否已经生效。这些操作与写入是并行的。

但是，这还不足以充分地描述线性化：如果与写入并发的读取操作可以返回旧值也可以返回新值，那么读者在写入过程中可以多次看到旧值和新值之间的来回翻转。这不是我们所期望的、可以模拟“单一数据副本”的系统。

为了使系统可线性化，我们需要添加另一个约束，如图9-3所示。

*图9-3 在任何一个读取操作返回新值之后，之后所有(在相同或其他客户端上的)读取操作也必须返回新值。*

In a linearizable system we imagine that there must be some point in time (between the start and end of the write operation) at which the value of x atomically flips from 0 to 1. Thus, if one client’s read returns the new value 1, all subsequent reads must also return the new value, even if the write operation has not yet completed. 

This timing dependency is illustrated with an arrow in Figure   9-3. Client A is the first to read the new value, 1. Just after A’s read returns, B begins a new read. Since B’s read occurs strictly after A’s read, it must also return 1, even though the write by C is still ongoing. (It’s the same situation as with Alice and Bob in Figure   9-1: after Alice has read the new value, Bob also expects to read the new value.)

We can further refine this timing diagram to visualize each operation taking effect atomically at some point in time. A more complex example is shown in Figure   9-4 [10].

In Figure   9-4 we add a third type of operation besides read and write:

* cas( x,   vold,   vnew)   ⇒   r means the client requested an atomic compare-and-set operation (see “Compare-and-set”). If the current value of the register x equals vold, it should be atomically set to vnew. If x   ≠   vold then the operation should leave the register unchanged and return an error. r is the database’s response (ok or error).

Each operation in Figure   9-4 is marked with a vertical line (inside the bar for each operation) at the time when we think the operation was executed. Those markers are joined up in a sequential order, and the result must be a valid sequence of reads and writes for a register (every read must return the value set by the most recent write).

The requirement of linearizability is that the lines joining up the operation markers always move forward in time (from left to right), never backward. This requirement ensures the recency guarantee we discussed earlier: once a new value has been written or read, all subsequent reads see the value that was written, until it is overwritten again.

*Figure 9-4. Visualizing the points in time at which the reads and writes appear to have taken effect. The final read by B is not linearizable.*

There are a few interesting details to point out in Figure   9-4: 
* First client B sent a request to read x, then client D sent a request to set x to 0, and then client A sent a request to set x to 1. Nevertheless, the value returned to B’s read is 1 (the value written by A). This is okay: it means that the database first processed D’s write, then A’s write, and finally B’s read. Although this is not the order in which the requests were sent, it’s an acceptable order, because the three requests are concurrent. Perhaps B’s read request was slightly delayed in the network, so it only reached the database after the two writes.

* Client B’s read returned 1 before client A received its response from the database, saying that the write of the value 1 was successful. This is also okay: it doesn’t mean the value was read before it was written, it just means the ok response from the database to client A was slightly delayed in the network.

* This model doesn’t assume any transaction isolation: another client may change a value at any time. For example, C first reads 1 and then reads 2, because the value was changed by B between the two reads. An atomic compare-and-set (cas) operation can be used to check the value hasn’t been concurrently changed by another client: B and C’s cas requests succeed, but D’s cas request fails (by the time the database processes it, the value of x is no longer 0).

* The final read by client B (in a shaded bar) is not linearizable. The operation is concurrent with C’s cas write, which updates x from 2 to 4. In the absence of other requests, it would be okay for B’s read to return 2. However, client A has already read the new value 4 before B’s read started, so B is not allowed to read an older value than A. Again, it’s the same situation as with Alice and Bob in Figure   9-1.

That is the intuition behind linearizability; the formal definition [6] describes it more precisely. It is possible (though computationally expensive) to test whether a system’s behavior is linearizable by recording the timings of all requests and responses, and checking whether they can be arranged into a valid sequential order [11].

> Linearizability Versus Serializability
>
> Linearizability is easily confused with serializability (see “Serializability”), as both words seem to mean something like “can be arranged in a sequential order.” However, they are two quite different guarantees, and it is important to distinguish between them:
>
> *Serializability*
>
> Serializability is an isolation property of transactions, where every transaction may read and write multiple objects (rows, documents, records) — see “Single-Object and Multi-Object Operations”. It guarantees that transactions behave the same as if they had executed in some serial order (each transaction running to completion before the next transaction starts). It is okay for that serial order to be different from the order in which transactions were actually run [12].
>
> *Linearizability*
>
> Linearizability is a recency guarantee on reads and writes of a register (an individual object). It doesn’t group operations together into transactions, so it does not prevent problems such as write skew (see “Write Skew and Phantoms”), unless you take additional measures such as materializing conflicts (see “Materializing conflicts”).
>
> A database may provide both serializability and linearizability, and this combination is known as strict serializability or strong one-copy serializability (strong-1SR) [4, 13]. Implementations of serializability based on two-phase locking (see “Two-Phase Locking (2PL)”) or actual serial execution (see “Actual Serial Execution”) are typically linearizable.
>
> However, serializable snapshot isolation (see “Serializable Snapshot Isolation (SSI)”) is not linearizable: by design, it makes reads from a consistent snapshot, to avoid lock contention between readers and writers. The whole point of a consistent snapshot is that it does not include writes that are more recent than the snapshot, and thus reads from the snapshot are not linearizable.
