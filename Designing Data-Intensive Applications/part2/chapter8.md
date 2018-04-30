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

### Cloud Computing and Supercomputing

There is a spectrum of philosophies on how to build large-scale computing systems: 

* At one end of the scale is the field of high-performance computing (HPC). Supercomputers with thousands of CPUs are typically used for computationally intensive scientific computing tasks, such as weather forecasting or molecular dynamics (simulating the movement of atoms and molecules). 

* At the other extreme is cloud computing, which is not very well defined [6] but is often associated with multi-tenant datacenters, commodity computers connected with an IP network (often Ethernet), elastic/ on-demand resource allocation, and metered billing. 

* Traditional enterprise datacenters lie somewhere between these extremes. 

With these philosophies come very different approaches to handling faults. In a supercomputer, a job typically checkpoints the state of its computation to durable storage from time to time. If one node fails, a common solution is to simply stop the entire cluster workload. After the faulty node is repaired, the computation is restarted from the last checkpoint [7, 8]. Thus, a supercomputer is more like a single-node computer than a distributed system: it deals with partial failure by letting it escalate into total failure — if any part of the system fails, just let everything crash (like a kernel panic on a single machine). 

In this book we focus on systems for implementing internet services, which usually look very different from supercomputers: 

* Many internet-related applications are online, in the sense that they need to be able to serve users with low latency at any time. Making the service unavailable — for example, stopping the cluster for repair — is not acceptable. In contrast, offline (batch) jobs like weather simulations can be stopped and restarted with fairly low impact. 

* Supercomputers are typically built from specialized hardware, where each node is quite reliable, and nodes communicate through shared memory and remote direct memory access (RDMA). On the other hand, nodes in cloud services are built from commodity machines, which can provide equivalent performance at lower cost due to economies of scale, but also have higher failure rates. 

* Large datacenter networks are often based on IP and Ethernet, arranged in Clos topologies to provide high bisection bandwidth [9]. Supercomputers often use specialized network topologies, such as multi-dimensional meshes and toruses [10], which yield better performance for HPC workloads with known communication patterns. 

* The bigger a system gets, the more likely it is that one of its components is broken. Over time, broken things get fixed and new things break, but in a system with thousands of nodes, it is reasonable to assume that something is always broken [7]. When the error handling strategy consists of simply giving up, a large system can end up spending a lot of its time recovering from faults rather than doing useful work [8]. 

* If the system can tolerate failed nodes and still keep working as a whole, that is a very useful feature for operations and maintenance: for example, you can perform a rolling upgrade (see Chapter   4), restarting one node at a time, while the service continues serving users without interruption. In cloud environments, if one virtual machine is not performing well, you can just kill it and request a new one (hoping that the new one will be faster). 

* In a geographically distributed deployment (keeping data geographically close to your users to reduce access latency), communication most likely goes over the internet, which is slow and unreliable compared to local networks. Supercomputers generally assume that all of their nodes are close together. 

If we want to make distributed systems work, we must accept the possibility of partial failure and build fault-tolerance mechanisms into the software. In other words, we need to build a reliable system from unreliable components. (As discussed in “Reliability”, there is no such thing as perfect reliability, so we’ll need to understand the limits of what we can realistically promise.) 

Even in smaller systems consisting of only a few nodes, it’s important to think about partial failure. In a small system, it’s quite likely that most of the components are working correctly most of the time. However, sooner or later, some part of the system will become faulty, and the software will have to somehow handle it. The fault handling must be part of the software design, and you (as operator of the software) need to know what behavior to expect from the software in the case of a fault. 

It would be unwise to assume that faults are rare and simply hope for the best. It is important to consider a wide range of possible faults — even fairly unlikely ones — and to artificially create such situations in your testing environment to see what happens. In distributed systems, suspicion, pessimism, and paranoia pay off.