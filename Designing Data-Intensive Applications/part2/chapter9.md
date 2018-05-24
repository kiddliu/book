# 第九章 一致性与共识

*活着但是是错的与正确但是死掉了，哪个更好？*

杰·克莱普斯，*关于Kafka与Jepsen的几个注意事项*（2013）

---

正如第8章所讨论的，分布式系统中许多事情会出错。处理这些故障最简单方法就是直接让整个服务失效，并向用户显示错误消息。如果这个解决方案不可接受，我们需要找到容错的方法——也就是即使某些内部组件出现故障，也可以保持服务正常运行。

在这一章里，我们将讨论构建可容错分布式系统算法与协议的一些例子。我们将假设第8章中的所有问题都会发生：网络中的数据包会丢失，会被重新排序，会重复或任意地延迟; 时钟最多是近似的; 并且节点可以暂停（例如，由于垃圾收集）或者随时崩溃。

构建容错系统的最佳方式是找到一些具有有用保证的通用的抽象概念，实现它们，然后让应用程序依赖这些保证。 这与我们在第7章中用于事务的方法相同：通过使用事务，应用程序可以假装没有崩溃（原子性），没有其他人同时访问数据库（隔离性），并且存储设备是完全可以依赖的（耐用性）。即使发生崩溃，竞态条件和磁盘故障确实会发生，但是事务抽象隐藏了这些问题所以应用程序不需要担心它们。

我们现在将继续沿着同样的路线前进，尝试寻找能够让应用程序忽略分布式系统部分问题的抽象概念。例如，分布式系统最重要的抽象之一就是共识：也就是说，让所有的节点都能达成一致。我们将在本章中看到，尽管存在网络故障和进程故障，可靠地达成一致是个令人惊讶的棘手问题。

一旦实现了协商一致，应用程序可以将其用于各种目的。例如，假设你有一个单主机复制的数据库。如果主机失效而你需要故障迁移到另一个节点，那么其余的数据库节点可以使用协商一致来选出新的主机。正如在“处理节点中断”中所讨论的，重要的是只有一个主机，并且所有节点都同意谁是主机。如果两个节点都认为自己是主机，那么这种情况就叫做裂脑，并且常常导致数据丢失。正确实现协商一致才可以避免此类问题。

在本章的后面，我们将在“分布式事务与协商一致”一节中研究解决协商一致以及相关问题的算法。但是首先，我们需要探索在分布式系统中可以提供的各种保证和抽象概念。

我们需要了解能做什么不能做什么的范围：在某些情况下，系统是可以容忍错误并继续工作的；在其他情况下，这就不可能了。在理论证明和实际实现中已经深入探讨了什么可能什么不可能的极限。我们将在这一章里概述这些基本限制。

分布式系统领域的研究人员几十年来一直在研究这些问题，因此有大量的材料——我们只会接触一些表面问题。在这本书中，我们没有空间去深入到正式模型与证明的细节，所以我们将坚持非正式的直觉。如果你感兴趣的话，这些参考文献提供了足够多的深度讨论。

## 一致性保证

在“复制延迟问题”一节中，我们研究了复制了的数据库中出现的一些计时问题。如果同时查看两个数据库节点，你很可能会在两个节点上看到不同的数据，因为写入请求在不同的时间到达不同的节点。无论数据库使用哪种复制方法（单主机、多主机或无主机复制），都会发生这些不一致的情况。

大多数复制了的数据库至少提供了最终一致性，这意味着如果停止对数据库的写入并等待一段时间，那么最终所有读取请求都会返回相同的值。换句话说，不一致是暂时的，最终是可以自己解决的（假设任何的网络故障也最终被修复了）。最终一致性的一个更好的名称可能是收敛，因为我们期望所有的副本最终收敛到相同的值。

然而，这是一个很弱的保证——它没有提到副本什么时候会收敛。在收敛之前，读取可以返回任何内容或什么都不返回。例如，如果写入一个值然后马上读取它，则无法保证你会看到刚才写入的值，因为读取请求可能被路由到另一个副本（见“读取你自己的写入”一节）。

对于应用程序开发人员来说最终一致性是很难，因为它与普通单线程程序中变量的行为非常不同。如果给变量赋值然后立即读取它，你不会期望读到旧值，或者读取失败。数据库表面上看起来像一个可以读写的变量，但实际上它有更复杂的语义。

当使用只提供弱保证的数据库时，你需要一直意识到它的局限性，而不是意外地假设太多。Bug通常很微妙，很难通过测试找到，因为大部分时间应用程序都能正常工作。最终一致性的边缘情况只有在系统中出现故障（例如网络中断）或高并发时才会变得明显。

在这一章里我们将探讨数据系统会选择提供的更强的一致性模型。它们并不是免费的：与有较弱保证的系统相比，具有更强保证的系统性能会更差，容错性会更低。尽管如此，因为更方便正确地使用，更有力的担保是有吸引力的。一旦你了解了几个不同的一致性模型，你就可以更好地决定哪一个最适合你的需要。

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

### 依赖线性化

在什么情况下线性化是有用的？查看一场体育比赛的最后比分也许是一个轻率的例子：在这种情况下，比赛结果晚了几秒更新不太可能造成任何真正的伤害。然而，在一些领域线性化是使系统正常工作的重要条件。

#### 锁定与主机选举

使用单主机复制的系统需要确保确实只有一个主机，而不是多个主机（裂脑）。选举主机的一种方法是使用锁：每一个节点启动的时候都试图获得锁，成功的节点将成为主机。不管这个锁是如何实现的，它必须是线性化的：所有节点都必须同意哪个节点拥有锁；不然没用。

像Apache ZooKeyer和etcd这样的协调服务通常用于实现分布式的锁与主机选举。他们使用协商一致算法以容错方式实现线性化操作（我们在本章后面的“容错协商一致”一节中讨论了这些算法）。正确实现锁和主机选举仍然有许多微妙的细节（见比如“主机与锁”中的围栏问题），而诸如Apache Curator这样的库，通过提供基于ZooKeeper的高级解决方案起到帮助作用。然而，线性化的存储服务是这些协调任务的基础。

分布式的锁定也用在一些分布式数据库中的更高粒度级别，例如Oracle Real Application Clusters （RAC）。由于多个节点共享对同一磁盘存储系统的访问，RAC在每个磁盘页上都用到锁。由于这些线性化的锁位于事务执行的关键路径上，因此RAC的部署通常有一个专用的集群互连网络，用于数据库节点之间的通信。

#### 约束与唯一性保证

唯一性约束在数据库中是很常见的：例如，用户名或电子邮件地址必须唯一地标识一个用户，而在文件存储服务中，不可能有两个路径和文件名相同的文件。如果要在写入数据时强制执行这个约束（如果两个人试图同时用同样的名字创建用户或者文件，其中一个将被返回错误），你需要线性化。

这种情况实际上与锁类似：当用户注册你的服务时，你可以想象成他们获得了他们选择的用户名上的“锁”。这个操作也非常类似原子性的比较后设置，如果用户名没有被占用，就把这个用户名设为申请该用户名用户的ID。

如果你想确保银行账户余额不会为负数，或者你不会卖出比仓库库存更多的物品，或者两个人不会同时在飞机上或剧院里预订相同的座位，就会出现类似的问题。这些约束都要求有一个所有节点都同意的最新值（帐户余额、库存水平、座位是否占用）。

在实际应用中，宽泛地对待这些约束有时是可以接受的（例如，如果航班超订，你可以把客人转移至不同的航班，并为带来的不便之处向他们进行赔偿）。在这种情况下，线性化也许不需要，我们会在“及时性和完整性”一节讨论这种的被宽泛解读的约束。

但是，硬唯一性约束，例如通常在关系型数据库中找到的那种，需要线性化。其他类型的约束，例如外键或属性约束，不需要线性化的情况下就可以实现。

#### 跨通道时序依赖

注意图9-1中的一个细节：如果爱丽丝没有报出分数，鲍勃就不会知道他的查询结果已经过时了。几秒钟后，他就会再次刷新页面，并最终看到了最后的比分。只有在系统中有另一条通信通道（爱丽丝的声音对到鲍勃的耳朵），才会注意到这种线性化违规。

计算机系统中也可能出现类似的情况。例如，假设你有一个网站，用户可以上传照片，背景进程调整照片大小到较低分辨率以便更快地下载（缩略图）。该系统的架构和数据流如图9-5所示.

需要显式地指示图像编辑器执行调整大小的任务，并且通过消息队列将此指令从Web服务器发送到编辑器（见第11章）。Web服务器不会将整个照片放在队列中，因为大多数消息代理都是专为小消息设计的，而一张照片的大小可能是几兆字节。取而代之的是，照片首先被写到文件存储服务中，一旦写入完成，给编辑器的指令就会放在队列上。

*图9-5 Web服务器和图像编辑器通过文件存储和消息队列进行通信，从而打开了竞争条件的潜在可能性。*

如果文件存储服务是线性化的，那么这个系统应该可以正常工作。如果消息队列不是线性化的，就有存在竞争条件的风险：消息队列（图9-5中的步骤3和步骤4）可能比存储服务中的内部复制的速度更快。在这种情况下，当编辑器获取图像（步骤5）时，它可能会看到图像的旧版本，或者什么也看不到。如果它处理图像的旧版本，则文件存储中的全尺寸的图片和大小调整后的图片就不一致了。

这个问题之所以出现，是因为Web服务器和编辑器之间有两个不同的通信通道：文件存储和消息队列。没有新近性线性化的保证，两个信道之间的竞争条件是可能的。这种情况类似于图9-1，图中两个通信通道之间也存在竞争条件：数据库复制，以及真实世界里爱丽丝嘴巴到鲍勃耳朵之间的音频通道。

线性化并不是避免这种竞赛条件的唯一方法，却是最容易理解的方式。如果你控制着多余的那条通信通道(比如消息队列的情况，但不是爱丽丝和鲍勃的情况)，你可以使用与我们在“读取自己的写操作”中讨论的类似替代方法，代价是额外的复杂性。

### 实现线性化系统

现在既然我们已经看了几个线性化有用的例子，那么让我们考虑一下如何实现提供线性化语义的系统吧。

由于线性化本质上意味着“表现地就像只有一份数据拷贝，而且对它的所有操作都是原子性的”，那么最简单的答案就是真的只用一份数据拷贝。但是，这种方法将没有办法容错：如果保存该副本的节点失效，数据将丢失，或者至少在节点再次上线之前是无法访问的。

使系统可以容错的最常见方法是使用复制。让我们重新回顾第5章中提到的复制方法，然后比较它们是否可以线性化：

*单主机复制（有可能可以线性化）*

在有着单主机复制的系统中（见“主机与从机”一节），主机拥有用于写入的数据的主副本，而从机在其他节点上维护数据的备份副本。如果你从主机或同步更新的从机中读取数据，那么它们有可能是线性化的。然而，并不是每个单主机数据库实际上都是线性化的，要么是因为设计（比如因为它使用快照隔离），要么是由于并发bug。

使用主机完成读取请求依赖于这样一个假设——你确实知道哪个是主机。正如在“真理由多数人定义”一节中所讨论的那样，节点很有可能认为它是主机，而实际上并非如此——如果妄想的主机继续响应请求，就很可能违反了线性化。使用异步复制，故障转移甚至可能会丢失已经提交的写入操作（见“处理节点离线”一节），这既违反了持久性也违反了线性化。

*协商一致算法（线性化的）*

一些我们将在本章稍后讨论到的协商一致算法，与单主机复制相似。然而，协商一致协议包含了防止裂脑和陈旧复本的措施。正是由于这些细节，协商一致算法可以安全地实现线性化存储。这就是例如ZooKeeper以及etcd的工作方式。

*单主机复制（非线性化的）*

具有多主机复制的系统通常是非线性化的，因为它们同时处理多个节点的写入请求，并异步地将它们复制到其他节点。由于这个原因，它们会产生需要解决的冲突写入请求（见“处理写入冲突”一节）。这种冲突是缺少数据单个拷贝的产物。

*无主机复制（大概不可以线性化）*

对于无主机复制的系统（Dynamo风格；见“无主机复制”一节），人们有时会说通过要求仲裁读写（*w + r > n*）你可以获得“强一致性”。取决于仲裁的确切配置，以及如何定义强一致性，这并不完全正确。

基于现世时钟的“以最后一次写入为准”的冲突解决方法（例如在Cassandra中；见“依赖同步时钟”一节）几乎肯定是非线性化的，因为时钟时间戳不能保证与由于时钟偏斜导致的实际事件顺序一致。草率的仲裁(“草率的定额值与提示交换”一节)也破坏了任何实现线性化的机会。即使有严格的仲裁，还是有可能出现非线性化行为，如下一节所示。

#### 线性化和仲裁

从直觉上看，似乎严格的仲裁读写应该在Dynamo风格的模型中是线性化的。然而，当我们有可变的网络延迟时，就有可能有竞争条件，如图9-6所示。

*图9-6 非线性化的执行，尽管使用了严格的仲裁。*

在图9-6中，*x*的初始值是0，并且写入器客户端通过将写入请求发送到所有三个副本（*n*＝3，*w*＝3）把*x*更新到1。同时，客户端A从两个节点的仲裁团中读取（*r* = 2），并且在其中一个节点上看到新值1。同时在写入过程中，客户端B从两个节点构成的另一个仲裁团中读取，并且从这两个节点取回旧值0。

仲裁条件（*w + r > n*）是满足了，但这种执行仍然不是线性化的：B的请求是在A的请求完成后开始的，但是B返回旧值的同时A返回了新值。（这又是图9-1中的爱丽丝和鲍勃的情况。）

有趣的是，可以在牺牲性能的情况下使Dynamo风格的仲裁线性化：在将结果返回到应用程序之前，读取器必须同步执行读修复（见“读取修复与反熵”一节），而编写器必须在发送其写入请求之前读取仲裁团节点的最新状态。然而由于性能损失，Riak不执行同步的读取修复。Cassandra确实在仲裁读取时等待读取修复完成，但是如果对相同的键有多个并发写入，就会失去线性化，因为它使用了以最后一次写入为准的冲突解决方案。

此外，只有线性化的读写操作才能以这种方式实现；线性化的比较后设置操作不能实现，因为它需要协商一致的算法。

总之，最安全的假设是具有Dynamo风格复制的无主机系统不提供线性化。

#### 线性化的代价

正是由于一些复制方法可以提供线性化而另一些不能，因此更深入地探讨线性化的利弊是很有意思的。

我们已经在第5章中讨论了不同复制方法的一些用例；例如，我们看到多主机复制通常是多数据中心复制的一个很好的选择(见“多数据中心操作”一节)。图9-7展示了这样一个部署的例子。

*图9-7 网络中断强制在线性化与可用性之间选择。*

考虑一下如果两个数据中心之间网络中断了会发生什么情况。让我们假设每个数据中心内的网络都在工作，客户端可以访问数据中心，但是数据中心之间不能相互连接。

使用多主机数据库，每个数据中心可以继续正常工作：由于来自一个数据中心的写入请求被异步地复制到另一个数据中心，因此在网络连接恢复的时侯，只需排起队来等待交换写入操作。

另一方面，如果使用单主机复制，那么主机必然位于其中一个数据中心。任何写入和任何可线性读取都必须发送给主机——因此，对于任何连接到从机数据中心的客户端，这些读写请求必须同步地通过网络发送到主机数据中心。

如果数据中心之间的网络在单主机设置下中断了，连接到从机数据中心的客户端无法与主机取得联系，因此他们不能对数据库发起任何写入请求，也不能发起任何线性化读取请求。他们仍然可以向从机发起读取请求，但数据可能是陈旧的（非线性化的）。如果应用程序需要线性化的读写，那么网络中断将导致应用程序在无法与主机取得联系的数据中心中不可用。

如果客户端可以直接连接到主机的数据中心，这并不是一个问题，因为应用程序继续在那里正常工作。但是，只能访问从机数据中心的客户端将经历断线，直到（数据中心之间的）网络链接被修复为止。

#### CAP定理

这个问题不仅仅是单主机和多主机复制的结果：任何线性化数据库都有这个问题，不管它是如何实现的。这个问题也不限于多数据中心部署，它可能发生在任何不可靠的网络上，甚至在一个数据中心内。权衡如下：

* 如果你的应用程序需要线性化，并且一些副本由于网络问题而与其他副本断开了连接，于是有些副本在断开连接时无法处理请求：它们必须要么等待网络问题的解决，要么返回错误（无论哪种方式，它们都变得不可用）。

* 如果你的应用程序不需要线性化，那么可以编写为每个副本都可以独立处理请求，即使它与其他副本（例如多主机）断开连接。在这种情况下，即使面对网络问题应用程序仍然可用，只是它的行为不是线性化的。

因此，不需要线性化的应用程序可以更好地容忍网络问题。这种洞察力被通称为CAP定理，由埃里克·布鲁尔于2000年命名，尽管自20世纪70年代以来分布式数据库的设计者就知道了这种取舍。

CAP最初是作为经验法则提出的，没有准确的定义，目的是为了开始讨论数据库中的取舍问题。当时，许多分布式数据库专注于在具有共享存储的设备集群上提供可线性化的语义[18]，CAP则鼓励数据库工程师探索更广泛的分布式无共享系统的设计空间，这些系统更适合用于实现大规模Web服务。CAP值得称赞的地方是这种文化的转变——见证了自2000年代中期以来新数据库技术的爆发（也就是NoSQL）。

> **没有用处的CAP定理**
>
> CAP有时会表示为*一致性、可用性、分区耐受性：从3项中选择2项*。然而，这样说会误导人，因为网络分区是一种故障，所以它们不是你有的选的东西：不管喜欢与否它们都会发生。
>
> 当网络正常工作时，系统可以提供一致性（线性化）和完全可用性。当网络发生故障时，你必须在线性化或完全可用性之间进行选择。因此，描述CAP更好的方法是*网络分区时要么有一致性要么有可用性*。更可靠的网络可以降低选择的频率，但在某些时候这种选择是不可避免的。
>
> 在CAP的讨论中，对于术语*可用性*有几个相互矛盾的定义，而形式化为定理[30]并不符合其通常的含义。许多所谓的“高度可用”（容错）的系统实际上不符合CAP对可用性的独具一格的定义。总之，围绕CAP存在许多误解和困惑，它不能帮助我们更好地理解系统，所以最好避免CAP。

正式定义的CAP定理范围很窄：它只考虑一个一致性模型（即线性化）和一种故障（*网络分区*，或者在线但是与其它节点断开的节点）。它没有提到任何关于网络延迟、离线节点或其他取舍的内容。因此，虽然CAP在历史上具有影响力，但它对系统的设计没有什么实际价值。

在分布式系统中有许多更有趣的不可能的结果，而CAP现在已经被更精确的结果所取代，因此它在今天更具有历史意义一些。

#### 线性化与网络延迟

虽然线性化是个有用的保证，但在实践中很少有系统实际上是线性化的。例如，即使是现代多核CPU上的RAM也不是线性化的：如果运行在一个CPU核心上的线程在某个内存地址写入数据，而另一个CPU核心上的线程随后读取相同的地址，是不能保证可以读取第一个线程所写的值的（除非使用内存屏障或栅栏）。

造成这种行为的原因是每个CPU内核都有自己的内存缓存和存储缓冲区。默认情况下，内存访问首先访问缓存，任何更改都异步写入主内存。由于访问高速缓存中的数据要比访问主内存快得多，这个特性对于在现代CPU上获得良好性能是必不可少的。然而，这样数据就有了几个副本(一个在主内存中，可能还有几个在各种缓存内)，这些副本都是异步更新的，因此失去了线性化。

为什么要做这个取舍？用CAP定理来证明多核内存一致性模型是没有意义的：在一台计算机中，我们通常假定通信是可靠的，并且我们不期望一个CPU核心在与计算机的其他部分断开连接的情况下能够继续正常工作。放弃线性化的原因是性能，而不是容错。

许多分布式数据库选择不提供线性化保证也是如此：它们这样做主要是为了提高性能，而不是为了容错。线性化很慢——而且一直都是这样的，而不仅仅是在网络故障的时候。

难道我们找不到一种更有效的实现线性化存储的方法吗？答案似乎是否定的：阿提亚和韦尔奇证明了如果想要线性化，读写请求的响应时间至少与网络中延迟的不确定性成正比。在具有高度可变延迟的网络中，就像大多数计算机网络（参见“超时和无界限延迟”一节）一样，线性化读写的响应时间不可避免地会很高。不存在更快的线性化算法，但是较弱的一致性模型可能会更快，因此这种取舍对于对延迟敏感的系统是很重要的。在第12章中我们将讨论在不牺牲正确性的情况下避免线性化的一些方法。

## 排序保证

我们之前说过，可线性化的寄存器的行为就好像数据只有一个副本，并且每个操作似乎在某个时间点原子性地生效。这个定义意味着操作是以某种定义好的顺序执行的。我们在图9-4中展示了这种排序，方法是把这些操作以它们执行的次序连接起来。

在这本书中排序一直是反复出现的主题，这表明它是一个重要的基本概念。让我们简要回顾一下关于排序我们讨论过的其它一些情况：

* 在第5章中，我们看到主机在单主机复制中的主要目的是确定复制日志中写入的顺序——也就是从机应用这些写入的顺序。如果没有单主机，由于并发操作就会引发冲突（见“处理写入冲突”一节）。

* 串行化，我们在第7章中讨论过，是为了确保事务的行为就好像它们是按照某种顺序执行的一样。它的实现既可以通过真的串行顺序执行事务，也可以在（通过锁定或中止）防止串行化冲突的前提下允许并发执行。

* 我们在第8章中讨论的在分布式系统中使用时间戳和时钟（见“依赖同步时钟”一节），是把顺序引入无序世界的又一次尝试，例如，确定两个写入请求哪一个是稍后发生的。

结果表明，排序、线性化与协商一致之间有着深刻的联系。虽然这个概念比这本书的其他部分更偏理论更抽象一些，但它对于澄清我们对系统能做什么不能做什么的理解是很有帮助的。我们将在接下来的几个小节中探讨这个主题。

### 排序与因果关系

排序持续出现的原因有很多，其中一个是它帮助维护了因果关系。在这本书中，我们已经看到了几个因果关系很重要的例子：

* 在“一致前缀读取”一节（图5-5）中，我们看到了一个例子，会话的观察者首先看到了问题的答案，然后才看到了被回答的问题。这是令人困惑的，因为它违背了我们对原因和结果的直觉：如果问题得到了回答，那么这个问题显然必须首先出现，因为给出答案的人一定看到了这个问题（假设他们不是灵媒，是看不到未来的）。我们说问题与答案之间是存在因果关系的。

* 图5-9中也出现了类似的模式，我们研究了三台主机之间的复制，并注意到一些写入请求可能因为网络延迟的缘故而“超越”其他写入请求。从一个副本的角度来看，好像有一个不存在的行更新。这里的因果关系意味着行必须先创建才能更新。

* 在“检测并发写入”一节中，我们观察到如果有两个操作A和B，那么有三种可能：A发生在B之前，B发生在A之前，以及A和B是并发的。这种*发生在...之前*关系是因果关系的另一种表达：如果A发生在B之前，这意味着B可能知道A，或建立在A之上，或依赖于A，如果A和B同时存在，它们之间就没有因果关系；换句话说，我们确信两者都不知道对方。

* 在事务的快照隔离（“快照隔离和可重复读”一节）的上下文中，我们说，事务从一致快照进行读取。但是，在这种情况下“一致”意味着什么呢？它的意思是*与因果关系一致*：如果快照包含一个答案，它还必须包含被回答的问题。在单个时间点观察整个数据库使其与因果关系一致：在该时间点之前发生的所有操作的结果都是可见的，但在此之后发生的任何操作都是不可见的。读偏（不可重复读，如图7-6所示）意味着在违反因果关系的状态下读取数据。

* 我们举的事务之间写偏的例子（见“写偏和幻影”一节）也显示了因果相关性：在图7-8中，爱丽丝被允许离岗是因为事务认为鲍勃仍然在值，反之亦然。在这种例子中，离岗的动作依赖于谁正在当值的观察结果。可序列化快照隔离（见“可序列化快照隔离（SSI）”）通过跟踪事务之间的因果依赖来检测写偏。

* 在爱丽丝和鲍勃看足球的例子中（图9-1），鲍勃在听到爱丽丝惊叹结果后从服务器那里得到了旧结果，这违反了因果关系：Alice的感叹在因果关系上取决于比分的宣布，所以鲍勃也应该能够在听到爱丽丝的声音之后看到这个分数。同样的模式再次出现在“跨通道计时依赖”一节，这一次是图片大小调整服务。

因果关系在事件之间强加了顺序：原因先于效果；消息在被收到之前已经被发送了；问题出现在答案之前。就像在现实生活中一样，一件事导致另一件事：一个节点读取了一些数据然后写入了一些东西作为结果，另一个节点读取了这个被写入的东西然后再依次写入了其他的东西，以此类推。这些因果相关的操作链定义了系统中的因果顺序——也就是什么发生在什么之前。

如果一个系统服从因果关系所施加的顺序，我们就说它在*因果一致*的。例如，快照隔离提供了因果一致性：当你从数据库中读取数据时，当你看到一些数据时，你还必须能够看到因果关系之前的任何数据（假设它在此期间没有被删除）。

#### 因果顺序不是全序

全序允许对任意两个元素进行比较，因此如果有两个元素，则始终可以说哪个更大哪个更小。例如，自然数是全序的：如果我给你任意两个数字，比如说5和13，你可以告诉我13大于5。

然而，数学集合是非全序的：{*a, b*}大于{*b, c*}吗？你根本不能比较它们，因为它们彼此都不互为子集。我们说它们是不可比较的，因此数学集合是偏序的：在某些情况下，一个集合大于另一个集合（如果一个集合包含另一个的所有元素），但在其他情况下它们是不可比较的。

全序与偏序之间的差异反映在不同的数据库一致性模型中：

*可线性化*

在一个可线性化的系统中，我们有一个操作全序：如果系统的行为好象数据只有一个拷贝，并且每一个操作都是原子性的，这就意味着对于任何两个操作，我们总是可以说哪一个是先发生的。该总序被展示为图9-4中的时间线。

*因果关系*

我们说过如果两个操作彼此都不是在另一个操作之前发生的，那么两个操作是并发的（见“在...之前发生关系与并发”一节）。换句话说，如果两个事件是因果相关的（一个发生在另一个事件之前）那么它们是有序的，但如果它们是并发的那么它们是不可比较的。这意味着因果关系定义的是偏序，而不是全序：有些操作是根据彼此的顺序排列的，但有些操作是不可比较的。

因此根据这个定义，在可线性化的数据存储中是不存在并发操作的：必然有一个时间线，所有的操作都是全序地沿着它排列。可能有几个请求在等待处理，但是数据存储保证每个请求没有任何并发地，在单个时间线的单个时间点上对数据的单个副本进行操作。

并发意味着时间线先分支，而后再合并——在这种情况下，不同分支上的操作是不可比较的（即并发的）。我们在第5章中看到了这种现象：例如，图5-14不是一条直线的全序，而是多个不同操作同时进行的大杂烩。图中的箭头表示因果关系——操作的偏序。

如果你熟悉分布式版本控制系统，比如Git，它们的版本历史非常类似因果依赖关系图。通常一次提交在另一次提交之后，在一条直线上进行的，但有时会有分支（当几个人同时在一个项目上工作时），而合并分支是在并发创建的提交在合并时创建的。

#### 线性化强于因果一致性

那么因果顺序和线性化之间有什么关系呢？答案是线性化意味着因果关系：任何可线性化的系统都将正确地保留因果关系。特别是，如果系统中有多个通信通道（例如图9-5中的消息队列和文件存储服务），线性化可以确保因果关系自动保持，而无需系统做任何特殊的处理（例如在不同组件之间传递时间戳）。

线性化确保因果关系使线性化系统易于理解并且有吸引力。然而，正如在“线性化的成本”一节中所讨论的，使系统线性化可能会损害其性能和可用性，特别是如果系统有显著的网络延迟(例如，它分布在不同地地理区域)。由于这个原因，一些分布式数据系统已经放弃了线性化，这使得它们能够获得更好的性能但也更难以使用。

好消息是折中方案是可行的。线性化不是保持因果关系的唯一途径——还有其他方法。系统可以是因果一致的，而无需引入线性化带来的性能影响（特别是CAP定理不适用了）。事实上，因果一致性是最强大的一致性模型，它不会因网络延迟而减慢，并且在网络故障面前仍然可用。

在许多情况下，看起来需要线性化的系统实际上只是要求因果一致性，而因果一致性可以更有效地实现。基于这一观察，研究人员正在探索保存因果关系的新型数据库，其性能和可用性特征与最终一致的系统类似。

由于这项研究是很近的，它还没有进入生产性系统中，而且还有一些挑战有待克服。然而，它是未来系统的一个有前途的方向。

#### 抓取因果关系

我们不会在这里讨论非线性系统可以如何保持因果一致性的所有细节，而只是简单地探讨一些关键的想法。

为了保持因果关系，你需要知道哪个操作发生在另外哪个操作之前。这是一个偏序：并发操作可以按任何顺序处理，但如果一个操作发生在另一个操作之前，则必须在每个副本上按照那个顺序处理它们。因此，当副本处理一个操作时，它必须确保所有因果前操作（之前发生的所有操作）都已被处理；如果缺少了某个前序操作，则后面的操作必须等待，直至前序操作被处理。

为了确定因果关系，我们需要某种方法来描述系统中节点上的“知识”。如果一个节点在发出了写入Y时已经看到了值X，那么X和Y可能是因果相关的。这一分析使用了在欺诈指控的刑事调查中你会想到的问题：在他们做出决定Y时，首席执行官*知道*X的情况吗？

用于确定哪个操作发生在另外哪个操作之前的技术与我们在“检测并发写入”一节中讨论的类似。那一节讨论了无主机数据存储中的因果关系，在那种情况下我们需要检测对同一个键的并发写入，以防止丢失更新。因果一致性更进一步：它需要跟踪整个数据库中的因果依赖，而不仅仅是单个键。版本向量可以推广到这里来完成这件事。

为了确定因果顺序，数据库需要知道应用程序读取了哪个版本的数据。这就是为什么在图5-13中，之前操作的版本号在写入时被传回数据库的原因。在SSI的冲突检测中也出现了类似的想法，正如在“可串行化快照隔离（SSI）”一节中所讨论的那样：当一个事务想要提交时，数据库检查它读取的数据版本是否仍然是最新的。为此，数据库会跟踪记录哪个数据被哪个事务读取了。

### 序号排序

虽然因果关系是一个重要的理论概念，但实际上，跟踪记录所有的因果关系是不切实际的。在许多应用程序中，客户端在写入某些内容之前会读取大量数据，然而并不清楚写入请求是否因果依赖于先前的所有还是部分读取请求。明确地跟踪记录已读取的所有数据意味着很大的开销。

然而，有更好的方法：我们可以使用序列号或时间戳对事件排序。时间戳不需要来自现世时钟（或物理时钟，它有许多在“不可靠的时钟”一节讨论到的问题）。取而代之的是逻辑时钟，它是一种生成数字序列从而标识操作的算法，通常使用计数器，每当发生一个操作就增加一。

这样的序列号或时间戳很紧凑（只有几个字节的大小），并且它们提供了全序：即，每个操作都有唯一的序列号，并且总是可以比较两个序列号来确定哪个更大（即，哪个操作发生得更晚）。

特别是，我们可以按照与因果关系一致的全序构建序列号：我们承诺，如果操作A发生在B之前，则A在全序中出现在B之前（A的序列号低于B）。并发操作可以任意地排序。这样的全序包含了所有的因果关系信息，但也强加了比因果关系更严格要求的次序。

在具有单主机复制（见“主机与从机”一节）的数据库中，复制日志定义了与因果性一致的写入操作全序。主机只用为每个操作递增计数器，从而为复制日志中的每个操作分配单调递增的序列号。如果从机按照在复制日志中出现的顺序应用写入，那么从机的状态总是因果一致的（即使状态落后于主机）。

#### 非因果序列号生成器

如果不是单主机(也许是因为你在使用的是多主机或无主机的数据库，或者因为数据库是分了区的)，那么如何为操作生成序列号就不那么清楚了。那么在实践中，用到了各种各样的方法：

* 每个节点可以生成自己独立的序列号集合。比如说，如果有两个节点，一个节点只能生成奇数，而另一个节点只能生成偶数。通常，你可以在序列号的二进制表示中保留一些位，以包含唯一的节点标识符，这样可以确保两个不同的节点永远不会生成相同的序列号。

* 你可以为每一个操作附加来自现世时钟（物理时钟）的时间戳。这样的时间戳并不是连续的，但是如果它们具有足够高的精度，它们可能就足以完全全序操作。在以最后一次写入为准的冲突解决方法中用到了这个事实（见“为事件排序的时间戳”一节）。

* 你可以预先分配序列号块。例如，节点A可能要求从1到1，000之间的序列号块，而节点B可能要求从1，001到2，000之间的块。然后每个节点可以独立地从各自的块分配序列号，并在可用序号减少时分配到一个新块。

相比于把所有的操作都推送到单主机从而使计数器加一，这三个选项执行效果更好，也更具有伸缩性。它们为每个操作生成一个唯一的、近似增加的序列号。然而，它们都有一个问题：它们产生的序列号与因果关系不一致。

之所以会出现因果关系问题，是因为这些序列号生成器无法正确捕获不同节点之间操作的顺序：

* 每个节点每一秒可以处理不同数量的操作。因此，如果一个节点产生偶数而另一个节点产生奇数，偶数计数器可能落后于奇数的计数器，抑或反之。如果你有一个奇数的运算和偶数的运算，你并不能准确地分辨出哪一个是先发生的。

* 来自物理时钟的时间戳受时钟偏差的影响，这可能使它们与因果关系不一致。例如图8-3，其中显示了一个场景，在该场景中因果关系中稍后发生的操作实际上被分配了一个较低的时间戳。

* 在块分配器的情况下，可以给一个操作在1001到2000的范围内的序列号，而随后的操作可以给出范围为1到1000的数字。在这里同样的，序列号与因果关系不一致。

#### 兰波特时间戳

虽然刚才描述的三个序列号生成器与因果关系不一致，但是实际上*有*一种与因果关系相一致的简单的序列号生成方法。它被称为*兰波特时间戳*，由莱斯利·兰波特于1978年提出，现在是分布式系统领域中被引用最多的论文之一。

兰波特时间戳的使用如图9-8所示。每个节点都有一个唯一的标识符，每个节点都有一个记录了它处理了的操作数的计数器。兰波特时间戳只是一对（*计数器，节点ID*）。两个节点有时可能有相同的计数器值，但是通过在时间戳中包含节点ID，每个时间戳都是唯一的。

*图9-8 兰波特时间戳提供了与因果关系一致的全序。*

兰波特时间戳与物理现世时钟没有关系，但它提供了全序：如果你有两个时间戳，那么计数器值越大，时间戳越大；如果计数器值相同，那么节点ID越大，时间戳就越大。

到目前为止，这种描述基本上与上一节中描述的偶数/奇数计数器相同。兰波特时间戳的关键思想是：每个节点和每个客户端都跟踪记录它到目前为止看到的最大计数器值，并在每个请求中包含这个最大值。当节点接收到最大计数器值大于其自身计数器值的请求或响应时，它立即将自己的计数器增大到最大。

如图9-8所示，客户端A从节点2接收计数器值5，然后将最大值5发送给节点1。那时节点1的计数器只有1，但它立即向前移动到5，因此下一次操作的计数器值递增为6。

只要在每次操作中都携带最大计数器值，这个方案就能确保来自兰波特时间戳的排序与因果关系一致，因为每个因果依赖关系都会导致时间戳的变大。

兰波特时间戳有时会与我们在“检测并发写入”一节中看到的版本向量相混淆。虽然它们有一些相似之处，但它们有一个不同的目的：版本向量可以区分两个操作是并发的还是一个是因果依赖的，而兰波特时间戳总是强制执行总排序。从兰波特时间戳的总排序中，你无法判断两个操作是并发的还是因果相关的。兰波特时间戳相对于版本向量的优点是它们更紧凑。

#### 时间戳排序是不够的

尽管兰波特时间戳定义了与因果关系一致的操作的全序，但是它们并不完全足以解决分布式系统中的许多常见问题。

例如，考虑需要确保用户名唯一标识用户帐户的系统。如果两个用户同时尝试创建具有相同用户名的帐户，那么两个用户中的一个应该成功另一个应该失败。（我们之前在“主机与锁”中提到过这个问题。）

乍一看，似乎全序操作(例如，使用兰波特时间戳)就足以解决这个问题：如果创建了两个具有相同用户名的帐户，则选择具有较低时间戳的帐户作为赢家(先获取用户名的帐户)，然后让具有较大时间戳的帐户失败。由于时间戳是全序的，这种比较总是有效的。

这种方法适用于事后确定赢家：一旦收集了系统中所有的用户名创建操作，你就可以比较它们的时间戳了。但是，当节点刚刚收到用户创建用户名的请求，并且需要立即决定请求是成功还是失败时，这是不够的。此时，这个节点不知道另一个节点是否同时正在创建一个具有相同用户名的帐户，也不知道另外那个节点可能为该操作分配的时间戳是什么。

为了确保没有其他节点在并发创建具有相同用户名和较低时间戳的帐户的过程中，你必须检查其他节点，看看它们正在做什么。如果其他节点中的一个由于网络问题而失败或无法到达，该系统将陷入瘫痪。这不是我们需要的那种容错系统。

问题是只有在收集了所有操作之后，操作的全序才会出现。如果另一个节点生成了一些操作，但你还不知道它们是什么，那么就无法构造操作的最终顺序：这些来自另一个节点的未知操作可能需要按全序插入在不同位置。

总结一下：为了实现关于用户名的唯一性约束，只有操作的全序是不够的——你还需要知道排序何时完成。如果你有一个创建用户名的操作，并且你确信没有其他节点能够在操作之前按全序插入相同用户名的声明，那么你可以安全地声明操作成功。

知道全序何时完成的这个理念正是全序广播的主题。

### 全序广播

如果你的程序只运行在在单个CPU核心上，那么定义操作的全序是很简单的：这就是CPU执行它们的顺序。然而，在分布式系统中，让所有节点都同意相同的操作顺序是很困难的。在最后一节中我们讨论了按时间戳或序列号排序，但发现它不如单主机复制（如果使用时间戳排序来实现唯一性约束，则不能容忍任何错误）强大。

我们之前讨论过，单主机复制通过选择一个节点作为主机并把所有的操作排列在主机的单个CPU核心上来决定操作的全序的。接下来的挑战是，如果吞吐量超过单个主机可以处理的量应该如何缩放系统，以及如果主机失败如何处理故障转移（见“处理节点中断”一节）。在分布式系统的文献中，这个问题被称为全序广播或原子广播。

> **排序保证的范围**
>
> 每个分区只有单主机的分区数据库通常只维护每个分区的顺序，这意味着它们不能提供跨分区的一致性保证(例如，一致快照、外键引用)。实现跨所有分区的全序是可能的，但需要额外的协调工作。

全序广播通常被描述为用于节点之间交换消息的协议。它非正式地要求两个安全属性必须总是满足：

可靠地送达

消息不会丢失：如果一个消息被传递到一个节点，那么说明它被传递到了所有节点。

全序地送达

消息以相同的顺序传递给每个节点。

正确的全序广播算法必须确保可靠性和排序属性总是满足的，即使节点或网络有故障。当然，网络中断的时候消息无法传递，但是算法可以继续重试，这样当网络最终被修复时，消息就会传递过去（并且它们仍然必须以正确的顺序送达）。

#### 使用全序广播

协商一致服务，比如ZooKeeper和etcd，实际上实现了全序广播。这个事实暗示了全序广播与协商一致之间有着密切的联系，我们将在本章稍后部分加以探讨。

全序广播正是数据库复制所需的：如果每个消息代表了对数据库的一次写入，并且每个副本以相同的顺序处理相同的写入，那么副本之间将保持一致（除了任何暂时的复制滞后）。这个原理被称为状态机复制，我们将在第11章中再讨论它。

同样地，全序广播可以用来实现串行化事务：正如在“实际串行执行”一节中所讨论的，如果每个消息代表了一个要作为存储过程执行的确定性事务，而且如果每个节点以相同的顺序处理这些消息，那么数据库的分区和副本会彼此保持一致。

全序广播的一个重要方面是顺序在消息发送时是固定的：如果后续地消息已经送出，则不允许节点回溯性地将消息插入到顺序中的先前位置。这使得全序广播比时间戳排序更强。

看待全序广播的另一种方式，那就是，这是一种创建日志（如复制日志、事务日志或预写入日志）的方式：传递消息就像写入日志一样。因为所有节点都必须以相同的顺序传递相同的消息，所以所有的节点都可以读取日志，看到相同的消息序列。

全序广播对于实现提供栅栏令牌的锁服务也很有用（见“栅栏令牌”一节）。获取锁的每一个请求都会作为消息附加到日志中，并且所有消息都会按照它们在日志中出现的顺序编号。之后，序列号可以用作栅栏令牌，因为它是单调递增的。在ZooKeeper中，这个序列号称为`zxid`。

#### 利用全序广播实现线性化存储

如图9-4所示，在一个线性化系统中有一个关于操作的全序。这是不是意味着线性化与全序广播是同一回事？不完全是，但是在两者之间有着密切的联系。

全序广播是异步的：消息保证是以固定的顺序可靠地传递的，但是没有办法保证什么时侯传递消息（因此，一个接收者可能落后于其他接收者）。相比之下，线性化是种新近性保证：读取操作保证可以看到最新写入的值。

然而如果有了全序广播，你可以在它的基础上构建线性化存储。例如，你可以确保用户名是唯一地标识用户帐户的。

想象一下，对于每一个可能的用户名，你可以通过原子性的比较后设置操作拥有一个线性化寄存器。每个寄存器最初的值为`null`（指明用户名还没有取）。当用户想要创建用户名时，在寄存器上为该用户名执行比较后设置操作，将其设置为用户帐户ID，条件是前一个寄存器值为`null`。如果多个用户尝试并发地获取同一个用户名，那么只可能有一个比较后设置操作会成功，因为其他用户（由于线性化）将看到一个不是`null`的值。

你可以通过把全序广播作为只添加日志来实现这样一个线性化的比较后设置操作，如下所示：

1. 在日志中添加一条消息，试探性地指出要求的用户名。

2. 读取日志，然后等待被添加的消息发送回来。

3. 检查任何要求你要的用户名的消息。如果对应你想要的用户名的第一条消息是你自己的消息，那么就成功了：你可以提交用户名声明（也许是通过在日志中附加另一条消息）并且向客户端确认。如果第一条消息来自另一个用户，那么操作中止。

因为日志条目以相同的顺序传递给所有节点的，如果有几个并发写入请求，那么所有节点都会同意哪条日志是第一条。选择冲突写入中的第一个作为胜利者然后中止后面的写入，可以确保所有节点就写入是否已提交或中止达成一致。类似的方法可以用于在日志基础之上实现串行化的多对象事务。

虽然这个过程确保了线性化写入，但是它并不保证线性化读取——如果从一个异步更新于日志的存储中读取，那么有可能会过期。（准确地说，此处所述的过程提供了顺序一致性，有时也称为时间线一致性，相比于线性化这个保证略弱一些。)要使读取线性化，有几个选项：

* 你可以通过添加消息，读取日志为读取排序，当消息传递回来时执行真正的读取动作。因此，消息在日志中的位置定义了读取发生的时间点。（etcd中的仲裁团读取与这个有些类似）。

* 如果日志允许以线性化的方式获取最新日志消息的位置，你可以查询这个位置，等待所有在此之前的条目送达，然后执行读取。（这就是ZooKeeper的`sync()`操作背后的理念。）

* 你可以向写入同步更新的副本发起读取，因此肯定是最新的。（该技术用于链复制；见“关于复制的研究”一节）。

#### 利用线性化存储实现全序广播

上一节展示了如何使用全序广播建立一个线性化的比较后设置操。我们也可以反过来，假设已经有了线性化存储，展示如何用它构建全序广播。

最简单的方法是假设你有一个线性化的寄存器，它可以存储一个整数，并且有原子性的加一后获取的操作。亦或是原子性的比较后设置操作。

算法很简单：对于要通过全序广播发送的每一条消息，你都会加一后获取那个线性化的整数，然后将从寄存器获得的值作为序列号附加到消息中。然后你可以将消息发送到所有节点（重发任何丢失了的消息），接收方将按照序列号连续地传递消息。

值得注意的是与兰波特时间戳不同，通过递增线性化寄存器获得的数字构成了一个没有间隙的序列。因此，如果节点已经传递了消息4并接收到序列号为6的消息，它知道必须等到消息5之后才能传递消息6。用兰波特时间戳的情况就不一样了——事实上，这就是全序广播和时间戳排序之间的关键区别。

让线性化整数有原子性的加一后获取运算是有多难呢？和往常一样，如果从来没有失败过，那就很容易了：你可以将它保存在一个节点上的一个变量中。问题在于处理到该节点的网络连接中断时的情况，以及节点失效时如何恢复值。一般来说，如果对线性化的序列号生成器考虑得够仔细，你就不可避免地需要协商一致的算法。

这不是巧合：可以证明，线性化的比较后设置(或加一后获取)寄存器和全序广播都等同于协商一致。也就是说，如果你能解决其中的一个问题，你可以把它转化为其他问题的解决方案。这是一个相当深刻和令人惊讶的洞察力！

现在是时候直接解决协商一致的问题了，我们将在本章的其余部分中完成它。

## 分布式事务与协商一致

协商一致是分布式计算中最重要和最基本的问题之一。表面上看它似乎很简单：非正式地说，目标就是让几个节点就某件事达成一致。你可能认为这不应该太难。然而，许多蹩脚的系统错误地认为这个问题是容易解决的。

虽然协商一致是非常重要的，关于它的章节之所以出现在这本书的很后边是因为这个话题是相当微妙，而欣赏这种微妙之处需要一些先决的知识。即使在学术界，对协商一致的理解也是经过几十年才逐渐形成的，其间也存在着许多误解。既然我们已经讨论了复制（第5章）、事务（第7章）、系统模型（第8章）、线性化和全序广播（这一章），我们终于准备好解决协商一致问题了。

在许多情况下，节点必须达成一致意见。例如：

*主机的选举*

在具有单主机复制的数据库中，所有节点需要一起商定哪个节点是主机。如果一些节点由于网络故障而无法与其他节点通信，那么主机的地位可能会出现争议。在这种情况下，协商一致在防止出现恶劣的故障迁移非常重要，因为这会导致两个节点都认为自己是领导者的裂脑状态（见“处理节点中断”一节）。如果有两位领导者，他们都会接受写作，他们的数据也会出现分歧，从而导致不一致和数据丢失。


*原子性提交*

在支持事务跨越多个节点或分区的数据库中，我们有这样一个问题：一个事务在某些节点上可能失败，而在另一些节点上会成功。如果我们想维持事务原子性（在ACID的意义上，见“原子性”一节），我们必须让所有节点就事务的结果达成一致：要么它们都中止/回滚（如果出了什么问题），要么它们都提交（如果没有出任何问题）。这个协商一致的例子被称为*原子提交*问题。

> **不可能达成的共识**
>
> 你可能已经听说过FLP结果——以作者菲舍尔、林奇和帕特森命名——它证明了如果存在节点可能会崩溃的风险，那么没有算法总是能够达成一致。在分布式系统中，我们必须假设节点可能会崩溃，因此可靠的一致性是不可能的了。然而，我们现在在这里，正在讨论了达成一致的算法。到底是怎么一回事？
>
> 答案是，FLP结果在异步系统模型(见“系统模型与现实”一节)中证明的，这是一个非常严格的模型，它假设了确定性算法是不能使用任何时钟或超时。如果允许算法使用超时，或者使用其他方法识别可疑的崩溃节点(即使怀疑有时是错误的)，那么协商一致就可以解决。即使只允许算法使用随机数，也足以绕过这个不可能的结果。
>
> 因此，虽然不可能达成协商一致的FLP结果有着重要的理论意义，但分布式系统在实际应用中通常是可以达到共识的。

在这一节中，我们首先会更详细地研究原子提交问题。我们会特别讨论两阶段提交（2PC）算法，这是最常见的解决原子提交的方法，并在各种数据库、消息系统和应用服务器中实现。事实证明，2PC是一种协商一致的算法——但不是最好的那个。

通过学习2PC算法，我们将了解更好的协商一致算法，比如那些用于ZooKeeper（Zab）和etcd（Raft）里的那些算法。

### Atomic Commit and Two-Phase Commit (2PC)

In Chapter   7 we learned that the purpose of transaction atomicity is to provide simple semantics in the case where something goes wrong in the middle of making several writes. The outcome of a transaction is either a successful commit, in which case all of the transaction’s writes are made durable, or an abort, in which case all of the transaction’s writes are rolled back (i.e., undone or discarded).

Atomicity prevents failed transactions from littering the database with half-finished results and half-updated state. This is especially important for multi-object transactions (see “Single-Object and Multi-Object Operations”) and databases that maintain secondary indexes. Each secondary index is a separate data structure from the primary data — thus, if you modify some data, the corresponding change needs to also be made in the secondary index. Atomicity ensures that the secondary index stays consistent with the primary data (if the index became inconsistent with the primary data, it would not be very useful).

#### From single-node to distributed atomic commit

For transactions that execute at a single database node, atomicity is commonly implemented by the storage engine. When the client asks the database node to commit the transaction, the database makes the transaction’s writes durable (typically in a write-ahead log; see “Making B-trees reliable”) and then appends a commit record to the log on disk. If the database crashes in the middle of this process, the transaction is recovered from the log when the node restarts: if the commit record was successfully written to disk before the crash, the transaction is considered committed; if not, any writes from that transaction are rolled back.

Thus, on a single node, transaction commitment crucially depends on the order in which data is durably written to disk: first the data, then the commit record [72]. The key deciding moment for whether the transaction commits or aborts is the moment at which the disk finishes writing the commit record: before that moment, it is still possible to abort (due to a crash), but after that moment, the transaction is committed (even if the database crashes). Thus, it is a single device (the controller of one particular disk drive, attached to one particular node) that makes the commit atomic.

However, what if multiple nodes are involved in a transaction? For example, perhaps you have a multi-object transaction in a partitioned database, or a term-partitioned secondary index (in which the index entry may be on a different node from the primary data; see “Partitioning and Secondary Indexes”). Most “NoSQL” distributed datastores do not support such distributed transactions, but various clustered relational systems do (see “Distributed Transactions in Practice”).

In these cases, it is not sufficient to simply send a commit request to all of the nodes and independently commit the transaction on each one. In doing so, it could easily happen that the commit succeeds on some nodes and fails on other nodes, which would violate the atomicity guarantee:

* Some nodes may detect a constraint violation or conflict, making an abort necessary, while other nodes are successfully able to commit.

* Some of the commit requests might be lost in the network, eventually aborting due to a timeout, while other commit requests get through.

* Some nodes may crash before the commit record is fully written and roll back on recovery, while others successfully commit.

If some nodes commit the transaction but others abort it, the nodes become inconsistent with each other (like in Figure   7-3). And once a transaction has been committed on one node, it cannot be retracted again if it later turns out that it was aborted on another node. For this reason, a node must only commit once it is certain that all other nodes in the transaction are also going to commit.

A transaction commit must be irrevocable — you are not allowed to change your mind and retroactively abort a transaction after it has been committed. The reason for this rule is that once data has been committed, it becomes visible to other transactions, and thus other clients may start relying on that data; this principle forms the basis of read committed isolation, discussed in “Read Committed”. If a transaction was allowed to abort after committing, any transactions that read the committed data would be based on data that was retroactively declared not to have existed — so they would have to be reverted as well.

(It is possible for the effects of a committed transaction to later be undone by another, compensating transaction [73, 74]. However, from the database’s point of view this is a separate transaction, and thus any cross-transaction correctness requirements are the application’s problem.)

#### Introduction to two-phase commit

Two-phase commit is an algorithm for achieving atomic transaction commit across multiple nodes — i.e., to ensure that either all nodes commit or all nodes abort. It is a classic algorithm in distributed databases [13, 35, 75]. 2PC is used internally in some databases and also made available to applications in the form of XA transactions [76, 77] (which are supported by the Java Transaction API, for example) or via WS-AtomicTransaction for SOAP web services [78, 79].

The basic flow of 2PC is illustrated in Figure   9-9. Instead of a single commit request, as with a single-node transaction, the commit/ abort process in 2PC is split into two phases (hence the name).

*Figure 9-9. A successful execution of two-phase commit (2PC).*

> **Don’t confuse 2PC and 2PL**
>
> Two-phase commit (2PC) and two-phase locking (see “Two-Phase Locking (2PL)”) are two very different things. 2PC provides atomic commit in a distributed database, whereas 2PL provides serializable isolation. To avoid confusion, it’s best to think of them as entirely separate concepts and to ignore the unfortunate similarity in the names.

2PC uses a new component that does not normally appear in single-node transactions: a coordinator (also known as transaction manager). The coordinator is often implemented as a library within the same application process that is requesting the transaction (e.g., embedded in a Java EE container), but it can also be a separate process or service. Examples of such coordinators include Narayana, JOTM, BTM, or MSDTC.

A 2PC transaction begins with the application reading and writing data on multiple database nodes, as normal. We call these database nodes participants in the transaction. When the application is ready to commit, the coordinator begins phase 1: it sends a prepare request to each of the nodes, asking them whether they are able to commit. The coordinator then tracks the responses from the participants:

* If all participants reply “yes,” indicating they are ready to commit, then the coordinator sends out a commit request in phase 2, and the commit actually takes place.

* If any of the participants replies “no,” the coordinator sends an abort request to all nodes in phase 2.

This process is somewhat like the traditional marriage ceremony in Western cultures: the minister asks the bride and groom individually whether each wants to marry the other, and typically receives the answer “I do” from both. After receiving both acknowledgments, the minister pronounces the couple husband and wife: the transaction is committed, and the happy fact is broadcast to all attendees. If either bride or groom does not say “yes,” the ceremony is aborted [73].

#### A system of promises

From this short description it might not be clear why two-phase commit ensures atomicity, while one-phase commit across several nodes does not. Surely the prepare and commit requests can just as easily be lost in the two-phase case. What makes 2PC different?

To understand why it works, we have to break down the process in a bit more detail:

1. When the application wants to begin a distributed transaction, it requests a transaction ID from the coordinator. This transaction ID is globally unique.

2. The application begins a single-node transaction on each of the participants, and attaches the globally unique transaction ID to the single-node transaction. All reads and writes are done in one of these single-node transactions. If anything goes wrong at this stage (for example, a node crashes or a request times out), the coordinator or any of the participants can abort.

3. When the application is ready to commit, the coordinator sends a prepare request to all participants, tagged with the global transaction ID. If any of these requests fails or times out, the coordinator sends an abort request for that transaction ID to all participants.

4. When a participant receives the prepare request, it makes sure that it can definitely commit the transaction under all circumstances. This includes writing all transaction data to disk (a crash, a power failure, or running out of disk space is not an acceptable excuse for refusing to commit later), and checking for any conflicts or constraint violations. By replying “yes” to the coordinator, the node promises to commit the transaction without error if requested. In other words, the participant surrenders the right to abort the transaction, but without actually committing it.

5. When the coordinator has received responses to all prepare requests, it makes a definitive decision on whether to commit or abort the transaction (committing only if all participants voted “yes”). The coordinator must write that decision to its transaction log on disk so that it knows which way it decided in case it subsequently crashes. This is called the commit point.

6. Once the coordinator’s decision has been written to disk, the commit or abort request is sent to all participants. If this request fails or times out, the coordinator must retry forever until it succeeds. There is no more going back: if the decision was to commit, that decision must be enforced, no matter how many retries it takes. If a participant has crashed in the meantime, the transaction will be committed when it recovers — since the participant voted “yes,” it cannot refuse to commit when it recovers.

Thus, the protocol contains two crucial “points of no return”: when a participant votes “yes,” it promises that it will definitely be able to commit later (although the coordinator may still choose to abort); and once the coordinator decides, that decision is irrevocable. Those promises ensure the atomicity of 2PC. (Single-node atomic commit lumps these two events into one: writing the commit record to the transaction log.)

Returning to the marriage analogy, before saying “I do,” you and your bride/ groom have the freedom to abort the transaction by saying “No way!” (or something to that effect). However, after saying “I do,” you cannot retract that statement. If you faint after saying “I do” and you don’t hear the minister speak the words “You are now husband and wife,” that doesn’t change the fact that the transaction was committed. When you recover consciousness later, you can find out whether you are married or not by querying the minister for the status of your global transaction ID, or you can wait for the minister’s next retry of the commit request (since the retries will have continued throughout your period of unconsciousness).

#### Coordinator failure

We have discussed what happens if one of the participants or the network fails during 2PC: if any of the prepare requests fail or time out, the coordinator aborts the transaction; if any of the commit or abort requests fail, the coordinator retries them indefinitely. However, it is less clear what happens if the coordinator crashes.

If the coordinator fails before sending the prepare requests, a participant can safely abort the transaction. But once the participant has received a prepare request and voted “yes,” it can no longer abort unilaterally — it must wait to hear back from the coordinator whether the transaction was committed or aborted. If the coordinator crashes or the network fails at this point, the participant can do nothing but wait. A participant’s transaction in this state is called in doubt or uncertain.

The situation is illustrated in Figure   9-10. In this particular example, the coordinator actually decided to commit, and database 2 received the commit request. However, the coordinator crashed before it could send the commit request to database 1, and so database 1 does not know whether to commit or abort. Even a timeout does not help here: if database 1 unilaterally aborts after a timeout, it will end up inconsistent with database 2, which has committed. Similarly, it is not safe to unilaterally commit, because another participant may have aborted.

*Figure 9-10. The coordinator crashes after participants vote “yes.” Database 1 does not know whether to commit or abort.*

Without hearing from the coordinator, the participant has no way of knowing whether to commit or abort. In principle, the participants could communicate among themselves to find out how each participant voted and come to some agreement, but that is not part of the 2PC protocol.

The only way 2PC can complete is by waiting for the coordinator to recover. This is why the coordinator must write its commit or abort decision to a transaction log on disk before sending commit or abort requests to participants: when the coordinator recovers, it determines the status of all in-doubt transactions by reading its transaction log. Any transactions that don’t have a commit record in the coordinator’s log are aborted. Thus, the commit point of 2PC comes down to a regular single-node atomic commit on the coordinator.

#### Three-phase commit

Two-phase commit is called a blocking atomic commit protocol due to the fact that 2PC can become stuck waiting for the coordinator to recover. In theory, it is possible to make an atomic commit protocol nonblocking, so that it does not get stuck if a node fails. However, making this work in practice is not so straightforward.

As an alternative to 2PC, an algorithm called three-phase commit (3PC) has been proposed [13, 80]. However, 3PC assumes a network with bounded delay and nodes with bounded response times; in most practical systems with unbounded network delay and process pauses (see Chapter   8), it cannot guarantee atomicity.

In general, nonblocking atomic commit requires a perfect failure detector [67, 71] — i.e., a reliable mechanism for telling whether a node has crashed or not. In a network with unbounded delay a timeout is not a reliable failure detector, because a request may time out due to a network problem even if no node has crashed. For this reason, 2PC continues to be used, despite the known problem with coordinator failure.