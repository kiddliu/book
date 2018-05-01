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

### Synchronous Versus Asynchronous Networks 

Distributed systems would be a lot simpler if we could rely on the network to deliver packets with some fixed maximum delay, and not to drop packets. Why can’t we solve this at the hardware level and make the network reliable so that the software doesn’t need to worry about it? 

To answer this question, it’s interesting to compare datacenter networks to the traditional fixed-line telephone network (non-cellular, non-VoIP), which is extremely reliable: delayed audio frames and dropped calls are very rare. A phone call requires a constantly low end-to-end latency and enough bandwidth to transfer the audio samples of your voice. Wouldn’t it be nice to have similar reliability and predictability in computer networks? 

When you make a call over the telephone network, it establishes a circuit: a fixed, guaranteed amount of bandwidth is allocated for the call, along the entire route between the two callers. This circuit remains in place until the call ends [32]. For example, an ISDN network runs at a fixed rate of 4,000 frames per second. When a call is established, it is allocated 16 bits of space within each frame (in each direction). Thus, for the duration of the call, each side is guaranteed to be able to send exactly 16 bits of audio data every 250 microseconds [33, 34]. 

This kind of network is synchronous: even as data passes through several routers, it does not suffer from queueing, because the 16 bits of space for the call have already been reserved in the next hop of the network. And because there is no queueing, the maximum end-to-end latency of the network is fixed. We call this a bounded delay.

#### Can we not simply make network delays predictable? 

Note that a circuit in a telephone network is very different from a TCP connection: a circuit is a fixed amount of reserved bandwidth which nobody else can use while the circuit is established, whereas the packets of a TCP connection opportunistically use whatever network bandwidth is available. You can give TCP a variable-sized block of data (e.g., an email or a web page), and it will try to transfer it in the shortest time possible. While a TCP connection is idle, it doesn’t use any bandwidth.ii 

If datacenter networks and the internet were circuit-switched networks, it would be possible to establish a guaranteed maximum round-trip time when a circuit was set up. However, they are not: Ethernet and IP are packet-switched protocols, which suffer from queueing and thus unbounded delays in the network. These protocols do not have the concept of a circuit. 

Why do datacenter networks and the internet use packet switching? The answer is that they are optimized for bursty traffic. A circuit is good for an audio or video call, which needs to transfer a fairly constant number of bits per second for the duration of the call. On the other hand, requesting a web page, sending an email, or transferring a file doesn’t have any particular bandwidth requirement — we just want it to complete as quickly as possible. 

If you wanted to transfer a file over a circuit, you would have to guess a bandwidth allocation. If you guess too low, the transfer is unnecessarily slow, leaving network capacity unused. If you guess too high, the circuit cannot be set up (because the network cannot allow a circuit to be created if its bandwidth allocation cannot be guaranteed). Thus, using circuits for bursty data transfers wastes network capacity and makes transfers unnecessarily slow. By contrast, TCP dynamically adapts the rate of data transfer to the available network capacity. 

There have been some attempts to build hybrid networks that support both circuit switching and packet switching, such as ATM.iii InfiniBand has some similarities [35]: it implements end-to-end flow control at the link layer, which reduces the need for queueing in the network, although it can still suffer from delays due to link congestion [36]. With careful use of quality of service (QoS, prioritization and scheduling of packets) and admission control (rate-limiting senders), it is possible to emulate circuit switching on packet networks, or provide statistically bounded delay [25, 32].

> Latency and Resource Utilization
>
> More generally, you can think of variable delays as a consequence of dynamic resource partitioning.
>
> Say you have a wire between two telephone switches that can carry up to 10,000 simultaneous calls. Each circuit that is switched over this wire occupies one of those call slots. Thus, you can think of the wire as a resource that can be shared by up to 10,000 simultaneous users. The resource is divided up in a static way: even if you’re the only call on the wire right now, and all other 9,999 slots are unused, your circuit is still allocated the same fixed amount of bandwidth as when the wire is fully utilized.
>
> By contrast, the internet shares network bandwidth dynamically. Senders push and jostle with each other to get their packets over the wire as quickly as possible, and the network switches decide which packet to send (i.e., the bandwidth allocation) from one moment to the next. This approach has the downside of queueing, but the advantage is that it maximizes utilization of the wire. The wire has a fixed cost, so if you utilize it better, each byte you send over the wire is cheaper.
>
> A similar situation arises with CPUs: if you share each CPU core dynamically between several threads, one thread sometimes has to wait in the operating system’s run queue while another thread is running, so a thread can be paused for varying lengths of time. However, this utilizes the hardware better than if you allocated a static number of CPU cycles to each thread (see “Response time guarantees”). Better hardware utilization is also a significant motivation for using virtual machines.
>
> Latency guarantees are achievable in certain environments, if resources are statically partitioned (e.g., dedicated hardware and exclusive bandwidth allocations). However, it comes at the cost of reduced utilization — in other words, it is more expensive. On the other hand, multi-tenancy with dynamic resource partitioning provides better utilization, so it is cheaper, but it has the downside of variable delays. Variable delays in networks are not a law of nature, but simply the result of a cost/ benefit trade-off.

However, such quality of service is currently not enabled in multi-tenant datacenters and public clouds, or when communicating via the internet.iv Currently deployed technology does not allow us to make any guarantees about delays or reliability of the network: we have to assume that network congestion, queueing, and unbounded delays will happen. Consequently, there’s no “correct” value for timeouts — they need to be determined experimentally.