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

### 按键的哈希分区

由于倾斜和热点的风险，许多风不是数据存储使用哈希函数来决定给定键的分区。

一个好的哈希函数接受倾斜的数据并使它均匀分布。假设你有个返回值为32位的接受字符串的哈希函数。每当你给它一个新字符串，它返回一个看似随机的，在0到2<sup>32</sup> - 1之间的数字。哪怕输入的字符串非常类似，它们的哈希值还是在数字范围内平均分布。

出于分区的目的，哈希函数没有必要是加密级别的强度：举个例子，Cassandra与MongoDB用的是MD5，而Voldemort用的是Fowler-Noll-Vo函数。许多编程语言都有内奸的简单哈希算法（因为被用在哈希表上），但是它们不太适合用来分区：举个例子，在Java的`Object.hashcode()`以及Ruby的`Object#hash`中，同样的键在不同的进程中有着不同的哈希值。

一旦你有了合适的用作键的哈希函数，你可以指定每个分区的哈希范围（而不是键的范围），哈希落在分区范围内的键将被存在那个分区里。如图6-3所示。

*图6-3 按键的哈希分区*

这个技巧可以很好地把键公平地分布到不同分区。分区边界可以均匀间隔的，或者可以假随机地选择（在这种情况下这个技巧被叫做*一致哈希*）

> 一致哈希
>
> 一致哈希，由戴维·卡尔热等人定义，是一种在互联网范围内的缓存系统——比如内容分发网络（CDN）——中均匀分布负载的方法。它使用随机选择的分区范围来避免对集中控制或者是分布式共识的需要。值得注意的是一致在这里与复制的一致性（见第五章）或是ACID一致性（见第七章）无关，它只是描述了再平衡的一种特别方式。
>
> 如我们即将在“再平衡分区”一节看到的，这种特别方式实际上对数据来说工作得并不是很好，所以事件中很少用到（一些数据库的说明文档仍然引用了一致哈希，但是通常并不准确）。因为这太令人困惑了，最好还是避免术语*一致哈希*而只是把它叫做哈希分区。

然而不幸的是，使用以键的哈希分区我们丢掉了以键的范围分区的一个良好特性：进行高效范围查询的能力。先前相邻的键现在分散到了所有分区里，于是排序次序丢失。在MongoDB中，如果你启用了基于哈希的分片模式，任何范围查询都必须发送到所有分区。Riak、Couchbase和Voldemort根本不支持主键上的范围查询。

Cassandra采取了两种分区策略的折中方案。Cassandra中的表声明时可以带上由数个列组成的复合主键。只有键的首部分哈希后用来决定分区，而其它列被用作Cassandra SSTable中数据排序的连接索引。因此查询请求不能搜索符合键首列中的一些列值，但是如果它为首列指明了一个固定值，他可以在键的其它列执行高效地范围扫描。

连接索引的方法使我们可以为一对多关系使用一种优雅的数据模型。举个例子，在社交媒体网站上，用户可以发布许多状态更新。如果更新的主键选的是`(user_id, update_timestamp)`，那么你可以高效地获取特定用户某个时间段内所有的状态更新，按时间戳排序。不同的用户可以储存在不同的分区，但是对于同一个用户，状态更新按照时间戳排序保存在单个分区上。

### 倾斜的工作负载与热点缓解

如上所述，以键的哈希来判定它的分区可以帮助减少热点。然而，它不能完全的避免特点：在极端情况下，所有的读写都是针对同一个键的，最终所有的请求还是被发送到了同一个分区。

这一类工作负载也许是不常见的，但是不是没有听说过：举个例子，在社交媒体网站上，当拥有数百万粉丝的名人用户做了些什么的时候可以导致一场活动风暴。这个事件可以引起大量对同一个键的写入请求（键也许是这个名人的用户ID，或者是人们评论的动作的ID）。取键的哈希没有帮助，因为两个同样ID的哈希仍然是一样的。

今天，绝大部分数据系统都无法为这样极度倾斜的工作负载自动补偿，所以降低倾斜是应用程序的职责。举个例子，如果一个键已知很热了，一个简单的技巧是添加一个随机数到键的开始或者末尾。只有两个数字的十进制随机数把对这个键的写入请求分到了100个不同的键上，使得这些键分不到了不同的分区。

然而，把写入分到了不同的键，任何读取请求现在需要做更多的工作了，因为它们必须读取所有100个键的数据并结合起来。这个技巧也需要更多的记录，对少数热键添加随机数字才有意义；对于大多数有着很低写入吞吐量的键这样做会产生没有必要的消耗。因而，你也需要某种记录哪些键被分开的方式。

也许在未来，数据系统可以自动检测并补偿倾斜的工作负载；但现在，你需要为你自己的应用程序仔细权衡。

## 分区与二级索引

我们目前讨论了的分区方法依赖键值对数据模型。如果记录只用主键访问过，我们可以从那个键判定分区，并且用它把读取和写入请求发送到键所对应的分区。

如果牵扯到二级索引的话情况就变得更复杂了（见“其它索引结构”）。二级索引通常不用来唯一标识一条记录，而只是一种搜索特定值的方式：找出用户123所有的操作，找到所有包含词语`hogwash`的文章，找出所有红色的车，等等等等。

二级索引是关系型数据库的核心产品，它在文档型数据库中也很常见。许多键值对存储（比如HBase与Voldemort）不支持二级索引是因为它增加了实现的复杂度，但是有些（比如Riak）开始支持它，因为它对于数据模型很有用。最后，二级索引是Solr和Elasticsearch等搜索服务器的*存在理由*。

二级索引的问题在于它们不能整齐地映射到分区。用二级索引对数据库分区有两种主要方式：基于文档分区与基于术语分区。

### 根据文档对二级索引分区

举个例子，设想你在运营一个售卖二手车的网站（如图6-4所示）。每个清单有一个唯一ID——叫它*文档ID*吧——然后你用文档ID把数据库分区了（举个例子，从0到499的ID在分区0，从500到999的ID在分区1，以此类推）。

你想让用户通过搜索找车，允许他们通过颜色和厂商过滤，所以你需要`color`和`make`的二级索引（在文档型数据库中这些可以是字段；在关系型数据库中这些是列）。如果你声明了索引，数据库可以自动建立索引。举个例子，每当一台红色的车被录入数据库，数据库分区自动把它添加到索引条目`color:red`的文档ID列表中。

*图6-4 根据文档对二级索引分区*

在这种索引方法中，每个分区是完全独立的：每个分区维护自己的二级索引，只覆盖分区中的文档。它不关心在其它分区储存了什么数据。每当需要写入数据库时——添加、删除或更新文档——你只需要处理包含着你在写入的文档ID的分区。由于这个原因，文档分区的索引也被称为*本地索引*（与下一节将描述的*全局索引*相对应）。

然而，从文档分区的索引读取需要很小心：除非对文档ID做了特殊处理，那么没有理由把所有具有特定颜色或特定品牌的车都会在同一个分区中。在图6-4中，红色的车出现在了分区0和分区1。因而，如果你要搜索红色的车，你需要发送查询请求到所有分区，然后把返回的所有结果结合起来。

这种查询分了区的数据库的方法叫做*分散/收集*，而这使得向二级索引发起读取查询的代价非常高。即使并发查询分区，分散/收集很容易出现尾部延迟放大效应（详见“实践中的百分位”）。然而，它被广泛地使用着：MongoDB、Riak、Cassandra、Elasticsearch、SolrCloud以及VoltDB全都使用文档分了区的二级索引。大多数数据库厂商推荐你构建自己的分区方法于是二级索引查询可以由单个分区响应，但是这部总是可能的，尤其是在你在单个查询中使用了多个二级索引（比如说同时按颜色和厂商过滤车）。

*图6-5 根据术语对二级索引分区*

### 根据字词对二级索引分区

与其每个分区有自己的二级索引（*本地索引*），我们可以构建覆盖所有分区数据的*全局索引*。然而，我们不能把这个索引只存在一个节点上，因为这很有可能变成平静进而失去了分区的意义。全局索引也必须被分区，但是可以与主键索引的分区不同。

图6-5展示了这大概会是什么样子：来自所有分区的红色的车都出现在索引的`color:red`下边，但是索引也被分区了，于是以字母*a*到*r*的开头的颜色出现在分区0，以字母*s*到*z*开头的颜色出现在分区1。汽车厂商的索引也是类似分区的（分区边界为*f*和*h*）。

我们把这种索引叫做*按字词分区*的索引，因为我们寻找的字词决定了索引的分区。在这个例子中，字词是`color:red`。*字词*这个名字来源于全文索引（一种特别的二级索引），在这里字词就是所有文章中出现过的词。

跟先前一样，我们可以按字词对索引分区，或者使用字词的哈希值。按字词本身分区在范围扫描（比如，对一个数字属性，比如询问车价）时很有用，而按字词的哈希值分区可以得到更平均分配的负载。

相比于按文档分区的索引，全局（按字词分区）索引的优点是它让读取更有效率：与其在所有分区上做分散/收集，客户端只需要向包含所需字词的分区发起请求。然而，全局索引的缺点在于写入更慢更复杂了，因为对单个文档的写入请求现在会影响索引的多个分区（文档中的每一个字词都会在不同的分区，不同的节点上）。

在理想的世界里，索引总是最新的，每个写入数据库的文档会立即反映在索引中。然而，在按字词分区的索引中，这需要一个横跨所有受写入影响的分区的分布式的事务，但是并不是所有数据库都支持它（见第七与第九章）。

在实践中，对全局二级索引的更新经常是异步的（那就是，如果你在写入之后没多久读取，你刚刚做出的变更不一定会体现在索引中）。举个例子，亚马逊的DynamoDB声明它的全局二级索引正常情况下会在几分之一秒钟内更新，但是如果基础架构发生故障，就很可能遇到更长的传播延迟。

全局按字词分区的索引的其它用途包括了Riak的搜索功能以及Oracle的数据仓库，它允许你在本地与全局索引之间选择。我们会在第十二章回到如何实现按字词分区的二级索引这个主题。

## 再平衡分区

随着时间的推移，数据库中的东西发生变化：

* 查询吞吐量增加，所以你要添加更多的CPU来应对负载。

* 数据集大小增加，所以你要添加更多的磁盘和内存来储存它。

* 设备故障，而其它设备需要接管故障设备的职责。

所以这些变化都需要把数据和请求从一个节点转移到另一个节点。把负载从集群中的一个系欸按移到另一个的过程叫做*再平衡*。

不管使用的是哪种分区方法，再平衡通常都要求满足某些最低标准：

* 再平衡之后，负载（数据存储，读写请求）应当在集群中的节点之间平均分配。

* 当再平衡进行时，数据库应当继续接受读写请求。

* 只有必须移动的数据才会在节点之间转移，从而使得再平衡过程很快并且只占用最少的带宽与磁盘I/O负载。

### 再平衡的策略

There are a few different ways of assigning partitions to nodes [23]. Let’s briefly discuss each in turn.

#### 如何不这样做: 哈希值余N

当按照键的哈希值分区时，我们之前说过（图6-3）最好把可能的哈希值分成不同的范围，并把每个范围指定到一个分区（比如，如果0 ≤ *hash(key)* < b<sub>0</sub>就把键指定到分区0，如果b<sub>0</sub> ≤ *hash(key)* < b<sub>1</sub>就指定到分区1，以此类推）。

也许你回想为什么不用*取余*（许多编程语言中的%操作符）。举个例子，*hash(key) mod* 10会返回会返回0到9之间的数字（如果我们把哈希值写作十进制的数字，那么哈希值取10的*余数*就是最后一位数字）。如果我们有10个节点，编号0到9，这看起来是一种指定键到节点的简单方法。

*取N余数*的方式问题在于如果节点数N发生变化，大多数的键需要从一个节点转移到另外一个节点。举个例子，假如*hash(key)* = 123456。如果一开始有10个节点，那么键最初在节点6（因为对123456取10的余数是6）。当增加到11个节点的时候，键需要转移到节点3（对123456取11的余数是3），当增加到12个节点的时候，键需要转移到节点0（对123456取12的余数是0）。如此频繁地迁移使得再平衡代价相当得高。

我们需要一种只在必要时移动数据地方法。

#### 固定数量的分区

幸运的是，有一个相当简单的解决方案：创建比节点数多很多的分区，并指定数个分区到每个节点。举个例子，一个运行在拥有10个节点的集群上的数据库从一开始就被拆分成1000个分区，于是大约100个分区被指定到了各个节点上。

现在，如果一个节点被添加到急群众，新节点可以从每个已有节点上偷一些分区过来，直到分区再一次平均分配为止。这个过程如图6-6所示。如果一个节点从集群中删除，同样的事情会反着发生。



Only entire partitions are moved between nodes. The number of partitions does not change, nor does the assignment of keys to partitions. The only thing that changes is the assignment of partitions to nodes. This change of assignment is not immediate — it takes some time to transfer a large amount of data over the network — so the old assignment of partitions is used for any reads and writes that happen while the transfer is in progress.

*Figure 6-6. Adding a new node to a database cluster with multiple partitions per node.*

In principle, you can even account for mismatched hardware in your cluster: by assigning more partitions to nodes that are more powerful, you can force those nodes to take a greater share of the load. 

This approach to rebalancing is used in Riak [15], Elasticsearch [24], Couchbase [10], and Voldemort [25]. 

In this configuration, the number of partitions is usually fixed when the database is first set up and not changed afterward.

Although in principle it’s possible to split and merge partitions (see the next section), a fixed number of partitions is operationally simpler, and so many fixed-partition databases choose not to implement partition splitting. Thus, the number of partitions configured at the outset is the maximum number of nodes you can have, so you need to choose it high enough to accommodate future growth. However, each partition also has management overhead, so it’s counterproductive to choose too high a number. 

Choosing the right number of partitions is difficult if the total size of the dataset is highly variable (for example, if it starts small but may grow much larger over time). Since each partition contains a fixed fraction of the total data, the size of each partition grows proportionally to the total amount of data in the cluster. If partitions are very large, rebalancing and recovery from node failures become expensive. But if partitions are too small, they incur too much overhead. The best performance is achieved when the size of partitions is “just right,” neither too big nor too small, which can be hard to achieve if the number of partitions is fixed but the dataset size varies.

#### Dynamic Partitioning

For databases that use key range partitioning (see “Partitioning by Key Range”), a fixed number of partitions with fixed boundaries would be very inconvenient: if you got the boundaries wrong, you could end up with all of the data in one partition and all of the other partitions empty. Reconfiguring the partition boundaries manually would be very tedious. 

For that reason, key range– partitioned databases such as HBase and RethinkDB create partitions dynamically. When a partition grows to exceed a configured size (on HBase, the default is 10   GB), it is split into two partitions so that approximately half of the data ends up on each side of the split [26]. Conversely, if lots of data is deleted and a partition shrinks below some threshold, it can be merged with an adjacent partition. This process is similar to what happens at the top level of a B-tree (see “B-Trees”). 

Each partition is assigned to one node, and each node can handle multiple partitions, like in the case of a fixed number of partitions. After a large partition has been split, one of its two halves can be transferred to another node in order to balance the load. In the case of HBase, the transfer of partition files happens through HDFS, the underlying distributed filesystem [3]. 

An advantage of dynamic partitioning is that the number of partitions adapts to the total data volume. If there is only a small amount of data, a small number of partitions is sufficient, so overheads are small; if there is a huge amount of data, the size of each individual partition is limited to a configurable maximum [23]. 

However, a caveat is that an empty database starts off with a single partition, since there is no a priori information about where to draw the partition boundaries. While the dataset is small — until it hits the point at which the first partition is split — all writes have to be processed by a single node while the other nodes sit idle. To mitigate this issue, HBase and MongoDB allow an initial set of partitions to be configured on an empty database (this is called pre-splitting). In the case of key-range partitioning, pre-splitting requires that you already know what the key distribution is going to look like [4, 26]. 

Dynamic partitioning is not only suitable for key range– partitioned data, but can equally well be used with hash-partitioned data. MongoDB since version 2.4 supports both key-range and hash partitioning, and it splits partitions dynamically in either case.

#### Partitioning proportionally to nodes

With dynamic partitioning, the number of partitions is proportional to the size of the dataset, since the splitting and merging processes keep the size of each partition between some fixed minimum and maximum. On the other hand, with a fixed number of partitions, the size of each partition is proportional to the size of the dataset. In both of these cases, the number of partitions is independent of the number of nodes. 

A third option, used by Cassandra and Ketama, is to make the number of partitions proportional to the number of nodes — in other words, to have a fixed number of partitions per node [23, 27, 28]. In this case, the size of each partition grows proportionally to the dataset size while the number of nodes remains unchanged, but when you increase the number of nodes, the partitions become smaller again. Since a larger data volume generally requires a larger number of nodes to store, this approach also keeps the size of each partition fairly stable. 

When a new node joins the cluster, it randomly chooses a fixed number of existing partitions to split, and then takes ownership of one half of each of those split partitions while leaving the other half of each partition in place. The randomization can produce unfair splits, but when averaged over a larger number of partitions (in Cassandra, 256 partitions per node by default), the new node ends up taking a fair share of the load from the existing nodes. Cassandra 3.0 introduced an alternative rebalancing algorithm that avoids unfair splits [29]. 

Picking partition boundaries randomly requires that hash-based partitioning is used (so the boundaries can be picked from the range of numbers produced by the hash function). Indeed, this approach corresponds most closely to the original definition of consistent hashing [7] (see “Consistent Hashing”). Newer hash functions can achieve a similar effect with lower metadata overhead [8].