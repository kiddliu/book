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

在这一张，我们会研究许多会出错的事情，并且探索数据库用来针对这些问题的算法。我们会特别深入到并发控制的领域，讨论可能发生的各种种类的竞争条件以及数据库是如何实现比如*提交读*、*快照隔离*以及*可序列化*这样的隔离界别的。

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

Most databases are accessed by several clients at the same time. That is no problem if they are reading and writing different parts of the database, but if they are accessing the same database records, you can run into concurrency problems (race conditions). 

Figure   7-1 is a simple example of this kind of problem. Say you have two clients simultaneously incrementing a counter that is stored in a database. Each client needs to read the current value, add 1, and write the new value back (assuming there is no increment operation built into the database). In Figure   7-1 the counter should have increased from 42 to 44, because two increments happened, but it actually only went to 43 because of the race condition. 

Isolation in the sense of ACID means that concurrently executing transactions are isolated from each other: they cannot step on each other’s toes. The classic database textbooks formalize isolation as serializability, which means that each transaction can pretend that it is the only transaction running on the entire database. The database ensures that when the transactions have committed, the result is the same as if they had run serially (one after another), even though in reality they may have run concurrently [10].

*Figure 7-1. A race condition between two clients concurrently incrementing a counter.*

However, in practice, serializable isolation is rarely used, because it carries a performance penalty. Some popular databases, such as Oracle 11g, don’t even implement it. In Oracle there is an isolation level called “serializable,” but it actually implements something called snapshot isolation, which is a weaker guarantee than serializability [8, 11]. We will explore snapshot isolation and other forms of isolation in “Weak Isolation Levels”.

#### 持久性

The purpose of a database system is to provide a safe place where data can be stored without fear of losing it. Durability is the promise that once a transaction has committed successfully, any data it has written will not be forgotten, even if there is a hardware fault or the database crashes. 

In a single-node database, durability typically means that the data has been written to nonvolatile storage such as a hard drive or SSD. It usually also involves a write-ahead log or similar (see “Making B-trees reliable”), which allows recovery in the event that the data structures on disk are corrupted. In a replicated database, durability may mean that the data has been successfully copied to some number of nodes. In order to provide a durability guarantee, a database must wait until these writes or replications are complete before reporting a transaction as successfully committed. 

As discussed in “Reliability”, perfect durability does not exist: if all your hard disks and all your backups are destroyed at the same time, there’s obviously nothing your database can do to save you.

> **Replication and Durability**
>
> Historically, durability meant writing to an archive tape. Then it was understood as writing to a disk or SSD. More recently, it has been adapted to mean replication. Which implementation is better? 
>
> The truth is, nothing is perfect:
>
> * If you write to disk and the machine dies, even though your data isn’t lost, it is inaccessible until you either fix the machine or transfer the disk to another machine. Replicated systems can remain available.
>
> * A correlated fault — a power outage or a bug that crashes every node on a particular input — can knock out all replicas at once (see “Reliability”), losing any data that is only in memory. Writing to disk is therefore still relevant for in-memory databases.
>
> * In an asynchronously replicated system, recent writes may be lost when the leader becomes unavailable (see “Handling Node Outages”).
>
> * When the power is suddenly cut, SSDs in particular have been shown to sometimes violate the guarantees they are supposed to provide: even fsync isn’t guaranteed to work correctly [12]. Disk firmware can have bugs, just like any other kind of software [13, 14].
>
> * Subtle interactions between the storage engine and the filesystem implementation can lead to bugs that are hard to track down, and may cause files on disk to be corrupted after a crash [15, 16]. 
>
> * Data on disk can gradually become corrupted without this being detected [17]. If data has been corrupted for some time, replicas and recent backups may also be corrupted. In this case, you will need to try to restore the data from a historical backup.
>
> * One study of SSDs found that between 30% and 80% of drives develop at least one bad block during the first four years of operation [18]. Magnetic hard drives have a lower rate of bad sectors, but a higher rate of complete failure than SSDs.
>
> * If an SSD is disconnected from power, it can start losing data within a few weeks, depending on the temperature [19].
>
> In practice, there is no one technique that can provide absolute guarantees. There are only various risk-reduction techniques, including writing to disk, replicating to remote machines, and backups — and they can and should be used together. As always, it’s wise to take any theoretical “guarantees” with a healthy grain of salt.