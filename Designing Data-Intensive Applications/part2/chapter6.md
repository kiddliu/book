# 第六章 分区

*显然，我们必须脱离顺序同时不去限制计算机。 我们必须声明定义并提供数据的优先级和描述。 我们必须说明关系，而不是过程。*

葛丽丝·穆雷·霍普, *未来的管理与计算机* (1962)

---

在第五章我们讨论了复制——也就是，在不同节点上有同样数据的多个拷贝。对于非常大的数据集，或者非常高的查询吞吐量，这是不够的：我们需要把数据*分区*，也叫做*分片*。

> 术语上的混乱
>
> 我们这里称之为*parition*的东西在MongoDB、Elasticsearch以及SolrCloud中叫做*shard*；在HBase叫做*region*，在Bigtable中叫做*tablet*，在Cassandra和Riak中叫做vnode，在Couchbase中叫做vBucket。然而，分区时最成熟的名词，所以我们会继续用它。

通常，分区时这样定义的，每条数据（每条记录，每个行或者每个文档）只属于一个分区。有许多方式可以实现这个目标，我们会在本章深入讨论它们。实际上，每个分区都是一个小数据库，虽然数据库支持同时涉及多个分区的操作。

想要对数据分区的主要原因是*可扩展性*。不同分区可以放在无共享集群中的不同的节点上（*无共享*的定义详见第二部分的介绍）。因而，大的数据集分布在多个磁盘，查询负载分布在多个处理器上。

对于在单个分区上操作的查询请求，每个节点可以在自己的分区上独立执行查询请求，所以查询吞吐量可以通过添加更多节点扩展。大型复杂的查询请求可以并行在多个节点上执行，虽然这样会变得非常困难。

分区数据库的先行者是1980年代的产品，比如Teradata和Tandem NonStop SQL，而最近被NoSQL数据库和基于Hadoop的数据仓库重新发现。有些系统为事务性工作量设计，而其它是为了分析（见“事务处理，还是分析？”一节）：这个差别影响了系统如何调优，但是分区的基本原理适用于两种工作负载。

在这一章中我们将首先了解大型数据集分区的不同方式，并观察为数据建立索引是如何与分区互动的。然后我们会聊到再平衡，如果你要添加或移除集群中的节点的话是很必要的。最后我们会概览数据库是如何路由请求到正确的分区而后执行查询请求的。

## 分区与复制

分区通常会结合复制从而每个分区的拷贝存放于数个节点。这意味着，即使每条记录只属于一个分区，为了容错性它仍可能被储存在几个不同的节点上。

一个节点可以储存多个分区。如果使用了领机-从机复制模型，分区与复制结合之后看起来如图6-1一般。每个分区的领机被分配到一个节点，它的从机被分配到其它节点。每个节点可以是某些分区的领机也可以是其它分区的从机。

我们在第五章讨论的所有关于数据库复制的东西同样适用于分区的复制。分区方法的选择很大程度上独立于复制方法的选择，所以我们在本章会忽略复制，让事情保持简单。

*图6-1 结合复制与分区：每个节点对于某些分区来说是领机，对于其它分区来说是从机。*

## 键值对数据的分区

假设你有大量的数据，你想要对它分区。如何决定哪些条目储存在哪些节点上？我们分区的目标是把数据与查询请求负载平分到多个节点上。如果每个节点承担同样的份额，那么——理论上——10个节点应该可以应对10被于单个节点的数据以及读写吞吐量（暂时忽略复制）。

如果分区不是平均分配的，那么某些分区相比其它分区有更多的数据以及查询请求，我们把它叫做*倾斜*。倾斜的出现使得分区效果变差。在极端情况下，所有的负载可能都在一个分区上，以至于10个节点中的9个是空闲的而最忙碌的那个节点成为了瓶颈。有着不成比例的高负载分区叫做*热点*。

避免热点的最简单的方式是随机地分配条目到结点，但是这样做有个大缺点：当你尝试读取特定的条目，你没有办法知道它在哪个节点上，所以你需要并行查询所有节点。

我们可以做得更好。让我们暂时假设你有一个简单的键值对数据模型，你总是通过主键访问条目。举个例子，在过时了的纸质百科全书中，你通过标题查找条目；由于所有的条目是按照标题的字母顺序排序的，你可以很快地找到你要找的那条。

### 按键的范围分区

一种分区的方式是指定一段连续范围的键（从某个最小值到某个最小值）到各个分区，就好像纸质百科全书（图6-2）的分册一样。如果你知道范围的边界，你可以很容易地判断哪个分区包含给定的键。如果你也知道哪个分区被指定到了哪个节点，那么你可以直接向恰当的节点直接发起请求（或者，在百科全书的例子中，从书架上选正确的那本）。

*图6-2 印刷版百科全书是按照键的范围分册的。*

键的范围不必要平均间隔开，因为你的数据不一定是平均分布的。举个例子，在图6-2中，第一册包含了以A和B开头的词，但是第12册包含了以T、U、V、X、Y和Z开头的词。如果每一册都只覆盖两个字母的话将导致某些册比其它一些要大很多。为了使数据均匀的分布，分区的范围需要适应数据本身。

分区边界可以是管理员手动选的，也可以数据自动选择的（我们会在“再平衡分区”一节详细讨论分区边界的选择）。这种分区策略用在了Bigtable，它的开源版本HBase，RethinkDB以及2.4版本前的MongoDB。

在每个分区中，我们可以把键按排好的顺序保存（见“SSTable与LSM树”）。这样做使得范围扫描变得容易，并且你可以把键当作连接后的索引从而在一个查询请求中获取多个相关的条目（见“多列索引”）。举个例子，设想一个储存传感器网络数据的应用，它的键是测量（*年-月-日-时-分-秒*）的时间戳。范围扫描在这种情况下特别有用，因为它让你很容易获得，比如，特定月份的所有读数。

然而，按键的范围分区的缺点是特定访问的模式会导致热点出现。如果键是时间戳，那么分区对应了时间范围——比如，每天对应一个分区。不幸的是，当测量发生时我们把来自传感器的数据写入数据库，所有写入最终都进入到了同一个分区（对应今天的那个），所以那个分区会被写入请求过载，而其它分区都闲置了。

为了避免传感器数据库里的这个问题，你需要使用时间戳意外的东西作为键的首元素。举个例子，你可以为每个时间戳加上传感器名称前缀，这样分区首先按照传感器名称再是时间。假设同时有许多活动的传感器，写入负载最终会更平均地分布在许多分区上。现在，当你需要获取某个时间段内多个传感器的值，你需要执行对每个传感器名称执行独立的范围查询。