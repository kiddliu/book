# 第三部分 衍生数据

在本书的第一部分和第二部分中，我们收集了所有与分布式数据库有关的主要考虑因素，从磁盘上的数据布局一直到出现故障时分布式一致性的限制。然而，这种讨论假定应用程序中只有一个数据库。

实际上，数据系统往往更为复杂。在大型应用程序中，你通常需要能够以多种不同的方式访问处理数据，而没有数据库能够同时满足所有这些不同的需求。因此，应用程序通常使用由几个不同的数据存储、索引、缓存、分析系统等组成的组合，并且实现将数据从一个存储转移到另一个存储的机制。

在本书的这最后一部分，我们将研究关于集成多个不同数据系统成为连贯的应用程序架构的问题，这些系统可能有着不同的数据模型，并且针对不同的访问模式进行了优化。这一种系统构建常常被那些声称它们的产品能满足所有需求的厂商忽视。在现实中，集成不同的系统是复杂应用需要完成的最重要的事情之一。

# 记录系统与衍生数据

在高层次上，存储和处理数据的系统可以分为两大类：

*记录系统*

记录系统，也被称为事实来源，存储着数据的权威版本。当新数据输入时，例如用户输入，它首先写入到这里。每个事实精确地只表示一次（表现形式通常是规范化了的）。如果另一个系统和记录系统之间有任何差异，那么（根据定义）记录系统中的值是正确的。

*衍生数据系统*

衍生系统中的数据是从另一个系统获取一些现有数据并以某种方式转换或处理的结果。如果衍生数据丢失了，你可以从原始源重新创建它。一个典型的例子就是缓存：如果数据出现在缓存中，那么可以直接提供数据，但是如果缓存没有需要的内容，可以回退到底层数据库获取数据。非规范化值、索引和物化视图也属于这个类别。在推荐系统中，预测性的汇总数据通常来自使用日志。

从技术上讲，派生数据是冗余的，因为它复制了现有的信息。然而，它在读取查询上获得良好的性能通常是必不可少的。它通常是非正规化的。你可以从单个源派生出几个不同的数据集，使你能够从不同的“视角”查看数据。

并不是所有的系统都在架构中明确区分了记录系统和派生数据，但是这是一个非常有用的区别，因为它阐明了整个系统的数据流：它明确了系统的哪些部分有哪些输入和输出，而它们又是如何相互依赖的。

绝大多数数据库、存储引擎以及查询语言本质上既不是记录系统，也不是派生系统。数据库只是一个工具：你如何使用它取决于你自己。记录系统和派生数据系统之间的差别不取决于工具，而是取决于你如何在应用程序中使用它。

通过弄清楚哪些数据是从哪个其他数据派生出来的，你就可以让令人困惑的系统架构变得清晰起来。这一点将是贯穿本书这一部分的主题。

# 章节总览

从第10章开始我们将研究面向批处理的数据流系统，比如MapReduce，看看它们如何为我们提供构建大规模数据系统的好工具和原则。在第11章中，我们将把这些想法应用到数据流中，这样我们就可以用更低的延迟来做同样的事情。第12章总结了整本书，探讨我们如何使用这些工具来构建可靠的、可扩展的和可维护的应用程序的想法。