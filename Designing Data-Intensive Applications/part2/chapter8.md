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

然而，如果在执行程序时出现意外停顿该怎么办？例如，假设线程在`lease.isValid()`这一行周围停止15秒然后才终于继续执行。在这种情况下，很可能在处理请求时租约已经过期，而另一个节点已经接管为领导者。 然而，没有什么可以告诉这个线程它已经暂停了这么久了，所以这段代码直到循环的下一次迭代之前都不会注意到租约已经过期——这时通过处理请求它可能已经做了一些不安全的事。

假设某个线程可能会暂停很长时间是疯了吗？然而并不是。它可能发生的原因有很多种：

* 许多编程语言运行时（比如Java虚拟机）都有一个垃圾回收器（GC），它偶尔需要停止所有正在运行的线程。这些“停止一切”的GC暂停有时会持续几分钟！即使像HotSpot JVM中CMS那样的所谓的“并发”垃圾回收器也不能完全与应用程序代码并行运行——即使它们需要时不时地停止所有事。尽管通常可以通过改变分配模式或调整GC设置来减少暂停，但如果我们想要提供健壮性保证，就必须假设最差的场景。

* 在虚拟化环境中，虚拟机可以被暂停（暂停执行所有进程并将内存内容保存到磁盘）然后被恢复（恢复内存内容并继续执行）。这个暂停可以发生在进程执行的任何时间，并且可以持续任意长度的时间。这个功能有时用于将虚拟机从一台主机实时迁移到另一台主机而无需重新启动，在这种情况下，暂停的长度取决于进程写入内存的速率。

* 在类似笔记本电脑的终端用户设备上，执行也会被任意地暂停和恢复，例如，当用户合上他们的笔记本电脑的盖子时。

* 当操作系统上下文切换到另一个线程时，或者当管理程序切换到其他虚拟机时（在虚拟机中运行时），当前正在运行的线程可以在代码中的任意点暂停。在虚拟机的情况下，在其他虚拟机中花费的CPU时间被称为窃取时间。如果机器负载很重——即，如果有很长的线程队列等待运行——可能需要一些时间才能使暂停的线程再次运行。

* 如果应用程序执行同步的磁盘访问，则线程可能会暂停而等待慢速磁盘I/O操作完成。在许多语言中，即使代码没有明确提到文件访问，磁盘访问也会出人意料地发生——例如，Java类加载器会延迟加载类文件，直至类第一次被使用到的时候，而这可能会在程序执行过程中任意时刻发生。I/O暂停和GC暂停甚至可能同时发生从而它们的延迟结合到了一起。如果磁盘实际上是一个网络文件系统或网络块设备（比如亚马逊的EBS），那么I/O延迟还会进一步受到网络延迟变化的影响。

* 如果操作系统被配置为允许内存交换到磁盘（分页），则简单的内存访问可能会导致页面错误，需要将磁盘中的页面加载到内存中。这个缓慢的I/O操作发生时线程被暂停了。如果内存压力很高，这可能又需要将另一个的页面换出到磁盘。在极端的情况下，操作系统可能会花费大部分时间将页面换入换出内存，从而只做了很少的实际工作（这被称为系统颠簸）。为了避免这个问题，通常在服务器机器上禁用分页（如果你宁愿杀死一个进程来释放内存，也不愿意系统颠簸）。

* 可以通过发送SIGSTOP信号来暂停Unix进程，例如通过在shell中按下Ctrl-Z。该信号立即停止进程，防止获得更多的CPU周期直到使用SIGCONT恢复它，此时它会继续从上一次暂停的地方执行。即使您的环境通常不使用SIGSTOP，也可能由运营工程师意外发送。

所有这些事件都可以在任何时候抢占正在运行的线程，并在稍后继续运行，而线程甚至不会注意到它。这个问题类似于使多线程代码在单个机器上线程安全：你不能假定任何关于时序的事情，因为会发生任意的上下文切换和并行计算。

在单个机器上编写多线程代码时，我们有相当不错的工具来实现线程安全：互斥量，信号量，原子计数器，无锁数据结构，阻塞队列等等。然而，这些工具不能直接用于分布式系统，因为分布式系统没有共享内存——只有通过不可靠网络发送的消息。

分布式系统中的节点必须假定它的执行过程可以在任何时刻暂停很长一段时间，即使在函数中间也是如此。 在暂停期间，其他系统一直在执行，甚至可能因为没有响应而宣布已暂停的节点失效。最终，暂停的节点可能会继续运行，甚至没有注意到之前一直处于睡眠状态，直到后来检查时钟的时候。

#### 相应时间保证

在许多编程语言与操作系统中，我们讨论过，线程与进程可能暂停无限长的时间。如果足够努力，这些导致暂停的原因都可以消除。

一些软件运行在指定时间内未能响应可能导致严重后果的环境中：控制飞机，火箭，机器人，汽车和其他物体的计算机必须对其传感器输入做出快速和可预测的响应。在这些系统中，软件必须在特定期限之前响应；如果超出了期限，可能会导致整个系统的故障。这些都是所谓的硬实时系统。

> 实时是真的么？
>
> 在嵌入式系统中，实时意味着系统是经过仔细设计和测试的以满足所有场景下对时间的特定保证。这个含义与Web上实时术语的模糊使用相反，它描述了服务器将数据推送到客户端以及流处理而没有严格的响应时间限制（见第11章）。

例如，如果汽车车载传感器检测到您目前正在经历碰撞，你肯定不希望系统由于不合时宜的GC暂停导致安全气囊延迟弹出。

在系统中提供实时保证需要软件堆栈在每个级别的支持：需要一个实时操作系统（RTOS），它允许在指定的时间间隔内为线程分配CPU时间; 库函数必须记录最坏情况下的执行时间; 动态内存分配会受到限制或完全不允许（实时垃圾收集器存在，但应用程序仍必须确保它不会给GC太多工作）; 并且必须进行大量的测试和测量以确保达到了保证条件。

所有这些都需要大量的额外工作，并且严重限制了可以使用的编程语言、库以及工具的范围（因为大多数语言和工具不提供实时性保证）。由于这些原因，开发实时系统非常昂贵，并且它们最常用于安全关键的嵌入式设备。此外，“实时”与“高性能”不是一回事——实际上，实时系统的吞吐量可能比较低，因为它们必须把及时响应优先于其它一切（见“延迟和资源利用率”一节）。

对于大多数服务器端数据处理系统来说，实时保证既不经济也不合适。因此，这些系统必须承受在非实时环境中运行时产生的暂停和时钟不稳定。

#### 限制垃圾回收机制的影响

进程暂停产生的负面影响可以在不使用昂贵的实时调度情况下就得以缓解。编程语言运行时在计划垃圾收集时有一定的灵活性，因为它们可以随着时间的推移，跟踪对象分配的速率以及剩余的可用内存容量。

一种新理念是把GC暂停当作节点计划内的短暂离线，而在一个节点收集垃圾的同时让其他节点处理来自客户端的请求。如果运行时可以警告应用程序节点很快需要GC暂停，应用程序可以停止向该节点发送新请求，等待它完成处理未完成的请求，然后在没有请求正在进行的情况下执行GC。这个技巧隐藏了来自客户端的GC暂停并降低了高百分比位的响应时间。一些对延迟敏感的金融交易系统使用这种方法。

这个理念的一个变种是只对短期对象（收集它们很快）上使用垃圾收集器，而在积累了足够多长期对象时不触发GC，而是定期重启进程。一次可以重启一个节点，并且可以在计划重新启动之前将流量从节点导开，就像滚动升级一样（见第4章）。

这些措施不能完全防止垃圾收集暂停，但它们可以有效地降低它们对应用程序的影响。

## 知识，真想与谎言

本章到目前为止我们已经探索了分布式系统与运行在单台计算机上的程序不同的各个方式：没有共享内存，只有通过不可靠网络传递的、具有可变延迟的消息，系统还要忍受来自部分失效、不可靠的时钟和处理暂停问题。

如果您不习惯于分布式系统，这些问题导致的后果会使你陷入极度混乱的状态。网络中的节点无法确切知道任何事情——它只能根据通过网络接收（或未接收）到的消息进行猜测。节点只能通过交换消息来找出另一个节点所处的状态（它存储了哪些数据，是否正常工作等）。如果远程节点没有响应，则无法知道它处于什么状态，因为没有办法可靠地将网络中的问题与节点上的问题区分开来。

关于这些系统的讨论落入了哲学范畴：在我们的系统中我们知道的何为真何为假？如果知觉和测量的机制不可靠，我们对这些知识有多少信心？软件系统应该遵守我们对物理世界的期望，例如因果关系吗？

幸运的是，我们不需要进一步去理解生命的意义。在分布式系统中，我们可以描述我们对行为（系统模型）的假设，并以满足这些假设的方式为目标设计实际的系统。算法可以被证明在某个系统模型中正确运行。这意味着即使底层系统模型只提供了极少的保证，可靠的行为也时可以实现的。

然而，尽管可以使软件在不可靠的系统模型中表现良好，但这并不容易。在本章的其余部分中，我们会进一步探讨分布式系统中知识和真相的观念，这将有助于我们思考我们可以做出的各种假设以及我们可能希望提供的保证。在第9章中，我们将继续研究一些分布式系统的例子，以及在特定假设下提供特殊保证的算法。

### 真理是由多数人定义的

想象一个具有不对称故障的网络：节点能够接收发送给它的所有消息，但是从该节点发出的任何消息都被丢弃或延迟了。即使节点工作得很好，并且正在接收来自其他节点的请求，但是其他节点无法接收到它的响应。在超时之后，其他节点声明它已失效，因为它们没有从节点收到任何反馈。情况像一场噩梦般展开：半断开的节点被拖到墓地，脚在乱踢、惊声尖叫：“我还没死！”——但没有人能听到它的尖叫声，葬礼游行队伍继续肃穆的前行。

在稍微不那么噩梦的场景中，半断开的节点可能注意到它正在发送的消息没有被其他节点确认，于是意识到网络中肯定发生故障了。然而，节点被其他节点错误地宣告失效，而半断开的节点对此什么都做不了。

作为第三种情况，想象一个节点经历了一个长时间的完全停止的垃圾收集暂停。所有节点的线程都被GC抢占了，暂停了一分钟，因此不处理任何请求，也不发送任何响应。其他节点等待，重试，开始变得不耐烦，最终宣布节点失效并将其运到灵车上。最后，GC完成，节点的所有线程继续进行，就好像什么都没有发生一样。其他节点感到惊讶的是，已经失效的节点突然从棺材里抬起头来，完全健康，并开始愉快地与旁观者聊天。一开始，正在GC的节点甚至没有意识到整整一分钟已经过去了，并因此被宣布失效——从它的角度来看，自从最后一次与其他节点交谈以来时间几乎就没走啊。

这些故事的寓意是，节点不一定可以相信自己对形势的判断。分布式系统不能完全依赖于单个节点，因为节点在任何时间都可能失效，可能使得系统卡顿且无法恢复。相反，许多分布式算法依赖于仲裁，即节点之间投票(见“读写仲裁”一节)：决策需要来自几个节点的某个最小票数，从而减少对任何一个特定节点的依赖。

决策包括声明节点失效的决定。如果一组节点声明另一个节点失效了，那么它必须被认为是失效的，即使这个节点仍然感觉可以正常工作。单个节点必须遵守仲裁结果，然后离线。

最常见的是，仲裁票数超过一半节点，是绝对多数(尽管其他类型的仲裁也是可能的)。多数仲裁允许系统在单个节点出现故障时继续工作(有三个节点，可以容忍其中一个失效；对于五个节点，可以容忍其中两个失效)。然而，这仍然是安全的，因为系统中只能有一个多数——不可能出现有两个多数同时又有相互冲突的决定。在第九章讨论一致算法时，我们会更详细地讨论仲裁的使用。

#### 主机与锁

很多时候，系统要求某些东西只能有一个在其中。比如：

* 数据库分区的主机只允许有一个，以防止裂脑（见“处理节点离线”一节）。

* 特定资源或对象的锁只允许一个事务或客户端持有，以防止对它并发写入、数据破坏。

* 每个用户名只允许一个用户注册，因为用户名必须唯一标识一个用户。

在分布式系统中实现它们需要很小心：即使节点相信自己就是“天选者”（分区中的主机，锁的拥有者，用户成功抢注了用户名的请求处理逻辑），也并不意味着众多节点组成的仲裁团同意！节点也许之前当过主机，但是如果其它节点同时宣布它失效（比如，因为网络中断或者垃圾回收暂停），它很有可能被降级，而另一个主机也许已经被选出了。

如果节点继续以天选者的姿态行动，即使大多数节点已经宣布它失效，它会在那些没有仔细设计的系统中导致问题。这样的节点可以以自定义的容量向其它节点发送消息，那么如果其它节点选择相信它的话，系统作为一个整体可能会做一些不正确的事。

比如，图8-4展示了由于错误的锁定实现导致的数据破坏bug。（这个bug不只是理论性的，HBase过去有这样的问题。）假设你想要保证存储服务中的文件一次只可以被一个客户端访问，因为如果多个客户端尝试写入到它的话，文件数据会被破坏。你尝试通过要求客户端在访问文件之前必须从锁定服务处获取租约来实现它。

*图8-4 分布式锁不正确的实现：客户端1相信它仍然持有有效的租约，即使事实上已经过期了，并因此破坏了存储中的文件数据。*

这个问题就是我们在“进程暂停”一节讨论过的一个例子：如果持有租约的客户端暂停了太长时间，租约过期。另一个客户端可以获取同一个文件的租约，并开始写入这个问题。当客户端从暂停中恢复，它（错误地）相信自己有合法的租约并且也继续向那个文件写入数据。结果是，客户端的写入冲突，文件被破坏了。

#### 栅栏令牌

当使用锁或者租约来保护对某个资源的访问时，例如图8-4中的文件存储，我们需要确保错以为自己是“天选者”的节点不能破坏系统的其余部分。一种实现这个目标相当简单的技术叫做栅栏，如图8-5所示。

*图8-5 通过只允许以提升栅栏令牌的顺序的写入，使得对存储的访问变得安全*

让我们假设每一次锁服务器授出一个锁或一份租约，也同时返回一个栅栏令牌，它是一个数字，每一次锁被授出数字都会增加（比如，通过锁服务每次加一）。这样我们可以要求每一次客户端发送写入请求到存储服务时，请求都必须包含当前的栅栏令牌。

在图8-5中，客户端1获取了租约，令牌为33，但是之后它进入了一个长暂停于是租约过期了。客户端2获取了租约，令牌34（数字一致增加），然后发送它的写入请求到存储服务，其中包含34这个令牌。之后，客户端1恢复并发送它的请求到存储服务，其中包含令牌值33。然而，存储服务器记得它已经处理了附带更高令牌数字（34）的写入请求，于是拒绝了有令牌33的请求。

如果使用ZooKeeper作为锁服务，那么事务ID`zxid`或者节点版本`cversion`可以用作栅栏令牌。由于它们保证为单调增的，它们满足要求的条件。

值得注意的是，这种机制要求资源自己在检查令牌时，主动拒绝任何有着比已经处理的请求更老的令牌的写入请求——依赖客户端自己检查锁状态是不足够的。对于不明确支持栅栏令牌的资源，您大概仍然可以绕过此限制（例如，在文件存储服务的场景下，可以将栅栏令牌包含在文件名中）。然而，为了避免在锁的保护之外处理请求，某些检查是必要的。

在服务器端检查令牌看起来像是一个缺点，但它可以说是一件好事：对于服务来说，假定客户端总是表现正常并不明智，因为运行客户端的人们的优先级与运行服务的人们的优先级是非常不同的。因此，对于任何服务来说，保护自己免受意外滥用的客户端是个好主意。

### Byzantine Faults

Fencing tokens can detect and block a node that is inadvertently acting in error (e.g., because it hasn’t yet found out that its lease has expired). However, if the node deliberately wanted to subvert the system’s guarantees, it could easily do so by sending messages with a fake fencing token.

In this book we assume that nodes are unreliable but honest: they may be slow or never respond (due to a fault), and their state may be outdated (due to a GC pause or network delays), but we assume that if a node does respond, it is telling the “truth”: to the best of its knowledge, it is playing by the rules of the protocol.

Distributed systems problems become much harder if there is a risk that nodes may “lie” (send arbitrary faulty or corrupted responses) — for example, if a node may claim to have received a particular message when in fact it didn’t. Such behavior is known as a Byzantine fault, and the problem of reaching consensus in this untrusting environment is known as the Byzantine Generals Problem [77].

> The Byzantine Generals Problem
>
> The Byzantine Generals Problem is a generalization of the so-called Two Generals Problem [78], which imagines a situation in which two army generals need to agree on a battle plan. As they have set up camp on two different sites, they can only communicate by messenger, and the messengers sometimes get delayed or lost (like packets in a network). We will discuss this problem of consensus in Chapter   9.
>
> In the Byzantine version of the problem, there are n generals who need to agree, and their endeavor is hampered by the fact that there are some traitors in their midst. Most of the generals are loyal, and thus send truthful messages, but the traitors may try to deceive and confuse the others by sending fake or untrue messages (while trying to remain undiscovered). It is not known in advance who the traitors are. >
>
> Byzantium was an ancient Greek city that later became Constantinople, in the place which is now Istanbul in Turkey. There isn’t any historic evidence that the generals of Byzantium were any more prone to intrigue and conspiracy than those elsewhere. Rather, the name is derived from Byzantine in the sense of excessively complicated, bureaucratic, devious, which was used in politics long before computers [79]. Lamport wanted to choose a nationality that would not offend any readers, and he was advised that calling it The Albanian Generals Problem was not such a good idea [80].

A system is Byzantine fault-tolerant if it continues to operate correctly even if some of the nodes are malfunctioning and not obeying the protocol, or if malicious attackers are interfering with the network. This concern is relevant in certain specific circumstances. For example:

* In aerospace environments, the data in a computer’s memory or CPU register could become corrupted by radiation, leading it to respond to other nodes in arbitrarily unpredictable ways. Since a system failure would be very expensive (e.g., an aircraft crashing and killing everyone on board, or a rocket colliding with the International Space Station), flight control systems must tolerate Byzantine faults [81, 82].

* In a system with multiple participating organizations, some participants may attempt to cheat or defraud others. In such circumstances, it is not safe for a node to simply trust another node’s messages, since they may be sent with malicious intent. For example, peer-to-peer networks like Bitcoin and other blockchains can be considered to be a way of getting mutually untrusting parties to agree whether a transaction happened or not, without relying on a central authority [83].

However, in the kinds of systems we discuss in this book, we can usually safely assume that there are no Byzantine faults. In your datacenter, all the nodes are controlled by your organization (so they can hopefully be trusted) and radiation levels are low enough that memory corruption is not a major problem. Protocols for making systems Byzantine fault-tolerant are quite complicated [84], and fault-tolerant embedded systems rely on support from the hardware level [81]. In most server-side data systems, the cost of deploying Byzantine fault-tolerant solutions makes them impractical.

Web applications do need to expect arbitrary and malicious behavior of clients that are under end-user control, such as web browsers. This is why input validation, sanitization, and output escaping are so important: to prevent SQL injection and cross-site scripting, for example. However, we typically don’t use Byzantine fault-tolerant protocols here, but simply make the server the authority on deciding what client behavior is and isn’t allowed. In peer-to-peer networks, where there is no such central authority, Byzantine fault tolerance is more relevant.

A bug in the software could be regarded as a Byzantine fault, but if you deploy the same software to all nodes, then a Byzantine fault-tolerant algorithm cannot save you. Most Byzantine fault-tolerant algorithms require a supermajority of more than two-thirds of the nodes to be functioning correctly (i.e., if you have four nodes, at most one may malfunction). To use this approach against bugs, you would have to have four independent implementations of the same software and hope that a bug only appears in one of the four implementations.

Similarly, it would be appealing if a protocol could protect us from vulnerabilities, security compromises, and malicious attacks. Unfortunately, this is not realistic either: in most systems, if an attacker can compromise one node, they can probably compromise all of them, because they are probably running the same software. Thus, traditional mechanisms (authentication, access control, encryption, firewalls, and so on) continue to be the main protection against attackers.

#### Weak forms of lying

Although we assume that nodes are generally honest, it can be worth adding mechanisms to software that guard against weak forms of “lying” — for example, invalid messages due to hardware issues, software bugs, and misconfiguration. Such protection mechanisms are not full-blown Byzantine fault tolerance, as they would not withstand a determined adversary, but they are nevertheless simple and pragmatic steps toward better reliability. For example:

* Network packets do sometimes get corrupted due to hardware issues or bugs in operating systems, drivers, routers, etc. Usually, corrupted packets are caught by the checksums built into TCP and UDP, but sometimes they evade detection [85, 86, 87]. Simple measures are usually sufficient protection against such corruption, such as checksums in the application-level protocol.

* A publicly accessible application must carefully sanitize any inputs from users, for example checking that a value is within a reasonable range and limiting the size of strings to prevent denial of service through large memory allocations. An internal service behind a firewall may be able to get away with less strict checks on inputs, but some basic sanity-checking of values (e.g., in protocol parsing [85]) is a good idea.

* NTP clients can be configured with multiple server addresses. When synchronizing, the client contacts all of them, estimates their errors, and checks that a majority of servers agree on some time range. As long as most of the servers are okay, a misconfigured NTP server that is reporting an incorrect time is detected as an outlier

### System Model and Reality

Many algorithms have been designed to solve distributed systems problems — for example, we will examine solutions for the consensus problem in Chapter   9. In order to be useful, these algorithms need to tolerate the various faults of distributed systems that we discussed in this chapter.

Algorithms need to be written in a way that does not depend too heavily on the details of the hardware and software configuration on which they are run. This in turn requires that we somehow formalize the kinds of faults that we expect to happen in a system. We do this by defining a system model, which is an abstraction that describes what things an algorithm may assume.

With regard to timing assumptions, three system models are in common use:

*Synchronous model*

The synchronous model assumes bounded network delay, bounded process pauses, and bounded clock error. This does not imply exactly synchronized clocks or zero network delay; it just means you know that network delay, pauses, and clock drift will never exceed some fixed upper bound [88]. The synchronous model is not a realistic model of most practical systems, because (as discussed in this chapter) unbounded delays and pauses do occur.

*Partially synchronous model*

Partial synchrony means that a system behaves like a synchronous system most of the time, but it sometimes exceeds the bounds for network delay, process pauses, and clock drift [88]. This is a realistic model of many systems: most of the time, networks and processes are quite well behaved — otherwise we would never be able to get anything done — but we have to reckon with the fact that any timing assumptions may be shattered occasionally. When this happens, network delay, pauses, and clock error may become arbitrarily large.

*Asynchronous model*

In this model, an algorithm is not allowed to make any timing assumptions — in fact, it does not even have a clock (so it cannot use timeouts). Some algorithms can be designed for the asynchronous model, but it is very restrictive.

Moreover, besides timing issues, we have to consider node failures. The three most common system models for nodes are:

*Crash-stop faults*

In the crash-stop model, an algorithm may assume that a node can fail in only one way, namely by crashing. This means that the node may suddenly stop responding at any moment, and thereafter that node is gone forever — it never comes back.

*Crash-recovery faults*

We assume that nodes may crash at any moment, and perhaps start responding again after some unknown time. In the crash-recovery model, nodes are assumed to have stable storage (i.e., nonvolatile disk storage) that is preserved across crashes, while the in-memory state is assumed to be lost.

*Byzantine (arbitrary) faults*

Nodes may do absolutely anything, including trying to trick and deceive other nodes, as described in the last section.

For modeling real systems, the partially synchronous model with crash-recovery faults is generally the most useful model. But how do distributed algorithms cope with that model?

#### Correctness of an algorithm

To define what it means for an algorithm to be correct, we can describe its properties. For example, the output of a sorting algorithm has the property that for any two distinct elements of the output list, the element further to the left is smaller than the element further to the right. That is simply a formal way of defining what it means for a list to be sorted.

Similarly, we can write down the properties we want of a distributed algorithm to define what it means to be correct. For example, if we are generating fencing tokens for a lock (see “Fencing tokens”), we may require the algorithm to have the following properties:

*Uniqueness*

No two requests for a fencing token return the same value.

*Monotonic sequence*

If request x returned token tx, and request y returned token ty, and x completed before y began, then tx   <   ty.

*Availability*

A node that requests a fencing token and does not crash eventually receives a response.

An algorithm is correct in some system model if it always satisfies its properties in all situations that we assume may occur in that system model. But how does this make sense? If all nodes crash, or all network delays suddenly become infinitely long, then no algorithm will be able to get anything done.

#### Safety and liveness

To clarify the situation, it is worth distinguishing between two different kinds of properties: safety and liveness properties. In the example just given, uniqueness and monotonic sequence are safety properties, but availability is a liveness property.

What distinguishes the two kinds of properties? A giveaway is that liveness properties often include the word “eventually” in their definition. (And yes, you guessed it — eventual consistency is a liveness property [89].)

Safety is often informally defined as nothing bad happens, and liveness as something good eventually happens. However, it’s best to not read too much into those informal definitions, because the meaning of good and bad is subjective. The actual definitions of safety and liveness are precise and mathematical [90]:

* If a safety property is violated, we can point at a particular point in time at which it was broken (for example, if the uniqueness property was violated, we can identify the particular operation in which a duplicate fencing token was returned). After a safety property has been violated, the violation cannot be undone — the damage is already done.

* A liveness property works the other way round: it may not hold at some point in time (for example, a node may have sent a request but not yet received a response), but there is always hope that it may be satisfied in the future (namely by receiving a response).

An advantage of distinguishing between safety and liveness properties is that it helps us deal with difficult system models. For distributed algorithms, it is common to require that safety properties always hold, in all possible situations of a system model [88]. That is, even if all nodes crash, or the entire network fails, the algorithm must nevertheless ensure that it does not return a wrong result (i.e., that the safety properties remain satisfied).

However, with liveness properties we are allowed to make caveats: for example, we could say that a request needs to receive a response only if a majority of nodes have not crashed, and only if the network eventually recovers from an outage. The definition of the partially synchronous model requires that eventually the system returns to a synchronous state — that is, any period of network interruption lasts only for a finite duration and is then repaired.

#### Mapping system models to the real world

Safety and liveness properties and system models are very useful for reasoning about the correctness of a distributed algorithm. However, when implementing an algorithm in practice, the messy facts of reality come back to bite you again, and it becomes clear that the system model is a simplified abstraction of reality.

For example, algorithms in the crash-recovery model generally assume that data in stable storage survives crashes. However, what happens if the data on disk is corrupted, or the data is wiped out due to hardware error or misconfiguration [91]? What happens if a server has a firmware bug and fails to recognize its hard drives on reboot, even though the drives are correctly attached to the server [92]?

Quorum algorithms (see “Quorums for reading and writing”) rely on a node remembering the data that it claims to have stored. If a node may suffer from amnesia and forget previously stored data, that breaks the quorum condition, and thus breaks the correctness of the algorithm. Perhaps a new system model is needed, in which we assume that stable storage mostly survives crashes, but may sometimes be lost. But that model then becomes harder to reason about.

The theoretical description of an algorithm can declare that certain things are simply assumed not to happen — and in non-Byzantine systems, we do have to make some assumptions about faults that can and cannot happen. However, a real implementation may still have to include code to handle the case where something happens that was assumed to be impossible, even if that handling boils down to printf(" Sucks to be you") and exit( 666) — i.e., letting a human operator clean up the mess [93]. (This is arguably the difference between computer science and software engineering.)

That is not to say that theoretical, abstract system models are worthless — quite the opposite. They are incredibly helpful for distilling down the complexity of real systems to a manageable set of faults that we can reason about, so that we can understand the problem and try to solve it systematically. We can prove algorithms correct by showing that their properties always hold in some system model.

Proving an algorithm correct does not mean its implementation on a real system will necessarily always behave correctly. But it’s a very good first step, because the theoretical analysis can uncover problems in an algorithm that might remain hidden for a long time in a real system, and that only come to bite you when your assumptions (e.g., about timing) are defeated due to unusual circumstances. Theoretical analysis and empirical testing are equally important.