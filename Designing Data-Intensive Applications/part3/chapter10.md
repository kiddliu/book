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

批处理系统接受大量的输入数据，启动一个作业来处理它，然后产生一些输出数据。作业通常需要花费一段时间（从几分钟到几天不等），因此通常不会有用户等待作业完成。取而代之的是，批处理作业通常定期运行（比如一天一次）。批处理作业的主要性能衡量指标通常是吞吐量（处理特定大小输入数据集所花费的时间）。在这一章里，我们将讨论批处理。

*流处理系统（近似实时系统）*

流处理介于在线处理和离线/批处理之间（因此有时被称为近似实时或接近线的处理）。与批处理系统类似，流处理器消耗输入并产生输出（而不是响应请求）。然而，流作业在事件发生之后不久就会运行，而批处理作业则在固定的输入数据集上操作。这种差异使得流处理系统比等效的批处理系统具有更低的延迟时间。由于流处理构建在批处理的基础上，我们将在第11章中讨论它。

正如我们将在本章中看到的，批处理是构建可靠的、可扩展的和可维护的应用程序的重要组成部分。举个例子，2004年发布的批处理算法MapReduce，被称为“使谷歌如此大规模可扩展的算法”。之后在各种开放源码数据系统中都有实现，包括Hadoop、CouchDB和MongoDB。

与许多年前为数据仓库开发的并行处理系统相比，MapReduce是一个相当低层级的编程模型，但是它在商业性硬件上可以实现的处理规模方面是向前迈进了一大步。尽管现在MapReduce的重要性正在下降，但仍然值得去理解它，因为它清楚地说明了批处理为什么以及如何有用。

事实上，批处理是一种非常古老的计算形式。早在可编程数字计算机被发明之前，穿孔卡片制表机——如1890年美国人口普查中使用的何乐礼机——实现了一种半机械化的批处理形式，用以从大量的输入计算总的统计数据。并且MapReduce与二十世纪四五十年代广泛用于商业数据处理的机电式IBM卡片分类机有着惊人的相似之处。如往常一样，历史有重复自己的倾向。

在本章中，我们将研究MapReduce和其他几个批处理算法和框架，并探讨它们是如何在现代数据系统中使用的。但是首先，我们先使用标准Unix工具看看数据处理。即使你已经很熟悉它们了，还是要提醒你一下Unix哲学还是很有价值的，因为Unix的思想和经验传递给了大规模、异构的分布式数据系统。

## 用Unix工具进行批处理

让我们从一个简单的例子开始。假设您有一个Web服务器，每次服务了请求之后都会在日志文件添加一行。举个例子，使用nginx默认访问日志格式，其中一行可能如下所示：

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

各种工具可以读取这些日志文件并生成关于你的网站流量的漂亮报告，但为了便于练习，让我们使用基本的Unix工具构建自己的日志文件。例如，假设你想在你的网站上找到最受欢迎的五个页面。您可以在Unix命令行中这样做，如下所示：

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

#### Chain of commands versus custom program

Instead of the chain of Unix commands, you could write a simple program to do the same thing. For example, in Ruby, it might look something like this:

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

➊ `counts` is a hash table that keeps a counter for the number of times we’ve seen each URL. A counter is zero by default.

➋ From each line of the log, we take the URL to be the seventh whitespace-separated field (the array index here is 6 because Ruby’s arrays are zero-indexed).

➌ Increment the counter for the URL in the current line of the log.

➍ Sort the hash table contents by counter value (descending), and take the top five entries.

➎ Print out those top five entries.

This program is not as concise as the chain of Unix pipes, but it’s fairly readable, and which of the two you prefer is partly a matter of taste. However, besides the superficial syntactic differences between the two, there is a big difference in the execution flow, which becomes apparent if you run this analysis on a large file.