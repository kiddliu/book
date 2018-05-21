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

### 依赖线性化

在什么情况下线性化是有用的？查看一场体育比赛的最后比分也许是一个轻率的例子：在这种情况下，比赛结果晚了几秒更新不太可能造成任何真正的伤害。然而，在一些领域线性化是使系统正常工作的重要条件。

#### 锁定与主机选举

使用单主机复制的系统需要确保确实只有一个主机，而不是多个主机（裂脑）。选举主机的一种方法是使用锁：每一个节点启动的时候都试图获得锁，成功的节点将成为主机。不管这个锁是如何实现的，它必须是线性化的：所有节点都必须同意哪个节点拥有锁；不然没用。

像Apache ZooKeyer和etcd这样的协调服务通常用于实现分布式的锁与主机选举。他们使用协商一致算法以容错方式实现线性化操作（我们在本章后面的“容错协商一致”一节中讨论了这些算法）。正确实现锁和主机选举仍然有许多微妙的细节（见比如“主机与锁”中的围栏问题），而诸如Apache Curator这样的库，通过提供基于ZooKeeper的高级解决方案起到帮助作用。然而，线性化的存储服务是这些协调任务的基础。

分布式的锁定也用在一些分布式数据库中的更高粒度级别，例如Oracle Real Application Clusters （RAC）。由于多个节点共享对同一磁盘存储系统的访问，RAC在每个磁盘页上都用到锁。由于这些线性化的锁位于事务执行的关键路径上，因此RAC的部署通常有一个专用的集群互连网络，用于数据库节点之间的通信。

#### 约束与唯一性保证

唯一性约束在数据库中是很常见的：例如，用户名或电子邮件地址必须唯一地标识一个用户，而在文件存储服务中，不可能有两个路径和文件名相同的文件。如果要在写入数据时强制执行这个约束（如果两个人试图同时用同样的名字创建用户或者文件，其中一个将被返回错误），你需要线性化。

这种情况实际上与锁类似：当用户注册你的服务时，您可以想象成他们获得了他们选择的用户名上的“锁”。这个操作也非常类似原子性的比较后设置，如果用户名没有被占用，就把这个用户名设为申请该用户名用户的ID。

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

在有着单主机复制的系统中（见“主机与从机”一节），主机拥有用于写入的数据的主副本，而从机在其他节点上维护数据的备份副本。如果您从主机或同步更新的从机中读取数据，那么它们有可能是线性化的。然而，并不是每个单主机数据库实际上都是线性化的，要么是因为设计（比如因为它使用快照隔离），要么是由于并发bug。

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

如果一个系统服从因果关系所施加的顺序，我们就说它在*因果一致*的。例如，快照隔离提供了因果一致性：当您从数据库中读取数据时，当您看到一些数据时，您还必须能够看到因果关系之前的任何数据（假设它在此期间没有被删除）。

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

如果您熟悉分布式版本控制系统，比如Git，它们的版本历史非常类似因果依赖关系图。通常一次提交在另一次提交之后，在一条直线上进行的，但有时会有分支（当几个人同时在一个项目上工作时），而合并分支是在并发创建的提交在合并时创建的。

#### 线性化强于因果一致性

那么因果顺序和线性化之间有什么关系呢？答案是线性化意味着因果关系：任何可线性化的系统都将正确地保留因果关系。特别是，如果系统中有多个通信通道（例如图9-5中的消息队列和文件存储服务），线性化可以确保因果关系自动保持，而无需系统做任何特殊的处理（例如在不同组件之间传递时间戳）。

线性化确保因果关系使线性化系统易于理解并且有吸引力。然而，正如在“线性化的成本”一节中所讨论的，使系统线性化可能会损害其性能和可用性，特别是如果系统有显著的网络延迟(例如，它分布在不同地地理区域)。由于这个原因，一些分布式数据系统已经放弃了线性化，这使得它们能够获得更好的性能但也更难以使用。

好消息是折中方案是可行的。线性化不是保持因果关系的唯一途径——还有其他方法。系统可以是因果一致的，而无需引入线性化带来的性能影响（特别是CAP定理不适用了）。事实上，因果一致性是最强大的一致性模型，它不会因网络延迟而减慢，并且在网络故障面前仍然可用。

在许多情况下，看起来需要线性化的系统实际上只是要求因果一致性，而因果一致性可以更有效地实现。基于这一观察，研究人员正在探索保存因果关系的新型数据库，其性能和可用性特征与最终一致的系统类似。

由于这项研究是很近的，它还没有进入生产性系统中，而且还有一些挑战有待克服。然而，它是未来系统的一个有前途的方向。

#### 抓取因果关系

我们不会在这里讨论非线性系统可以如何保持因果一致性的所有细节，而只是简单地探讨一些关键的想法。

为了保持因果关系，您需要知道哪个操作发生在另外哪个操作之前。这是一个偏序：并发操作可以按任何顺序处理，但如果一个操作发生在另一个操作之前，则必须在每个副本上按照那个顺序处理它们。因此，当副本处理一个操作时，它必须确保所有因果前操作（之前发生的所有操作）都已被处理；如果缺少了某个前序操作，则后面的操作必须等待，直至前序操作被处理。

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

* 您可以为每一个操作附加来自现世时钟（物理时钟）的时间戳。这样的时间戳并不是连续的，但是如果它们具有足够高的精度，它们可能就足以完全全序操作。在以最后一次写入为准的冲突解决方法中用到了这个事实（见“为事件排序的时间戳”一节）。

* 您可以预先分配序列号块。例如，节点A可能要求从1到1，000之间的序列号块，而节点B可能要求从1，001到2，000之间的块。然后每个节点可以独立地从各自的块分配序列号，并在可用序号减少时分配到一个新块。

这三个选项都比把所有操作通过递增计数器的单主机者推动所有操作更具伸缩性。它们为每个操作生成一个唯一的、近似增加的序列号。然而，它们都有一个问题：它们产生的序列号与因果关系不一致。

These three options all perform better and are more scalable than pushing all operations through a single leader that increments a counter. They generate a unique, approximately increasing sequence number for each operation. However, they all have a problem: the sequence numbers they generate are not consistent with causality.

The causality problems occur because these sequence number generators do not correctly capture the ordering of operations across different nodes:

* Each node may process a different number of operations per second. Thus, if one node generates even numbers and the other generates odd numbers, the counter for even numbers may lag behind the counter for odd numbers, or vice versa. If you have an odd-numbered operation and an even-numbered operation, you cannot accurately tell which one causally happened first.

* Timestamps from physical clocks are subject to clock skew, which can make them inconsistent with causality. For example, see Figure   8-3, which shows a scenario in which an operation that happened causally later was actually assigned a lower timestamp.viii

* In the case of the block allocator, one operation may be given a sequence number in the range from 1,001 to 2,000, and a causally later operation may be given a number in the range from 1 to 1,000. Here, again, the sequence number is inconsistent with causality.

#### Lamport timestamps

Although the three sequence number generators just described are inconsistent with causality, there is actually a simple method for generating sequence numbers that is consistent with causality. It is called a Lamport timestamp, proposed in 1978 by Leslie Lamport [56], in what is now one of the most-cited papers in the field of distributed systems.

The use of Lamport timestamps is illustrated in Figure   9-8. Each node has a unique identifier, and each node keeps a counter of the number of operations it has processed. The Lamport timestamp is then simply a pair of (counter, node ID). Two nodes may sometimes have the same counter value, but by including the node ID in the timestamp, each timestamp is made unique.

*Figure 9-8. Lamport timestamps provide a total ordering consistent with causality.*

A Lamport timestamp bears no relationship to a physical time-of-day clock, but it provides total ordering: if you have two timestamps, the one with a greater counter value is the greater timestamp; if the counter values are the same, the one with the greater node ID is the greater timestamp.

So far this description is essentially the same as the even/ odd counters described in the last section. The key idea about Lamport timestamps, which makes them consistent with causality, is the following: every node and every client keeps track of the maximum counter value it has seen so far, and includes that maximum on every request. When a node receives a request or response with a maximum counter value greater than its own counter value, it immediately increases its own counter to that maximum.

This is shown in Figure   9-8, where client A receives a counter value of 5 from node 2, and then sends that maximum of 5 to node 1. At that time, node 1’ s counter was only 1, but it was immediately moved forward to 5, so the next operation had an incremented counter value of 6.

As long as the maximum counter value is carried along with every operation, this scheme ensures that the ordering from the Lamport timestamps is consistent with causality, because every causal dependency results in an increased timestamp.

Lamport timestamps are sometimes confused with version vectors, which we saw in “Detecting Concurrent Writes”. Although there are some similarities, they have a different purpose: version vectors can distinguish whether two operations are concurrent or whether one is causally dependent on the other, whereas Lamport timestamps always enforce a total ordering. From the total ordering of Lamport timestamps, you cannot tell whether two operations are concurrent or whether they are causally dependent. The advantage of Lamport timestamps over version vectors is that they are more compact.

#### Timestamp ordering is not sufficient

Although Lamport timestamps define a total order of operations that is consistent with causality, they are not quite sufficient to solve many common problems in distributed systems.

For example, consider a system that needs to ensure that a username uniquely identifies a user account. If two users concurrently try to create an account with the same username, one of the two should succeed and the other should fail. (We touched on this problem previously in “The leader and the lock”.)

At first glance, it seems as though a total ordering of operations (e.g., using Lamport timestamps) should be sufficient to solve this problem: if two accounts with the same username are created, pick the one with the lower timestamp as the winner (the one who grabbed the username first), and let the one with the greater timestamp fail. Since timestamps are totally ordered, this comparison is always valid.

This approach works for determining the winner after the fact: once you have collected all the username creation operations in the system, you can compare their timestamps. However, it is not sufficient when a node has just received a request from a user to create a username, and needs to decide right now whether the request should succeed or fail. At that moment, the node does not know whether another node is concurrently in the process of creating an account with the same username, and what timestamp that other node may assign to the operation.

In order to be sure that no other node is in the process of concurrently creating an account with the same username and a lower timestamp, you would have to check with every other node to see what it is doing [56]. If one of the other nodes has failed or cannot be reached due to a network problem, this system would grind to a halt. This is not the kind of fault-tolerant system that we need.

The problem here is that the total order of operations only emerges after you have collected all of the operations. If another node has generated some operations, but you don’t yet know what they are, you cannot construct the final ordering of operations: the unknown operations from the other node may need to be inserted at various positions in the total order.

To conclude: in order to implement something like a uniqueness constraint for usernames, it’s not sufficient to have a total ordering of operations — you also need to know when that order is finalized. If you have an operation to create a username, and you are sure that no other node can insert a claim for the same username ahead of your operation in the total order, then you can safely declare the operation successful.

This idea of knowing when your total order is finalized is captured in the topic of total order broadcast.

### Total Order Broadcast

If your program runs only on a single CPU core, it is easy to define a total ordering of operations: it is simply the order in which they were executed by the CPU. However, in a distributed system, getting all nodes to agree on the same total ordering of operations is tricky. In the last section we discussed ordering by timestamps or sequence numbers, but found that it is not as powerful as single-leader replication (if you use timestamp ordering to implement a uniqueness constraint, you cannot tolerate any faults).

As discussed, single-leader replication determines a total order of operations by choosing one node as the leader and sequencing all operations on a single CPU core on the leader. The challenge then is how to scale the system if the throughput is greater than a single leader can handle, and also how to handle failover if the leader fails (see “Handling Node Outages”). In the distributed systems literature, this problem is known as total order broadcast or atomic broadcast [25, 57, 58]. ix

> Scope of ordering guarantee
>
> Partitioned databases with a single leader per partition often maintain ordering only per partition, which means they cannot offer consistency guarantees (e.g., consistent snapshots, foreign key references) across partitions. Total ordering across all partitions is possible, but requires additional coordination [59].

Total order broadcast is usually described as a protocol for exchanging messages between nodes. Informally, it requires that two safety properties always be satisfied:

Reliable delivery

No messages are lost: if a message is delivered to one node, it is delivered to all nodes.

Totally ordered delivery

Messages are delivered to every node in the same order.

A correct algorithm for total order broadcast must ensure that the reliability and ordering properties are always satisfied, even if a node or the network is faulty. Of course, messages will not be delivered while the network is interrupted, but an algorithm can keep retrying so that the messages get through when the network is eventually repaired (and then they must still be delivered in the correct order).

#### Using total order broadcast

Consensus services such as ZooKeeper and etcd actually implement total order broadcast. This fact is a hint that there is a strong connection between total order broadcast and consensus, which we will explore later in this chapter.

Total order broadcast is exactly what you need for database replication: if every message represents a write to the database, and every replica processes the same writes in the same order, then the replicas will remain consistent with each other (aside from any temporary replication lag). This principle is known as state machine replication [60], and we will return to it in Chapter   11.

Similarly, total order broadcast can be used to implement serializable transactions: as discussed in “Actual Serial Execution”, if every message represents a deterministic transaction to be executed as a stored procedure, and if every node processes those messages in the same order, then the partitions and replicas of the database are kept consistent with each other [61].

An important aspect of total order broadcast is that the order is fixed at the time the messages are delivered: a node is not allowed to retroactively insert a message into an earlier position in the order if subsequent messages have already been delivered. This fact makes total order broadcast stronger than timestamp ordering.

Another way of looking at total order broadcast is that it is a way of creating a log (as in a replication log, transaction log, or write-ahead log): delivering a message is like appending to the log. Since all nodes must deliver the same messages in the same order, all nodes can read the log and see the same sequence of messages.

Total order broadcast is also useful for implementing a lock service that provides fencing tokens (see “Fencing tokens”). Every request to acquire the lock is appended as a message to the log, and all messages are sequentially numbered in the order they appear in the log. The sequence number can then serve as a fencing token, because it is monotonically increasing. In ZooKeeper, this sequence number is called zxid [15].

#### Implementing linearizable storage using total order broadcast

As illustrated in Figure   9-4, in a linearizable system there is a total order of operations. Does that mean linearizability is the same as total order broadcast? Not quite, but there are close links between the two.x

Total order broadcast is asynchronous: messages are guaranteed to be delivered reliably in a fixed order, but there is no guarantee about when a message will be delivered (so one recipient may lag behind the others). By contrast, linearizability is a recency guarantee: a read is guaranteed to see the latest value written.

However, if you have total order broadcast, you can build linearizable storage on top of it. For example, you can ensure that usernames uniquely identify user accounts.

Imagine that for every possible username, you can have a linearizable register with an atomic compare-and-set operation. Every register initially has the value null (indicating that the username is not taken). When a user wants to create a username, you execute a compare-and-set operation on the register for that username, setting it to the user account ID, under the condition that the previous register value is null. If multiple users try to concurrently grab the same username, only one of the compare-and-set operations will succeed, because the others will see a value other than null (due to linearizability).

You can implement such a linearizable compare-and-set operation as follows by using total order broadcast as an append-only log [62, 63]: 

1. Append a message to the log, tentatively indicating the username you want to claim.

2. Read the log, and wait for the message you appended to be delivered back to you.xi

3. Check for any messages claiming the username that you want. If the first message for your desired username is your own message, then you are successful: you can commit the username claim (perhaps by appending another message to the log) and acknowledge it to the client. If the first message for your desired username is from another user, you abort the operation.

Because log entries are delivered to all nodes in the same order, if there are several concurrent writes, all nodes will agree on which one came first. Choosing the first of the conflicting writes as the winner and aborting later ones ensures that all nodes agree on whether a write was committed or aborted. A similar approach can be used to implement serializable multi-object transactions on top of a log [62].

While this procedure ensures linearizable writes, it doesn’t guarantee linearizable reads — if you read from a store that is asynchronously updated from the log, it may be stale. (To be precise, the procedure described here provides sequential consistency [47, 64], sometimes also known as timeline consistency [65, 66], a slightly weaker guarantee than linearizability.) To make reads linearizable, there are a few options:

* You can sequence reads through the log by appending a message, reading the log, and performing the actual read when the message is delivered back to you. The message’s position in the log thus defines the point in time at which the read happens. (Quorum reads in etcd work somewhat like this [16].)

* If the log allows you to fetch the position of the latest log message in a linearizable way, you can query that position, wait for all entries up to that position to be delivered to you, and then perform the read. (This is the idea behind ZooKeeper’s sync() operation [15].)

* You can make your read from a replica that is synchronously updated on writes, and is thus sure to be up to date. (This technique is used in chain replication [63]; see also “Research on Replication”.)

#### Implementing total order broadcast using linearizable storage

The last section showed how to build a linearizable compare-and-set operation from total order broadcast. We can also turn it around, assume that we have linearizable storage, and show how to build total order broadcast from it.

The easiest way is to assume you have a linearizable register that stores an integer and that has an atomic increment-and-get operation [28]. Alternatively, an atomic compare-and-set operation would also do the job.

The algorithm is simple: for every message you want to send through total order broadcast, you increment-and-get the linearizable integer, and then attach the value you got from the register as a sequence number to the message. You can then send the message to all nodes (resending any lost messages), and the recipients will deliver the messages consecutively by sequence number.

Note that unlike Lamport timestamps, the numbers you get from incrementing the linearizable register form a sequence with no gaps. Thus, if a node has delivered message 4 and receives an incoming message with a sequence number of 6, it knows that it must wait for message 5 before it can deliver message 6. The same is not the case with Lamport timestamps — in fact, this is the key difference between total order broadcast and timestamp ordering.

How hard could it be to make a linearizable integer with an atomic increment-and-get operation? As usual, if things never failed, it would be easy: you could just keep it in a variable on one node. The problem lies in handling the situation when network connections to that node are interrupted, and restoring the value when that node fails [59]. In general, if you think hard enough about linearizable sequence number generators, you inevitably end up with a consensus algorithm.

This is no coincidence: it can be proved that a linearizable compare-and-set (or increment-and-get) register and total order broadcast are both equivalent to consensus [28, 67]. That is, if you can solve one of these problems, you can transform it into a solution for the others. This is quite a profound and surprising insight!

It is time to finally tackle the consensus problem head-on, which we will do in the rest of this chapter.