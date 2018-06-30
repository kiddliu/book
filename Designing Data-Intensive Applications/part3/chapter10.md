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

* Unix命令的输入文件通常被视为不可变的。这意味着你可以根据需要运行命令，尝试各种命令行选项，而不会损坏输入文件。

* 你可以在任何时候结束管道，将输出输送到`less`，然后查看它是否具有预期的形式。这种检查能力对调试是很好的。

* 你可以将管道某一阶段的输出写入到文件，然后把这个文件用作下一个阶段的输入。这使得你可以重新启动后期阶段，而无需重新运行整个管道。

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

MapReduce是一个编程框架，有了它你可以编写代码来处理在类似HDFS这样的分布式文件系统中的大型数据集。理解它的最简单方法是参考“简单日志分析”一节中的Web服务器日志分析案例。MapReduce中的数据处理模式非常类似于这个示例：

1. 读取一组输入文件，并将其分解为记录。在Web服务器日志示例中，每条记录都是日志中的一行（也就是说，`\n`是记录分隔符）。

2. 调用映射函数从每个输入记录中提取键和值。在前面的示例中，映射函数是`awk ‘{print $7}’`：它提取URL（`$7`）作为键，而值留空。

3. 按键对所有键值对进行排序。在日志示例中，这是通过第一个`sort`命令完成的。

4. 调用归纳函数来迭代排了序的键值对。如果同一键多次出现，排序让它们在列表中相邻，因此很容易组合这些值而不必将大量状态保存在内存中。在前面的示例中，归纳函数由命令`uniq -c`实现，该命令使用相同的键计算相邻记录的数量。

这四个步骤可以由一个MapReduce任务执行。步骤2（映射）和步骤4（归纳）是编写自定义数据处理代码的地方。步骤1（将文件分解为记录）由输入格式解析器处理。步骤3排序步骤，在MapReduce中是隐式的——你不用写它，因为映射函数的输出总是排序之后再交给归纳函数。

要创建MapReduce任务，你需要实现两个回调函数，即映射函数和归纳函数，它们的行为如下（也参考“MapReduce查询”一节）：

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

归纳函数从映射函数中获取文件，并将它们合并在一起，保持排序顺序。因此，如果不同的映射函数使用相同的键生成记录，那么它们将在合并后的归纳函数输入中相邻。归纳函数是用一个键和一个迭代器调用的，迭代器用相同的键递增地扫描所有记录（在某些情况下，内存是无法放下所有的记录的）。

归纳函数可以使用任意逻辑来处理这些记录，并且可以生成任意数量的输出记录。这些输出记录被写入分布式文件系统上的文件（通常，在运行归纳函数的设备本地磁盘上有一个副本，在其他机器上有其它副本）。

#### MapReduce工作流

使用单个MapReduce任务可以解决的问题范围是有限的。回到日志分析示例，单个MapReduce任务可以确定每个URL的页面浏览量，但无法确定最流行的URL，因为这需要进行第二轮排序。

因此，MapReduce任务被链接到工作流中是非常常见的，这样任务的输出就变成了下一个任务的输入。Hadoop MapReduce框架不支持任何特定的工作流，因此链接是通过目录名称隐式完成的：必须配置第一个任务将其输出写入HDFS中的指定目录，而第二个任务必须配置为读取同样的目录。从MapReduce框架的角度来看，它们是两个独立的任务。

因此，链式MapReduce作业就不太像Unix命令的管道（这些命令只使用内存中的一个小缓冲区，将一个进程的输出直接传递给另一个进程作为输入）而更像是命令序列，每个命令的输出被写入一个临时文件而下一个命令从临时文件中读取。这种设计有它的优缺点，我们将在“中间状态的物化”一节中讨论这个问题。

批处理任务的输出只有在任务成功完成时才被认为有效（MapReduce会丢弃失败任务的部分输出）。因此，工作流中的任务只有在先前的作业——即生成其输入目录的任务——成功完成时才能启动。为了处理任务执行之间的这些依赖关系，开发了各种Hadoop工作流的调度程序，包括Oozie、Azkaban、Luigi、Airflow和Pinball。

这些调度程序还有管理功能，在维护大量批处理任务时非常有用。在构建推荐系统时，包含50到100个MapReduce任务组成的工作流很常见，而在一个大型组织中，许多不同的团队会运行不同的任务来读取彼此的输出。工具支持对于管理这样复杂的数据流非常重要。

Hadoop的各种高级工具，如Pig、Hive、Cascading、Crunch和FlumeJava，也设置了包含多个可以自动连在一起的MapReduce阶段的工作流。

### 归纳侧的连接与分组

我们在第2章中研究数据模型和查询语言时讨论了连接，但我们还没有深入研究连接是如何实现的。现在是我们重新开始讨论它的时候了。

在许多数据集中，一条记录与另一条记录有关联是很常见的：关系模型中的*外键*，文档模型中的*文档引用*，亦或是图形模型中的*边*。每当一些代码需要访问关联两边的记录（包含引用的记录与被引用的记录）时，连接都是必需的。正如在第2章中所讨论的，去正规化可以减少对连接的需求，但通常无法完全不依赖它。

在数据库中，如果执行的查询只涉及到少量的记录，数据库通常会使用索引快速定位感兴趣的记录（见第3章）。如果查询涉及连接，就会需要多个索引查找。然而，MapReduce没有索引的概念——至少通常意义上并非如此。

当给予MapReduce任务一组文件作为输入时，它会读取所有这些文件的全部内容；数据库将此操作称为*全表扫描*。如果你只想读取少量的记录，那么相比于索引查找，全表扫描代价要高得多。然而在分析性查询中（见“事务处理还是分析？”一节），想要计算大量记录的聚合是很常见的。在这种情况下，扫描整个输入可能是相当合理的事情，尤其是如果可以在多台机器上并行处理的话。

当我们在批处理主题中讨论连接时，我们是指解决数据集中所有的某种关联。例如，我们假设一项任务同时为所有用户处理数据，而不仅仅是查找某个特定用户的数据（使用索引可以更有效地完成这一任务）。

#### 案例：用户活动事件的分析

批处理任务中连接的典型案例如图10-2所示。左边是描述已登录的用户在网站上的操作事件日志（称为*活动事件*或*点击流数据*），右边是用户数据库。你可以把这个案例视为星型模式的一部分（见“星型与雪花型：用于分析的模式”一节）：事件日志是事实表，而用户数据库是维度之一。

*图10-2 在用户活动事件的日志与用户资料的数据库之间的连接。*

分析任务会需要把用户活动与用户资料信息相关联：例如，如果资料包含用户的年龄或出生日期，系统可以判断哪些页面在哪个年龄组最流行。然而，活动事件只包含用户ID，而不是完整的用户资料信息。在每个活动事件中都嵌入用户资料信息太浪费了。因此，活动事件需要与用户资料数据库连接。

这个连接最简单实现是挨个检查活动事件，然后（在远程服务器上）查询用户数据库中遇到的每个用户ID。这是可能的，但是性能可能会非常差：处理的吞吐量受到访问数据库服务器往返时间的限制，本地缓存的有效性很大程度上取决于数据的分布，而并行执行大量的查询很容易使数据库拥堵。

为了在批处理过程中获得良好的吞吐量，计算必须（尽可能多地）发生在本地单台设备上。通过网络对你想要处理的每一条记录进行随机访问请求太慢了。此外，查询远程数据库意味着批处理作业变得不确定起来，因为远程数据库中的数据可能会发生变化。

因此，更好的方法是获取用户数据库的副本（比如使用ETL进程从数据库备份中提取——见“数据仓库”一节），并把它放在用户活动事件日志所在的同一个分布式文件系统中。这样，用户数据库放在HDFS中的一组文件中而用户活动记录放在另一组文件中，于是可以使用MapReduce将所有相关记录集中在同一个地方，从而有效地处理它们。

#### 归并连接

回想一下，映射函数的目的是从每个输入记录中提取一个键和值。在图10-2的情况中，这个键是用户ID：一组映射函数将遍历活动事件（提取用户ID作为键而活动事件作为值），而另一组映射函数将遍历用户数据库（提取用户ID作为键而用户的出生日期作为值）。这个过程如图10-3所示.

*图10-3 归纳侧在用户ID上排序合并连接。如果输入数据集被分成数个文件，每一个文件可以用多个映射函数并行处理。*

当MapReduce框架把映射函数的输出按键分区，然后对键值对进行排序时，结果是所有活动事件和具有相同用户ID的用户记录在归纳函数的输入中彼此相邻。MapReduce任务甚至可以为记录排序，使得归纳函数总是先看到用户数据库的记录，然后才是按时间戳顺序排列的活动事件——这种技术被称为次级排序。

之后，归纳函数可以很容易地执行实际的连接逻辑：对每个用户ID都调用一次归纳函数，因为有了次级排序，第一个值应该是来自用户数据库的出生日期记录。归纳函数将出生日期存储在一个局部变量中，然后使用相同的用户ID迭代活动事件记录，输出一对浏览的URL与用户年龄。随后的MapReduce任务可以计算每个URL的用户年龄分布，并按年龄进行分组聚合计算。

由于归纳函数一次处理了特定用户ID的所有记录，因此在任何时候它都只需要在内存中保存一个用户记录，而且不需要通过网络发出任何请求。这个算法被称为*归并连接*，因为映射函数的输出是按键排序的，然后归纳函数将连接两边排好序的记录列表合并在一起。

#### 把相关数据放在一起

在归并连接中，映射函数和排序过程保证把执行特定用户ID连接操作所需的所有数据集中在同一个位置：这样只需要调用一次归纳函数。事先排列好所有需要的数据之后，归纳函数可以是一段相当简单的单线程代码，以很高的吞吐量和很低的内存开销遍历这些记录。

看待这个体系架构的一种方式是映射函数“发送消息”给归纳函数。当映射函数发送一个键值对时，键的作用类似于值被发送到的目标地址。尽管键只是一个任意字符串（而不是有着IP地址和端口号那样的真实网络地址），但它表现地像一个地址：所有具有相同键的键值对都将被发送到相同的目的地（对归纳函数的调用）。

使用MapReduce编程模型把真实世界中网络通信方面的计算（把数据发送到正确的设备）从应用程序逻辑（一旦获取数据就处理它）分离出来。这种分离与数据库的典型使用形成鲜明对比：从数据库获取数据的请求经常发生在应用程序代码的深处。由于MapReduce处理所有网络通信，这使得应用程序代码不用担心部分失效问题，比如其它节点的崩溃：MapReduce自动重试失败的任务而不影响应用程序逻辑。

#### GROUP BY

除了连接以外，“把相关数据放在一起”模式的另一种常见用法是按某个键（如SQL中的`GROUP BY`子句）对记录进行分组。所有具有相同键的记录构成一个组，下一步通常是在每个组内执行某种聚合操作——比如：

* 对每组中记录的数目计数（就像我们统计页面浏览量示例中，你可以用SQL中的`COUNT(*)`表示它）

* 用SQL把某个特定字段的值都加起来（`SUM(fieldname)`）

* 根据某个排序函数选出前*k*个记录

用MapReduce实现这种分组操作最简单方法是设置映射函数，从而生成的键值对使用期望的分组键。之后，分区与排序过程把有着相同键的所有记录放在同一个归纳函数中。因此，在MapReduce之上实现分组和连接看起来非常相似。

分组的另一种常见用途是整理特定用户会话的所有活动事件，方便找出用户采取的动作序列——这叫做*会话化*过程。例如，这样的分析可以用来计算那些浏览你的网站新版本的用户是否比那些浏览旧版本（A/B测试）的用户更愿意消费，或是计算一些营销活动是否物有所值。

如果有多个Web服务器处理用户请求，那么特定用户的活动事件很可能分散在不同服务器的日志文件中。你可以使用会话cookie、用户ID或类似的标识符作为分组键来实现会话化，并且把特定用户的所有活动事件集中在一个地方，同时把不同用户的事件分布在不同的分区中。

#### 处理倾斜

如果有大量数据与单个键有关，“把所有具有相同键的记录放在一起”的模式就会出问题。举个例子，在社交网络中，大多数用户只会与几百人有联系，但少数名人会拥有数百万粉丝。这种不成比例的活动数据库记录被称为*关键对象*或*热键*。

在单个归纳函数种收集与一个名人相关的所有活动（例如，对他们发表内容的回复）会导致明显的倾斜（也被称为*热点*）——也就是说，某个归纳函数必须比其他归纳函数处理了更多的记录（参见“倾斜的工作负载和缓解热点”）。因为MapReduce任务只有在所有的映射函数和归纳函数完成后才算完成，任何后续任务都必须等待最慢的归纳函数完成之后才可以进行。

如果连接的输入有热键，现在有了一些算法可以用来进行补偿。举个例子，Pig中的偏斜联接方法首先执行一个抽样任务从而判断哪些键是热键。在执行实际连接时，映射函数把与热键有关的任何记录发送给任意选择的几个归纳函数中的一个（与传统的MapReduce不同，后者基于键的散列值确定地选择归纳函数）。对于连接的其他输入，与热键相关的记录需要被复制到所有处理那个键的归纳函数中。

这种技术把处理热键的工作分散到多个归纳函数上，使得可以更好地并行化处理，而代价是不得不把其它的连接输入复制到多个归纳函数。Crunch中的分片连接方法也很类似，但需要显式地指定热键而不是使用抽样作业。这种技术也与我们在“倾斜工作负载和缓解热点”中讨论的技术非常类似，它使用了随机化来缓解分区数据库中的热点。

Hive的倾斜连接优化采用了另一种方法。它要求在表的元数据中显式指定热键，之后它会把与这些键相关的记录存储在单独的文件中，与其它记录隔离开。在该表上执行连接时，它对热键使用映射侧连接（见下一节）。

当用热键分组记录并对它们执行聚合操作时，可以用两个阶段执行分组。第一个MapReduce阶段把记录发送到一个随机归纳函数，从而每个归纳函数只在一部分热键记录上执行分组，输出更紧凑的聚合值。然后，第二个MapReduce任务把所有第一阶段归纳函数的值组合成每个键的单个值。

### 映射侧连接

上一节中描述的连接算法在归纳函数中执行实际的连接逻辑，因此被称为归纳侧连接。映射函数负责准备输入数据：从每条输入记录中提取键和值，把键值对分配给归纳函数分区，然后按键进行排序。

归纳侧方式的好处在于你不用对输入数据进行任何假设：无论它的属性和结构如何，映射函数都能为连接做好准备。然而，缺点是所有这些排序、复制到归纳函数然后合并归纳函数的输入，代价都是非常高的。根据可用的内存缓冲区大小，数据在通过MapReduce的各个阶段时有可能会多次写入磁盘。

另一方面，如果你可以对输入数据做出某些假设，使用所谓的映射侧连接可以使连接速度更快。这种方法使用了一个缩减了的MapReduce任务，其中没有归纳函数也没有排序。取而代之的是每个映射器只从分布式文件系统读取一个输入文件块，并只写入一个输出文件到文件系统——仅此而已。

#### 广播哈希连接

执行映射侧连接最简单的方式是在一个大数据集连接一个小数据集的时候。特别是，小数据集需要足够小，从而每个映射函数都可以把它整体加载到内存中。

例如在图10-2中，假设用户数据库足够小到可以放在内存中。在这种情况下，当映射函数启动时，它首先可以把用户数据库从分布式文件系统读入内存中的哈希表。一旦完成，映射函数就可以扫描用户活动事件，查找哈希表中每个时间的用户ID。

这里还有几个映射任务：一个是用于连接的每个大型输入的文件块（在图10-2的例子中，活动事件是大型输入）。每个映射函数都把小型输入完全加载到内存中。

这个简单但有效的算法称为广播哈希连接：广播这个词反映了这样一个事实：用于大型输入分区的每个映射函数都读取小型输入的全部内容（因此，小型输入有效地“广播”到了大型输入的所有分区），而散列这个词反映了它对哈希表的使用。这种连接方法由Pig（名为“复制连接”）、Hive（“MapJoin”）、Cascading和Crunch支持。它还用于数据仓库查询引擎，比如Impala。

与把小型连接输入加载到内存中的哈希表不同，另一种方法是把小型连接输入储存在本地磁盘上的只读索引中。索引中经常使用的部分会保留在操作系统的页面缓存中，因此这种方法可以提供几乎与内存哈希表一样快的随机访问查找，而实际上不需要把数据集放入内存。

#### 分区哈希连接

如果映射侧连接的输入也以相同的方式进行分区，那么哈希连接方法可以独立地应用于每个分区。在图10-2的场景中，你可以根据用户ID的最后一个十进制数字对活动事件和用户数据库进行分区（因此，任意一边都有10个分区）。例如，映射函数3首先把ID以3结尾的所有用户加载到哈希表中，然后扫描每个ID以3结尾的用户的所有活动事件。

如果分区操作正确，可以确保所有你希望连接的记录都位于同一个编号分区中，因此每个映射函数只从每个输入数据集中读取一个分区就足够了。这样做的优点是每个映射函数可以把较少的数据加载到它的哈希表中。

这种方法只有在两个连接输入都有相同数量的分区时才可以工作，并根据相同的键和相同的哈希函数把记录分配给分区。如果输入是由已经执行此分组的现有MapReduce任务生成的，那么这是一个合理的假设。

在Hive中，分区哈希连接称为分桶映射连接。

#### 映射侧合并连接

如果输入数据集不仅以相同的方式进行分区，而且根据相同的键进行排序，那么映射侧连接的另一个变体也适用。在这种情况下，输入是否足够小到可以放在内存里并不重要，因为映射函数可以执行通常由归纳函数执行的合并操作：递增地读取两个输入文件，按升键顺序，然后使用相同的键匹配记录。

如果映射侧合并连接是可能的，这可能意味着先前的MapReduce任务首先将输入数据集引入到这个分了区且排了序的表单中。原则上，这个连接可以在先前任务的归纳阶段执行。但是，在一个单独的、只有映射的任务中执行合并联接仍然是合适的，比如分了区且排了序的数据集除了这个特定的连接之外，还用于其他目的。

#### 有映射侧连接的MapReduce工作流

当MapReduce连接的输出被下游任务消耗时，映射侧或归纳侧连接的选择会影响输出的结构。归纳侧连接的输出通过连接键进行分区和排序，而映射端连接的输出以与大型输入相同的方式进行分区和排序（因为对于连接的大型输入的每个文件块都启动了一个映射任务，而不管是不是使用了分区连接还是广播连接）。

正如前面所讨论的，映射侧连接还对输入数据集的大小、排序和分区做出了更多假设。在优化连接策略时，了解分布式文件系统中数据集的物理布局变得很重要：仅仅知道存储数据目录的编码格式以及名称是不够的；你还必须知道分区的数量以及数据被分区和排序的键。

在Hadoop生态系统中，这类关于数据集划分的元数据通常保存在HCatalog和Hive元存储中。

### 批处理工作流的输出

我们已经讨论了许多实现MapReduce任务工作流的算法，但是我们忽略了一个重要问题：一旦完成了这些处理，结果是什么？为什么我们首先要执行所有这些任务？

对于数据库查询，我们将事务处理（OLTP）的目的与分析目的区分开（见“事务处理还是分析？”一节）。我们看到，OLTP查询通常使用索引按键查找少量记录，以便将它们呈现给用户（比如在网页上）。另一方面，分析性查询通常会扫描大量记录，执行分组和聚合操作，并且输出通常以报告的形式出现：可以是一张显示某项指标随时间的变化的图表，或者是根据某种排名规则排列的前10项，要么是将一定数量的记录细分为几个子类。这类报告的使用者通常是需要作出业务决策的分析师或是经理。

批处理适合出现在哪里？它既不是事务处理，也不是分析任务。它更接近于分析任务，在这个过程中批处理通常会扫描输入数据集的大部分。然而，MapReduce任务的工作流与用于分析目的SQL查询不一样（见“将Hadoop与分布式数据库进行比较”一节）。批处理过程的输出通常不是报表，而是某种其他类型的结构。

#### 构建搜索索引

谷歌最初使用MapReduce是为了它的搜索引擎建立索引，它是由5到10个MapReduce作业的工作流实现的。尽管谷歌后来不再使用MapReduce来实现这个目标，但是如果从构建搜索索引的角度来看，它能帮助你了解MapReduce。（即使在今天，Hadoop MapReduce仍然是为Lucene/Solr构建索引的好方法。）

我们在“全文搜索和模糊索引”一节中简要地看到了全文搜索索引，比如Lucene，是如何工作的：它是一个文件（术语字典），在该文件中你可以有效地查找特定关键字，并找到包含这个关键字的所有文档ID列表（公告列表）。这是搜索索引一个非常简单的视图——实际上它需要许多额外的数据，以便按照相关性对搜索结果进行排序，纠正拼写错误，解析同义词等等——但原则是一直不变的。

如果你需要对一组固定的文档执行全文搜索，那么批处理过程是构建索引的一种非常有效的方法：映射函数根据需要划分文档集合，每个归纳函数为它处理的分区构建索引，之后索引文件被写入分布式文件系统中。构建这样的文档分区索引（见“分区与次级索引”一节)可以非常好地并行化。由于按关键字查询搜索索引是只读操作，这些索引文件一旦创建就不可变。

如果已经建立索引的文档集合变了，一个选择是定期重新运行整个文档集的索引建立工作流，并在完成时用新的索引文件整体替换先前的索引文件。如果只是少量文档发生了变化，这种方法在计算上代价会很高，但是优点是索引建立过程非常容易解释：输入文档，输入索引。

或者可以增量构建索引。如第3章所述，如果想在索引中添加、删除或更新文档，Lucene将写出新的段文件，并在后台异步合并、压缩段文件。我们会在第11章中看到更多关于这种增量处理的信息。

#### Key-value stores as batch process output

Search indexes are just one example of the possible outputs of a batch processing workflow. Another common use for batch processing is to build machine learning systems such as classifiers (e.g., spam filters, anomaly detection, image recognition) and recommendation systems (e.g., people you may know, products you may be interested in, or related searches [29]).

The output of those batch jobs is often some kind of database: for example, a database that can be queried by user ID to obtain suggested friends for that user, or a database that can be queried by product ID to get a list of related products [45].

These databases need to be queried from the web application that handles user requests, which is usually separate from the Hadoop infrastructure. So how does the output from the batch process get back into a database where the web application can query it?

The most obvious choice might be to use the client library for your favorite database directly within a mapper or reducer, and to write from the batch job directly to the database server, one record at a time. This will work (assuming your firewall rules allow direct access from your Hadoop environment to your production databases), but it is a bad idea for several reasons:

* As discussed previously in the context of joins, making a network request for every single record is orders of magnitude slower than the normal throughput of a batch task. Even if the client library supports batching, performance is likely to be poor.

* MapReduce jobs often run many tasks in parallel. If all the mappers or reducers concurrently write to the same output database, with a rate expected of a batch process, that database can easily be overwhelmed, and its performance for queries is likely to suffer. This can in turn cause operational problems in other parts of the system [35].

* Normally, MapReduce provides a clean all-or-nothing guarantee for job output: if a job succeeds, the result is the output of running every task exactly once, even if some tasks failed and had to be retried along the way; if the entire job fails, no output is produced. However, writing to an external system from inside a job produces externally visible side effects that cannot be hidden in this way. Thus, you have to worry about the results from partially completed jobs being visible to other systems, and the complexities of Hadoop task attempts and speculative execution.

A much better solution is to build a brand-new database inside the batch job and write it as files to the job’s output directory in the distributed filesystem, just like the search indexes in the last section. Those data files are then immutable once written, and can be loaded in bulk into servers that handle read-only queries. Various key-value stores support building database files in MapReduce jobs, including Voldemort [46], Terrapin [47], ElephantDB [48], and HBase bulk loading [49].

Building these database files is a good use of MapReduce: using a mapper to extract a key and then sorting by that key is already a lot of the work required to build an index. Since most of these key-value stores are read-only (the files can only be written once by a batch job and are then immutable), the data structures are quite simple. For example, they do not require a WAL (see “Making B-trees reliable”).

When loading data into Voldemort, the server continues serving requests to the old data files while the new data files are copied from the distributed filesystem to the server’s local disk. Once the copying is complete, the server atomically switches over to querying the new files. If anything goes wrong in this process, it can easily switch back to the old files again, since they are still there and immutable [46].

#### Philosophy of batch process outputs

The Unix philosophy that we discussed earlier in this chapter (“ The Unix Philosophy”) encourages experimentation by being very explicit about dataflow: a program reads its input and writes its output. In the process, the input is left unchanged, any previous output is completely replaced with the new output, and there are no other side effects. This means that you can rerun a command as often as you like, tweaking or debugging it, without messing up the state of your system.

The handling of output from MapReduce jobs follows the same philosophy. By treating inputs as immutable and avoiding side effects (such as writing to external databases), batch jobs not only achieve good performance but also become much easier to maintain:

* If you introduce a bug into the code and the output is wrong or corrupted, you can simply roll back to a previous version of the code and rerun the job, and the output will be correct again. Or, even simpler, you can keep the old output in a different directory and simply switch back to it. Databases with read-write transactions do not have this property: if you deploy buggy code that writes bad data to the database, then rolling back the code will do nothing to fix the data in the database. (The idea of being able to recover from buggy code has been called human fault tolerance [50].)

* As a consequence of this ease of rolling back, feature development can proceed more quickly than in an environment where mistakes could mean irreversible damage. This principle of minimizing irreversibility is beneficial for Agile software development [51].

* If a map or reduce task fails, the MapReduce framework automatically re-schedules it and runs it again on the same input. If the failure is due to a bug in the code, it will keep crashing and eventually cause the job to fail after a few attempts; but if the failure is due to a transient issue, the fault is tolerated. This automatic retry is only safe because inputs are immutable and outputs from failed tasks are discarded by the MapReduce framework.

* The same set of files can be used as input for various different jobs, including monitoring jobs that calculate metrics and evaluate whether a job’s output has the expected characteristics (for example, by comparing it to the output from the previous run and measuring discrepancies).

* Like Unix tools, MapReduce jobs separate logic from wiring (configuring the input and output directories), which provides a separation of concerns and enables potential reuse of code: one team can focus on implementing a job that does one thing well, while other teams can decide where and when to run that job.

In these areas, the design principles that worked well for Unix also seem to be working well for Hadoop — but Unix and Hadoop also differ in some ways. For example, because most Unix tools assume untyped text files, they have to do a lot of input parsing (our log analysis example at the beginning of the chapter used {print $ 7} to extract the URL). On Hadoop, some of those low-value syntactic conversions are eliminated by using more structured file formats: Avro (see “Avro”) and Parquet (see “Column-Oriented Storage”) are often used, as they provide efficient schema-based encoding and allow evolution of their schemas over time (see Chapter   4).

### Comparing Hadoop to Distributed Databases

As we have seen, Hadoop is somewhat like a distributed version of Unix, where HDFS is the filesystem and MapReduce is a quirky implementation of a Unix process (which happens to always run the sort utility between the map phase and the reduce phase). We saw how you can implement various join and grouping operations on top of these primitives.

When the MapReduce paper [1] was published, it was — in some sense — not at all new. All of the processing and parallel join algorithms that we discussed in the last few sections had already been implemented in so-called massively parallel processing (MPP) databases more than a decade previously [3, 40]. For example, the Gamma database machine, Teradata, and Tandem NonStop SQL were pioneers in this area [52].

The biggest difference is that MPP databases focus on parallel execution of analytic SQL queries on a cluster of machines, while the combination of MapReduce and a distributed filesystem [19] provides something much more like a general-purpose operating system that can run arbitrary programs.

#### Diversity of storage

Databases require you to structure data according to a particular model (e.g., relational or documents), whereas files in a distributed filesystem are just byte sequences, which can be written using any data model and encoding. They might be collections of database records, but they can equally well be text, images, videos, sensor readings, sparse matrices, feature vectors, genome sequences, or any other kind of data.

To put it bluntly, Hadoop opened up the possibility of indiscriminately dumping data into HDFS, and only later figuring out how to process it further [53]. By contrast, MPP databases typically require careful up-front modeling of the data and query patterns before importing the data into the database’s proprietary storage format.

From a purist’s point of view, it may seem that this careful modeling and import is desirable, because it means users of the database have better-quality data to work with. However, in practice, it appears that simply making data available quickly — even if it is in a quirky, difficult-to-use, raw format — is often more valuable than trying to decide on the ideal data model up front [54].

The idea is similar to a data warehouse (see “Data Warehousing”): simply bringing data from various parts of a large organization together in one place is valuable, because it enables joins across datasets that were previously disparate. The careful schema design required by an MPP database slows down that centralized data collection; collecting data in its raw form, and worrying about schema design later, allows the data collection to be speeded up (a concept sometimes known as a “data lake” or “enterprise data hub” [55]).

Indiscriminate data dumping shifts the burden of interpreting the data: instead of forcing the producer of a dataset to bring it into a standardized format, the interpretation of the data becomes the consumer’s problem (the schema-on-read approach [56]; see “Schema flexibility in the document model”). This can be an advantage if the producer and consumers are different teams with different priorities. There may not even be one ideal data model, but rather different views onto the data that are suitable for different purposes. Simply dumping data in its raw form allows for several such transformations. This approach has been dubbed the sushi principle: “raw data is better” [57].

Thus, Hadoop has often been used for implementing ETL processes (see “Data Warehousing”): data from transaction processing systems is dumped into the distributed filesystem in some raw form, and then MapReduce jobs are written to clean up that data, transform it into a relational form, and import it into an MPP data warehouse for analytic purposes. Data modeling still happens, but it is in a separate step, decoupled from the data collection. This decoupling is possible because a distributed filesystem supports data encoded in any format.

#### Diversity of processing models

MPP databases are monolithic, tightly integrated pieces of software that take care of storage layout on disk, query planning, scheduling, and execution. Since these components can all be tuned and optimized for the specific needs of the database, the system as a whole can achieve very good performance on the types of queries for which it is designed. Moreover, the SQL query language allows expressive queries and elegant semantics without the need to write code, making it accessible to graphical tools used by business analysts (such as Tableau).

On the other hand, not all kinds of processing can be sensibly expressed as SQL queries. For example, if you are building machine learning and recommendation systems, or full-text search indexes with relevance ranking models, or performing image analysis, you most likely need a more general model of data processing. These kinds of processing are often very specific to a particular application (e.g., feature engineering for machine learning, natural language models for machine translation, risk estimation functions for fraud prediction), so they inevitably require writing code, not just queries.

MapReduce gave engineers the ability to easily run their own code over large datasets. If you have HDFS and MapReduce, you can build a SQL query execution engine on top of it, and indeed this is what the Hive project did [31]. However, you can also write many other forms of batch processes that do not lend themselves to being expressed as a SQL query.

Subsequently, people found that MapReduce was too limiting and performed too badly for some types of processing, so various other processing models were developed on top of Hadoop (we will see some of them in “Beyond MapReduce”). Having two processing models, SQL and MapReduce, was not enough: even more different models were needed! And due to the openness of the Hadoop platform, it was feasible to implement a whole range of approaches, which would not have been possible within the confines of a monolithic MPP database [58].

Crucially, those various processing models can all be run on a single shared-use cluster of machines, all accessing the same files on the distributed filesystem. In the Hadoop approach, there is no need to import the data into several different specialized systems for different kinds of processing: the system is flexible enough to support a diverse set of workloads within the same cluster. Not having to move data around makes it a lot easier to derive value from the data, and a lot easier to experiment with new processing models.

The Hadoop ecosystem includes both random-access OLTP databases such as HBase (see “SSTables and LSM-Trees”) and MPP-style analytic databases such as Impala [41]. Neither HBase nor Impala uses MapReduce, but both use HDFS for storage. They are very different approaches to accessing and processing data, but they can nevertheless coexist and be integrated in the same system.

#### Designing for frequent faults

When comparing MapReduce to MPP databases, two more differences in design approach stand out: the handling of faults and the use of memory and disk. Batch processes are less sensitive to faults than online systems, because they do not immediately affect users if they fail and they can always be run again.

If a node crashes while a query is executing, most MPP databases abort the entire query, and either let the user resubmit the query or automatically run it again [3]. As queries normally run for a few seconds or a few minutes at most, this way of handling errors is acceptable, since the cost of retrying is not too great. MPP databases also prefer to keep as much data as possible in memory (e.g., using hash joins) to avoid the cost of reading from disk.

On the other hand, MapReduce can tolerate the failure of a map or reduce task without it affecting the job as a whole by retrying work at the granularity of an individual task. It is also very eager to write data to disk, partly for fault tolerance, and partly on the assumption that the dataset will be too big to fit in memory anyway.

The MapReduce approach is more appropriate for larger jobs: jobs that process so much data and run for such a long time that they are likely to experience at least one task failure along the way. In that case, rerunning the entire job due to a single task failure would be wasteful. Even if recovery at the granularity of an individual task introduces overheads that make fault-free processing slower, it can still be a reasonable trade-off if the rate of task failures is high enough.

But how realistic are these assumptions? In most clusters, machine failures do occur, but they are not very frequent — probably rare enough that most jobs will not experience a machine failure. Is it really worth incurring significant overheads for the sake of fault tolerance?

To understand the reasons for MapReduce’s sparing use of memory and task-level recovery, it is helpful to look at the environment for which MapReduce was originally designed. Google has mixed-use datacenters, in which online production services and offline batch jobs run on the same machines. Every task has a resource allocation (CPU cores, RAM, disk space, etc.) that is enforced using containers. Every task also has a priority, and if a higher-priority task needs more resources, lower-priority tasks on the same machine can be terminated (preempted) in order to free up resources. Priority also determines pricing of the computing resources: teams must pay for the resources they use, and higher-priority processes cost more [59].

This architecture allows non-production (low-priority) computing resources to be overcommitted, because the system knows that it can reclaim the resources if necessary. Overcommitting resources in turn allows better utilization of machines and greater efficiency compared to systems that segregate production and non-production tasks. However, as MapReduce jobs run at low priority, they run the risk of being preempted at any time because a higher-priority process requires their resources. Batch jobs effectively “pick up the scraps under the table,” using any computing resources that remain after the high-priority processes have taken what they need.

At Google, a MapReduce task that runs for an hour has an approximately 5% risk of being terminated to make space for a higher-priority process. This rate is more than an order of magnitude higher than the rate of failures due to hardware issues, machine reboot, or other reasons [59]. At this rate of preemptions, if a job has 100 tasks that each run for 10 minutes, there is a risk greater than 50% that at least one task will be terminated before it is finished.

And this is why MapReduce is designed to tolerate frequent unexpected task termination: it’s not because the hardware is particularly unreliable, it’s because the freedom to arbitrarily terminate processes enables better resource utilization in a computing cluster.

Among open source cluster schedulers, preemption is less widely used. YARN’s CapacityScheduler supports preemption for balancing the resource allocation of different queues [58], but general priority preemption is not supported in YARN, Mesos, or Kubernetes at the time of writing [60]. In an environment where tasks are not so often terminated, the design decisions of MapReduce make less sense. In the next section, we will look at some alternatives to MapReduce that make different design decisions.

