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

Apache Thrift与Protocol Buffers（protobuf）是基于同样原则二进制编码库。Protocol Buffers最初由谷歌开发，Thrift最初由Facebook开发，二者在2007到08年之间都开源了。

对于编码的数据，Thrift与Protocol Buffers二者都需要一个模式定义。为了用Thrift编码示例4-1中的数据，需要用Thrift接口定义语言（IDL）描述模式定义，类似这样：

```IDL
struct Person {
    1: required string          userName,
    2: optional i64             favoriteNumber,
    3: optional list < string > interests
}
```

等价的Protocol Buffers的模式定义看起来很类似：

```protobuf
message Person {
    required string user_name       = 1;
    optional int64  favorite_number = 2;
    repeated string interests       = 3;
}
```

Thrift与Protocol Buffers都自带代码生成工具，它接受一个类似这样的模式定义，然后用不同编程语言生成实现这个模式定义的类。应用程序可以调用这个生成的代码来编解码模式定义对应的记录。

用这个模式定义编码的数据是什么样子的？令人困惑的是，Thrift有两种不同的二进制编码格式，分别叫做*BinaryProtocol*与*CompactProtocol*。我们先来看BinaryProtocol。用这种格式编码示例4-1需要59个字节，如图4-2所示。

*图4-2 用Thrift的BinaryProtocol编码示例条目*

与图4-1类似，每个字段有一个类型标记（指出这是一个字符串、整型数、链表等等）以及必有的长度标记（字符串长度，链表中项目个数）。数据中出先得字符串（“Martin”、“Daydreaming”、“hacking”）也被编码为ASCII码（或者，UTF-8），与之前类似。

与图4-1的最大不同在于这里没有字段名（`userName`、`favoriteNumber`、`interests`）。相反，编码后的数据包含字段标签，它们是数字（`1`、`2`和`3`）。这些数字出现在模式定义中。字段标签类似字段的别名——这是一种描述字段的紧凑方式，不需要把字段名描述出来。

Thrift的CompactProtocol编码语义上与BinaryProtocol等价，但如你在图4-3中看到的，它打包同样的信息只用了34个字节。它通过把字段类型与标签打包到一个字节做到的，以及使用变化长度的整型数。数字1337不是使用全部八个字节，而是被编码为两个字节，其中每个字节的高位用来表示之后是否还有更多的字节需要处理。这意味着表示位于-64到63的数字只需要一个字节，表示位于-8192到8191的数字只需要两个字节，等等。更大的数字需要更多的字节。

*图4-3 用Thrift的CompactProtocol编码示例条目*

最后，Protocol Buffers（只有一种二进制编码格式）编码同样的数据如图4-4所示。它做打包位的方式有些许不同，但是与Thrift的CompactProtocol非常类似。Protocol Buffers用了33个字节编码了同样的条目。

*图4-4 用Protocol Buffers编码示例条目*

注意一个细节：早先展示的模式定义中，每个字段都标记了`required`或者`optional`，但是这不影响字段是如何编码的（二进制数据中没有办法表示一个字段是否为必须的）。区别只是在于`required`可以启动运行时检查，如果字段没有设置将会失败，方便捕获bug。

#### 字段标签与模式定义演进

我们之前说，模式定义无法避免地随着时间推移需要变更。我们把这叫做模式定义演进。Thrift与Protocol Buffers是如何处理模式定义变更且同时保持向后以及向前兼容地呢？

正如之前例子看到的，编码后的记录只是编码后字段的连接。每个字段通过它的标签号码标记（示例模式定义中的数字1，2，3）并用数据类型标注（例如字符串或者整型数）。如果一个字段值没有设，它只是简单地被编码后的条目忽略了。从这里你可以看出字段标签对于编码后数据的含义至关重要。你可以在模式定义中改变一个字段的名字，毕竟编码后的数据从不引用字段名字，但是你不能改变一个字段的标签，因为这将使得所有已经存在的编码后数据无效。

你可以添加新的字段到模式定义中，前提是你给每一个字段一个新的标签号码。如果旧代码（不知道你添加的新标签好吗）尝试读取新代码写入的数据，其中包含它不能识别的标签号码对应的字段，它可以简单地忽略那个字段。数据类型标记允许解析器判定多少字节需要忽略。这保持了向前兼容性：旧代码可以读取新代码写入的数据。

那么向后兼容性呢？只要每一个字段都有一个唯一的标签号码，新代码就总可以读取旧数据，因为标签号码依旧有相同的含义。唯一的细节在于如果你添加一个新字段，你无法要求它是强制的。否则，强制检查会失败，因为就代码不会写你新添加的字段。因而为了保持向后兼容性，每个模式定义初次部署之后添加的字段都必须是可选的，或者必须有默认值。

删除字段与添加字段类似，只是向前向后兼容性考虑反过来了。这意味着你只可以移除可选字段（强制字段不能被移除），也用用不能再使用同一个标签号码（因为有可能还有某处的数据包含旧的标签号码，而新代码必须忽略这个字段）。

#### 数据类型与模式定义演进

那改变字段的数据类型呢？这是由可能的——检查说明文档获取详细信息——但是有值丢失精度或者被截断的风险。举个例子，假设你把32位整型数变为64位整型数。新的代码可以很容易地读取旧代码写入地数据，因为解析器可以在缺失的位填入零。然而，如果就代码读取新代码写入的数据，旧代码仍然用32位变量储存值。如果解码后的64位值用32位放不下，它就会被截断了。

Protocol Buffers一个奇怪的细节是它没有链表或者数组数据类型，取而代之的是一个字段的`repeated`标记（它是与`required`和`optional`平齐的第三选项）。如你在图4-4中看到的，`repeated`字段的编码正如它的字面意思：条目中同一个字段标签出现了多次。这样做有一个不错的效果，就是可以把`optional`（单值）字段变成一个`repeated`（多值）字段。新代码读取旧数据看到的是一个有着零或一个元素的链表（取决于字段是否出现）；旧代码读取新数据只看到链表的最后一个元素。

Thrift有专门的链表数据类型，它可以被链表元素的数据类型参数化。类似Protocol Buffers的单值到多值演化是不允许的，但是它有它的优势：支持嵌套的链表。

### Avro

Apache Avro是另外一种二进制编码格式，与Protocol Buffers以及Thrift不同。作为Hadoop的子项目它启动于2009年，原因是Thrift不适合Hadoop的使用场景。

Avro也使用模式定义指明被编码数据的结构。它有两种模式定义语言：一种（Avro接口定义语言）是为了方便人们编辑，另一种（基于JSON）机器读取时更容易。

我们的示例模式定义，用Avro接口定义语言写的话，大概是这个样子的：

```Avro
record Person {
    string                  userName;
    union { null, long }    favoriteNumber = null;
    array<string>           interests;
}

```

与这个模式定义等价的JSON表示如下：

```JSON
{
    "type": "record",
    "name": "Person",
    "fields": [
        {"name": "userName",       "type": "string"},
        {"name": "favoriteNumber", "type": ["null", "long"], "default": null},
        {"name": "interests",      "type": {"type": "array", "items": "string"}}
    ]
}
```

首先，注意模式定义中没有标签号码。如果用这个模式定义编码示例条目（示例4-1），Avro二进制编码只有32个字节长——所有我们看到的编码中最紧凑的。编码后的字节序列的拆解如图4-5所示。

如果仔细检查字节序列，你会发现没有信息表示字段或是字段数据类型。编码只是简单地把值连接在了一起。字符串只是长度前缀外跟UTF-8字节，但是编码后地数据中没有信息告诉你这是一个字符串。它也可以是一个整型数，或者完全另外一个东西。整型数使用的是变长编码（这与Thrift的CompactProtocol一致）。

*图4-5 用Avro编码的示例条目*

为了解析二进制数据，按它们在模式定义中的次序遍历所有字段，然后使用模式定义告诉你每个字段的数据类型。这意味着二进制数据只有在读取数据的代码使用写入数据代码使用的一摸一样的模式定义才能正确解码。任何读写模式定义的不匹配都意味着解码错误。

那么Avro如何支持模式定义演进呢？

#### 写者模式定义与读者模式定义

在Avro，当应用想要编码某些数据时（把它写入到文件或者数据库，通过网络把它发送出去，等等），它使用它知道的任意版本的模式定义编码数据——比如说，编译到应用内部的模式定义。这叫做写者模式定义。

当应用要解码某些数据时（从文件或者数据库中读取，从网络上获得，等等），我们希望数据是符合某种模式定义的，它被称为读者模式定义。这是应用代码依赖的模式定义——代码也许是应用构建过程中由模式定义生成的。

Avro的核心思想是写者模式定义与读者模式定义不必完全相同——它们只需要兼容。当数据被解码（被读取）时，Avro库通过比对写者模式定义与读者模式定义解析差异，然后把数据从写者模式定义转化为读者模式定义。Avro的规范明确定义了解析如何工作，如图4-6所示。

举个例子，写者模式定义与读者模式定义的字段顺序不同是不会有问题的，因为模式定义解析通过字段名匹配。如果读取数据的代码遇到了一个出现在写者模式定义但是读者模式定义没有的字段，就忽略它。如果读取数据的代码期望某个字段，但是写者模式定义不包含名称对应的字段，那么字段会被填入读者模式定义声明的默认值。

*图4-6 Avro读者解析写者模式定义与读者模式定义*

#### 模式定义演进规则

在Avro，向前兼容意味着可以有新版本的模式定义作为写者，而旧版本的模式定义作为读者。相反的，向后兼容意味着可以有新版本的模式定义作为读者，而旧版本的作为写者。

为了维持兼容性，你只能添加或者删除有默认值的字段。（在示例的Avro模式定义中字段`favoriteNumber`有默认值为`null`。）举个例子，假设你添加一个有默认值的字段，那么这个新字段在新模式定义中存在而旧的没有。当读者用新模式定义读取用旧模式定义写入的记录，缺少的字段将填入默认值。

如果要添加一个没有默认值的字段，新的读者没有能力读取旧写者写入的数据，于是破坏了向后兼容性。如果删除了一个没有默认值的字段，旧的读者没有能力读取新写者写入的数据，于是破坏了向前兼容性。

在一些编程语言中，`null`是任意变量都可以接受的默认值，但是对于Avro来说不是的：如果要允许一个字段为空，你必须用*联合类型*。举个例子，`union { null, long, string } field;`表示这个字段可以是数字，字符串，或者空。只有在它是联合中的一个分支时你才可以用`null`作为默认值。这比所有字段都可以默认为空要冗余了些，但是通过明确什么可以为空什么不可以达到了防止错误的目的。

结果是，Avro没有Protocol Buffers和Thrift中有的`optional`和`required`标记（取而代之的是联合类型与默认值）。

改变字段的数据类型是可能的，前提是Avro可以转换这个类型。改变字段的名称是可能的，但是稍微复杂一些：读者的模式定义可以包含字段名称的别名，于是可以把旧写者的模式定义名称与这些别名匹配。这意味改变字段名称是向后兼容的，但不是向前兼容的。类似的，添加一个分支到联合类型是向后兼容但不是向前兼容的。

#### 但是，什么是写者模式定义？

有一个重要问题之前被我们一边带过：读者是怎么知道编码特定数据的写者模式定义的？我们不可能把整个模式定义包含在每条记录里，因为模式定义很可能比编码后的数据大多了，使得所有由于二进制编码而节省下来的空间都白费了。

答案取决于使用Avro的上下文。下面给出了一些例子：

*有许多条记录的大文件*

Avro的一种常见使用方式——尤其是在Hadoop上下文中——是用来存储包含上百万条记录的大文件，这些记录都用同样的模式定义进行了编码。（我们将在第十章讨论这种情况。）在这种情况下，文件的写者只用在文件开头包含一次写者模式定义。Avro定义了一个文件格式（对象容器文件）来做这件事。

*包含独立写入记录的数据库*

在数据库中，不同的记录也许是用不同写者模式定义在不同的时间点写入的——你不能假设所有的记录都有同样的模式定义。最简单的解决办法是在每条编码记录前加上版本号，同时在数据库中保存模式定义版本列表。读者能获取一条记录，抽取版本号，然后从数据库中获取对应的写者模式定义。使用这个写者模式定义，它可以解码其余的记录。（比如说，Espresso就是这样工作的。）

*通过网络连接发送记录*

当两个进程通过双向网络连接通信时，它们可以在建立连接时协商模式定义版本，之后在这个连接生命周期内使用这个定义。Avro远程过程调用协议（见“TODO：透过服务的数据流：REST和RPC”）是这样工作的。

任何情况下数据库的模式定义版本都是有用的，因为它就像说明文档，给机会检查模式定义兼容性。作为版本号，你可以使用简单的自增整型数，或者使用模式定义的哈希值。

#### 动态生成的模式定义

相比于Protocol Buffers和Thrift，Avro方式的优势是模式定义不包含任何标签号码。但是为什么这很重要呢？在模式定义中保留几个号码有什么问题呢？

差别在于Avro对动态生成的模式定义更加友好。举个例子，假设有一个关系型数据库，想把内容导出到文件，而且想用二进制格式从而避免文本格式（JSON、CSV、SQL）前面提到的问题。如果用Avro，从关系型模式定义生成Avro模式定义（用先前看到的JSON表现形式）是相当简单的，再用这个模式定义编码数据库内容，并导出到Avro对象容器文件。每个数据库表生成一个记录模式定义，而每个列变成了记录中的一个字段。数据库中列的名字对应成为了Avro中的字段名字。

现在，假如数据库模式定义变了（比如，表中添加了一列又删除了一列），你只需要从更新了的数据库模式定义生成一个新的Avro模式定义，然后用新的Avro模式定义导出数据。数据导出过程不需要关心模式定义的变化——只是每次运行简单地做模式定义转换。任何读取新数据文件的人将看到记录的字段发生了变化，然而由于字段是通过名字标记的，更新了的写者模式定义仍然可以与旧的读者定义匹配。

相比之下，如果用Thrift或者Protocol Buffers，字段标签很可能需要手动分配：每一次数据库模式定义变化的时候，管理员必须手动更新从数据库列名称到字段标签的映射关系。（这个过程有可能可以自动化，但是模式定义生成器必须非常小心，不能分配先前使用过的字段标记。）这种动态生成的模式定义不是Thrift或者Protocol Buffers的设计目标，但是是Avro的。

#### 代码生成与动态类型语言

Thrift与Protocol Buffers依赖代码生成：模式定义完毕之后，可以用你选择的编程语言生成实现了这个模式定义的代码。这对于比如Java、C++或C#之类的静态类型语言很有用，因为它可以使用高效的内存结构储存解码后的数据，在编写访问数据结构的代码时也使IDE中的类型检查与自动完成可用。

在类似JavaScript、Ruby或Python的动态类型语言中，代码生成就没有太大意义了，因为没有编译时类型检查问题。代码生成在这些语言中经常被忽视，因为它们明确地不需要编译这一步。此外，对于动态生成的模式定义（比如从数据库表中生成的Avro模式定义），代码生成对于获取数据来说完全没有必要。

虽然Avro对静态类型编程语言提供了可选的代码生成功能，但是没有它也可以良好工作。如果你有一个对象容器文件（其中嵌入了写者的模式定义），你可以用Avro库打开它然后用看JSON文件的同样方式看这些数据。这个文件是自描述的，因为它包含了所有必须的元数据。

这个特性在结合例如Apache Pig这样的动态类型数据处理语言时特别有用。在Pig中，你只需要打开Avro文件，开始分析它们，然后用Avro格式写出导出的数据集到输出文件而完全不用在意模式定义。

### 模式的优点

如我们所见，Protocol Buffers、Thrift以及Avro都使用模式定义来描述二进制编码格式。它们的模式定义语言要比XML模式或是JSON模式简单得多，后者支持许多细节验证规则（比如，“这个字段的字符串值必须符合这个正则表达式”或者“这个字段的整型数值必须在0到100之间”）。由于Protocol Buffers、Thrift以及Avro实现简单用起来也简单，它们已经支持了相当多数的编程语言。

这些编码所基于的理念绝非新鲜。举个例子，它们与ASN.1，一种在1984年首次标准化了的模式定义语言，有许多共同点。它被用来定义了许多网络协议，比如它的二进制编码（DER）仍然被用来编码SSL证书（X.509）。ASN.1通过标签号码支持模式演进，这与Protocol Buffers以及Thrift类似。然而，它也很复杂，文档很差，所以ASN.1也许对于新的应用来说不是一个很好的选择。

许多数据系统也为它们的数据实现了某些专有二进制编码。比如说，绝大部分关系型数据库都定义了网络协议，通过它可以发送查询请求到数据库并获取响应。这些协议通常是针对特定数据库的，而数据库厂商提供驱动（比如通过ODBC或者JDBC的API）把来自数据库网络协议的响应解码为内存数据结构。

所以，我们看到虽然文本的数据格式例如JSON、XML以及CSV被广泛使用，基于模式定义的二进制编码也是一个可选项。它们有一些不错的特性：

* 它们比诸多“二进制JSON”变种要紧凑得多，因为它们在编码后的数据中忽略字段名。

* 模式定义是有价值的文档形式，由于解码时模式定义是必须的，你可以确定它一定是最新的（而人为维护的文档很容易与实际情况不符）。

* 保存模式定义的数据库方便在部署任何东西之前检查模式定义变更的向前与向后兼容性。

* 对于静态类型编程语言的用户，从模式定义生成代码的功能很有用，因为它允许在编译时进行类型检查。

总的来说，模式定义演进提供了与无模式定义/读时模式定义JSON数据库提供的同等灵活性（见“文档模型中的模式定义灵活性”），同时提供了数据更好保证以及更好的工具。

### 数据流的模式

在本章开始时我们说无论何时传输数据到另外一个不共享内存的进程——比如，无论何时通过网络传输数据或者写入数据到文件——你需要把它编码成字节序列。然后为了做这件事，我们讨论了许多不同的编码。

我们谈到了向前和向后兼容性，这对演进能力（通过允许独立升级系统内不同部件，而不必同时升级所有部件使得变更变得简单）很重要。兼容性是编码数据进程与解码数据进程之间的关系。

这是相当抽象的想法——数据从一个进程流向另一个进程有许多种方式。谁编码了数据，而谁解码了它？本章的余下部分我们会探索几种最常见的进程间数据流动的方式：

* 通过数据库（见“通过数据库的数据流”）

* 通过服务调用（见“通过服务的数据流：REST与RPC”）

* 通过异步消息传递（见“消息传递的数据流”）

#### 通过数据库的数据流

在数据库中，写入数据库的进程编码数据，然后读取数据库的进程解码它。也许只有一个进程在访问数据库，这时读者只是相同进程的稍后版本——这种情况下你可以认为存储数据到数据库就是发送信息给未来的自己。

向后兼容在这里自然是必须的；否则未来的自己没办法解码之前写入的数据。一般地，几个不同进程同时访问数据库是很常见的。这些进程也许是几个不同的应用程序或者服务，或者它们是同一个服务的几个示例（为了扩展性或容错性并行运行着）。任何一种方式，在应用变化的环境中，很有可能一些进程使用新代码访问数据库而另外一些运行着旧代码——比如说因为新版本正在以滚动升级的方式部署，那么其中一些实例已经更新了而其它的还没有。

这意味着数据库中的值有可能是新代码写入的，而之后被仍在运行的就代码读取。因此向前兼容对于数据库来说也常常是必须的。

然而，还有另外一个障碍。假设添加字段到记录模式定义，然后新代码为这个新字段写值到数据库。然后，旧代码（对新字段一无所知）读取记录，更新它，并写回数据库。在这种情况下，理想的行为通常是就代码保持新字段不变，即使它无法被解读。

之前讨论的编码格式都支持这种未知字段保留，但是有时你需要在应用级别处理它，如图4-7所示。举个例子，如果你在应用中解码数据库值到模型对象，之后重新编码这些模型对象，在这个转换过程中未知字段有可能被丢掉。解决这个问题并不难，你只是需要注意到这个问题。

*图4-7 当旧版本的应用更新先前被新版本应用写入的数据时，如果不小心的话数据会丢失*

#### 不同时间写入的不同的值

数据库通常允许在任意时间更新任意值。这意味着在一个数据库中某些值是五毫秒前写入的，而有些值是五年前写入的。

当部署应用的新版本时（至少是服务端），也许在几分钟之内就用新版本完全替换了旧版本。对于数据库内容来说不是这样的：五年了的数据会依然在那，还是原来的编码，除非那个时候就明确地重写了。这一观察结果有时总结为*数据较代码长寿*。

重写（*迁移*）数据到新的模式定义当然是可能的，但是在大的数据集上这样做是代价昂贵的，所以绝大多数数据库尽可能避开它。绝大多数关系型数据库允许允许简单的模式定义改变，比如添加新的默认值为`null`的列而不复写已经存在的数据。当旧的行被读取，数据库将为磁盘上编了码的数据缺失的列填充`null`。LinkedIn的文档数据库Espresso为存储而使用了Avro，使得它可以使用Avro的模式定义演进规则。

模式定义演进因而让整个数据库看起来仿佛被用单个模式定义编码，即使底层存储也许包含了多种历史版本模式定义编码的记录。

#### 档案存储

也许你会时不时地获取你的数据库的快照，比如为了备份或是为了加载到数据仓库中（见“数据仓库”一节）。在这种情况下，导出的数据一般使用最新的模式定义编码，即使源数据库中原始的编码包含了不同时期不同版本的模式定义混合。既然无论如何都在拷贝数据，也许你也同样地编码了数据的拷贝。

由于导出的数据一次性写出，因而是不可改变的，类似Avro对象容器文件的格式很合适。同样这也是一个好机会来用适合分析的面向列的格式编码数据，比如Parquet（见“列压缩”一节）。

在第十章我们将讨论使用档案存储中的数据。

### 通过服务的数据流：REST与RPC

当进程需要通过网络通讯时，有几种不同的方式部署这种通讯。最常见的部署是由两个角色：*客户端*与*服务端*。服务器在网络中暴露API，而客户端可以连接服务器从而向API发出请求。服务器暴露的API被称作*服务*。

网络是这样工作的：客户端（浏览器）发起请求到网络服务器，发起`GET`请求下载HTML、CSS、JavaScript、图片等等，发起`POST`请求提交数据给服务器。API包含一套标准化的协议与数据格式（HTTP、URL、SSL/TLS、HTML等等）。因为网络浏览器，网络服务器以及网站作者在这些标准上达成大致一致，你可以用任意网络浏览器访问任意网站（至少理论上是这样！）。

网络浏览器不是客户端仅有的类型。举个例子，运行在移动设备或是桌面电脑上的原生应用也可以发起网络请求到服务器，而运行在网络浏览器中的客户端JavaScript应用可以用XMLHttpRequest变成HTTP客户端（这种技术叫做*Ajax*）。在这种情况下，服务器的响应通常不是可以显示给人看的HTML，而是为了方便客户端应用代码进一步处理，编码了的数据（比如JSON）。虽然HTTP可以被用做传输层协议，然而实现在其上层的API是针对特定应用的，而客户端与服务端需要在API的细节上达成一致。

此外，服务器自己可以是其它服务的客户端（比如，典型网络应用服务器是数据库的客户端）。这种方式通常用来把大型应用根据应用领域拆解为小的服务，比如一个服务当它需要来自其它服务的功能或者数据，于是向另外一个服务发起请求。这种构建应用的方式传统上叫做*面向服务的架构*（SOA），最近又被提炼加工为*微服务架构*。

某种方式上，服务与数据库类似：它们通常允许客户端提交、查询数据。然而数据库允许使用第二章讨论的查询语言进行任意的查询，但是服务暴露的应用程序特定的API只允许服务的业务逻辑预先确定的输入输出。这种限制提供了一定程度的封装：服务可以对客户端能做和不能做什么施加精细的限制。

面向服务或是微服务架构的关键设计目标是通过使服务可独立部署和可演进，使应用容易修改和维护。举个例子，每个服务应该被一个小组负责，这个小组可以频繁发布服务的新版本，而无须与其它小组协调。换句话说，我们应该期望新旧版本的服务端与客户端同时运行，于是服务端与客户端使用的数据编码必须在多个版本服务API之间兼容——这正是我们在这一章讨论的东西。

#### 网络服务

当HTTP被用作与服务沟通的底层协议，这叫做*网络服务*。也许这稍稍用词不当，因为网络服务并不是只用于网络，也用于几种不同的上下文中。比如：

1. 运行在用户设备上的客户端程序通过HTTP发出请求到服务。这些请求通常跨越公共互联网。

2. 一个服务向同一组织拥有的另一个服务发起请求，这些服务经常位于同一个数据中心内，作为面向服务/微服务架构的一部分。（支持这种使用场景的软件叫做*中间件*）

3. 一个服务向其它组织拥有的一个服务发起请求，通常经过互联网。这经常用于不同组织后台系统之间的数据交换。这一类包括在线服务提供的公用API，比如信用卡处理系统，或者数据用户共享访问的OAuth。

网络服务有两种受欢迎的方式：*REST*与*SOAP*。从哲学角度看它们几乎完全相反，经常成为它们各自支持者之间激烈辩论的主题。

REST不是一种协议，而是一种基于HTTP原则的设计哲学。它强调简单的数据格式，利用URL标识资源并使用HTTP功能进行缓存控制、身份验证以及内容类型协商。相比于SOAP，REST越来越受欢迎，至少是在跨组织服务继承的时候，并且常常与微服务结合在一起。根据REST原则设计的API被叫做*RESTful*。

相比之下，SOAP是种基于XML的协议，用来发起网络API请求。虽然最常用于HTTP，但是它的目标是独立于HTTP，避免使用大部分HTTP功能。取而代之的是，它带有庞大而复杂的多种相关标准（网络服务框架，称为WS-*）用于添加各种功能。

SOAP网络服务的API是用一种基于XML的语言描述的，叫做网络服务描述语言，或是WSDL。WSDL支持代码生成，于是客户端可以通过本地类和方法调用（被编码为XML消息，稍后再被框架解码）访问远端服务。这对静态类型编程语言很有用，但是对动态类型语言就不是那么有用了（见“代码生成与动态类型语言”一节）。

由于WSDL不是为人类可读设计的，而SOAP消息对手动构造来说经常太过复杂，SOAP用户非常依赖工具的支持，代码生成，以及IDE。对于SOAP厂商不支持的编程语言的用户来说，与SOAP服务的集成是很困难的。

虽然SOAP与它众多的扩展表面上标准化了，但是不同厂商实现之间的交互经常导致问题。因为这些原因，虽然SOAP仍然在许多大型企业中使用，在大部分小公司中已经失宠了。

RESTful API更倾向选择简单的方式，通常更少依赖代码生成和自动化工具。定义格式比如OpenAPI，也叫做Swagger，可以被用来描述RESTful API并且生成说明文档。

#### 远程过程调用（RPC）的问题

Web服务仅仅是通过网络进行API请求的一系列技术的最新版本，其中许多技术接受了大量炒作但是存在严重的问题。企业级JavaBean（EJB）以及Java的Remote Method Invocation（RMI）仅限Java使用。Distributed Component Object Model（DCOM）仅限微软平台。通用对象请求代理架构（CORBA）非常得复杂，也不提供向前向后兼容性

所有这些都是基于*远程过程调用*（RPC）的理念，它再1970年代就出现了。RPC模型尝试把向远端网络服务发起的请求看上去与调用同一个进程内的（这种抽象叫做*位置透明*）函数或者方法一样。虽然一开始RPC看起来很方便，但是这种方式本质上有缺陷。网络请求与本地函数调用是非常不一样的：

* 本地函数调用是可预测的，成功或者失败只取决于你控制的参数。网络请求是不可预测的：请求或者响应也许会因为网络原因丢失，或者是远端设备运行缓慢或者不可用，而这些问题全部超出你控制的范围。网络问题很常见，所以你必须预计它们的出现，比如说需要重试失败了的请求。

* 本地函数调用要么返回一个值，要么抛出一个一场，或者永远不返回（因为进入了一个无限循环或者进程崩溃）。网络请求有另外一种可能结果：返回没有结果，因为超时了。在这种情况下，你完全不知道发生了什么：如果你无法从远端服务获得一个响应，你没办法知道到底请求发送过去了没有（我们会在第八章具体讨论这个问题。）

* 如果重发一个失败的网络请求，可能发生的是请求实际上发送过去了，只是响应被丢掉了。这种情况下，重试会导致动作执行好几遍，除非构建一种去重复（幂等）的机制到协议中。本地函数调用没有这样的问题（我们会在第十一章具体讨论幂等）

* 每一次调用本地函数，执行通常会花去同样的时间。网络请求比函数调用要慢得多，而且它得延迟也是变化非常的：好的时候也许可以在一毫秒之内完成，但是当网络拥挤或者远端服务过载时也许会花去几秒钟才能完成一样的事。

* 调用本地函数的时候，你可以高效地传入本地内存中地对象引用。当发送网络请求时，所有这些参数需要被编码为可以通过网络发送出去的字节序列。如果参数是比如数字或者字符串这样的原始数据类型时是可以的，但是对于更大的对象来说这马上变成了一个问题。

* 客户端与服务可以使用不同的变成语言实现，所以RPC框架必须把数据类型从一种语言翻译成另外一种。这项工作可以变得很丑陋，因为不是所有的语言有着一模一样的类型——回忆一下比如JavaScript中数字大于253的问题（见“JSON，XML，和二进制变体”一节）。这个问题不会在单一语言编写的单个进程中出现。

所有这些因素意味着尝试把远端服务看上去像是所选择的编程语言的本地对象是没有意义的，因为本质上就是不同的东西。REST的部分吸引力在于它并不试图隐藏它是一个网络协议的事实（尽管这似乎并没有阻止人们在REST之上构建RPC库）。

#### 当前RPC的方向

虽然有这么多的问题，但是RPC并没有离开我们。许多RPC框架都构建于这一章提到的所有编码：比如说，Thrift与Avro自带对RPC的支持，gRPC是一种使用Protocol Buffers的RPC实现，Finagle也使用Thrift，而Rest.li使用HTTP上的JSON。

新一代的RPC框架更明确了远端请求与本地函数调用不同的这个事实。举个例子，Finagle与Rest.li使用*future*（*promise*）来封装可能失败的异步操作。Future也简化了需要并行地向多个服务发起请求并结合所有返回的结果。gRPC支持*流*，一次调用不只是包含一个请求和一个响应，而是在一段时间内的一系列的请求和响应。这些框架中的一部分还支持*服务发现*——那就是允许客户端找出在哪一个IP地址和端口它能找到特定的服务。我们会在“请求路由”一节回来看这个问题。

有着二进制编码格式的自定义RPC协议相比与诸如HTTP上的JSON这种通用的格式有着更好的性能。然而，RESTful API有其他显著的优势：对于实验与调试来说是很好的（用网络浏览器或者命令行工具`curl`就能发送请求，而不需要生成任何代码或者安装任何软件），它被所有主流编程语言和平台支持，并且有一个工具的庞大生态系统可用（服务器、缓存、负载平衡器、代理、防火墙、监测工具、调试工具、测试工具等等）。

由于这些原因，REST似乎成为了公共API的主流风格。RPC框架的主要聚焦点是同一组织内服务之间的请求，通常在同一个数据中心内。

#### RPC的数据编码与演进

对于可演进性来说，RPC客户端与服务器可以独立变更和部署是很重要的。相比于数据流经数据库（上一节描述的），我们可以对数据流经服务做一个简单的假设：合理地假设所有服务器首先更新，所有客户端之后更新。这样，请求只需要向后兼容，而响应只需要向前兼容。

RPC方案的向前向后兼容性数据继承于它使用的编码：

* Thrift、gRPC（Protocol Buffer）以及Avro RPC可以根据对应编码格式的兼容性规则演进。

* SOAP的请求与响应都是用XML模式定义规定的。它们可以演进，但是有一些很微妙的陷阱。

* RESTful API绝大部分通常使用JSON（没有正式规定的模式定义）做响应，用JSON或者URI编码/表格编码的请求参数做请求。添加可选的请求参数以及添加新的字段到响应对象通常被认为是保持了兼容性的变更。

由于RPC经常用做跨组织的通信，实现服务兼容性变得更困难，所以服务供应商通常对客户端没有控制，不能强迫它们升级。因此兼容性需要维持很长一段时间，也许没有期限。如果需要做破坏兼容性的变革，服务供应商一般最终会提供并维护多版本的服务API。

并没有规定API版本应该如何做（比如客户端如何指出它要用哪一个版本的API）。对于RESTful API，一般的做法是把版本号码放在URL或者HTTP的Accept头中。对于使用API密钥表示特定客户端的服务来说，另一个选择是把客户端请求的API版本存储在服务器端，并且允许通过单独的管理接口更新选择的版本。

### 消息传递的数据流

我们已经了解许多编码后数据从一个进程流向另一个进程的不同方式。截至目前，我们讨论了REST和RPC（一个进程通过网络发送请求到另外一个进程，期望尽快得到响应），以及数据库（一个进程写编码后的数据，另一个进程稍后再读取它）。

在本章最后这一节，我们将简要了解*异步消息传递系统*，它介于RPC与数据库之间。它们与RPC很类似的地方在于客户端请求（通常叫做*消息*）发送到另一个进程时延迟很低。与数据库很类似的方面在于消息并不是通过直接网络连接发送出去的，而是通过一个中间叫做*消息代理*（也叫做*消息队列*或者*面向消息的中间件*）的中介，它会临时储存这个消息。

与直接RPC相比，使用消息代理有以下几个好处：

* 如果接收者不可用或者过载，它可以作为缓冲，因而增加了系统的可靠性。

* 它可以自动重发消息到崩溃的进程，因而防止消息丢失。

* 它避免了发送者需要知道接收者的IP地址与端口（这在云部署中特别有用，因为虚拟机经常来来去去的）。

* 它允许一个消息发送到多个接收者。从逻辑上解耦了发送者与接收者（发送者只发布消息而不关心谁处理了它们）。

然而，与RPC的差别是消息传递通信通常是单向的：发送者一般不期望接收消息的回复。进程可能会发送一个响应，但是通常会在一个独立的渠道完成。这种通信模式是异步的：发送者不等待消息的发送，而只是发送它，之后不再关心。

#### 消息代理

过去，例如TIBCO、IBM WebSphere以及webMethods这样商业性企业软件占据了消息代理市场的主导地位。最近，开源实现，比如RabbitMQ、ActiveMQ、HornetQ、NATS和Apache Kafka，变得越来越受欢迎。我们会在第十一章详细比较它们。

具体的传递语义因实现与配置各异，但是通常，消息代码以下列方式使用：一个进程发送消息到命名*队列*或是*主题*，而代理保证消息被传递给一个或者多个队列或是主题的*消费者*或是*订阅者*。在同一个主题上可以有许多生产者和许多消费者。

主题只能提供单向的数据流。然而，消费者自己也会发布消息到其它主题（于是可以把它们连在一起，我们会在第十一章看到），或者发布到一个回复队列，之后被原消息的发送着消费掉（从而构成一个请求/响应数据流，与RPC类似）。

消息代理通常不强制任何特定的数据模型——消息只是有着一些元数据的字节序列，于是你可以使用任何编码类型。如果编码是向前向后兼容的，就有了最大的灵活性独立变更发布者和消费者，并以任意顺序部署它们。

如果消费者重新发布消息到其它主题，你需要注意保留未知字段，防止之前描述数据库时遇到的问题（图4-7）。

#### 分布式参与者模式

参与者模式是单进程中并发的编程模型。不是直接处理线程（以及相关的竞争条件、锁定以及死锁问题），而是逻辑被封装在*参与者*中。每一个参与者通常代表一个客户端或者实体，它也许有一些本地状态（不与其它参与者共享），并且通过发送和接收异步消息与其它参与者通信。消息传递并不能保证：在特定的错误场景中，消息会丢失。由于每个参与者同一时间只处理一个消息，他并不需要担心线程，框架可以独立调度每个参与者。

在*分布式参与者框架*中，这个编程模型被用来在多节点范围内扩展应用程序。使用了同样的消息传递机制，而不在乎发送者与接收者是否在同一个节点或是不同节点。如果它们在不同的节点上，消息会悄无声息地编码为字节序列，通过网络发送，之后在另外一侧解码。

相比于RPC，位置透明在参与者模式中工作得更好，因为参与者模式已经假设消息有可能会丢，即便是在一个进程中。虽然延迟在网络中要比在同一进程内高，但是使用参与者模式时本地与远端通信之间本质得差别会少一些。

分布式参与者框架本质上把消息代理和参与者编程模型集成到了一个框架中。然而，如果要滚动升级基于参与者模型的引用，仍然需要考虑向前向后兼容性问题，因为消息也许从新版本节点发送到了旧版本节点，或者是正好反过来。

三种流行的分布式参与者框架处理消息编码如下：

* *Akka*默认使用Java内建的序列化，它不提供向前向后兼容性。然而，你可以把它换成其它的比如Protocol Buffers，并因而获得支持滚动升级的能力。

* *Orleans*默认使用一种自定义的数据编码格式，它不支持滚动升级部署；为了部署新版本的应用，需要设置新的簇，把流量从旧的簇导向到新的簇，然后关闭旧的。与Akka类似，可以使用自定义的序列化插件。

* 在*Erlang OTP*中对记录模式定义的变更是出奇得难（尽管系统有许多高可用性的功能）；滚动升级是可能的，但是需要仔细地计划。一种实验性地新`maps`数据类型（类似JSON的结构，在2014年Erlang R17引入）也许在将来可以使它变得简单一些。

## 总结

这一章我们了解几种把数据结构转化为网络上或磁盘上字节的方法。我们看到了这些编码的细节是如何影响效率，跟重要的，是如何影响了应用的架构以及部署它们的选择的。

特别的是，许多服务需要支持滚动升级，也就是服务的新版本每次渐渐地部署到一部分节点而不是同时部署到所有节点。滚动升级允许服务地新版本发布时没有下线时间（因而鼓励频繁的小更新而不是稍有的大更新）也让部署风险更低（可以检测出错的版本，从而在影响大量用户之前回滚）。这些特性对于可进化性来说是有极大好处的，使得程序变更更简单。

滚动升级的过程中，或者因为许多其它原因，我们必须假设不同的节点运行着不同版本的代码。因此，流经系统的所有数据以提供向后兼容性（新代码可以读取旧数据）和向前兼容性（旧代码可以读取新数据）的方式编码是很重要的。

我们讨论几种数据编码格式以及它们的兼容性特性：

* 特定编程语言的编码受限于单个编程语言，而且经常不能提供向前向后兼容性。

* 文本格式例如JSON、XML以及CSV广泛广泛，而它们的兼容性取决于如何使用它们。它们有可选的模式定义语言，它们有些时候有帮助，有些时候是障碍。这些格式某种程度上对待数据类型是很模糊的，所以处理数字和二进制字符串的时候需要特别小心。

* 二进制模式定义驱动的格式例如Thrift、Protocol Buffers以及Avro允许紧凑高效的编码，且有着明确定义的向前先后兼容性语义。模式定义对于静态类型语言的说明文档以及代码生成很有用。然而，缺点是数据需要解码才能是人类可读的。

我们还讨论了几种数据流的模式，描绘了在不同场景中数据编码的重要性：

* 数据库，这里写入数据库的进程编码数据，而读取数据库的进程解码它。

* RPC与REST API，这里客户端编码请求，服务端解码请求并编码响应，最终客户端解码响应。

* 异步消息传递（使用消息代理或是参与者），这里节点通信是通过发送消息，发送者编码消息而接收者解码它。

我们可以总结出这样的结论，小心的话，向前向后兼容性以及滚动升级都是可以达成的。祝你的应用可以快速演进，部署更频繁。