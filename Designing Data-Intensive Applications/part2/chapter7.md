# 第七章 事务

*有些作者声称一般性的二阶段提交支持起来代价太大，因为它带来了性能或者可用性问题。我们认为让应用程序开发者处理由于过度使用事务而出现瓶颈的性能问题会更好，而不应该在编程时完全没有事务可用。*

詹姆斯·科贝特等，*Spanner：谷歌的全球级分布式数据库*（2012）

---

数据系统的残酷现实世界中，很多东西都可能会出错：

* 数据库软件或是硬件可能在任何时间崩溃（包括在一次写入过程当中）。

* 应用程序可能在任何时间崩溃（包括在执行一系列操作的中途）。

* 网络中断可以意外地将应用程序从数据库断开，或者把一个数据库节点与另外一个节点断开。

* 数个客户端可以同时写入数据库，覆盖彼此提交的变更。

* 客户端会读取到没有意义的数据，因为先前只被部分地更新了。

* 客户端之间的竞争条件可以导致以外的bug。

为了可靠，系统需要处理这些故障并且保证它们不会导致整个系统灾难性的故障。然而，实现容错机制需要大量的工作。它需要对所有可能出错的事进行仔细的考虑，以及大量的测试从而保证解决方案是实际可行的。

数十年来，*事务*都是简化这些问题的首选机制。事务是应用程序把数个读写请求组合为一个逻辑单元的方式。从概念上讲，事务中的所有读写操作执行时被当作一次操作：整个事务要么成功（*提交*）要么失败（*取消*，*回滚*）。如果失败，应用可以安全地重试。有了事务，错误处理对应用程序来说变得相当简单，因为它不再需要担心部分失败——也就是（无论出于任何原因）某些操作成功了而某些失败了的情况。

如果你已经有了使用事务许多年的经验，它们看起来很浅显，但是我们不应该视它们为理所应当。事务不是一条自然规则；它们被发明出来是有目的的，就是为了在应用程序访问数据库时*简化编程模型*。通过使用事务，应用程序可以自如地忽略某些潜在的错误场景以及并发问题，因为数据库会替你处理它们（我们把这叫做*安全性保证*）。

不是所有的应用程序都需要用到事务，有些时候弱化事务性保证或是干脆完全抛弃它也是有好处的（比如，为了实现更高的性能或是更高的可用性）。某些安全属性是可以不通过事务实现的。

那么如何判断你需不需要事务呢？为了回答这个问题，我们首先需要理解到底事务可以提供什么样的安全性保证，以及随之而来的代价是什么。虽然乍一看事务简单明了，但是实际上有很多细微但是重要的细节在起作用。

在这一张，我们会研究许多会出错的事情，并且探索数据库用来针对这些问题的算法。我们会特别深入到并发控制的领域，讨论可能发生的各种种类的竞争条件以及数据库是如何实现比如*提交读*、*快照隔离*以及*可串行化*这样的隔离界别的。

这一章既适用于单节点数据库也适用于分布式数据库；在第八章我们将着重讨论只在分布式系统中出现的难题。

## 事务难以掌握的概念

今天，几乎所有的关系型数据库，以及一些非关系型数据库，都支持事务。它们中的绝大多数都沿用第一款SQL数据库，IBM System R在1975年引入的风格。虽然某些实现细节发生了变化，但是总体思路在40年内基本保持不变：MySQL、PostgreSQL、Oracle、SQL Server等对事务的支持与System R的惊人地相似。

在2000年代末，非关系型（NoSQL）数据库开始变得流行起来。它们旨在通过提供新的数据模型来改善关系性数据库的现状（见第二章），并且默认包含了复制（见第五章）以及分区（见第六章）。事务是这场运动的主要受害者：许多新一代的数据库完全放弃了事务，或者重新定义了这个次用以描述比先前所理解的要弱得多的一套保证。

围绕着对这一类新的分布式数据库的炒作，出现了一种普遍的看法，就是事务是站在可扩展性的对立面的，任何大型系统都必须放弃事务从而维持良好的性能和高可用性。另一方面，事务性保证有时被数据库厂商呈现为“有价值数据”的“严肃的应用程序”的基本要求。

事实可没有那么简单：就像任何其它技术设计选择一样，事务也有优点与限制。为了理解这些权衡，让我们看看事务可以提供的保证的细节——无论是在正常运行还是各种极端的（却也现实）的情况之下。

### ACID的含义

事务提供的安全性保证常常用知名的缩写ACID描述，它代表了*原子性*、*一致性*、*隔离性*和*持久性*。它由Theo Härder和Andreas Reuter于1983年创造，旨在为数据库中的容错机制建立精确的术语。

然而，在现实中，一个数据库的ACID实现与另外一个的实现并不等同。举个例子，之后我们会看到，关于*隔离*的含义就有许多歧义。在高层次的理念合理的，但是问题处在了细节上。今天，当一个系统号称“符合ACID”，实际上尚不清楚有哪些保证。ACID非常不幸地变成了一个商业用词。

（不满足ACID标准地系统有时被称作BASE，它代表了*基本可用*、*软状态*以及*最终一致性*，这比ACID的定义还要模糊。看起来BASE唯一合理的定义是“非ACID”；也就是说它可以意味着任何事。）

让我们深入研究原子性、一致性、隔离性和持久性的定义，因为这会让我们改进我们对事务的理解。

#### 原子性

通常，*原子性*指的是无法分割成更小部件的东西。这个词在计算机学不同的分支内指代的是相似但又稍微不同的东西。举个例子，在多线程编程中，如果一个线程执行原子操作，那意味着另外一个线程没有办法看到这个操作一半的结果。系统只会要么处于操作之前的状态或者操作之后的状态，而不是介于二者之间。

相比之下，在ACID的上下文中，原子性与并发*无关*。它也不是描述如果几个进程同时访问同一个数据会发生什么，因为那是字母*I*涵盖的，即*隔离性*（详见“隔离性”一节）。

相反的事，ACID原子性描述的是当客户端想要进行数个写入请求，但是在某些写入请求处理结束后发生了故障时——比如，进程崩溃，网络连接中断，磁盘写满，或者是违反了某些完整性约束——会发生什么。如果写入请求被归为一个原子事务，而事务由于故障不能完成（*提交*），那么事务会*中止*，而数据库必须丢弃或者复原任何迄今为止事务完成的写入。

没有原子性，如果在进行多次更改的过程中发生错误，那么很难知道哪些变化已经生效而哪些没有。应用可以再次尝试写入，但是写入同样变更两次带来的分线，会导致重复或是不正确的数据。原子性简化了这个问题：如果事务被中止，应用程序可以确定它没有改变任何东西，所以它可以安全地重试。

ACID原子性的定义特征是能够在出错的时候放弃事务并把所有事务的写入放弃。相比于*原子性*来说也许*可取消性*是一个更好的词，但是我们会继续用*原子性*，因为那才是常用词。

#### 一致性

*一致性*这个词已经被重用地太厉害了：

* 在第五章我们讨论了*复制一致性*，以及异步复制系统中出现的*最终一致性*问题（见“复制延迟的问题”一节）。

* *一致性哈希*是某些系统用于再平衡的分区方法。（见“一致性哈希”）。

* 在CAP定理中（见第九章），*一致性*一词是用来指*可线性化*（见“可线性化”）。

* 在ACID的语境下，*一致性*指的是一种应用程序特定的概念——数据库目前处于“良好的状态”。

* In the context of ACID, consistency refers to an application-specific notion of the database being in a “good state.”

同一个词用来表达至少四种不同的含义实在是不幸。

ACID一致性的理念是关于数据的某些描述（不变量）必须始终为真——举个例子，在账户系统中，所有账户的借贷必须平衡。如果事务起始时数据库满足那些不变量，而事务处理期间的任何写入都保持有效性，那么可以确定不变量还是满足的。

然而，这种一致性的理念取决于应用程序的不变量定义，正确定义事务使得它们保持一致性。这也是应用程序的责任。这不是数据库可以保证的：如果你写入了错误的数据破坏了这些不变量，数据库也不能阻止你。（某些特定种类的不变量可以由数据库检查，比如使用外键限制或是唯一性限制。然而通常，引用程序定义哪些数据合法哪些数据不合法——数据库只负责存储它们。）

原子性、隔离性和持久性是数据库的特性，而一致性（在ACID意义上）是应用程序特性。应用程序依赖数据库的原子性与隔离性来实现一致性，但是这不只取决于数据库。因而，字母C其实不属于ACID。

#### 隔离性

大多数数据库可以同时为多个客户端访问。如果它们读写的是数据库不同的部分那是没问题的，然而如果它们访问的是同一个数据库条目，就会碰到并发问题（竞争条件）。

图7-1是这种问题的一个简单示例。假设有两个客户端在持续递增存储在数据库中的计数器值。每个客户端需要读取当前的值，增加1，然后把新值写回去（假设数据库没有内建递增操作）。在图7-1中计数器应该从42增加到了44，因为发生了两次递增，但是由于竞争条件实际上只到了43。

ACID意义上的隔离性是指并发执行的事务相互之间是隔离的：它们不会互相干预彼此的操作。经典数据库教科书把隔离性形式化为可串行化性，意味着每个事务都可以假装自己是整个数据库上唯一正在执行的事务。数据库保证在事务提交之后，结果与它们顺序执行（一个接一个）的结果是一样的，哪怕实际上它们是并发执行的。

*图7-1. 两个并发递增计数器的客户端之间的竞争条件。*

然而在实际当中，可串行化的隔离性很少用到，因为它会带来性能损失。一些流行的数据库，比如Oracle 11g，根本没实现它。在Oracel数据库中有一个隔离界别叫做“可串行化”，但是实际上实现的是*快照隔离*，相比可串行化它是一个弱保证。我们会在“弱隔离级别”一节中探索快照隔离以及其它形式的隔离。

#### 持久性

数据库系统存在的目的是为了提供一个安全的储存数据的场所而不用担心丢失它。持久性承诺一旦事务提交成功，它写入的任何数据不会被遗忘，哪怕是发生了硬件故障或是数据库崩溃了。

在单节点数据库中，持久性通常意味着数据库已经被写入到诸如机械硬盘或是固态硬盘这样的非易失性存储中。它常常也涉及到预写入日志或是类似的东西（见“让B树变得可靠”一节），它在磁盘上的数据结构被破坏之后可以进行恢复。在一个复制数据库中，持续性意味着数据被成功地复制到了一定数量的节点。为了保证持久性，数据库必须等待所有的写入或是复制完成以后才能报告事务成功地提交了。

就如“可靠性”一节讨论到地，完美地持久性是不存在的：如果所有的硬盘与所有的备份同时被破坏了，显然没有什么数据库可以帮到你的了。

> **复制与持久性**
>
> 历史上，持久性是指写入到备份磁带上。之后被理解为写入到磁盘或是固态硬盘。最近，它开始意味着复制。那种实现更好呢？
>
> 事实是，没有什么是完美的：
>
> * 如果写入到了磁盘而设备死掉了，虽然数据没有丢失，但是直到你修复了设备或是把磁盘转移到另一台设备，数据都是无法访问的。复制了的系统则继续可用。
>
> * 相关性故障——断电或是由于特定输入导致所有节点崩溃的bug——可以立刻让所有副本下线（见“可靠性”一节），丢失任何还在内存中的数据。因此对于内存数据库来说，写入到磁盘仍然是相关的操作。
>
> * 在异步复制系统中，当领机离线时最新的写入请求都会丢失（见“处理节点离线”一节）。
>
> * 当电源突然被掐断，固态硬盘尤其表现出有时会违反它们本应提供的保证：哪怕`fsync`也无法保证工作正常。硬盘固件会有bug，就像任何其它类型的软件一样。
>
> * 存储引擎与文件系统实现之间的细微互动可以导致很难跟踪的bug，并会导致磁盘上的文件在崩溃后被破坏。 
>
> * 磁盘上的数据会慢慢地不知不觉地被破坏。如果数据被破坏已经有了一段时间，副本以及最新地备份也可能已经破坏了。在这种情况下，你需要尝试从一份老的备份中恢复数据。
>
> * 一份关于固态硬盘的研究发现在使用的头四年中产生至少一个坏块的可能性在30%到80%。机械硬盘相比于固态硬盘有更低的坏扇区率，但是有更高的完全故障率。
>
> * 如果SSD被断电，在几个礼拜之内就会开始丢失数据，而它取决于温度。
>
> 在实践中，没有任何一种技巧可以提供绝对的保证。只有各种降低风险的技巧，包括写入到磁盘，复制到远端设备以及备份——且它们可以也应该一起使用。与往常一样，用怀疑的眼光看待任何理论性的“保证”总是明智的。

### 单对象与多对象操作

概括一下，在ACID中，原子性与隔离性描述了在同一个事务中如果客户端发起了几个写入请求数据库应该做什么：

*原子性*

如果在执行一系列写入中途发生了错误，事务应当中止，而已经完成了的写入应当被丢弃。换句话说，数据库通过提供全有或者全无的保证，使你不用担心发生部分故障。

*隔离性*

并发执行的事务不应该互相干预彼此。举个例子，如果一个事务发起了数个写入请求，之后的另一个事务应该要么看到所有的写入，要么什么都没看到，但是不会看到其中一部分。

这些定义都假设你一次要修改数个对象（行，文档，记录）。如果好几份数据需要保持同步的话就经常需要这种*多对象事务*。图7-2展示了一个来自电子邮件应用的例子。为了显示用户未读邮件的数量，你会进行类似这样的查询：

```SQL
SELECT COUNT(*) FROM emails WHERE recipient_id = 2 AND unread_flag = true
```

然而，如果邮件太多你会发现这个查询太慢了，于是决定把未读消息的数量储存在一个单独的字段里（一种反规范化）。现在，每当收到一封新邮件，你也必须为未读计数器加一，而每当一封邮件被标记为已读，你也必须为唯独计数器减一。

在图7-2中，用户2经历了一个异常现象：邮箱列表显示有一封未读邮件，但是因为计数器加一还没发生，计数器显示零个未读邮件。隔离性可以通过保证用户2要么同时看到插入的邮件和更新了的计数器，要么什么都没有，而不是一个不一致的中间状态来预防这个问题。

*图7-2 违反隔离性：一个事务读到了另一个事务未提交的写入（“脏读”）。*

图7-3展示了对原子性的需要：如果在事务的过程中发生了错误，邮箱的内容与未读计数器就变得不同步了。在一个原子性事务中，如果对计数器的更新失败了，事务会终止且被插入的邮件会回滚。

*图7-3 原子性保证如果发生了错误任何来自事务之前的写入都会被撤销，以避免不一致的状态。*

多对象事务需要某些方式判定哪些读写操作属于同一个事务。在关系型数据库中，这通常是基于客户端到数据库服务器的TCP连接完成的：在任意特定的连接上，在`BEING TRANSACTION`与`COMMIT`之间的所有语句被认为属于同一个事务。

另一方面，许多非关系型数据库没有这种把操作分组到一起的方式。即使有多对象的API（举个例子，键值对存储可以有一次操作更新好几个键值的*多赋值*操作），但是这不意味它有事务的语义：这个命令有可能成功更新了某些键值而其它的失败了，使数据库出在了部分更新的状态。

#### 单对象写入

当单对象被更新时原子性与隔离性也适用。举个例子，假设你正在写入20KB大的JSON文档到数据库：

* 如果网络连接在发送10KB之后中断了，数据库会存储这无法解析的10KB JSON片段么？

* 如果数据库正在覆盖磁盘上的前一个值时断电了，你最终得到的是新旧值拼接在一起的值么？

* 如果另一个客户端在写入进行时读取那个文档，它看到的是一个部分更新的值么？

这些问题会令人难以置信地困惑，所以存储引擎几乎普遍提供单节点上的单对象（比如键值对）级别的原子性与隔离性。原子性在针对崩溃恢复时可以用日志实现（见“使B树稳定”一节），而隔离性可以用每个对象的锁来实现（同一时刻只允许一个线程访问对象）。

一些数据库孩提攻了更复杂的原子操作，比如增量操作，它消除了对如图7-1所示的读取-修改-写入循环的需求。类似的流行操作还有比较并设置操作，它使得写入只有在值没有并发地被其它人修改才会发生（见“比较并设置”一节）。

这些单对象操作很有用，因为它们可以防止好几个客户端并发尝试写入同一个对象时丢失更新的数据（见“防止丢失更新的数据”）。然而，它们不是通常意义上的事务。比较并设置以及其它单对象操作由于商业目的已经被称为“轻量级事务”，甚至是“ACID”，但是这些术语都是误导人的。事务通常被理解为一种把在多个对象上的多个操作分组成单个操作单元的机制。

#### 多对象事务的需要

许多分布式数据存储放弃了多对象事务，因为在多分区上实现很困难，而且在某些需要非常高的可用性和性能的场景中会很碍事。然而，本质上没有任何东西阻拦分布式数据库使用事务，我们会在第九章讨论分布式事务的实现。

但是我们到底需不需要多对象事务呢？只用键值对数据模型和单对象操作是不是就可以实现任意的应用程序呢？

某些使用场景中单对象插入、更新、删除就足够了。然而在许多其他场景中我们需要协调对好几个不同对象的写入操作：

* 在关系型数据模型中，一张表中的一行经常有一个指向其它表中一行的外键。（类似的在图数据模型中，一个顶点有到其它顶点的边。）多对象事务允许你保证这些引用保持有效：当插入几条互相指向的记录，那外键必须正确而且保持更新，否则数据就没有意义了。

* 在文档型数据模型中，需要被一起更新的字段经常是在同一个文档里的，它们被视为单个对象——在更新单个文档是不需要多对象事务。然而，文档型数据库没有连接功能，也鼓励反规范化（见“今天关系型 vs 文档型数据库”一节）。当反规范化的信息需要更新时，比如图7-2中的例子，你需要一次更新好几个文档。事务在这种场景下非常有用，防止非规范化的数据不同步。

* 在有二级索引的数据库（几乎所有数据库都有，除了纯键值对存储）中，每当你更新一个值时索引也需要更新。从事务的角度上看这些索引是不同的数据库对象：举个例子，没有事务隔离性，一条记录出现在一个索引中但是不在另外一个索引中是可能的，因为对第二个索引的更新还没有发生。

这样的应用程序仍然可以不用事务实现。然而没有原子性的话错误处理变得非常复杂，没有隔离性则会导致并发问题。我们会在“弱隔离级别”一节讨论这些问题，并在第十二章探索其它替代方案。

#### 处理错误与中止

事务的一个核心功能是当错误发生时他可以被终止然后安全地重试。符合ACID地数据库是基于这样的哲学地：如果数据库面临原子性、隔离性或者持久性被破坏的危险，它宁愿放弃整个事务也不会允许它部分执行。

然而并不是所有的系统都遵从这个哲学。特别是，无领机复制的数据库（见“无领机复制”一节）更多的是基于“尽最大努力”信条，它可以被总结为“数据库会尽可能地执行语句，当遇到问题时，他不会撤销它已经完成了的结果”——所以从错误中恢复编程了应用程序的责任。

错误的发生难以避免，但是许多软件开发者更倾向于只考虑不出错的路径而不是复杂的错误处理。举个例子，非常流行的对象关系映射（ORM）框架，比如Rail的ActiveRecord以及Django，不会重试中止了的事务——错误通常引发堆栈异常，所以任何用户输入都被丢弃而用户获得一条错误信息。这很可惜，因为中止的全部意义就是为了可以安全的重试。

虽然重试中止了的事务是一个既简单又有效的错误处理机制，但是它并不完美：

* 如果事务实际成功了，但是服务器试图向服务器确认提交成功时网络瘫痪了（于是客户端认为它失败了），那么再次事务重试会导致它被执行两次——除非你已经有了另外一个应用级别的去重机制。

* 如果错误是因为负载过高，重试事务会导致问题更加严重，而不会更好。为了避免这种反馈循环，你可以限制重试的次数，使用指数回退，并且（可能的话）把负载过高相关错误的处理机制与其它的区别开来。

* 只有在发生了瞬态错误（比如说死锁、违反了隔离性、临时网络中断以及故障迁移）之后才值得重试；而发生了永久错误（比如违反了约束）之后重试是没有意义的。

* 如果事务还会导致数据库之外的副作用，即使事务被中止了这些副作用也许已经发生了。举个例子，如果你再发送电子邮件，你不会想要在重试事务时每次重新发送这封电子邮件。如果你想要确定好几个不同的系统要么一起提交或者中止了事务，两段提交可以有所帮助（我们会在“原子提交与两段提交（2PC）”一节中讨论它）。

* 如果客户端进程在重试时崩溃，它尝试写入到数据库的任何数据都会丢失。

## 弱隔离级别

如果两个事务没有访问同一份数据，那么它们是可以安全地并行执行地，因为互相没有依赖。并发问题（竞争条件）只有在一个事务读取另外一个事务正在修改地数据时，或者当两个事务尝试同时修改同一份数据才会出现。

并发bug很难通过测试找到，因为这样的bug只有在时间不对的时候才会触发。这样的时间问题非常少发生，通常很难重现。并发也非常难以推理，尤其是在大型应用中你都不知道还有什么地方的代码也在同时访问数据库。应用开发在一次只有一个用户就已经很困难了；有许多并发用户让它变得更加困难，因为任何一段数据都可能在任何时候意外地发生变化。

由于这个原因，数据库一直试图通过提供*事务隔离性*向应用程序开发者隐藏并发问题。理论上，隔离性可以通过假设没有并发正在发生简化问题：*可串行化*隔离性意味着数据库保证事务有着顺序执行同样的效果（即一次一个，没有任何并发情形）。

实践中，可惜隔离性没有那么简单。可串行化隔离性有性能损耗，而许多数据库不打算付出这样的代价。因此通常系统会使用稍弱的隔离级别，他能防范*某些*并发问题，但不是全部的。这些隔离级别更难理解，并且会导致很微妙的bug，但是事件中它们依然在使用。

由弱事务隔离性导致的并发性bug不只是理论性问题。它们曾经导致了大量资金的损失，财务审计师进行调查，以及客户数据被破坏。关于这些问题启示的一条受欢迎的评论是“处理财务数据时要使用ACID数据库！”——但是这没有意义。甚至许多流行的关系型数据库系统（通常被认为是“ACID”的）使用弱隔离，所以他们不一定会防止这些错误的发生。

与其盲目地依赖工具，我们需要提高对存在的各种并发问题的认识，以及如何防止它们。之后我们才能用手边的工具构建可靠和正确的应用。

在这一节我们会思考好几个实践中用到的弱（非串行化）的隔离级别，并详细讨论那种竞争条件可以或者不会发生，所以你可以决定那种级别是适合你的应用的。一旦完成，我们会详细讨论可串行化（见“可串行化”一节）。我们对隔离级别的讨论是随意的，并使用示例。如果你需要对其属性进行严格的定义和分析，你可以在学术文献中找到它们。

### 提交读

事务隔离性最基本的级别时提交读。它做出了两项保证：

1. 从数据库读取时，你只会看到已经被提交了的数据（没有脏读）。

2. 当写入到数据库时，你只会覆盖已经被提交了的数据（没有脏写）。接下来让我们详细讨论这两项保证吧。

#### 没有脏读

设想一个事务写入了某些数据到数据库，但是事务还没有提交或是中止。另外一个事务可以看到那些未提交的数据吗？如果是，那就叫做脏读。

运行在提交读隔离级别的事务必须防止脏读的发生。这意味着事务的任何写入只有在提交之后才对其它事务可见（之后它的所有写入立刻变得可见了）。这如图7-4所示，其中用户1设*x* = 3，但是用户2*获取x*任然得到的是旧值，2，因为用户1还没有提交。

*图7-4 没有脏读：用户2只有在用户1的事务提交之后才能看到x的新值。*

为什么防止脏读很有用，这里有几个理由：

* 如果事务需要更新好几个对象，脏读意味着另外一个事务会看到部分更新而其它的看不到。举个例子，在图7-2中，用户看到新的未读邮件但是没有看到更新了的计数器。这是邮件的脏读。看到数据库处在部分更新的状态对用户来说很困惑，也会导致其它事务做出错误的决定。

* 如果事务中止，任何它已经完成了的写入都需要回滚（如图7-3中那样）。如果数据库允许脏读，那意味着事务会看到之后被混滚了的数据——也就是那些从来没有被实际提交到数据库中的数据。关于后果的推理很快会变得让人费解。

#### 没有脏写

如果两个事务并发地尝试更新数据库中同一个对象时会发生什么？我们不知道写入的发生顺序，但是一般我们假设稍后的写入会覆盖稍早的写入。

然而，如果稍早的写入是还没有提交的事务的一部分，那么稍后的写入会覆盖未提交的数据么？这被称为脏写。运行在提交读隔离级别的事务必须防止脏写的发生，通常是通过延迟第二个写入直到第一个写入的事务提交或是中止完成的。

通过防止脏写，这个隔离级别避免了一些类型的并发问题：

* 如果事务更新数个对象，脏写回导致恶劣的后果。举个例子，考虑一下图7-5，它展示一个二手车销售网站，其中两个人Alice和Bob，同时尝试购买同一部车。买一部车需要两次数据库写入：网站列表需要被更新来体现买家，而销售发票需要发送给买家。在图7-5的场景中，销售结果授予了Bob（因为她对销售列表所在的表进行了最终写入），但是发票却寄给了Alice（因为她对发票所在的表进行了最终写入）。提交读防止了这样的事故。

* 然而，提交读不不能防止图7-1中的两个计数器增加一的竞争条件。在这种情况下，第二次写如发生在第一个事务提交之后，所以这不是脏写。但是它仍然不对，却是由于另外一个原因——在“防止丢失的更新”我们回讨论如何使这样的计数器加一是安全的。

*图7-5 有脏写的话，来自不同事务的冲突写入会混在一起。*

#### 实现提交读

提交读是非常受欢迎的隔离级别。在Oracle 11g、PostgreSQL、SQL Server 2012、MemSQL以及许多其它数据库中都是默认设置。

最常见的是，为了防止脏写数据库使用了行级别的锁：当事务要修改特定对象（行或是文档）时，它必须首先获取对象的锁。然后它必须占有那个锁直至事务被提交或是被中止。只有一个事务可以占有属于任何给定对象的锁；如果另一个事务想要写入同一个对象，它首先必须等待直到第一个事务提交或者中止，然后它才能获取锁并继续执行。在提交读模式（或是更强的隔离级别）这种锁动作是由数据库自动完成的。

我们如何防止脏读呢？一种选择是使用同样的锁，并且要求任何要读取对象的事务简短地获取锁然后在读取之后再次释放它。这保证了读取无法在对象有修改后未提交的值时发生（因为在那个时候锁正在被发起写入的事务占有）。

然而，实践中获取读的锁的方式工作得并不是很好，因为一次长时间写入得事务可以强制许多只读的事务等待它完成。这会危及只读事务的相应时间，对于运营来说是很恶劣的：程序一个部分的缓慢对完全不同的另一部分有连锁反应，只是因为在等待锁。

由于这个原因，许多数据库防止脏读使用了图7-4中展示的方法：对于每个被写入的对象，数据库同时记住旧值与当前占据写入锁的事务设置的新值。当事务进行的时候，任何其它读取对象的事务获得旧的值。只有在新的值被提交了以后所有事务才会切换到读取新的值。

### 快照隔离与可重复读

如果只看提交读隔离的表面，认为它已经做了事务需要的所有事情，可以原谅：它允许中止（原子性需要），它避免了读取事务不完整的结果，也防止并发写入混在一起。确实，这些都是有用的功能，同时相比于那些不提供事务的系统提供的保证要强多了。

然而，使用这种隔离级别还是有许多种方式遇到并发bug的。举个例子，图7-6展示了使用提交读会发生的问题。

*图7-6 阅读扭曲：Alice观察到数据库处于不一致的状态。*

假设Alice在银行存了1000块，拆分到两个账户，每个账户500块。现在一个事务分别从其中一个账户转100块到另外一个账户。如果事务被处理的同时她不幸看到了账户余额列表，她会看到一个账户余额处于收款到达之前的状态（余额500块），另一个账户处于付款之后的状态（新余额400块）。对于Alice来说现在看起来她的账户加起来只有900块了——好像那100块凭空消失了。

这个异常现象叫做不可重复读或者读偏：如果Alice在事务结束再一次读取账户1的余额，他会看到另一个值（600块）而不是前一次查询到的值。读偏在提交读隔离中被认为是可接受的：Alice看到的账户余额在她读取的时候确实是提交过了的。

> 注意
>
> 术语*偏*不幸地被赋予了新的含义：之前我们把它用在了热点的不平衡负载方面，而在这里它指的是时间上的异常现象。

在Alice的场景中，这不是一个持久的问题，因为她一定会在几秒钟后重新加载在线银行网站看到一致的账户余额。然而，一些场景无法忍受这种临时的不一致性：

*备份*

做备份需要制作整个数据库的拷贝，在大型数据库上会花几个小时的时间。在备份过程中，写入请求会继续发送到数据库。这样，最终备份的某部分包含的是旧版本的数据，而其它部分包含的是新版本的。如果你需要从这样的备份恢复，不一致性（比如消失了的钱）就变成了永久的。

*分析性查询与完整性检查*

有时，你会想要执行一个扫描大部分数据库的查询操作。这样的查询操作在分析时很常见（见“事务处理还是分析？”一节），亦或者是周期性的完整性检查一切是否正常（检测数据损坏）的一部分。如果这些查询在不同的时间点观察数据库的某些部分，就很可能会返回没有意义的结果。

*快照隔离*是这个问题最常见的解决方案。它的理念是每个事务读自数据库的一致快照——也就是，事务看到所有事务开始时被提交到数据库的所有数据。即使数据之后被另一个事务修改了，从那个特定时间点起，每个事务只看到旧数据。

对于诸如备份和分析这种长时间运行的只读查询来说，快照隔离是一个好事情。如果查询执行过程中数据也在同时变化，那么推理查询的意义就很难了。当事务可以看到数据库的一致快照。冻结在某个特定时间点，就很容易理解了。快照隔离是个受欢迎的功能：PostgreSQL、使用InnoDB存储引擎的MySQL、Oracle、SQL Server以及其它数据库都支持。

#### Implementing snapshot isolation

Like read committed isolation, implementations of snapshot isolation typically use write locks to prevent dirty writes (see “Implementing read committed”), which means that a transaction that makes a write can block the progress of another transaction that writes to the same object. However, reads do not require any locks. From a performance point of view, a key principle of snapshot isolation is readers never block writers, and writers never block readers. This allows a database to handle long-running read queries on a consistent snapshot at the same time as processing writes normally, without any lock contention between the two. 

To implement snapshot isolation, databases use a generalization of the mechanism we saw for preventing dirty reads in Figure   7-4. The database must potentially keep several different committed versions of an object, because various in-progress transactions may need to see the state of the database at different points in time. Because it maintains several versions of an object side by side, this technique is known as multi-version concurrency control (MVCC). 

If a database only needed to provide read committed isolation, but not snapshot isolation, it would be sufficient to keep two versions of an object: the committed version and the overwritten-but-not-yet-committed version. However, storage engines that support snapshot isolation typically use MVCC for their read committed isolation level as well. A typical approach is that read committed uses a separate snapshot for each query, while snapshot isolation uses the same snapshot for an entire transaction. 

Figure   7-7 illustrates how MVCC-based snapshot isolation is implemented in PostgreSQL [31] (other implementations are similar). When a transaction is started, it is given a unique, always-increasingvii transaction ID (txid). Whenever a transaction writes anything to the database, the data it writes is tagged with the transaction ID of the writer.

*Figure 7-7. Implementing snapshot isolation using multi-version objects.*

Each row in a table has a created_by field, containing the ID of the transaction that inserted this row into the table. Moreover, each row has a deleted_by field, which is initially empty. If a transaction deletes a row, the row isn’t actually deleted from the database, but it is marked for deletion by setting the deleted_by field to the ID of the transaction that requested the deletion. At some later time, when it is certain that no transaction can any longer access the deleted data, a garbage collection process in the database removes any rows marked for deletion and frees their space. 

An update is internally translated into a delete and a create. For example, in Figure   7-7, transaction 13 deducts $ 100 from account 2, changing the balance from $ 500 to $ 400. The accounts table now actually contains two rows for account 2: a row with a balance of $ 500 which was marked as deleted by transaction 13, and a row with a balance of $ 400 which was created by transaction 13.

#### Visibility rules for observing a consistent snapshot

When a transaction reads from the database, transaction IDs are used to decide which objects it can see and which are invisible. By carefully defining visibility rules, the database can present a consistent snapshot of the database to the application. This works as follows: 

1. At the start of each transaction, the database makes a list of all the other transactions that are in progress (not yet committed or aborted) at that time. Any writes that those transactions have made are ignored, even if the transactions subsequently commit. 

2. Any writes made by aborted transactions are ignored. 

3. Any writes made by transactions with a later transaction ID (i.e., which started after the current transaction started) are ignored, regardless of whether those transactions have committed. 

4. All other writes are visible to the application’s queries. 

These rules apply to both creation and deletion of objects. In Figure   7-7, when transaction 12 reads from account 2, it sees a balance of $ 500 because the deletion of the $ 500 balance was made by transaction 13 (according to rule 3, transaction 12 cannot see a deletion made by transaction 13), and the creation of the $ 400 balance is not yet visible (by the same rule). Put another way, an object is visible if both of the following conditions are true: 

* At the time when the reader’s transaction started, the transaction that created the object had already committed. 

* The object is not marked for deletion, or if it is, the transaction that requested deletion had not yet committed at the time when the reader’s transaction started. 

A long-running transaction may continue using a snapshot for a long time, continuing to read values that (from other transactions’ point of view) have long been overwritten or deleted. By never updating values in place but instead creating a new version every time a value is changed, the database can provide a consistent snapshot while incurring only a small overhead.

#### Indexes and snapshot isolation

How do indexes work in a multi-version database? One option is to have the index simply point to all versions of an object and require an index query to filter out any object versions that are not visible to the current transaction. When garbage collection removes old object versions that are no longer visible to any transaction, the corresponding index entries can also be removed. 

In practice, many implementation details determine the performance of multi-version concurrency control. For example, PostgreSQL has optimizations for avoiding index updates if different versions of the same object can fit on the same page [31]. Another approach is used in CouchDB, Datomic, and LMDB. Although they also use B-trees (see “B-Trees”), they use an append-only/ copy-on-write variant that does not overwrite pages of the tree when they are updated, but instead creates a new copy of each modified page. Parent pages, up to the root of the tree, are copied and updated to point to the new versions of their child pages. Any pages that are not affected by a write do not need to be copied, and remain immutable [33, 34, 35]. 

With append-only B-trees, every write transaction (or batch of transactions) creates a new B-tree root, and a particular root is a consistent snapshot of the database at the point in time when it was created. There is no need to filter out objects based on transaction IDs because subsequent writes cannot modify an existing B-tree; they can only create new tree roots. However, this approach also requires a background process for compaction and garbage collection.

#### Repeatable read and naming confusion

Snapshot isolation is a useful isolation level, especially for read-only transactions. However, many databases that implement it call it by different names. In Oracle it is called serializable, and in PostgreSQL and MySQL it is called repeatable read [23]. 

The reason for this naming confusion is that the SQL standard doesn’t have the concept of snapshot isolation, because the standard is based on System R’s 1975 definition of isolation levels [2] and snapshot isolation hadn’t yet been invented then. Instead, it defines repeatable read, which looks superficially similar to snapshot isolation. PostgreSQL and MySQL call their snapshot isolation level repeatable read because it meets the requirements of the standard, and so they can claim standards compliance. 

Unfortunately, the SQL standard’s definition of isolation levels is flawed — it is ambiguous, imprecise, and not as implementation-independent as a standard should be [28]. Even though several databases implement repeatable read, there are big differences in the guarantees they actually provide, despite being ostensibly standardized [23]. There has been a formal definition of repeatable read in the research literature [29, 30], but most implementations don’t satisfy that formal definition. And to top it off, IBM DB2 uses “repeatable read” to refer to serializability [8]. 

As a result, nobody really knows what repeatable read means.