# 第八章 困扰分布式系统的麻烦

*Hey I just met you*
*The network’s laggy*
*But here’s my data*
*So store it maybe*

凯尔·金斯伯里，[*卡莉·蕾·杰普森与网络分区的危险*](https://www.youtube.com/watch?v=I6fD3FhwDio)

---

在过去的几章中一个重复的主题一直是系统如何处理出错的事情。比如说，我们讨论了副本的故障迁移（在“处理节点离线”一节），复制延迟（在“复制延迟的问题”一节），以及事务的并发控制（在“弱隔离级别”一节中）。随着我们慢慢理解现实系统中会发生的各种边缘情况，我们也能更好地处理它们。

然而，即使我们已经谈论了很多关于故障的问题，过去的几章仍然太乐观了。现实是更加黑暗的。我们现在会走向最悲观的一面，假设所有可能会出错的事都会出错。（有经验的系统运营工程师会告诉你这是一个合理的假设。如果你很有礼貌地问，它们很可能在抚摸过去战斗的伤痕同时告诉你一些惊悚的故事。）

使用分布式系统从本质上区别于在单台计算机上写软件——最大的区别在于事务出错多了许多新的、令人兴奋的方式。在这一章中，我们将体会实践中出现的问题，并且理解哪些可以依赖而哪些不能。

最后，我们作为工程师的任务是构建能够完成工作的系统（比如，满足用户希望的保证），哪怕一切都出错。在第九章，我们会看到那些在分布式系统中可以提供这样保证的算法的示例。但是首先，在这一章，我们必须理解我们面对的挑战是什么。

这一章对于分布式系统中可能出现的问题进行了彻底的悲观和沮丧的概述。我们会看到网络的问题（在“不可靠的网络”一节）；时钟与计时问题（在“不可靠的时钟”一节）；也会讨论它们可以避免到什么样的程度。所有这些问题的结果都令人迷惑，所以我们将探讨如何思考分布式系统的状态以及如何推理已经发生了的事情（在“知识，真相和谎言”一节）。

## 故障与部分失效

你在单台计算机上写应用程序时，它通常表现地相当可预测：要么它工作，要么它不能。有bug的软件会给人一种感觉，计算机有时“有糟糕的一天”（通常重启就可以解决的问题），但是那很可能是写得很差的软件导致的结果。

单台设备上的软件本质上没有理由不靠谱：只要硬件工作正常，同样的操作总是应该产生同样的结果（这是确定性）。如果出了硬件问题（比如内存破坏或是接口松动），结果常常是整个系统崩溃了（比如，核心崩溃，“蓝屏”，无法启动）。单个计算机配上好的软件通常要么完全正常要么完全不正常，而不会处于介于两者之间的某个状态。

这是计算机设计中的有意选择：如果发生了内部故障，我们倾向于计算机完全崩溃而不是返回错误的结果，因为错误的结果处理的时候很困难，也很让人困惑。因此，计算机隐藏了它们实现所基于的模糊物理现实，而是呈现出了一种理想化的系统模型，执行起来有着数学一般地完美。CPU指令总之做同样地事情；如果你把数据写入到内存或者磁盘，数据保持不变并且不会被随机地破坏。这种始终正确计算的设计目标可以一直追溯到第一台数字计算机。

当你在写运行在好几台用网络连接在一起的计算机上的软件时，情况从本质上就是不同的。在分布式系统中，我们不再运行在一个理想化的系统模型中了——我们别无选择，只能直面现实世界里的混乱现实。而在现实世界中，各种各样的事情都会出错，如这一则轶事所示：

*在我有限的经验中我处理过单个数据中心（DC）中的长活网络分区，PDU（配电单元）故障，交换机故障，整个机架的意外电源周期，整个数据中心的主干故障，整个数据中心的电力故障以及一位低血糖司机把他的福特皮卡开进了数据中心的暖通空调[供暖，通风和空调]系统。而我甚至不是一个运维人员。*

*寇达·黑尔*

在分布式系统中，系统的某些组件可能会以某种不可预知的方式坏掉，即使系统的其他部分仍然工作正常。这被称为部分失效。困难在于部分失效是非确定性的：如果你尝试做任何涉及多个节点和网络的事情，它有时可能工作正常，有时会毫无征兆的失效。我们会看到，你甚至不知道事情到底是成功了还是失败了，因为消息通过网络传输所花的时间也是非确定性的。这种不确定性以及部分失败的可能性正是分布式系统难以应付的原因。

### 云计算与超计算

关于如何构建大型计算系统有一系列哲学：

* 一个方向是*高性能计算*（HPC）领域。拥有上千颗CPU的超级计算机通常用于计算密集型科学计算任务，比如天气预报或者分子动力学（模拟原子和分子的运动）。

* 另一个方面是云计算，它不是很好定义但通常与多租户数据中心，连接了IP网络（通常是以太网）的商品计算机，弹性/按需资源分配和计费计费相关联。

* 传统的企业数据中心位于这两者之间

有了这些不同的哲学处理故障的方法也非常不同。在超级计算机中，任务通常会设置核查点时不时地把它的计算状态转到持久存储中。如果一个节点失效，常用的解决方案就是停下整个集群的工作负载。当故障的节点维修完毕，计算会从上一次核查点重启。因此，相比于分布式系统超级计算机与单节点计算机更类似：它处理部分失效的方式是把它升级为完全失效——如果系统的任意一部分失效，就让整个系统崩溃（与单个设备上的核心崩溃类似）。

在这本书里我们关注的是实现互联网服务的系统，它们一般与超计算机非常不一样：

* 许多与互联网相关的应用是在线的，因为它们需要能够随时为用户提供低延迟的服务。让服务离线——比如，由于维护而整个集群停机——是不能接受的。相比之下，离线（批处理）任务例如天气模拟可以暂停重启，影响相当小。

* 超级计算机通常是用专用的硬件搭建的，每个节点都相当可靠，节点之间是通过共享内存以及远程直接内存访问（RDMA）通信的。零一方面，云服务的节点是用商用设备构建的，由于规模经济可以以较低的成本提供等价的性能，但也具有较高的故障率。

* 大型数据中心网络常常是基于IP与以太网的，它以Clos拓扑组织在一起以提供高二等分带宽。超级计算机常常使用专用的网络拓扑结构，比如多维网格与环面，它为有着已知通信模式的HPC工作负载提供了更好的性能。

* 系统越大，其中一个组件坏掉的可能性就越大。随着时间推移，坏掉的被修好而又有新的东西坏掉，但是在一个有着数千个节点的系统中，假设有些东西总是坏的是很合理的。当错误处理册率包含直接放弃的时候，大型系统最终花费了大量时间从故障中恢复而不是做有用的工作。

* 如果系统可以接受失效的节点并继续作为一个整体工作，这对于运维来说是非常有用的功能：举个例子，你可以执行滚动升级（见第四章），每次重启一个节点而不会中断继续向用户提供服务。在云环境中，如果一个虚拟主机工作不正常，你只需要干掉它再请求一个新的（并且期望新的会更快一些）。

* 在一个地理位置上分布式的部署（让数据从地理位置上更靠近用户来降低访问延迟）中，通信基本是走互联网的，相比于本地网络它更慢也更不可靠。超级计算机一般假设所有的节点都是在一起的。

如果我们想要分布式系统正常工作，我们必须接受部分失效的可能性并且在软件中构建容错机制。换句话说，我们需要使用不可靠的组件构建可靠的系统。（我们在“可靠性”一节中讨论过，没有什么是完美可靠的，所以我们需要理解我们可以做哪些实际承诺。）

即使在只有几个节点的小型系统中，考虑部分失效也是很重要的。在小型系统中，大部分组件在大多数时间都工作正常。然而，迟早，系统的某些部分将会出现故障，而软件必须以某种方式处理它。故障处理必须是软件设计的一部分，而你（软件的运营者）需要知道当故障发生时应从软件中得到什么样的行为。

假定缺陷是罕见的而只是希望最好的事情发生是不明智的。考虑各种可能的故障——甚至是不太可能的错误——以及在测试环境中人为地构建这些情况来看看会发生什么是非常重要的。在分布式系统中，怀疑，悲观和偏执都会有回报的。

> **使用不可靠的组件构建可靠的系统**
>
> 也许你在想中到底有意义么——直观地看系统似乎只能如其最不可靠的组件（它最薄弱的环节）一样可靠。事实不是这样的：实际上，在计算科学中基于不太可靠的底层基础构建更可靠的系统是一个古老的想法。举个例子：
>
> * 纠错码使得数字信息在通信通道中精确传输的同时偶尔允许某些位出错，比如由于无线网络上的无线电干扰
>
> * IP协议（网际协议）是不可靠的：它会丢弃，延迟，复制或重新排列数据包。TCP（传输控制协议）在IP协议之上提供了一个更可靠的传输层：它确保丢失的数据包被重新传输，消除重复，并且数据包被重新组装为它们的发送顺序。
>
> 虽然系统可以比它的基础部分更可靠，但这种可靠性总是有限度的。例如，纠错码可以处理少量的单比特错误，但是如果信号被干扰所淹没，那么通过通信通道可以获得的数据量有一个基本的限制。TCP可以隐藏数据包丢失，重复和重新排序，但它不能神奇地消除网络中的延迟。
>
> 虽然更可靠的高级别的系统并不完美，但是它仍然很有用，因为它可以处理一些棘手的低级故障，因此通常可以更轻松地解决和处理其余的故障。 我们将在“TODO:端到端的论点”中进一步探讨这个问题。

## 不可靠的网络

正如我们在第二部分介绍中讨论到的，这本书中我们关注的分布式系统是无共享系统：比如说，一堆通过网络连接的设备。网络是这些设备唯一的通信方式——我们假设每台设备都有自己的内存于磁盘，而一台设备不能访问另一台设备的内存与磁盘（除非是通过网络向服务发起请求）。

无共享不是构建系统的唯一方式，但是它变成了构建互联网服务的占主导地位的方式是因为下边几个原因：它相对更便宜因为他不需要特殊的硬件，它可以使用商品化的云计算服务，并且它可以通过遍布多个地理位置的分布式数据中心的冗余性实现高可靠性。

互联网以及绝大多数数据中心的内部网络（常常是以太网）是异步包网络。在这种网络中，一个节点可以发送消息（包）到另一个节点，但是网络不保证什么时候它会到达，或者它会不会到达。如果你发送请求并期待响应，许多事都会出错（其中一部分如图8-1所示）：

1. 你的请求也许已经丢失了（也许有人拔掉了网线）。

2. 你的请求也许正在队列中等待而稍后会发送（也许事因为网络或是接收者过载了）。

3. 远端节点也许已经失效了（大概是崩溃了或者关机了）。

4. 远端节点也许暂时停止响应了（大概是正在经历长时间的垃圾回收暂停；见“进程暂停”一节），但是稍后它会再次开始响应。

5. 远端节点也许已经处理了你的请求，但是响应在网络中被丢掉了（也许网络交换机配置错误了）。

6. 远端节点也许已经处理了你的请求，但是响应被延迟了而稍后会被发送（也许网络或者自己的设备过载了）。

*图8-1 如果你发送了请求但是没有收到响应，我们没有办法区别到底是（a）请求丢失了，（b）远端节点离线还是（c）响应丢失了。*

发送者甚至无法区别包有没有被发送：接收者唯一的选项就是发送响应消息，而它也可能会丢失或延迟。这些问题在异步网络中是无法区别的：唯一的信息那就是你还没有收到响应。如果你发送请求到另外一个节点而没有收到响应，那是不可能说明原因的。

处理这个问题的常用方式是超时：超过某个时间你放弃等待并假设响应不可能到来了。然而，当超时发生时，你还是不知道远端节点到底有没有收到你的请求（如果请求仍然在某处排队，它仍有可能被发送到接收者，即使发送者已经放弃它了）。

### 实践中的网络故障

我们已经构建了计算机网络几十年了——人们可能希望现在我们已经知道如何使它们变得可靠。然而，看样子我们还没有成功。

一些系统的研究以及大量的轶事证据表明，即使在由一家公司运营的数据中心这样的受控环境中，网络问题也出人意料地普遍存在。一个关于中等规模数据中心的研究显示每个月会发生大约12次网络故障，其中一半是因为断开了单台设备，而另一半断开了整个机架。另一项研究测量了架顶式交换机，汇聚交换机和负载平衡器等组件的故障率[16]。它发现添加冗余的网络设备不会像你所希望的那样减少故障，因为它不能防范人为错误（例如，配置错误的交换机），这是造成中断的主要原因。

公共云服务，比如EC2，因为频繁地出现短暂的网络故障而臭名昭着，而管理良好的专用数据中心网络可能是更稳定的环境。尽管如此，没有人能够避免网络问题：例如，交换机软件升级期间的问题可能会触发网络拓扑重新配置，在此期间网络数据包可能会延迟超过一分钟。鲨鱼可能咬住海底电缆并损坏它们。其他令人惊讶的故障还包括网络接口有时会丢弃所有入站数据包，但成功发送出站数据包：仅仅由于网络链接在一个方向上工作并不能保证它也在相反的方向工作。

> 网络分区
>
> 当网络的一部分由于网络故障而与其余部分断开时，有时称为*网络分区*或*网络分割*。在本书中，我们通常会坚持使用更一般的术语*网络故障*，以避免与第六章讨论的存储系统的分区（碎片）混淆。

即使网络故障在你的环境中很少发生，但可能会发生故障的事实意味着你的软件需要能够处理它们。只要有任何通讯发生在网络上，它就有可能会失败——这是无法绕过的。

如果网络故障的错误处理机制没有定义也没有测试，则可能会发生任何错误：例如，即使网络恢复，集群可能会死锁并永久无法为请求提供服务，甚至有可能会删除你的全部数据。如果软件被置于意料之外的情况下，它可能会做出各种意想不到的事来。

处理网络故障并不一定意味着容忍它们：如果你的网络通常相当的可靠，有效的方法是在网络遇到问题时只是向用户显示错误消息。但是，你确实需要知道你的软件如何对网络问题做出反应，并确保系统能够从中恢复。主观故意触发网络问题并测试系统响应是有意义的（这是Chaos Monkey背后的理念；见“可靠性”一节）。

### 检测故障

许多系统都有自动检测故障节点的需要。例如：

* 负载均衡器需要停止向离线的节点发送请求（即，把它*从轮转中拿出来*）。

* 在单主机复制的分布式数据库中，如果主机失效，一个从机需要被提升为新的主机（见“处理节点停机”一节）。 

然而，网络的不确定性使得判断节点是否在工作变得很困难。在某些特定情况下，你可能会收到一些明确告诉你有些东西不起作用的反馈信息：

* 如果可以直接访问运行节点的设备，但没有进程正在侦听目标端口（例如，因为进程崩溃了），操作系统将回复`RST`或`FIN`数据包来帮助关闭或拒绝TCP连接。然而如果节点在处理请求时发生崩溃，你没有办法知道远程节点实际处理了多少数据。

* 如果节点进程崩溃（或者被管理员杀死了）但是该节点的操作系统仍在运行，可以用脚本通知其他节点崩溃的消息，这样另一个节点可以快速接管而无需等待超时。例如，HBase就是这么做的。

* 如果你有访问数据中心网络交换机管理界面的权限，你可以通过查询来检测硬件级别的链路故障（例如，远程设备已关机）。如果你是通过互联网连接，或者你处于共享数据中心而无法访问交换机，或者由于网络问题而无法访问管理界面，这个选择就被排除了。

* 如果路由器可以确定你尝试连接的IP地址无法访问，它可能会用ICMP协议目标无法访问的数据包回复您。然而路由器并没有神奇的故障检测能力——它与网络其他参与者有着相同的限制。

远程节点关闭的快速反馈很有用，但你不能指望它。即使TCP确认了数据包已发送，应用程序在处理数据之前可能已经崩溃了。如果你想确定请求是成功的，你需要应用程序本身的积极响应。

相反地，如果出现了问题，你可能会在堆栈的某个层上得到错误响应，但通常你必须假设你根本没有收到任何回应。你可以重试几次（TCP重试是透明的，但你也可以在应用程序级别重试），等待超时过去，最终如果在超时时间内都没有收到响应则声明该节点已经崩溃了。

### 超时和无限制的延迟

如果超时是检测故障唯一的可靠方法，那么超时时间应该多长？然而没有简单的答案。

长时间超时意味着需要长时间等待直到节点被宣告失效（在此期间，用户不得不等待或查看错误消息）。短暂超时可以更快地检测到故障，但当节点实际上只是暂时运行缓慢（例如，由于节点或网络上的负载峰值）而导致错误地宣布失效的风险更高。

过早地声明一个节点已经失效是有问题的：如果节点实际上处于活动状态并且正在执行一些操作（例如，发送电子邮件），而另一个节点接管，那么该操作最终可能会执行两次。我们将在“知识，真相与谎言”以及第9和11章中更详细地讨论这个问题。

当一个节点被宣布失效时，它的职责需要转移到其他节点，这会给其他节点和网络带来额外的负担。如果系统已经处于高负载状态，过早地宣布节点失效会使问题变得更糟。特别的是，节点可能实际上并未失效而只是因为过载而响应缓慢; 将其负载转移到其他节点可能会导致级联失效（在极端情况下，所有节点都宣告对方失效，最终一切都停止工作）。

想象一个有着可以保证数据包的最大延迟的网络的虚拟系统——每个数据包要么在一段时间*d*内交付，要么丢失，但交付永远不会比*d*更长。此外，假设你可以保证一个非故障节点总是在一段时间*r*内处理请求。在这种情况下，你可以保证每个成功的请求都会在*2d + r*时间内收到响应——如果你在该时间内没有收到响应，你知道要么网络要么远程节点不工作了。如果这为真，*2d + r*将是合理的超时时间。

然而我们所使用的大多数系统都没有这些保证：异步网络有*无限制的延迟*（也就是它们尽可能快地发送数据包，但数据包到达所需的时间没有上限），并且大多数服务器实现不能保证他们可以在某个最大时间内处理请求（见“响应时间保证”一节）。对于故障检测，系统大部分时间都在快速运行是不够的：如果超时时间很短，则往返时间只需要一个瞬间峰值就可以使系统失去平衡。

#### 网络拥塞和排队

开车的时候，由于交通堵塞道路网

* 如果几个不同的节点同时尝试发送数据包到同一个目的地，网络交换机必须把它们排队并将它们逐个发送到目标的网络链路（如图8-2所示）。在繁忙的网络链路上，数据包需要等待一段时间才能获得一个空挡（这称为网络拥塞）。如果传入的数据太多交换机队列填满了，数据包将被丢弃，因此需要重新发送——即使这时网络功能正常。

* 当数据包到达目标设备时，如果所有的CPU核心当前都处于繁忙状态，来自网络的请求将被操作系统排队，直到应用程序准备好处理它为止。根据机器的负载情况，这可能需要一段任意长度的时间。

* 在虚拟化环境中，正在运行的操作系统通常会暂停几十毫秒同时另一个虚拟机使用CPU核心。在此期间，虚拟机无法从网络中处理任何数据，因此输入的数据被虚拟机监视器排队（缓冲），进一步增加了网络延迟的可变性。

* TCP执行流量控制（也称为*拥塞避免*或*背压*），其中节点限制自己的发送速率以避免网络链路或接收节点过载。这意味着在数据在进入网络之前在发送端就会排额外的队。

*图8-2 如果好几台设备发送网络流量到同一个目的地，路由器队列会塞满。在这里，端口1，2，4都尝试向端口3发送包。*

此外，如果TCP包在某个超时时间内未得到确认（根据观察往返时间计算）则认为它丢失了，而丢失的数据包将自动重新发送。尽管应用程序没有看到数据包丢失和重新传输，但它看到了所产生的延迟（等待超时到期，之后等待重新传输的数据包得到确认）。

> TCP vs UDP
>
> 一些对延迟敏感的应用程序，例如视频会议和IP语音（VoIP），使用UDP而不是TCP。这是可靠性与延迟可变性之间的折衷：由于UDP不执行流量控制并且不重传丢失的数据包，所以它避免了可变网络延迟的一些原因（尽管它仍然易受交换机队列和调度延迟的影响）。
>
> 在延迟了的数据毫无价值的情况下UDP是一个不错的选择。例如在VoIP电话呼叫中，可能没有足够的时间在数据即将在扬声器上播放之前重新传输丢失的数据包。在这种情况下，重发数据包没有意义——应用程序必须用静音填充丢失数据包的时隙（导致声音短暂中断），然后在数据流中继续。取而代之的是重试发生在人类这一层。（“可以请你再说一遍吗？声音刚刚断了一会儿。”）

所有这些因素都会造成网络延迟的变化。当系统接近其最大容量时排队延迟的范围特别广泛：拥有大量可用容量的系统可轻松排空队列，而在利用度高的系统中长队列很快就填满了。

在公有云和多租户数据中心中，资源是在许多客户之间共享的：包括网络链接和交换机，甚至每台计算机的网络接口和CPU（当系统在虚拟机上运行时）都是共享的。批处理工作负载，比如MapReduce（见第10章）可以轻松地使网络链接饱和。由于你无法控制或是深入了解其他客户对共享资源的使用情况，如果您附近的某个人（嘈杂的邻居）正在使用大量资源，网络延迟可能会变化很大。

在这种环境中，您只能通过实验选择超时时间：衡量长时间内网络往返时间的分布情况，以及多台机器的情况，以确定期望的延迟可变性。然后，考虑到应用程序的特性，您可以确定故障检测延迟与过早超时风险之间的适当折衷方案。

更好的方案是，系统不使用配置好的常量超时时间，而是连续测量响应时间及其变化（抖动），并根据观察到的响应时间分布自动调整超时时间。这可以用Phi Accrual故障检测器完成，该检测器用在了Akka和Cassandra中。TCP重传超时也是以类似方式工作的。

### 同步 vs 异步网络

如果我们可以依靠网络来传递具有固定最大延迟的数据包，也不会丢弃数据包，那么分布式系统就会简单得多。为什么我们不能在硬件级别解决这个问题使网络变得可靠，这样软件就不用担心网络了？

为了回答这个问题，将数据中心网络与非常可靠的传统固定电话网络（非蜂窝，非VoIP）进行比较是很有趣的：延迟音频帧和掉线都是非常罕见的。电话呼叫需要始终较低的端到端延迟以及足够的带宽来传输声音的音频样本。难道在计算机网络中具有类似的可靠性和可预测性不是很好吗？

当你通过电话网络拨打电话时，它会建立一线路：为呼叫分配了固定的有保证的带宽量，在两个呼叫者之间的整个线路上都是这样。线路始终存在直到通话结束。例如，ISDN网络以每秒4000帧的固定速率运行传输。呼叫建立后，每个帧内（每个方向）分配16位空间。因此在通话期间，每一方都保证能够每250微秒发送一个精确的16位音频数据。

这种网络是同步的：即使数据通过多个路由器，也不会受到排队的影响，因为呼叫的16位空间已经在网络的下一段中保留下来了。而且由于没有排队，网络中端到端的最大延迟是固定的。我们称之为有限的延迟。

#### 我们可不可以简单地使网络延迟变得可预测？

值得注意的是电话网络中线路与TCP连接有很大不同：线路是固定数量的预留带宽，线路建立时无人可以使用，而TCP连接的数据包有机会使用任何可用的网络带宽。你可以为TCP提供可变大小的数据块（例如电子邮件或网页），并且会尽可能在最短的时间内传输它。当TCP连接空闲时，它不使用任何带宽。

如果数据中心网络和互联网是线路交换网络，那么建立线路时可以确保最大往返时间。 然而，它们并不是：以太网和IP是分组交换协议，它们受到排队的影响从而导致网络无限制的延迟。这些协议是没有线路概念的。

为什么数据中心网络和互联网使用分组交换呢？ 答案是因为它们针对突发流量进行了优化。线路适用于音频或视频通话，它们需要在通话期间每秒传输相当恒定的比特数。另一方面，请求网页、发送电子邮件或传输文件没有任何特定的带宽要求——我们只是希望它尽快完成。

如果你想通过线路传输文件，你就必须猜测带宽的分配。如果你猜的太低，传输速度就会太慢，导致网络容量浪费了。如果你猜得太高，线路就无法建立（因为如果无法保证其带宽分配，网络不能建立线路）。因此，使用线路进行突发数据传输会浪费网络容量，并导致传输缓慢。相比之下，TCP会动态调整数据传输速率以适应可用的网络容量。

已经有了一些构建同时支持电路交换和分组交换的混合网络的尝试，比如ATM。InfiniBand有一些相似之处：它在链路层实现了端到端的流量控制，这减少了在网络中排队的需要，尽管它仍然可能由于链路拥塞而遭受延迟。通过仔细使用服务质量（QoS，数据包的优先级和调度）和准入控制（对发送者限速），在分组网络上模仿线路交换是有可能的，提供统计上有界的延迟也是有可能的。

> 延迟和资源利用
>
> 更一般地说，你可以把可变延迟视为动态资源分区的结果。
>
> 假设两台电话交换机之间有一条线路，可以同时进行10,000个呼叫。通过此线路切换的每条电话线路都占用其中一个呼叫插槽。因此，你可以将线路视为可由多达10,000个并发用户共享的资源。资源以静态方式分配：即使现在只有你正在线路上进行通话，而余下的9999个插槽都未使用，你的电路仍将分配与导线充分利用时相同的固定数量的带宽。
>
> 相比之下，互联网动态地分享网络带宽。发送者互相争抢带宽以尽可能快地让它们的包通过网络，而网络交换机决定每一个时刻发送哪个包（即，带宽分配）。这种方法有排队的缺点，但优点是它最大限度地利用了线路。线路有着固定的成本，所以如果你可以更好地利用它，你通过线线发送的每个字节都更便宜。
>
> CPU也会出现类似情况：如果多个线程动态共享每个CPU核心，那么线程有时必须在另一个线程正在运行的同时在操作系统的运行队列中等待，线程暂停的时间长度不等。但是，相比于为每个线程分配了一个静态数量的CPU周期（见“响应时间保证”一节）这样做可以更好地利用硬件。更好的硬件利用率也是使用虚拟机的重要动机。
>
> 如果资源是静态分区的（例如，专用硬件和专用带宽分配），那么在某些环境中延迟保证是可实现的。然而这是以降低利用率为代价的——换句话说，它的成本更高。另一方面，动态资源分配的多租户（系统）提供了更好的利用率，所以它更便宜，但它具有可变延迟的缺点。网络中的可变延迟不是自然规律，而仅仅是成本/收益折衷的结果。

但是，目前在多租户数据中心和公共云或通过互联网进行通信时，这种服务质量控制尚未启用。当前部署了的技术不允许我们对网络的延迟或可靠性作出任何保证：我们必须假设网络拥塞，排队和无限制的延迟将会发生。因此，超时没有“正确”的值的——它们需要通过实验来确定。

## Unreliable Clocks

时钟和时间很重要。应用程序以各种方式依赖时钟来回答类似下边的问题：

1. 这个请求超时了吗？

2. 服务第99百分位的响应时间是多少？

3. 在过去五分钟之内服务平均每秒处理了多少个查询？

4. 用户在我们的网站上停留了多长时间？

5. 这篇文章是什么时侯发表的？

6. 在哪一天哪一个时间发送提醒邮件？

7. 这个缓存条目什么时候过期？

8. 日志文件中这个错误消息的时间戳是多少？

示例问题1到4测量的是持续时间（例如，正在发送的请求和正在接收的响应之间的时间间隔），而示例问题5到8描述的是时间点（在特定日期，特定时间发生的事件）。

在分布式系统中，时间是一件棘手的事情，因为通信是不即时的：消息通过网络从一台设备传输到另一台设备需要时间。收到消息的时间总是晚于发送消息的时间，但由于网络中的可变延迟，我们不知道会晚多久。在涉及多台设备的时候，有时这会使得确定事情发生的顺序变得很困难。

此外，网络上的每台机器都有自己的时钟，这是一个实际的硬件设备：通常是个石英晶体振荡器。这些设备并不完全准确，因此每台机器都有自己的时间，可能会比其他机器稍快或更慢一些。一定程度上同步这些时钟是可能的：最常用的机制是网络时间协议（NTP），它允许根据一组服务器报告的时间来调整计算机时钟。这些服务器则从更精确的时间源，比如GPS接收机，获取时间。

### 单调时钟 vs 现世时钟

现代计算机至少有两种不同的时钟：现世时钟和单调时钟。尽管它们都衡量时间，但是因为它们有不同的目的，所以区分这两者很重要。

#### 现世时钟

现世时钟是你直观地了解时钟的依据：它根据某个纪年法（也称为挂钟时间）返回当前日期和时间。 例如，Linux中的`clock_gettime(CLOCK_REALTIME)`以及Java中的`System.currentTimeMillis()`返回自公历1970年1月1日UTC时代以来的秒数（或毫秒），不包括闰秒。有些系统使用其他日期作为参考点。

现世时钟通常与NTP同步，这意味着来自一台机器的时间戳（在理想情况下）与另一台机器上的时间戳意义相同。但是，如下一节所述，现世时钟也具有各种各样的怪异特性。尤其是如果本地时钟时间在NTP服务器超前太多，它可能会被强制重置并显示跳回到先前的时间点。这些跳跃，以及它们经常忽略闰秒的事实，使得现世时钟并不适合用来测量消耗了的时间。

现世时钟还具有相当粗略的解析度，例如，在较早的Windows系统上是以10毫秒为单位前进的。在较新的系统中，这已经不是一个问题了。

#### 单调时钟

单调时钟适合用来测量持续时间（时间间隔），例如超时或者是服务的响应时间：举例来说Linux上的`clock_gettime(CLOCK_MONOTONIC)`和Java中的`System.nanoTime()`是单调时钟。 这个名字来源于他们保证总是前进的事实（而现世时钟可以在时间轴上回退）。您可以在某个时间点检查单调时钟的值，做一些事情，然后再次检查时钟。这两个值的差告诉你两次检查之间经过了多少时间。但是，时钟的绝对值是毫无意义的：它可能是计算机启动以来的纳秒数，或类似的任意值。特别是，比较来自两台不同计算机的单调时钟值是没有意义的，因为它们并不意味着同一件事。

在具有多个CPU插槽的服务器上，每个CPU可能有一个单独的定时器，但它不一定与其他CPU同步。操作系统弥补了任何差异并尝试将时钟的单调视图呈现给应用程序线程，即使它们是在不同的CPU上进行调度的。然而，对单调性的保证持半信半疑的态度是明智的。

如果检测到计算机的本地石英移动速度比NTP服务器移动速度更快或更慢，NTP则可以调整单调时钟向前移动的频率（这称为转动时钟）。 默认情况下，NTP允许时钟速率加快或减慢最多0.05％，但NTP不会导致单调时钟向前或向后跳转。单调时钟的解析度通常相当好：在大多数系统中，他们可以用几微秒或更短的单位测量时间间隔。

在分布式系统中，使用单调时钟来测量经过时间（例如，超时）通常是可以的，因为它不假定节点之间的会进行时钟同步，并且对测量的轻微不准确性不敏感。

### 时钟同步与准确度

单调时钟不需要同步，但需要根据NTP服务器或其他外部时间源设置现世时钟才使之有用。然而，我们让时钟报告正确时间的方法并不如你希望的那样可靠准确——硬件时钟和NTP变幻莫测。

举几个例子：

* 计算机中的石英钟不是很精确：它会漂移（运行的速度比设想的速度要么快要么慢）。时钟漂移的程度取决于机器的温度。谷歌假设其服务器的时钟漂移为200ppm（百万分之一），相当于每30秒钟与服务器重新同步的时钟的6毫秒漂移，或者每天重新同步时钟的17秒时间漂移。即使一切工作正常，这种漂移也会限制您可以达到的最佳可能精度。

* 如果计算机的时间与NTP服务器的时间差别太大，服务器会拒绝同步，或者本地时钟将被强制重置。任何观察时间重置前后的应用程序可能会看到时间倒退或者是突然地跳跃前进。

* 如果节点因为防火墙与NTP服务器意外隔开，错误的配置可能会在一段时间内不会被注意到。传闻证据表明在实践中这确实发生过。

* NTP同步只能与网络延迟一致，所以当你处于拥有包延迟多变的拥塞网络中时，其精度会受到限制。一项实验表明，通过互联网进行同步时，35毫秒的最小误差是可以实现的，尽管网络延迟中偶尔会出现的峰值导致大约一秒的误差。根据配置的不同，较高的网络延迟会导致NTP客户端完全放弃同步。

* 一些NTP服务器是错的或配置有问题，报告的时间偏差有几个小时之多。NTP客户端非常强大，它们会查询多个服务器并忽略异常值。尽管如此，在互联网上让陌生人告诉你时间来保证你的系统的正确性，让人总有点担心。

* 闰秒导致一分钟有59秒或者61秒长，这会使得那些没考虑闰秒的系统对时间的假设变得混乱。事实上闰秒曾经让许多大系统崩溃表明了对时钟的错误假设出现在系统里是多么的容易。处理闰秒的最佳方法是让NTP服务器“撒谎”，在一天中逐渐执行闰秒调整（这被称为拖尾），然而实践中NTP服务器的行为都很不一样。

* 在虚拟机中，硬件时钟被虚拟化了，这对需要精确计时的应用程序提出了额外的挑战。当CPU核心在虚拟机之间共享时，当另一个虚拟机正在运行的时候每个虚拟会暂停数十个毫秒。从应用程序的角度看，这种暂停表现为时钟突然向前跳跃。

* 如果在不能完全控制的设备上运行软件（比如，移动设备或嵌入式设备），你大概根本无法信任设备的硬件时钟。一些用户故意把硬件时钟设置为不正确的日期和时间，比如为了规避游戏中的时间限制。因此，时钟可能会设置到很久之前的过去或是很久以后的未来。

如果足够关注时钟精度并愿意投入大量资源，实现非常好的时钟精度是由可能的。 例如，针对金融机构的欧盟法规MiFID II草案要求所有高频交易基金必须把它们的时钟与UTC时间同步在100微秒之内，从而协助诊断市场异常现象，比如“闪现崩溃”以及对市场的操纵。

使用GPS接收机与精确时间协议（PTP），并且经过仔细的部署和监测就可以实现这种精确度。但是，这需要大量的努力和专业知识，并且时钟同步有许多种出错的方式。如果NTP守护进程配置错误，或者防火墙阻止NTP通信，由漂移引起的时间误差很快会变得变大。

### 依赖于同步时钟

时钟的问题在于虽然它们看起来很简单，使用也很方便，但它们有着惊人数量的缺陷：一天可能不是精确的86400秒，现世时钟可能会向后偏移，而一个节点上的时间可能与另一个节点上的时间完全不同。

在本章早些时候，我们讨论了网络丢包和任意延迟数据包的问题。尽管网络在大多数情况下都表现良好，但软件在设计时必须假定网络偶尔会出故障，软件必须正常处理这些故障。时钟也是如此：虽然它们大多数时间都工作得很好，还是需要健壮的软件来处理不正确的时钟。

问题的一部分在于不正确的时钟很容易被忽视。如果设备的CPU出现故障或者是网络配置错误，很可能设备根本无法工作，因此很快就会被注意到并被修复。另一方面，如果设备的石英时钟有缺陷或者NTP客户端配置错误，大多数情况下似乎都可以正常工作，然而它的时钟却渐渐地偏离现实。如果一些软件依赖于精确同步的时钟，结果更可能是静静地难以捉摸的数据丢失，而不是戏剧性地崩溃。

因此，如果您使用需要同步时钟的软件，则必须仔细监控所有机器之间的时钟偏移。任何时钟严重偏离其他节点的节点都应声明为失效并从群集中移除。这种监控可以确保在造成太大损失之前就注意到时钟已经坏了。

#### 用来为事件排序的时间戳

让我们考虑一个特定的情况，它对于时钟的依赖是诱人但却也危险的：跨越多个节点对事件进行排序。例如，如果两个客户端写入分布式数据库，谁先到达那里？哪一个是最新的？

图8-3显示了在具有多主机复制的数据库中对现世时钟的一种危险应用（这个例子与图5-9里的类似）。客户端A在节点1上写入*x* = 1; 写入被复制到节点3; 客户端B在节点3上对*x*加一（我们现在有*x* = 2）; 最后，这两个写入都被复制到节点2。

*图8-3 客户端B的写入在因果关系上晚于客户端A的写入，但是B的写入的时间戳更早一些。*

在图8-3中，当写入被复制到其他节点时，它会根据发生写入的节点上的现世时钟时间标记时间戳。在这个例子中时钟同步非常好：节点1和节点3之间的偏差小于3毫秒，比实践中可预期的更好。

然而图8-3中的时间戳无法正确排序事件：写入*x* = 1的时间戳为42.004秒，但写入*x* = 2的时间戳为42.003秒，即使*x* = 2明显是在稍后才发生的。当节点2收到这两个事件时，它会错误地推断出*x* = 1是更新的值而放弃写入*x* = 2。这样，客户端B的加一操作会丢失。

这种冲突解决策略被称为以最后一个写入为准（LWW），在多主机复制和无主机数据库，例如Cassandra和Riak中广泛使用（见“最后写入胜利（丢弃并发写入）”）。有些实现会在客户端而不是服务器上生成时间戳，但这不能改变LWW的本质问题：

* 数据库写入可能会神秘地消失：时钟滞后的节点没有办法覆盖先前被时钟超前的节点写入的值，直至节点间的时钟偏差被消除。这种情况会导致无数的数据被悄悄丢弃，而不会向应用程序报告任何错误。

* LWW无法区分连续快速的写入（在图8-3中，客户端B加一的动作必然发生在客户端A的写入之后）以及真正并发的写入（写入者彼此不知道对方）。为了防止破坏因果关系，需要使用额外的因果关系跟踪机制，例如版本向量（见“检测并发写入”一节）。

* 两个节点很可能独立地生成具有相同时间戳的写入，特别是当时钟仅具有毫秒级别解析度的时候。需要额外的决胜值（它可以只是一个大的随机数）来解决此类冲突，但这种方法也有可能导致违反因果关系。

因此，尽管通过保留最“新”的值而舍弃其他值来解决冲突是很有诱惑力的，但重要的是要意识到“新”的定义取决于本地的现世时钟，这很可能是不正确的。即便有着与NTP紧密同步的时钟，你也有可能在时间戳为100毫秒时（根据发送者的时钟）发送一个数据包，并在时间戳为99毫秒时（根据接收者的时钟）接收到发送的数据包——因此看起来好像数据包在它发送之前就抵达了，这是不可能的。

NTP同步是否可以足够得准确以至于不会发生这种不正确的排序？可能不会，因为NTP同步精度本身受到网络往返时间的限制，还有石英漂移等其他误差源。为了正确的排序，你需要的时钟源要比你测量的要准确得多（即网络延迟）。

基于增量计数器而不是振荡石英晶体的逻辑时钟，对于事件排序是种更安全的选择（见“检测并发写入”一节）。逻辑时钟不会测量一天中的时间或经过的秒数，而只会测量事件的相对顺序（无论一个事件发生在另一个事件之前还是之后）。相比之下，测量实际经过时间的现世时钟与单调时钟也被称为物理时钟。我们会在“顺序保证”一节中看看更多排序问题。

#### 时钟读数是有置信区间的

你可以以微秒或甚至纳秒的精度读取设备的现世时钟。但即使你可以得到如此细致的测量结果，这并不意味着这个值在这种精度上是实际精确的。事实上，它很可能不是——如之前所描述的，即使你每分钟都与本地网络上的NTP服务器同步，不精确的石英时钟导致的漂移也可能很容易达到几毫秒。对于公共互联网上的NTP服务器来说，最有可能达到的最佳精度大概是几十毫秒，而误差在网络拥塞时会轻松超过100毫秒。

因此，将时钟读数视为一个时间点就没有意义了——它更像是一个时间范围，在一个置信区间内：例如，一个系统现在的时间介于10.3秒和10.5秒之间有95%的可信度，但它并没有比这更精确的信息了。如果我们只知道正负100毫秒界别的时间，那么时间戳中的微秒位数本质上就是没有意义了。

不确定性的界限可以基于时间源进行计算。如果你的计算机上直接连接了GPS接收器或原子（铯）时钟，制造商会报告预期的误差范围。如果从服务器获取时间，则不确定性取决于自上次与服务器同步以来的预期石英漂移，再加上NTP服务器的不确定性以及到服务器的网络往返时间（第一次近似值，并假设你信任服务器）。

然而大多数系统并没有暴露这种不确定性：例如，当您调用`clock_gettime()`时，返回值没有告诉你时间戳的预期误差，因此你不知道这个置信区间是五毫秒还是五年。一个有趣的例外是Spanner中的谷歌TrueTime API，它明确报告本地时钟的置信区间。当你请求当前时间时，你会得到两个值：*`[earlist, lastest]`*，这是最早可能的时间戳和最晚可能的时间戳。根据其不确定性计算，时钟知道实际当前时间在该时间间隔内。这个间隔的宽度取决于其他因素，尤其是本地的石英钟最后一次与更准确的时钟源同步之后过了多长时间。

#### 为全局快照使用的同步时钟

在“快照隔离和可重复读取”一节中我们讨论了快照隔离，这是数据库中非常有用的功能，需要同时支持小型、快速读写事务和大型、长时间运行的只读事务（例如，用于备份或分析）。它使得只读事务可以看到在特定的时间点处于一致状态的数据库，而不会锁定数据库以及干扰其它读写事务。

快照隔离的最常见的实现需要一个单调递增的事务ID。如果写入发生的时间比快照晚（即，写入具有比快照更大的事务ID），那么写入对于快照事务是不可见的。在单节点数据库上，生成事务ID用简单的计数器就足够了。

然而当数据库分布在多台设备上，很可能位于多个数据中心内时，一个全局的、单调递增的事务ID（跨所有分区）就很难生成了，因为它需要协调。事务ID必须反映因果关系：如果事务B读取了事务A写入的值，那么B必须具有比A更大的事务ID——否则，快照就会不一致了。对于大量小规模、快速的事务，在分布式系统中创建交易ID成为难以承受的瓶颈。

我们可以使用同步之后的现世时钟时间戳作为事务ID吗？如果同步的结果足够好，他们将拥有正确的属性：后来的事务会有更大的时间戳。这个问题当然与时钟精度的不确定性有关。

Spanner就是用这种方式在数据中心之间实现了快照隔离。它使用了TrueTime API报告的时钟置信区间，并基于以下观察结果：如果您有两个置信区间，每个置信区间都包含可能的最早和最晚时间戳（*A* = [*A*<sub>earliest</sub>，*A*<sub>latest</sub>]和*B* = [*B*<sub>earliest</sub>，*B*<sub>latest</sub>]），如果这两个区间不重叠（即，*A*<sub>earliest</sub> < *A*<sub>latest</sub> < *B*<sub>earliest</sub> < *B*<sub>latest</sub>），那么B肯定发生在A之后——这是毫无疑问的。只有间隔重叠时，我们才无法确定A和B的发生顺序。

为了确保事务时间戳可以反映因果关系，Spanner在提交读写事务之前有意等待置信区间长度的时间。这样做，它可以确保任何可能读取数据的事务处于足够晚的时间，所以它们的置信区间不会重叠。为了尽可能缩短等待时间，Spanner需要保持时钟的不确定性尽可能的小; 为此，谷歌在每个数据中心都部署了GPS接收器或原子钟，把时钟同步在大约7毫秒之内。

为分布式事务语义使用时钟同步是一个活跃的研究领域。这些想法都很有趣，但是还没有在谷歌以外的主流数据库中实现。

### 进程暂停

让我们考虑另一个在分布式系统中危险地使用时钟的例子。假设你有一个数据库，每个分区只有一个主机。只允许主机接受写入。一个节点如何知道它仍然是主机（也就是它还没有被别人宣布失效），于是可以安全地接受写入？

一种选择是让主机从其他节点获取租约，就好像有超时限制的锁。任意一个时刻只有一个节点可以占有租约——因此，当节点获得租约时，它知道某段时间内它是主机，直到租约到期。为了保持主机地位，节点必须在到期前定期更新租约。如果节点失效，它将停止续租，于是另外一个节点在到期以后就可以接管。

可以想象请求处理循环看起来大概是这样的：

```JAVA
while (true) {
    request = getIncomingRequest();

    // Ensure that the lease always has at least 10 seconds remaining 
    if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000 ) {
        lease = lease.renew();
    }

    if (lease.isValid()) {
        process(request);
    }
}
```

这段代码有什么问题？首先，它依赖于同步了的时钟：租约的失效时间由另一台机器设置的（例如，可以按照当前时间加上30秒计算出到期时间），而且它会与本地的系统时钟进行比较。如果时钟不同步的程度超过了好几秒钟，这段代码将开始执行奇怪的事情。

其次，即使我们更改协议，只使用本地单调时钟，那也存在另一个问题：代码假设从检查时间（`System.currentTimeMillis()`）到请求被处理（`process(request)`），这中间只花去了很短的时间。 通常这段代码运行速度非常快，所以10秒缓冲区足以确保租期在处理请求的过程中不会过期。

Secondly, even if we change the protocol to only use the local monotonic clock, there is another problem: the code assumes that very little time passes between the point that it checks the time (System.currentTimeMillis()) and the time when the request is processed (process( request)). Normally this code runs very quickly, so the 10 second buffer is more than enough to ensure that the lease doesn’t expire in the middle of processing a request.

However, what if there is an unexpected pause in the execution of the program? For example, imagine the thread stops for 15 seconds around the line lease.isValid() before finally continuing. In that case, it’s likely that the lease will have expired by the time the request is processed, and another node has already taken over as leader. However, there is nothing to tell this thread that it was paused for so long, so this code won’t notice that the lease has expired until the next iteration of the loop — by which time it may have already done something unsafe by processing the request.

Is it crazy to assume that a thread might be paused for so long? Unfortunately not. There are various reasons why this could happen:

* Many programming language runtimes (such as the Java Virtual Machine) have a garbage collector (GC) that occasionally needs to stop all running threads. These “stop-the-world” GC pauses have sometimes been known to last for several minutes [64]! Even so-called “concurrent” garbage collectors like the HotSpot JVM’s CMS cannot fully run in parallel with the application code — even they need to stop the world from time to time [65]. Although the pauses can often be reduced by changing allocation patterns or tuning GC settings [66], we must assume the worst if we want to offer robust guarantees.

* In virtualized environments, a virtual machine can be suspended (pausing the execution of all processes and saving the contents of memory to disk) and resumed (restoring the contents of memory and continuing execution). This pause can occur at any time in a process’s execution and can last for an arbitrary length of time. This feature is sometimes used for live migration of virtual machines from one host to another without a reboot, in which case the length of the pause depends on the rate at which processes are writing to memory [67].

* On end-user devices such as laptops, execution may also be suspended and resumed arbitrarily, e.g., when the user closes the lid of their laptop.

* When the operating system context-switches to another thread, or when the hypervisor switches to a different virtual machine (when running in a virtual machine), the currently running thread can be paused at any arbitrary point in the code. In the case of a virtual machine, the CPU time spent in other virtual machines is known as steal time. If the machine is under heavy load — i.e., if there is a long queue of threads waiting to run — it may take some time before the paused thread gets to run again.

* If the application performs synchronous disk access, a thread may be paused waiting for a slow disk I/ O operation to complete [68]. In many languages, disk access can happen surprisingly, even if the code doesn’t explicitly mention file access — for example, the Java classloader lazily loads class files when they are first used, which could happen at any time in the program execution. I/ O pauses and GC pauses may even conspire to combine their delays [69]. If the disk is actually a network filesystem or network block device (such as Amazon’s EBS), the I/ O latency is further subject to the variability of network delays [29].

* If the operating system is configured to allow swapping to disk (paging), a simple memory access may result in a page fault that requires a page from disk to be loaded into memory. The thread is paused while this slow I/ O operation takes place. If memory pressure is high, this may in turn require a different page to be swapped out to disk. In extreme circumstances, the operating system may spend most of its time swapping pages in and out of memory and getting little actual work done (this is known as thrashing). To avoid this problem, paging is often disabled on server machines (if you would rather kill a process to free up memory than risk thrashing).

* A Unix process can be paused by sending it the SIGSTOP signal, for example by pressing Ctrl-Z in a shell. This signal immediately stops the process from getting any more CPU cycles until it is resumed with SIGCONT, at which point it continues running where it left off. Even if your environment does not normally use SIGSTOP, it might be sent accidentally by an operations engineer.

All of these occurrences can preempt the running thread at any point and resume it at some later time, without the thread even noticing. The problem is similar to making multi-threaded code on a single machine thread-safe: you can’t assume anything about timing, because arbitrary context switches and parallelism may occur.

When writing multi-threaded code on a single machine, we have fairly good tools for making it thread-safe: mutexes, semaphores, atomic counters, lock-free data structures, blocking queues, and so on. Unfortunately, these tools don’t directly translate to distributed systems, because a distributed system has no shared memory — only messages sent over an unreliable network.

A node in a distributed system must assume that its execution can be paused for a significant length of time at any point, even in the middle of a function. During the pause, the rest of the world keeps moving and may even declare the paused node dead because it’s not responding. Eventually, the paused node may continue running, without even noticing that it was asleep until it checks its clock sometime later.

#### Response time guarantees

In many programming languages and operating systems, threads and processes may pause for an unbounded amount of time, as discussed. Those reasons for pausing can be eliminated if you try hard enough. 

Some software runs in environments where a failure to respond within a specified time can cause serious damage: computers that control aircraft, rockets, robots, cars, and other physical objects must respond quickly and predictably to their sensor inputs. In these systems, there is a specified deadline by which the software must respond; if it doesn’t meet the deadline, that may cause a failure of the entire system. These are so-called hard real-time systems.

> Is real-time really real?
>
> In embedded systems, real-time means that a system is carefully designed and tested to meet specified timing guarantees in all circumstances. This meaning is in contrast to the more vague use of the term real-time on the web, where it describes servers pushing data to clients and stream processing without hard response time constraints (see Chapter   11).

For example, if your car’s onboard sensors detect that you are currently experiencing a crash, you wouldn’t want the release of the airbag to be delayed due to an inopportune GC pause in the airbag release system.

Providing real-time guarantees in a system requires support from all levels of the software stack: a real-time operating system (RTOS) that allows processes to be scheduled with a guaranteed allocation of CPU time in specified intervals is needed; library functions must document their worst-case execution times; dynamic memory allocation may be restricted or disallowed entirely (real-time garbage collectors exist, but the application must still ensure that it doesn’t give the GC too much work to do); and an enormous amount of testing and measurement must be done to ensure that guarantees are being met.

All of this requires a large amount of additional work and severely restricts the range of programming languages, libraries, and tools that can be used (since most languages and tools do not provide real-time guarantees). For these reasons, developing real-time systems is very expensive, and they are most commonly used in safety-critical embedded devices. Moreover, “real-time” is not the same as “high-performance” — in fact, real-time systems may have lower throughput, since they have to prioritize timely responses above all else (see also “Latency and Resource Utilization”).

For most server-side data processing systems, real-time guarantees are simply not economical or appropriate. Consequently, these systems must suffer the pauses and clock instability that come from operating in a non-real-time environment.

#### Limiting the impact of garbage collection

The negative effects of process pauses can be mitigated without resorting to expensive real-time scheduling guarantees. Language runtimes have some flexibility around when they schedule garbage collections, because they can track the rate of object allocation and the remaining free memory over time.

An emerging idea is to treat GC pauses like brief planned outages of a node, and to let other nodes handle requests from clients while one node is collecting its garbage. If the runtime can warn the application that a node soon requires a GC pause, the application can stop sending new requests to that node, wait for it to finish processing outstanding requests, and then perform the GC while no requests are in progress. This trick hides GC pauses from clients and reduces the high percentiles of response time [70, 71]. Some latency-sensitive financial trading systems [72] use this approach.

A variant of this idea is to use the garbage collector only for short-lived objects (which are fast to collect) and to restart processes periodically, before they accumulate enough long-lived objects to require a full GC of long-lived objects [65, 73]. One node can be restarted at a time, and traffic can be shifted away from the node before the planned restart, like in a rolling upgrade (see Chapter   4).

These measures cannot fully prevent garbage collection pauses, but they can usefully reduce their impact on the application.