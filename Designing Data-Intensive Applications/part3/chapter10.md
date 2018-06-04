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

It’s no coincidence that we were able to analyze a log file quite easily, using a chain of commands like in the previous example: this was in fact one of the key design ideas of Unix, and it remains astonishingly relevant today. Let’s look at it in some more depth so that we can borrow some ideas from Unix [10].

Doug McIlroy, the inventor of Unix pipes, first described them like this in 1964 [11]: “We should have some ways of connecting programs like [a] garden hose — screw in another segment when it becomes necessary to massage data in another way. This is the way of I/ O also.” The plumbing analogy stuck, and the idea of connecting programs with pipes became part of what is now known as the Unix philosophy — a set of design principles that became popular among the developers and users of Unix. The philosophy was described in 1978 as follows [12, 13]:

1. Make each program do one thing well. To do a new job, build afresh rather than complicate old programs by adding new “features”.

2. Expect the output of every program to become the input to another, as yet unknown, program. Don’t clutter output with extraneous information. Avoid stringently columnar or binary input formats. Don’t insist on interactive input.

3. Design and build software, even operating systems, to be tried early, ideally within weeks. Don’t hesitate to throw away the clumsy parts and rebuild them.

4. Use tools in preference to unskilled help to lighten a programming task, even if you have to detour to build the tools and expect to throw some of them out after you’ve finished using them.

This approach — automation, rapid prototyping, incremental iteration, being friendly to experimentation, and breaking down large projects into manageable chunks — sounds remarkably like the Agile and DevOps movements of today. Surprisingly little has changed in four decades.

The sort tool is a great example of a program that does one thing well. It is arguably a better sorting implementation than most programming languages have in their standard libraries (which do not spill to disk and do not use multiple threads, even when that would be beneficial). And yet, sort is barely useful in isolation. It only becomes powerful in combination with the other Unix tools, such as uniq.

A Unix shell like bash lets us easily compose these small programs into surprisingly powerful data processing jobs. Even though many of these programs are written by different groups of people, they can be joined together in flexible ways. What does Unix do to enable this composability?