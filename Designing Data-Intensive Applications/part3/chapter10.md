# 第十章 批处理

*如果系统受到一个人的强烈影响，那它是不可能成功的。一旦初始设计完成并且相当得健壮，真正的测试就开始了——拥有许多不同观点的人们开始进行他们自己的实验。*

高德纳

---

在本书的前两部分中，我们讨论了许多关于请求、查询以及对应的响应或结果的内容。这种风格的数据处理在许多现代数据系统中都假设：你请求什么，或者发送一条指令，稍后系统（但愿）给你一个回答。数据库、缓存、搜索索引、Web服务器以及许多其他系统都是以这样的方式工作的。

在这样的在线系统中，无论是请求页面的Web浏览器还是调用远程API的服务，我们通常假设请求是由用户触发的，并且这个用户正在等待响应。他们没有必要等待太久，所以我们非常关注这些系统的响应时间（见“描述性能”一节）。

互联网，以及越来越多的基于HTTP/REST的API使得请求/响应风格的交互变得如此普遍，以至于很容易认为这些是理所当然的。但我们应该记住，这不是构建系统的唯一方式，而且其它方法也有各自的优点。让我们区分三种不同类型的系统：

*服务（在线系统）*

服务等待来自客户端的请求或指令的到来。当接收到一条消息时，服务尝试尽可能快地处理它，并回复一个响应。响应时间通常是衡量服务性能的主要指标，而可用性通常也非常重要（如果客户端无法访问服务，用户可能会收到一条错误信息）。

*批处理系统（离线系统）*

批处理系统接受大量的输入数据，启动一个任务来处理它，然后产生一些输出数据。任务通常需要花费一段时间（从几分钟到几天不等），因此通常不会有用户等待任务完成。取而代之的是，批处理任务通常定期运行（比如一天一次）。批处理任务的主要性能衡量指标通常是吞吐量（处理特定大小输入数据集所花费的时间）。在这一章里，我们将讨论批处理。

*流处理系统（近似实时系统）*

流处理介于在线处理和离线/批处理之间（因此有时被称为近似实时或接近线的处理）。与批处理系统类似，流处理器消耗输入并产生输出（而不是响应请求）。然而，流任务在事件发生之后不久就会运行，而批处理任务则在固定的输入数据集上操作。这种差异使得流处理系统比等效的批处理系统具有更低的延迟时间。由于流处理构建在批处理的基础上，我们将在第11章中讨论它。

正如我们将在本章中看到的，批处理是构建可靠的、可扩展的和可维护的应用程序的重要组成部分。举个例子，2004年发布的批处理算法MapReduce，被称为“使谷歌如此大规模可扩展的算法”。之后在各种开放源码数据系统中都有实现，包括Hadoop、CouchDB和MongoDB。

与许多年前为数据仓库开发的并行处理系统相比，MapReduce是一个相当低层级的编程模型，但是它在商业性硬件上可以实现的处理规模方面是向前迈进了一大步。尽管现在MapReduce的重要性正在下降，但仍然值得去理解它，因为它清楚地说明了批处理为什么以及如何有用。

事实上，批处理是一种非常古老的计算形式。早在可编程数字计算机被发明之前，穿孔卡片制表机——如1890年美国人口普查中使用的何乐礼机——实现了一种半机械化的批处理形式，用以从大量的输入计算总的统计数据。并且MapReduce与二十世纪四五十年代广泛用于商业数据处理的机电式IBM卡片分类机有着惊人的相似之处。如往常一样，历史有重复自己的倾向。

在本章中，我们将研究MapReduce和其他几个批处理算法和框架，并探讨它们是如何在现代数据系统中使用的。但是首先，我们先使用标准Unix工具看看数据处理。即使你已经很熟悉它们了，还是要提醒你一下Unix哲学还是很有价值的，因为Unix的思想和经验传递给了大规模、异构的分布式数据系统。

## 用Unix工具进行批处理

让我们从一个简单的例子开始。假设你有一个Web服务器，每次服务了请求之后都会在日志文件添加一行。举个例子，使用nginx默认访问日志格式，其中一行可能如下所示：

Let’s start with a simple example. Say you have a web server that appends a line to a log file every time it serves a request. For example, using the nginx default access log format, one line of the log might look like this:

```TEXT
216.58.210.78 - - [27/Feb/2015: 17:55:11 + 0000] "GET /css/typography.css HTTP/1.1" 200 3377 "http://martin.kleppmann.com/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/40.0.2214.115 Safari/537.36"
```

(这实际上是一行；这里只是为了便于阅读才将它分解成多行。)这一行有很多信息。为了解读它，你需要查看日志格式的定义，如下所示：

```PHP  
$remote_addr - $remote_user [$time_local] "$request"
$status $body_bytes_sent "$http_referer" "$http_user_agent"
```

因此日志的这一行表示，在UTC时间2015年2月27日17时55分11秒，服务器收到了来自客户端IP地址为216.58.210.78对文件/css/typography.css的请求。用户没有经过身份验证，因此$Remote_user被设置为连字符（-）。响应状态为200（即请求成功），响应大小为3377字节。Web浏览器是Chrome 40，它加载该文件是因为它在网址http://martin.kleppmann.com/ 的页面中被引用了。

### 简单的日志分析

各种工具可以读取这些日志文件并生成关于你的网站流量的漂亮报告，但为了便于练习，让我们使用基本的Unix工具构建自己的日志文件。例如，假设你想在你的网站上找到最受欢迎的五个页面。你可以在Unix命令行中这样做，如下所示：

```BASH
cat /var/log/nginx/access.log | ➊
    awk '{print $7}'    | ➋
    sort                | ➌
    uniq -c             | ➍
    sort -r -n          | ➎
    head -n 5             ➏
```

➊ 读取日志文件。

➋ 用空格把每一行分割成许多字段，然后只输出每一行中的第七个字段，这是请求的URL。在我们的例子中，这个请求URL是*/css/typography.css*。

➌ 按字母顺序对请求的URL列表进行排序。如果某个URL被请求了*n*次，那么在排序之后，该文件在每一行包含了重复*n*次的相同URL。

➍ `uniq`命令通过检查相邻的两个行是否相同来过滤输入中的重复行。`-c`选项告诉命令还要输出计数器值：对于每个不同的URL，它都会报告该URL在输入中出现的次数。

➎ 第二次排序是根据每行开头的数字（`-n`）排序，即请求URL的次数。然后它以反向（`-r`）顺序返回结果，即最大值排在前面。

➏ 最后，`head`只输出输入的前五行（-n 5），而其余的被丢掉了。

这一系列命令的输出结果如下：

```BASH
4189 /favicon.ico
3631 /2013/05/24/improving-security-of-ssh-private-keys.html
2124 /2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html
1369 /
915 /css/typography.css
```

如果不熟悉Unix工具，虽然前面的命令行可能看起来有点生涩，却是非常得强大。它可以在数秒钟内处理千兆字节的日志文件，而且可以轻松地修改分析过程以满足需要。比如，如果要在报告中省略CSS文件，可以把`awk`的参数更改为`‘$7 !~ /\.css$/ {print$7}’`。如果要计算访问次数最多的客户端IP地址而不是访问次数最多的页面，可以把`awk`参数更改为`‘{print $1}’`。等等。

在这本书中我们没有篇幅来详细探讨Unix工具，但它们非常值得学习。令人惊讶的是，使用`awk`、`sed`、`grep`、`sort`、`uniq`和`xargs`的组合可以在几分钟内完成许多数据分析，而它们的性能出人意料地好。

#### 命令链 vs 定制程序

不使用Unix命令链，你还可以写一个简单的程序来完成同样的任务。举个例子，用Ruby这个程序大概是这个样子的：

```RUBY
counts = Hash.new(0) ➊

File.open('/var/log/nginx/access.log') do |file|
    file.each do |line|
        url = line.split[6] ➋
        counts[url] += 1    ➌
    end
end

top5 = counts.map{|url, count| [count, url] }.sort.reverse[0...5]   ➍
top5.each{|count, url| puts "#{count} #{url}" } ➎
```

➊ `counts` 一个哈希表，它为我们看到每个URL的次数计数。默认情况下，计数为零。

➋ 从日志的每一行中，我们将URL作为第七个按空格分隔的字段（这里的数组索引为6，因为Ruby的数组是以零为起始索引的）。

➌ 为日志当前行中URL的计数加一。

➍ 按计数器值（降序）对哈希表内容进行排序，并取前五项。

➎ 打印出前五项。

这个程序不像Unix管道链那样简洁，但是它的可读性相当强，而两者之中更偏好哪一个在一定程度上取决于你的喜好。但是除了这两者之间的表面语法差异之外，在执行流程上也有很大的差异，这一点在一个大文件上执行分析的时候就很明显了。

#### 排序 vs 内存内聚合

Ruby脚本保存了URL的内存哈希表，其中每个URL都映射到它被浏览的次数。Unix管道示例没有这样的哈希表而是依赖于排序URL列表，列表中同一个URL多次重复。

哪种方法更好？这取决于你有多少个不同的URL。对于绝大多数中小型网站，你大概可以把所有不同的URL，以及为每个URL设置的计数器，放在（例如）1GB内存中。在这个例子中，任务的工作集（任务需要随机访问的内存量）只取决于URL的数量：如果单个URL有一百万个日志条目，那么哈希表中所需的空间仍然是一个URL的长度加上计数器的大小。如果这个工作集足够小，那么内存哈希表就可以工作得很好了——哪怕是在一台笔记本电脑上。

另一方面，如果任务的工作集大于可用内存，则排序方法的优点是可以有效地利用磁盘。这与我们在“SSTable与LSM树”一节中讨论的原则相同：数据块可以在内存中排序并作为段文件写入磁盘，之后数个排好序的段可以合并到一个更大的排好序的文件中。合并排序的顺序访问模式在磁盘上表现良好。（请记住，对顺序I/O的优化是第3章中反复出现的主题。同样的模式在这里再次出现。）

GNU Coreutils (Linux)中的`sort`程序通过溢出到磁盘自动处理大于内存的数据集，并自动跨多个CPU核心进行并行排序。这意味着我们前面看到的简单的Unix命令链很容易扩展到大型数据集上，而不会耗尽内存。瓶颈很可能是从磁盘读取输入文件的速度。

### Unix哲学

我们能够很容易地使用前面示例中的一系列命令来分析日志文件，这并不是巧合：实际上，这是Unix的关键设计思想之一，今天它仍然具有惊人的相关性。让我们更深入地研究它，方便我们可以从Unix中借鉴一些点子。

道格拉斯·麦克罗伊，Unix管道的发明者，在1964年第一次这样描述它们：“我们应该有一些连接程序的方法，就像花园里浇水的软管一样——当必须以另一种方式修改数据的时候就再接入另一段（软管）。I/O也是同样的方式。”水管的比喻非常成功，用管道连接程序的理念成为了现在被称为Unix哲学的一部分——一套在Unix开发人员和用户中流行的设计原则。这一哲学在1978年被描述如下：

1. 让每个程序都做好一件事。要做一项新的工作，构建一个新程序而不是通过添加新的“功能”使旧程序复杂化。

2. 期望每一个程序的输出都可以成为另一个，也许未知，的程序的输入。不要杂乱地输出不相关的信息。避免严格的列式或二进制输入格式。不坚持使用交互式输入。

3. 设计和构建软件甚至是操作系统，都要尽早适用，最好是在几周之内就这样做。在扔掉和重建笨拙的组件时不要犹豫。

4. 优先使用工具而非不熟练的帮助来减轻编程任务，哪怕你必须专门去构建工具，并且在使用完这些工具后会扔掉其中一些。

这种方法——自动化、快速原型化、增量迭代、对实验友好，以及将大型项目分解成可管理的块——听起来非常像今天的敏捷和DevOps运动。令人惊讶的是，这四十年来几乎没有什么变化。

`sort`工具是很好的程序只做好一件事的例子。可以说它是一种比大多数编程语言在其标准库中实现的更好的排序实现（它不会溢出到磁盘，也不会使用多个线程，即使这样做是有益的）。然而，单单只用`sort`几乎没什么用。它只有与其他Unix工具，比如`uniq`，结合时才会变得强大。

Unix命令行，比如`bash`，让我们很容易将这些小程序*编写*成令人惊讶的强大的数据处理任务。虽然这些程序中有很多是由不同的人们编写的，但是它们可以灵活地结合在一起。实现这种可组合性Unix都做了些什么？

#### 统一的界面

如果你期望一个程序的输出变成另一个程序的输入，那意味着那些程序必须使用相同的数据格式——换句话说，一个兼容的接口。如果你想将任何程序的输出连接到任何程序的输入，这意味着所有的程序都必须使用相同的输入/输出接口。

在Unix中，这个接口是文件（或者更准确地说，是文件描述符）。文件只是一个有序的字节序列。因为这是一个简单的接口，所以可以使用相同的接口来表示许多不同的东西：文件系统上实际的文件、到另一个进程（Unix套接字、`stdin`、`stdout`）的通信通道、设备驱动程序（例如`/dev/audio`或`/dev/lp0`）、表示TCP连接的套接字，等等。我们很容易认为这是理所当然的，但事实上，这些非常不同的东西可以共享一个统一的接口是非常值得注意的，因此它们可以很容易地连接在一起。

按照惯例，许多（但不是所有）Unix程序将这个字节序列作为ASCII文本来处理。我们的日志分析示例利用了这个特点：`awk`、`sort`、`uniq`和`head`都将它们的输入文件视为由`\n`（新行，ASCII码`0x0A`）字符分隔的记录列表。选择`\n`是任意的——按理说，ASCII记录分隔符`0x1E`本来是更好的选择，因为它就是为此目的而设计的——但无论如何，所有这些程序在使用相同的记录分隔符的这一刻就标准化了，这使得它们可以进行相互操作。

对每条记录（即每行输入）的解析比较模糊。Unix工具通常通过空格或制表符将一行拆分为字段，但是也可以用CSV（逗号分隔）、管道分隔和其他编码。即使是像`xargs`这样相当简单的工具，也有六个命令行选项来指定如何解析其输入。

统一的ASCII文本接口大部分时间可以工作，但并不是很漂亮：我们的日志分析示例使用`{print $7}`来提取URL，这不是很易读。在一个理想的环境中，这可以是`{print $request_url}`或类似的东西。我们稍后再来谈谈这个想法。

虽然并不完美，但即使在几十年后，Unix统一接口仍然非常出色。没有多少软件可以像Unix工具一样互相操作和撰写：你无法轻松地通过自定义分析工具将电子邮件帐户的内容和在线购物历史记录输入电子表格，然后把结果发布到社交网络或一个wiki页面上。今天，可以像Unix工具一样顺利地协同工作的程序是一个例外，而不是正常情况。

即使是具有*相同数据模型*的数据库，也往往不容易把数据从一个库转移到另一个中。集成的缺乏导致了数据的碎片化。

#### 逻辑与连接的分离

Unix工具的另一个特点是它们使用标准输入（`stdin`）和标准输出（`stdout`）。如果你运行一个程序，而不指定任何其他设置，那么`stdin`来自键盘而`stdout`将转到屏幕上。但是，你也可以从文件中获取输入然后将输出重定向到文件。管道允许你将一个进程的`stdout`附加到另一个进程的`stdin`(使用一个小内存缓冲区，并且不会把整个中间数据流写入磁盘)。

如果需要的话程序仍然可以直接读写文件，但是如果程序不担心特定的文件路径而只是使用`stdin`和`stdout`，那么Unix方式的效果最好。这使得命令行用户可以用他们想要的任何方式连接输入和输出；程序不知道也不关心输入的来源和输出到哪里。（可以说，这是一种松散耦合、延迟绑定或控制反转的形式。）将输入/输出连接与程序逻辑分离，可以更容易地把小型工具组合到更大的系统中去。

你甚至可以编写自己的程序，然后把它们与操作系统提供的工具结合起来。你的程序只需要从`stdin`读取输入并将输出写入`stdout`，它就可以参与数据处理的管道中。在日志分析示例中，可以编写工具将用户代理字符串转换为更合理的浏览器标识符，或者一个可以把IP地址转换为国家代码的工具，然后把它插入到管道里。`sort`程序不关心是与操作系统的另一部分通信，还是与你编写的程序通信。

然而有了`stdin`和`stdout`能做的事情也是有限制的。需要多个输入输出的程序是存在的，但很棘手。你也不能把程序的输入通过管道发送到网络上。如果一个程序直接打开文件进行读写，或者启动另一个程序作为子进程，或者打开一个网络连接，那么这些I/O就会由程序自己连接起来。它仍然可以配置（比如通过命令行选项），但是在命令行中连接输入和输出的灵活性降低了。

#### 透明度与尝试

使Unix工具如此成功的部分原因在于它们让你很容易了解到正在发生的事情：

* Unix命令的输入文件通常被视为不可变的。这意味着您可以根据需要运行命令，尝试各种命令行选项，而不会损坏输入文件。

* 您可以在任何时候结束管道，将输出输送到`less`，然后查看它是否具有预期的形式。这种检查能力对调试是很好的。

* 您可以将管道某一阶段的输出写入到文件，然后把这个文件用作下一个阶段的输入。这使得您可以重新启动后期阶段，而无需重新运行整个管道。

因此尽管相比于关系数据库的查询优化器，Unix工具是相当简单的工具，但是它们仍然非常有用，特别是在尝试阶段。

然而Unix工具的最大限制在于它们只能在一台机器上运行——这就是Hadoop之类的工具出现的地方。

## MapReduce与分布式文件系统

MapReduce有点像Unix工具，但是可以分布在数千台机器上。就像Unix工具一样，它是一个相当简单、暴力，但又令人惊讶的有效工具。单个MapReduce任务可以与单个Unix进程相比较：它接受一个或多个输入，然后产生一个或多个输出。

与大多数Unix工具一样，运行MapReduce任务通常不会修改输入，除了产生输出之外没有任何副作用。输出文件只会按顺序写入一次（一旦写入，就不会再修改文件的现有部分）。

Unix工具使用`stdin`和`stdout`作为输入和输出，而MapReduce任务是在分布式文件系统上读写文件。在Hadoop的MapReduce实现中，这个文件系统称为HDFS（Hadoop Distributed File System），它是Google文件系统（GFS）的开源重新实现。

除了HDFS之外，还有其他各种分布式文件系统，比如GlusterFS和Quantcast文件系统（QFS）。对象存储服务，如Amazon S3、Azure Blob存储和OpenStack Swift在许多方面都是相似的。在本章中，我们将主要使用HDFS作为运行示例，但这些原则适用于任何分布式文件系统。

HDFS基于无共享原则（见第二部分的介绍），与网络附加存储（NAS）和存储区域网络（SAN）体系结构的共享磁盘方式形成对比。共享磁盘存储由集中式存储设备实现，通常使用自定义硬件和特殊的网络基础设施，比如光纤通道。另一方面，武功想方式不需要特殊硬件，只需要通过传统的数据中心网络连接起来的计算机。

HDFS由运行在每台机器上的守护进程组成，它公开了一个网络服务，允许其他节点访问存储在该机器上的文件（假设数据中心中的每台设备都附加了一些磁盘）。名为NameNode的中央服务器跟踪记录哪些文件区块存储在哪台设备上。因此，HDFS在概念上构建了一个大文件系统，它可以使用所有运行守护进程设备的磁盘上的空间。

为了容忍机器与磁盘故障，文件块被复制到多台设备上。复制可能仅仅意味着多台机器上相同数据的几份拷贝，如第5章中那样，或者是一个擦除编码方案，例如里德-所罗门码，它允许以比完全复制更低的存储开销来恢复丢失的数据。这些技术类似于RAID，它提供了连接到同一台计算机的多个磁盘的冗余性；不同之处在于分布式文件系统中，文件访问与复制是在没有特殊硬件的常规数据中心网络上完成的。

HDFS扩展性很好：在撰写这本书时，最大的HDFS部署运行在数万台设备上上，合计存储容量为数百PB。如此大的规模之所以变得可行，是因为在HDFS上使用商品化硬件以及开源软件存储和访问数据的成本，比使用专用存储设备获得等效容量的成本低得多。

### MapReduce任务的执行

MapReduce是一个编程框架，有了它您可以编写代码来处理在类似HDFS这样的分布式文件系统中的大型数据集。理解它的最简单方法是参考“简单日志分析”一节中的Web服务器日志分析案例。MapReduce中的数据处理模式非常类似于这个示例：

1. 读取一组输入文件，并将其分解为记录。在Web服务器日志示例中，每条记录都是日志中的一行（也就是说，`\n`是记录分隔符）。

2. 调用映射函数从每个输入记录中提取键和值。在前面的示例中，映射函数是`awk ‘{print $7}’`：它提取URL（`$7`）作为键，而值留空。

3. 按键对所有键值对进行排序。在日志示例中，这是通过第一个`sort`命令完成的。

4. 调用归纳函数来迭代排了序的键值对。如果同一键多次出现，排序让它们在列表中相邻，因此很容易组合这些值而不必将大量状态保存在内存中。在前面的示例中，归纳函数由命令`uniq -c`实现，该命令使用相同的键计算相邻记录的数量。

这四个步骤可以由一个MapReduce任务执行。步骤2（映射）和步骤4（归纳）是编写自定义数据处理代码的地方。步骤1（将文件分解为记录）由输入格式解析器处理。步骤3排序步骤，在MapReduce中是隐式的——您不用写它，因为映射函数的输出总是排序之后再交给归纳函数。

要创建MapReduce任务，您需要实现两个回调函数，即映射函数和归纳函数，它们的行为如下（也参考“MapReduce查询”一节）：

映射函数

映射函数对每个输入记录调用一次，任务是从输入记录中提取键和值。对于每个输入，它可能生成任意数量的键值对（也包括没有）。它不保留从一个输入记录到下一个输入记录的任何状态，因此每个记录都是独立处理的。

归纳函数

MapReduce框架采用映射函数生成的键值对，收集属于同一键的所有值，并在该值集合上迭代调用归纳函数。归纳函数可以产生输出记录（例如同一个URL的出现次数）。

在Web服务器日志案例中，我们在步骤5中有第二个`sort`命令，它根据请求的数量对URL进行排序。在MapReduce中，如果需要第二个排序阶段，可以通过编写第二个MapReduce任务并使用第一个作业的输出作为第二个作业的输入来实现它。这样看，映射函数的作用是通过将数据转换为适合排序的形式来准备数据，而归纳函数的作用是处理排好序的数据。

#### MapReduce的分布式执行

MapReduce与Unix命令管道的主要区别在于它可以在许多机器上并行化计算，而无需编写代码显式处理并行性。映射函数和归纳函数一次只对一条记录进行操作；它们不需要知道它们的输入来自何处或它们的输出将去何处，所以框架可以处理在机器之间移动数据的复杂性。

在分布式计算中是可以使用标准Unix工具作为映射函数和归纳函数，但更常见的是它们是使用传统编程语言的函数实现的。在Hadoop MapReduce中，映射函数和归纳函数是实现了特定接口的Java类。在MongoDB和CouchDB中，映射函数和归纳函数是JavaScript函数（见“MapReduce查询”一节）。

图10-1显示了Hadoop MapReduce任务中的数据流。它的并行化基于分区（见第6章）：任务的输入通常是HDFS中的文件目录，输入目录中的每个文件或文件块被视为单独的分区，可以通过单独的映射任务（由图10-1中的*m 1*、*m 2*和*m 3*标记）来处理。

每个输入文件的大小通常为数百MB。MapReduce调度程序（图中未显示）试图在存储输入文件副本的机器上执行每个映射函数，条件是该设备有足够的空闲内存和CPU资源来执行映射任务。这一原则被称为*把计算放在数据附近*：这样不用通过网络复制输入文件，减少了网络负载而增加了局部性。

*图10-1 有三个映射函数和三个归纳函数的MapReduce任务*

在大多数情况下，应该运行在映射任务中的应用程序代码还没有出现在被分配执行它的设备上，所以MapReduce框架首先拷贝代码（比如是Java程序情况的JAR文件）到适当的机器。然后，启动映射任务并开始读取输入文件，每次将一条记录传递给映射回调函数。映射函数的输出是由键值对组成的。

计算的归纳部分也是分区的。映射任务的数量取决于输入文件块的数量，而归纳任务的数量是由任务作者配置的（它可以与映射任务的数量不同）。为了确保所有具有相同键的键值对最终位于同一归纳函数上，框架使用键的哈希值来确定哪个归纳任务应该接收特定的键值对（见“按键的哈希值进行分区”一节）。

键值对必须进行排序，但数据集很可能太大，而无法在单台设备上使用常规排序算法进行排序。取而代之的是，排序是分阶段执行的。首先，每个映射任务根据键的哈希值把它的输出按归纳函数分区。每一个这样的分区都被写入映射器本地磁盘上排了序文件里，用到了类似于我们在“SSTable与LSM树”一节中讨论到的技术。

每当映射函数完成对其输入文件的读取并写入已排序的输出文件时，MapReduce调度程序会通知归纳函数，它们可以开始从归纳函数获取输出文件。归纳函数连接到每个映射函数，并从它们的分区下载排好序的键值对文件。这个通过归纳函数分区、排序，然后从映射函数拷贝数据分区到归纳函数的过程被称为混洗（这是一个让人困惑的词——与洗牌不同，MapReduce中没有随机性）。

归纳函数从映射函数中获取文件，并将它们合并在一起，保持排序顺序。因此，如果不同的映射函数使用相同的键生成记录，那么它们将在合并后的归纳函数输入中相邻。归纳函数是用一个键和一个迭代器调用的，迭代器用相同的键递增地扫描所有记录（在某些情况下，内存是无法放下所有的记录的)。

归纳函数可以使用任意逻辑来处理这些记录，并且可以生成任意数量的输出记录。这些输出记录被写入分布式文件系统上的文件(通常，在运行归纳函数的设备本地磁盘上有一个副本，在其他机器上有其它副本)。

#### MapReduce工作流

使用单个MapReduce任务可以解决的问题范围是有限的。回到日志分析示例，单个MapReduce任务可以确定每个URL的页面浏览量，但无法确定最流行的URL，因为这需要进行第二轮排序。

因此，MapReduce任务被链接到工作流中是非常常见的，这样任务的输出就变成了下一个任务的输入。Hadoop MapReduce框架不支持任何特定的工作流，因此链接是通过目录名称隐式完成的：必须配置第一个任务将其输出写入HDFS中的指定目录，而第二个任务必须配置为读取同样的目录。从MapReduce框架的角度来看，它们是两个独立的任务。

因此，链式MapReduce作业就不太像Unix命令的管道（这些命令只使用内存中的一个小缓冲区，将一个进程的输出直接传递给另一个进程作为输入）而更像是命令序列，每个命令的输出被写入一个临时文件而下一个命令从临时文件中读取。这种设计有它的优缺点，我们将在“中间状态的物化”一节中讨论这个问题。

批处理任务的输出只有在任务成功完成时才被认为有效（MapReduce会丢弃失败任务的部分输出）。因此，工作流中的任务只有在先前的作业——即生成其输入目录的任务——成功完成时才能启动。为了处理任务执行之间的这些依赖关系，开发了各种Hadoop工作流的调度程序，包括Oozie、Azkaban、Luigi、Airflow和Pinball。

这些调度程序还有管理功能，在维护大量批处理任务时非常有用。在构建推荐系统时，包含50到100个MapReduce任务组成的工作流很常见，而在一个大型组织中，许多不同的团队会运行不同的任务来读取彼此的输出。工具支持对于管理这样复杂的数据流非常重要。

Hadoop的各种高级工具，如Pig、Hive、Cascading、Crunch和FlumeJava，也设置了包含多个可以自动连在一起的MapReduce阶段的工作流。

### Reduce-Side Joins and Grouping

We discussed joins in Chapter   2 in the context of data models and query languages, but we have not delved into how joins are actually implemented. It is time that we pick up that thread again.

In many datasets it is common for one record to have an association with another record: a foreign key in a relational model, a document reference in a document model, or an edge in a graph model. A join is necessary whenever you have some code that needs to access records on both sides of that association (both the record that holds the reference and the record being referenced). As discussed in Chapter   2, denormalization can reduce the need for joins but generally not remove it entirely.v

In a database, if you execute a query that involves only a small number of records, the database will typically use an index to quickly locate the records of interest (see Chapter   3). If the query involves joins, it may require multiple index lookups. However, MapReduce has no concept of indexes — at least not in the usual sense.

When a MapReduce job is given a set of files as input, it reads the entire content of all of those files; a database would call this operation a full table scan. If you only want to read a small number of records, a full table scan is outrageously expensive compared to an index lookup. However, in analytic queries (see “Transaction Processing or Analytics?”) it is common to want to calculate aggregates over a large number of records. In this case, scanning the entire input might be quite a reasonable thing to do, especially if you can parallelize the processing across multiple machines.

When we talk about joins in the context of batch processing, we mean resolving all occurrences of some association within a dataset. For example, we assume that a job is processing the data for all users simultaneously, not merely looking up the data for one particular user (which would be done far more efficiently with an index).

#### Example: analysis of user activity events

A typical example of a join in a batch job is illustrated in Figure   10-2. On the left is a log of events describing the things that logged-in users did on a website (known as activity events or clickstream data), and on the right is a database of users. You can think of this example as being part of a star schema (see “Stars and Snowflakes: Schemas for Analytics”): the log of events is the fact table, and the user database is one of the dimensions.

*Figure 10-2. A join between a log of user activity events and a database of user profiles.*

An analytics task may need to correlate user activity with user profile information: for example, if the profile contains the user’s age or date of birth, the system could determine which pages are most popular with which age groups. However, the activity events contain only the user ID, not the full user profile information. Embedding that profile information in every single activity event would most likely be too wasteful. Therefore, the activity events need to be joined with the user profile database.

The simplest implementation of this join would go over the activity events one by one and query the user database (on a remote server) for every user ID it encounters. This is possible, but it would most likely suffer from very poor performance: the processing throughput would be limited by the round-trip time to the database server, the effectiveness of a local cache would depend very much on the distribution of data, and running a large number of queries in parallel could easily overwhelm the database [35].

In order to achieve good throughput in a batch process, the computation must be (as much as possible) local to one machine. Making random-access requests over the network for every record you want to process is too slow. Moreover, querying a remote database would mean that the batch job becomes nondeterministic, because the data in the remote database might change.

Thus, a better approach would be to take a copy of the user database (for example, extracted from a database backup using an ETL process — see “Data Warehousing”) and to put it in the same distributed filesystem as the log of user activity events. You would then have the user database in one set of files in HDFS and the user activity records in another set of files, and could use MapReduce to bring together all of the relevant records in the same place and process them efficiently.

#### Sort-merge joins

Recall that the purpose of the mapper is to extract a key and value from each input record. In the case of Figure   10-2, this key would be the user ID: one set of mappers would go over the activity events (extracting the user ID as the key and the activity event as the value), while another set of mappers would go over the user database (extracting the user ID as the key and the user’s date of birth as the value). This process is illustrated in Figure   10-3.

*Figure 10-3. A reduce-side sort-merge join on user ID. If the input datasets are partitioned into multiple files, each could be processed with multiple mappers in parallel.*

When the MapReduce framework partitions the mapper output by key and then sorts the key-value pairs, the effect is that all the activity events and the user record with the same user ID become adjacent to each other in the reducer input. The MapReduce job can even arrange the records to be sorted such that the reducer always sees the record from the user database first, followed by the activity events in timestamp order — this technique is known as a secondary sort [26].

The reducer can then perform the actual join logic easily: the reducer function is called once for every user ID, and thanks to the secondary sort, the first value is expected to be the date-of-birth record from the user database. The reducer stores the date of birth in a local variable and then iterates over the activity events with the same user ID, outputting pairs of viewed-url and viewer-age-in-years. Subsequent MapReduce jobs could then calculate the distribution of viewer ages for each URL, and cluster by age group.

Since the reducer processes all of the records for a particular user ID in one go, it only needs to keep one user record in memory at any one time, and it never needs to make any requests over the network. This algorithm is known as a sort-merge join, since mapper output is sorted by key, and the reducers then merge together the sorted lists of records from both sides of the join.

#### Bringing related data together in the same place

In a sort-merge join, the mappers and the sorting process make sure that all the necessary data to perform the join operation for a particular user ID is brought together in the same place: a single call to the reducer. Having lined up all the required data in advance, the reducer can be a fairly simple, single-threaded piece of code that can churn through records with high throughput and low memory overhead.

One way of looking at this architecture is that mappers “send messages” to the reducers. When a mapper emits a key-value pair, the key acts like the destination address to which the value should be delivered. Even though the key is just an arbitrary string (not an actual network address like an IP address and port number), it behaves like an address: all key-value pairs with the same key will be delivered to the same destination (a call to the reducer).

Using the MapReduce programming model has separated the physical network communication aspects of the computation (getting the data to the right machine) from the application logic (processing the data once you have it). This separation contrasts with the typical use of databases, where a request to fetch data from a database often occurs somewhere deep inside a piece of application code [36]. Since MapReduce handles all network communication, it also shields the application code from having to worry about partial failures, such as the crash of another node: MapReduce transparently retries failed tasks without affecting the application logic.

#### GROUP BY

Besides joins, another common use of the “bringing related data to the same place” pattern is grouping records by some key (as in the GROUP BY clause in SQL). All records with the same key form a group, and the next step is often to perform some kind of aggregation within each group — for example:

* Counting the number of records in each group (like in our example of counting page views, which you would express as a COUNT(*) aggregation in SQL)

* Adding up the values in one particular field (SUM( fieldname)) in SQL

* Picking the top k records according to some ranking function

The simplest way of implementing such a grouping operation with MapReduce is to set up the mappers so that the key-value pairs they produce use the desired grouping key. The partitioning and sorting process then brings together all the records with the same key in the same reducer. Thus, grouping and joining look quite similar when implemented on top of MapReduce.

Another common use for grouping is collating all the activity events for a particular user session, in order to find out the sequence of actions that the user took — a process called sessionization [37]. For example, such analysis could be used to work out whether users who were shown a new version of your website are more likely to make a purchase than those who were shown the old version (A/ B testing), or to calculate whether some marketing activity is worthwhile.

If you have multiple web servers handling user requests, the activity events for a particular user are most likely scattered across various different servers’ log files. You can implement sessionization by using a session cookie, user ID, or similar identifier as the grouping key and bringing all the activity events for a particular user together in one place, while distributing different users’ events across different partitions.

#### Handling skew

The pattern of “bringing all records with the same key to the same place” breaks down if there is a very large amount of data related to a single key. For example, in a social network, most users might be connected to a few hundred people, but a small number of celebrities may have many millions of followers. Such disproportionately active database records are known as linchpin objects [38] or hot keys.

Collecting all activity related to a celebrity (e.g., replies to something they posted) in a single reducer can lead to significant skew (also known as hot spots) — that is, one reducer that must process significantly more records than the others (see “Skewed Workloads and Relieving Hot Spots”). Since a MapReduce job is only complete when all of its mappers and reducers have completed, any subsequent jobs must wait for the slowest reducer to complete before they can start.

If a join input has hot keys, there are a few algorithms you can use to compensate. For example, the skewed join method in Pig first runs a sampling job to determine which keys are hot [39]. When performing the actual join, the mappers send any records relating to a hot key to one of several reducers, chosen at random (in contrast to conventional MapReduce, which chooses a reducer deterministically based on a hash of the key). For the other input to the join, records relating to the hot key need to be replicated to all reducers handling that key [40].

This technique spreads the work of handling the hot key over several reducers, which allows it to be parallelized better, at the cost of having to replicate the other join input to multiple reducers. The sharded join method in Crunch is similar, but requires the hot keys to be specified explicitly rather than using a sampling job. This technique is also very similar to one we discussed in “Skewed Workloads and Relieving Hot Spots”, using randomization to alleviate hot spots in a partitioned database.

Hive’s skewed join optimization takes an alternative approach. It requires hot keys to be specified explicitly in the table metadata, and it stores records related to those keys in separate files from the rest. When performing a join on that table, it uses a map-side join (see the next section) for the hot keys.

When grouping records by a hot key and aggregating them, you can perform the grouping in two stages. The first MapReduce stage sends records to a random reducer, so that each reducer performs the grouping on a subset of records for the hot key and outputs a more compact aggregated value per key. The second MapReduce job then combines the values from all of the first-stage reducers into a single value per key.