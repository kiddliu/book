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

Clocks and time are important. Applications depend on clocks in various ways to answer questions like the following: 

1. Has this request timed out yet?

2. What’s the 99th percentile response time of this service?

3. How many queries per second did this service handle on average in the last five minutes?

4. How long did the user spend on our site?

5. When was this article published?

6. At what date and time should the reminder email be sent?

7. When does this cache entry expire?

8. What is the timestamp on this error message in the log file?

Examples 1– 4 measure durations (e.g., the time interval between a request being sent and a response being received), whereas examples 5– 8 describe points in time (events that occur on a particular date, at a particular time).

In a distributed system, time is a tricky business, because communication is not instantaneous: it takes time for a message to travel across the network from one machine to another. The time when a message is received is always later than the time when it is sent, but due to variable delays in the network, we don’t know how much later. This fact sometimes makes it difficult to determine the order in which things happened when multiple machines are involved.

Moreover, each machine on the network has its own clock, which is an actual hardware device: usually a quartz crystal oscillator. These devices are not perfectly accurate, so each machine has its own notion of time, which may be slightly faster or slower than on other machines. It is possible to synchronize clocks to some degree: the most commonly used mechanism is the Network Time Protocol (NTP), which allows the computer clock to be adjusted according to the time reported by a group of servers [37]. The servers in turn get their time from a more accurate time source, such as a GPS receiver.

### Monotonic Versus Time-of-Day Clocks

Modern computers have at least two different kinds of clocks: a time-of-day clock and a monotonic clock. Although they both measure time, it is important to distinguish the two, since they serve different purposes.

#### Time-of-day clocks

A time-of-day clock does what you intuitively expect of a clock: it returns the current date and time according to some calendar (also known as wall-clock time). For example, clock_gettime( CLOCK_REALTIME) on Linuxv and System.currentTimeMillis() in Java return the number of seconds (or milliseconds) since the epoch: midnight UTC on January 1, 1970, according to the Gregorian calendar, not counting leap seconds. Some systems use other dates as their reference point.

Time-of-day clocks are usually synchronized with NTP, which means that a timestamp from one machine (ideally) means the same as a timestamp on another machine. However, time-of-day clocks also have various oddities, as described in the next section. In particular, if the local clock is too far ahead of the NTP server, it may be forcibly reset and appear to jump back to a previous point in time. These jumps, as well as the fact that they often ignore leap seconds, make time-of-day clocks unsuitable for measuring elapsed time [38].

Time-of-day clocks have also historically had quite a coarse-grained resolution, e.g., moving forward in steps of 10   ms on older Windows systems [39]. On recent systems, this is less of a problem.

#### Monotonic clocks

A monotonic clock is suitable for measuring a duration (time interval), such as a timeout or a service’s response time: clock_gettime( CLOCK_MONOTONIC) on Linux and System.nanoTime() in Java are monotonic clocks, for example. The name comes from the fact that they are guaranteed to always move forward (whereas a time-of-day clock may jump back in time). You can check the value of the monotonic clock at one point in time, do something, and then check the clock again at a later time. The difference between the two values tells you how much time elapsed between the two checks. However, the absolute value of the clock is meaningless: it might be the number of nanoseconds since the computer was started, or something similarly arbitrary. In particular, it makes no sense to compare monotonic clock values from two different computers, because they don’t mean the same thing.

On a server with multiple CPU sockets, there may be a separate timer per CPU, which is not necessarily synchronized with other CPUs. Operating systems compensate for any discrepancy and try to present a monotonic view of the clock to application threads, even as they are scheduled across different CPUs. However, it is wise to take this guarantee of monotonicity with a pinch of salt [40].

NTP may adjust the frequency at which the monotonic clock moves forward (this is known as slewing the clock) if it detects that the computer’s local quartz is moving faster or slower than the NTP server. By default, NTP allows the clock rate to be speeded up or slowed down by up to 0.05%, but NTP cannot cause the monotonic clock to jump forward or backward. The resolution of monotonic clocks is usually quite good: on most systems they can measure time intervals in microseconds or less.

In a distributed system, using a monotonic clock for measuring elapsed time (e.g., timeouts) is usually fine, because it doesn’t assume any synchronization between different nodes’ clocks and is not sensitive to slight inaccuracies of measurement.

### Clock Synchronization and Accuracy

Monotonic clocks don’t need synchronization, but time-of-day clocks need to be set according to an NTP server or other external time source in order to be useful. Unfortunately, our methods for getting a clock to tell the correct time aren’t nearly as reliable or accurate as you might hope — hardware clocks and NTP can be fickle beasts.

To give just a few examples:

* The quartz clock in a computer is not very accurate: it drifts (runs faster or slower than it should). Clock drift varies depending on the temperature of the machine. Google assumes a clock drift of 200   ppm (parts per million) for its servers [41], which is equivalent to 6   ms drift for a clock that is resynchronized with a server every 30 seconds, or 17 seconds drift for a clock that is resynchronized once a day. This drift limits the best possible accuracy you can achieve, even if everything is working correctly.

* If a computer’s clock differs too much from an NTP server, it may refuse to synchronize, or the local clock will be forcibly reset [37]. Any applications observing the time before and after this reset may see time go backward or suddenly jump forward.

* If a node is accidentally firewalled off from NTP servers, the misconfiguration may go unnoticed for some time. Anecdotal evidence suggests that this does happen in practice.

* NTP synchronization can only be as good as the network delay, so there is a limit to its accuracy when you’re on a congested network with variable packet delays. One experiment showed that a minimum error of 35   ms is achievable when synchronizing over the internet [42], though occasional spikes in network delay lead to errors of around a second. Depending on the configuration, large network delays can cause the NTP client to give up entirely.

* Some NTP servers are wrong or misconfigured, reporting time that is off by hours [43, 44]. NTP clients are quite robust, because they query several servers and ignore outliers. Nevertheless, it’s somewhat worrying to bet the correctness of your systems on the time that you were told by a stranger on the internet.

* Leap seconds result in a minute that is 59 seconds or 61 seconds long, which messes up timing assumptions in systems that are not designed with leap seconds in mind [45]. The fact that leap seconds have crashed many large systems [38, 46] shows how easy it is for incorrect assumptions about clocks to sneak into a system. The best way of handling leap seconds may be to make NTP servers “lie,” by performing the leap second adjustment gradually over the course of a day (this is known as smearing) [47, 48], although actual NTP server behavior varies in practice [49].

* In virtual machines, the hardware clock is virtualized, which raises additional challenges for applications that need accurate timekeeping [50]. When a CPU core is shared between virtual machines, each VM is paused for tens of milliseconds while another VM is running. From an application’s point of view, this pause manifests itself as the clock suddenly jumping forward [26].

* If you run software on devices that you don’t fully control (e.g., mobile or embedded devices), you probably cannot trust the device’s hardware clock at all. Some users deliberately set their hardware clock to an incorrect date and time, for example to circumvent timing limitations in games. As a result, the clock might be set to a time wildly in the past or the future.

It is possible to achieve very good clock accuracy if you care about it sufficiently to invest significant resources. For example, the MiFID II draft European regulation for financial institutions requires all high-frequency trading funds to synchronize their clocks to within 100 microseconds of UTC, in order to help debug market anomalies such as “flash crashes” and to help detect market manipulation [51].

Such accuracy can be achieved using GPS receivers, the Precision Time Protocol (PTP) [52], and careful deployment and monitoring. However, it requires significant effort and expertise, and there are plenty of ways clock synchronization can go wrong. If your NTP daemon is misconfigured, or a firewall is blocking NTP traffic, the clock error due to drift can quickly become large.

### Relying on Synchronized Clocks

The problem with clocks is that while they seem simple and easy to use, they have a surprising number of pitfalls: a day may not have exactly 86,400 seconds, time-of-day clocks may move backward in time, and the time on one node may be quite different from the time on another node.

Earlier in this chapter we discussed networks dropping and arbitrarily delaying packets. Even though networks are well behaved most of the time, software must be designed on the assumption that the network will occasionally be faulty, and the software must handle such faults gracefully. The same is true with clocks: although they work quite well most of the time, robust software needs to be prepared to deal with incorrect clocks.

Part of the problem is that incorrect clocks easily go unnoticed. If a machine’s CPU is defective or its network is misconfigured, it most likely won’t work at all, so it will quickly be noticed and fixed. On the other hand, if its quartz clock is defective or its NTP client is misconfigured, most things will seem to work fine, even though its clock gradually drifts further and further away from reality. If some piece of software is relying on an accurately synchronized clock, the result is more likely to be silent and subtle data loss than a dramatic crash [53, 54].

Thus, if you use software that requires synchronized clocks, it is essential that you also carefully monitor the clock offsets between all the machines. Any node whose clock drifts too far from the others should be declared dead and removed from the cluster. Such monitoring ensures that you notice the broken clocks before they can cause too much damage.

#### Timestamps for ordering events

Let’s consider one particular situation in which it is tempting, but dangerous, to rely on clocks: ordering of events across multiple nodes. For example, if two clients write to a distributed database, who got there first? Which write is the more recent one?

Figure   8-3 illustrates a dangerous use of time-of-day clocks in a database with multi-leader replication (the example is similar to Figure   5-9). Client A writes x   =   1 on node 1; the write is replicated to node 3; client B increments x on node 3 (we now have x   =   2); and finally, both writes are replicated to node 2.

*Figure 8-3. The write by client B is causally later than the write by client A, but B’s write has an earlier timestamp.*

In Figure   8-3, when a write is replicated to other nodes, it is tagged with a timestamp according to the time-of-day clock on the node where the write originated. The clock synchronization is very good in this example: the skew between node 1 and node 3 is less than 3   ms, which is probably better than you can expect in practice. 

Nevertheless, the timestamps in Figure   8-3 fail to order the events correctly: the write x   =   1 has a timestamp of 42.004 seconds, but the write x   =   2 has a timestamp of 42.003 seconds, even though x   =   2 occurred unambiguously later. When node 2 receives these two events, it will incorrectly conclude that x   =   1 is the more recent value and drop the write x   =   2. In effect, client B’s increment operation will be lost.

This conflict resolution strategy is called last write wins (LWW), and it is widely used in both multi-leader replication and leaderless databases such as Cassandra [53] and Riak [54] (see “Last write wins (discarding concurrent writes)”). Some implementations generate timestamps on the client rather than the server, but this doesn’t change the fundamental problems with LWW:

* Database writes can mysteriously disappear: a node with a lagging clock is unable to overwrite values previously written by a node with a fast clock until the clock skew between the nodes has elapsed [54, 55]. This scenario can cause arbitrary amounts of data to be silently dropped without any error being reported to the application.

* LWW cannot distinguish between writes that occurred sequentially in quick succession (in Figure   8-3, client B’s increment definitely occurs after client A’s write) and writes that were truly concurrent (neither writer was aware of the other). Additional causality tracking mechanisms, such as version vectors, are needed in order to prevent violations of causality (see “Detecting Concurrent Writes”).
* It is possible for two nodes to independently generate writes with the same timestamp, especially when the clock only has millisecond resolution. An additional tiebreaker value (which can simply be a large random number) is required to resolve such conflicts, but this approach can also lead to violations of causality [53].

Thus, even though it is tempting to resolve conflicts by keeping the most “recent” value and discarding others, it’s important to be aware that the definition of “recent” depends on a local time-of-day clock, which may well be incorrect. Even with tightly NTP-synchronized clocks, you could send a packet at timestamp 100   ms (according to the sender’s clock) and have it arrive at timestamp 99   ms (according to the recipient’s clock) — so it appears as though the packet arrived before it was sent, which is impossible.

Could NTP synchronization be made accurate enough that such incorrect orderings cannot occur? Probably not, because NTP’s synchronization accuracy is itself limited by the network round-trip time, in addition to other sources of error such as quartz drift. For correct ordering, you would need the clock source to be significantly more accurate than the thing you are measuring (namely network delay).

So-called logical clocks [56, 57], which are based on incrementing counters rather than an oscillating quartz crystal, are a safer alternative for ordering events (see “Detecting Concurrent Writes”). Logical clocks do not measure the time of day or the number of seconds elapsed, only the relative ordering of events (whether one event happened before or after another). In contrast, time-of-day and monotonic clocks, which measure actual elapsed time, are also known as physical clocks. We’ll look at ordering a bit more in “Ordering Guarantees”.

Clock readings have a confidence interval

You may be able to read a machine’s time-of-day clock with microsecond or even nanosecond resolution. But even if you can get such a fine-grained measurement, that doesn’t mean the value is actually accurate to such precision. In fact, it most likely is not — as mentioned previously, the drift in an imprecise quartz clock can easily be several milliseconds, even if you synchronize with an NTP server on the local network every minute. With an NTP server on the public internet, the best possible accuracy is probably to the tens of milliseconds, and the error may easily spike to over 100 ms when there is network congestion [57].

Thus, it doesn’t make sense to think of a clock reading as a point in time — it is more like a range of times, within a confidence interval: for example, a system may be 95% confident that the time now is between 10.3 and 10.5 seconds past the minute, but it doesn’t know any more precisely than that [58]. If we only know the time +/–   100   ms, the microsecond digits in the timestamp are essentially meaningless.

The uncertainty bound can be calculated based on your time source. If you have a GPS receiver or atomic (caesium) clock directly attached to your computer, the expected error range is reported by the manufacturer. If you’re getting the time from a server, the uncertainty is based on the expected quartz drift since your last sync with the server, plus the NTP server’s uncertainty, plus the network round-trip time to the server (to a first approximation, and assuming you trust the server).

Unfortunately, most systems don’t expose this uncertainty: for example, when you call clock_gettime(), the return value doesn’t tell you the expected error of the timestamp, so you don’t know if its confidence interval is five milliseconds or five years. An interesting exception is Google’s TrueTime API in Spanner [41], which explicitly reports the confidence interval on the local clock. When you ask it for the current time, you get back two values: [earliest, latest], which are the earliest possible and the latest possible timestamp. Based on its uncertainty calculations, the clock knows that the actual current time is somewhere within that interval. The width of the interval depends, among other things, on how long it has been since the local quartz clock was last synchronized with a more accurate clock source.

#### Synchronized clocks for global snapshots

In “Snapshot Isolation and Repeatable Read” we discussed snapshot isolation, which is a very useful feature in databases that need to support both small, fast read-write transactions and large, long-running read-only transactions (e.g., for backups or analytics). It allows read-only transactions to see the database in a consistent state at a particular point in time, without locking and interfering with read-write transactions.

The most common implementation of snapshot isolation requires a monotonically increasing transaction ID. If a write happened later than the snapshot (i.e., the write has a greater transaction ID than the snapshot), that write is invisible to the snapshot transaction. On a single-node database, a simple counter is sufficient for generating transaction IDs.

However, when a database is distributed across many machines, potentially in multiple datacenters, a global, monotonically increasing transaction ID (across all partitions) is difficult to generate, because it requires coordination. The transaction ID must reflect causality: if transaction B reads a value that was written by transaction A, then B must have a higher transaction ID than A — otherwise, the snapshot would not be consistent. With lots of small, rapid transactions, creating transaction IDs in a distributed system becomes an untenable bottleneck.vi

Can we use the timestamps from synchronized time-of-day clocks as transaction IDs? If we could get the synchronization good enough, they would have the right properties: later transactions have a higher timestamp. The problem, of course, is the uncertainty about clock accuracy.

Spanner implements snapshot isolation across datacenters in this way [59, 60]. It uses the clock’s confidence interval as reported by the TrueTime API, and is based on the following observation: if you have two confidence intervals, each consisting of an earliest and latest possible timestamp (A = [Aearliest, Alatest] and B = [Bearliest, Blatest]), and those two intervals do not overlap (i.e., Aearliest < Alatest < Bearliest < Blatest), then B definitely happened after A — there can be no doubt. Only if the intervals overlap are we unsure in which order A and B happened.

In order to ensure that transaction timestamps reflect causality, Spanner deliberately waits for the length of the confidence interval before committing a read-write transaction. By doing so, it ensures that any transaction that may read the data is at a sufficiently later time, so their confidence intervals do not overlap. In order to keep the wait time as short as possible, Spanner needs to keep the clock uncertainty as small as possible; for this purpose, Google deploys a GPS receiver or atomic clock in each datacenter, allowing clocks to be synchronized to within about 7   ms [41].

Using clock synchronization for distributed transaction semantics is an area of active research [57, 61, 62]. These ideas are interesting, but they have not yet been implemented in mainstream databases outside of Google.