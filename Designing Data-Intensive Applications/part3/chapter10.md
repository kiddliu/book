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

虽然并不完美，但即使在几十年后，Unix统一接口仍然非常出色。没有多少软件可以像Unix工具一样互相操作和撰写：您无法轻松地通过自定义分析工具将电子邮件帐户的内容和在线购物历史记录输入电子表格，然后把结果发布到社交网络或一个wiki页面上。今天，可以像Unix工具一样顺利地协同工作的程序是一个例外，而不是正常情况。

即使是具有*相同数据模型*的数据库，也往往不容易把数据从一个库转移到另一个中。集成的缺乏导致了数据的碎片化。

#### 逻辑与连接的分离

Unix工具的另一个特点是它们使用标准输入（`stdin`）和标准输出（`stdout`）。如果您运行一个程序，而不指定任何其他内容，`stdin`来自键盘而`stdout`将转到屏幕上。但是，您也可以从文件中获取输入和/或将输出重定向到文件。管道允许您将一个进程的stdout附加到另一个进程的stdin(使用一个小内存缓冲区，并且不将整个中间数据流写入磁盘)。

Another characteristic feature of Unix tools is their use of standard input (stdin) and standard output (stdout). If you run a program and don’t specify anything else, stdin comes from the keyboard and stdout goes to the screen. However, you can also take input from a file and/ or redirect output to a file. Pipes let you attach the stdout of one process to the stdin of another process (with a small in-memory buffer, and without writing the entire intermediate data stream to disk).

A program can still read and write files directly if it needs to, but the Unix approach works best if a program doesn’t worry about particular file paths and simply uses stdin and stdout. This allows a shell user to wire up the input and output in whatever way they want; the program doesn’t know or care where the input is coming from and where the output is going to. (One could say this is a form of loose coupling, late binding [15], or inversion of control [16].) Separating the input/ output wiring from the program logic makes it easier to compose small tools into bigger systems.

You can even write your own programs and combine them with the tools provided by the operating system. Your program just needs to read input from stdin and write output to stdout, and it can participate in data processing pipelines. In the log analysis example, you could write a tool that translates user-agent strings into more sensible browser identifiers, or a tool that translates IP addresses into country codes, and simply plug it into the pipeline. The sort program doesn’t care whether it’s communicating with another part of the operating system or with a program written by you.

However, there are limits to what you can do with stdin and stdout. Programs that need multiple inputs or outputs are possible but tricky. You can’t pipe a program’s output into a network connection [17, 18]. iii If a program directly opens files for reading and writing, or starts another program as a subprocess, or opens a network connection, then that I/ O is wired up by the program itself. It can still be configurable (through command-line options, for example), but the flexibility of wiring up inputs and outputs in a shell is reduced.

#### Transparency and experimentation

Part of what makes Unix tools so successful is that they make it quite easy to see what is going on:

* The input files to Unix commands are normally treated as immutable. This means you can run the commands as often as you want, trying various command-line options, without damaging the input files.

* You can end the pipeline at any point, pipe the output into less, and look at it to see if it has the expected form. This ability to inspect is great for debugging.

* You can write the output of one pipeline stage to a file and use that file as input to the next stage. This allows you to restart the later stage without rerunning the entire pipeline.

Thus, even though Unix tools are quite blunt, simple tools compared to a query optimizer of a relational database, they remain amazingly useful, especially for experimentation.

However, the biggest limitation of Unix tools is that they run only on a single machine — and that’s where tools like Hadoop come in.