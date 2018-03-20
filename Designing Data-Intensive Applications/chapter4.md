# 第四章 编码与演化

*世事皆变，无有静止*

以弗所的赫拉克利特，柏拉图在克堤拉斯篇引用的

---

应用程序不可避免地随时间而改变。因为新产品发布、用户需求的理解更加深入或者商业环境的改变，功能被添加被更改。在第一章中我们介绍了可进化性：我们应该着眼于构建可以轻松引入变更的系统（详见“可进化性：使变更变得简单”）。

在大多数情况下，应用功能的变更也需要对存储的数据变更：也许需要获取一个新的字段或者新的记录类型，也许也有数据需要以一种新的方式表达。

我们在第二章讨论的数据模型有不一样的方式应对这种变化。关系型数据库一般假设数据库中的所有数据符合一个模式定义：虽然这个模式定义可以改变（通过模式迁移；比如，`ALTER`语句），但是在任何一个时间点都只有一个模式定义生效。相比之下，只读模式（“无模式”）大数据不强制要求一个模式定义，所以数据库可以包含不同时间写入的新旧数据格式（见“文档模型中的模式定义灵活性”一节）。

当数据格式或是模式定义变化的时候，对应的应用代码也会发生变化（比如，为记录添加一个新字段，然后应用程序代码读写这个字段）。然而，在一个大的应用中，代码变化不会立即发生：

* 对服务器应用程序你大概想要执行*滚动升级*（也叫做*分阶段推出*），一次部署新版到一部分节点，检查新版本是不是平稳运行，然后逐步部署到所有的节点。这使得新版本的部署没有离线，因而估计更频繁的发布以及更好的演进。

* 对于客户端应用程序，您可能会受到用户的摆布，他们可能不会在一段时间内安装更新。

这意味着新旧版本代码，以及新旧版本的数据格式，很可能同一时间存在于系统中。为了使系统继续平稳运行，需要同时在两个方向保持兼容性：

*向后兼容*

新代码可以读取旧代码写入的数据。

*向前兼容*

旧代码可以读取新代码写入的数据。

向后兼容通常并不难实现：作为新代码的作者，你知道旧代码写入的数据格式，于是你可以显式处理它（如果必要只是让旧代码读取旧数据）。向前兼容会复杂一些，因为它需要旧代码忽略新代码带来的添加。

在这一章我们会了解几种数据编码的格式，包括JSON、XML、Protocol Buffers、Thrift以及Avro。特别的是，我们会了解它们是如何处理模式变更以及如何支持新旧数据与新旧代码需要共存的系统。然后我们会讨论这些格式是如何用于数据存储以及通讯的：在web服务中，具象状态传输（REST）与远程过程调用（RPC），以及消息传递系统，比如参与者与消息队列。

## 数据编码的格式

程序通常使用（至少）两种表现形式的数据：

1. 在内存中，数据以对象、结构体、链表、数组、哈希表、树等形式保存。这些数据结构都为CPU的高效访问与处理优化（一般通过使用指针）了。

2. 把数据写入文件或者通过网络发送的时候，必须把它编码为某种自成体系的字节序列（比如，JSON文档）。由于指针对其它进程没有任何意义，这个字节序列形式与内存中常见的数据结构非常不同。

因而，我们需要两种表现形式之间的转换方式。从内存表现形式转为字节序列叫做*编码*（也叫做*序列化*或者*编组*），相反的处理叫做*解码*（*解析，反序列化，反编组*）。

> ##### 术语冲突
>
> 不幸的是*序列化*在事务中（见第七章）也用到了，但是有完全不同的含义。为了避免重载这个词我们将在本书中继续使用*编码*这个词，即使*序列化*也许是更常见的术语。

由于这是一个常见问题，因此可以选择多种不同的库和编码格式。让我们来做一个简要的概述。

### 编程语言特有的格式

许多编程语言都内置支持把编码内存对象编码为字节序列。举个例子，Java有`java.io.Serializable`，Ruby有`Marshal`，Python有`pickle`，等等。也有许多第三方库，比如Java的Kryo。

这些编码库非常方便，因为它们允许用最少的代码保存和恢复内存中的对象。然而，它们也有一些深层次的问题：

* 编码经常与特定编程语言绑定，而用另外一种语言读取数据非常困难。如果用这样一种编码保存或传输数据，你投身于当前编程语言也许太长时间了，而且没有把你的系统与其他组织的系统（也许用了其他语言）集成在一起。

* 为了把数据恢复成同一种对象类型，解码过程需要能初始化任意类型。这常常是安全问题的源头：如果攻击者可以使你的应用解码任意字节流，它们可以初始化任意类型，这经常导致允许它们干坏事，比如远程执行任意代码。

* 在这些库中，版本控制数据往往是事后考虑的事情：因为它们旨在快速简便地编码数据，所以它们通常会忽略向前和向后兼容这样不便的问题。

* 效率（编解码花费的CPU时间，以及编码后结构的大小）往往也是事后考虑的问题。举个例子，Java内置的序列化因其糟糕的性能和臃肿的编码而恶名远扬。

由于这些原因，通常使用编程语言内建的编码都是一个糟糕的注意，除非是临时性的原因。

### JSON，XML，和二进制变体

把目光转到可以被许多编程语言读写的标准化编码，JSON与XML显然是竞争者。它们广为人知，被广泛支持，也几乎是广泛地不受欢迎。XML经常被人诟病太冗长，不必要地复杂。JSON受欢迎主要是因为浏览器的内建支持（由于是JavaScript的一个子集），相对于XML也简单。CSV是另一种流行的语言无关格式，尽管不那么强大。

JSON、XML以及CSV是基于文本的格式，因而某种程度上是可读的（虽然语法是一个热门争论话题）。除了表面的语法问题，还有一些细微的问题：

* 数字的编码有许多不确定性。在XML与CSV中，你无法区分数字与数字构成的字符串（除非通过引用外部的模式定义）。JSON区分字符串与数字，然而它不区分整型数与浮点数，而且也不会指明精度。

    处理大数字也有问题；比如说，大于253的整型数没有办法用IEEE 754双精度浮点数精确表示，于是这些数字被使用单精度数的语言（比如JavaScript）解析时就变得不精确了。这样的例子发生在推特，它们用一个64位数表示每一条推文。推特API返回的JSON包含了两个推文ID，一个是JSON数字，另一个是十进制数的字符串，来避开数字无法被JavaScript应用正确解析的事实。

* JSON与XML有着很好的Unicode字符串支持（也就是可读文本），但是它们不支持二进制字符串（没有字符编码的字节序列）。二进制字符串是很有用的功能，于是人们通过把二进制数据以Base64编码为文本从而绕开这个局限。之后模式定义指出这个值应当被解读为Base64编码的。这样可以工作，但是从某种程度上说这样效果不好，数据尺寸大小增加了33%。

* XML与JSON都支持可选的模式定义。模式定义语言非常强大，也因而学习实现圈起来非常复杂。XML模式定义的使用相当广泛，但是需要基于JSON的工具懒得去使用模式定义。由于数据正确的解读（比如数字与二进制字符串）取决于模式定义中的信息，不使用XML/JSON的应用需要硬编码合适的编解码逻辑。

* CSV没有任何模式定义，所以每一行每一列的含义取决于应用程序的定义。如果应用变更添加新的行或者列，你需要手动处理这个变更。CSV也是种相当模糊的格式（如果一个值包含逗号或者新行字符会发生什么？）。虽然转义字符规则已经正式定义了，并不是所有的解析器都正确地实现了它们。

除了这些瑕疵，JSON、XML以及CSV对于许多用途来说都足够好了。很有可能它们会继续流行下去，特别是作为数据交换格式（比如从一个组织向另外一个组织发送数据）。在这样的情况下，只要人们一致同意使用哪种格式，格式本身是否漂亮或者高效经常是无所谓的。让不同的组织一致同意某件事的难度可比其它问题高多了。

#### 二进制编码

对于那些只在组织内部使用的数据，必须使用最小公分母编码格式的压力小了很多。比如，可以选择一种更紧凑或者解析更快的格式。对于小数据集，收益可以忽略不计，然而一旦上涨到TB级别，数据格式的选择就会产生重大影响。

相比于XML，JSON没有那么冗长，但是相比于二进制格式二者还是用到了许多空格。这导致了针对JSON（比如MessagePack、BSON、BJSON、UBJSON、BISON和Smile）与XML（比如WBXML与Fast Infoset）的二进制编码的大量开发。这些格式被引入了众多生态中，但是它们中没有一个如文本版本的JSON与XML一样被广泛采用。

这些格式中的一部分扩展了数据类型集合（比如，区分了整型数与浮点数，或者添加了对二进制字符串的支持），其它的则保留了JSON/XML数据模型没有变。特别的是，由于没有规定模式定义，它们需要在编码后的数据中包含所有的对象字段名。也就是说，在示例4-1的JSON文档的二进制编码中，需要在某个地方包含字符串`userName`、`favoriteNumber`、`interests`。

*示例4-1 示例记录，这一章中我们会用几种二进制格式对它编码*

```JSON
{
    "userName": "Martin",
    "favoriteNumber": 1337,
    "interests": ["daydreaming", "hacking"]
}
```

让我们来看MessagePack，一种JSON的二进制编码的例子。图4-1展示了用MessagePack编码示例4-1中JSON文档后得到的字节序列。最初的几个字节是：

1. 第一个字节，`0x83`，表示紧跟着的是一个对象（高4位 = `0x80`），有三个字段（低4位 = `0x03`）。（加入你在想如果对象有超过15个字段会发生什么，那么字段数用4位表示不了了，于是它有另外一个类型标识，而字段数被编码为了2到4个字节。）

2. 第二个字节，`0xa8`，表示紧跟着的是一个字符串（高4位 = `0xa0`），有八个字节长（低4位 = `0x08`）。

3. 接下来八个字节是ASCII编码的字段名`userName`。由于长度先前指明了，于是不需要任何标记位来告诉我们字符串在哪里结束（或是任何转义）。

4. 解析来七个字节编码了六个字母的字符串值`Martin`以及前缀`0xa6`，等等。

这个二进制编码有66个字节长，只比81个字节长的文本JSON（去掉了空格）编码少了一点点。所有JSON的二进制编码在这方面都类似。不确定这么小的空间节省（以及解析速度的提升）值不值得放弃可读性。

在接下来的几节我们会看到如何做得更好，同样的记录只用32个字节编码。

*图4-1 用MessagePack编码示例记录（示例4-1）*

### Thrift与Protocol Buffers
