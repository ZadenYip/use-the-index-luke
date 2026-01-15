# Developers Need to Index
# 开发者需要索引

原文：[Preface](https://use-the-index-luke.com/sql/preface)

SQL performance problems are as old as SQL itself—some might even say that SQL is inherently slow. Although this might have been true in the early days of SQL, it is definitely not true anymore. Nevertheless SQL performance problems are still commonplace. How does this happen?
SQL 性能问题与 SQL 语言本身一样历史悠久——有些人甚至会说 SQL **天生**就是慢的。尽管在 SQL 诞生初期情况确实如此，但这绝对已成往事。然而，SQL 性能问题依然**司空见惯**。这是怎么回事呢？

The SQL language is perhaps the most successful fourth-generation programming language (4GL). Its main benefit is the capability to separate “what” and “how”. An SQL statement is a straight description of what is needed without instructions as to how to get it done. Consider the following example:
SQL 语言也许是最成功的第四代编程语言（4GL）。它的主要优势在于能够将「做什么」和「怎么做」分离开来。一条 SQL 语句是对所需内容的**直观描述**，而不包含关于如何完成该任务的指令。请看下面的例子：

```
SELECT date_of_birth
FROM employees
WHERE last_name = 'WINAND'
```

The SQL query reads like an English sentence that explains the requested data. Writing SQL statements generally does not require any knowledge about inner workings of the database or the storage system (such as disks, files, etc.). There is no need to tell the database which files to open or how to find the requested rows. Many developers have years of SQL experience yet they know very little about the processing that happens in the database.
这条 SQL 查询读起来就像是一个解释所需数据的英语句子。编写 SQL 语句通常不需要了解数据库或存储系统（如磁盘、文件等）的**内部运作机制**。不需要告诉数据库去打开哪些文件，也不需要告诉它如何找到被请求的行。许多开发者拥有多年的 SQL 经验，却对数据库内部发生的处理过程知之甚少。

The separation of concerns—what is needed versus how to get it—works remarkably well in SQL, but it is still not perfect. The abstraction reaches its limits when it comes to performance: the author of an SQL statement by definition does not care how the database executes the statement. Consequently, the author is not responsible for slow execution. However, experience proves the opposite; i.e., the author must know a little bit about the database to prevent performance problems.
关注点分离——即「需要什么」与「如何获取」的分离——在 SQL 中效果显著，但它并非完美无缺。当涉及到**性能**时，这种抽象机制便遇到了瓶颈：根据定义，SQL 语句的作者并不关心数据库如何执行该语句。因此，作者无需对执行缓慢负责。然而，经验证明恰恰相反；也就是说，作者必须了解一点数据库知识，以防止性能问题。

It turns out that the only thing developers need to learn is how to index. Database indexing is, in fact, a development task. That is because the most important information for proper indexing is not the storage system configuration or the hardware setup. The most important information for indexing is how the application queries the data. This knowledge—about the access path—is not very accessible to database administrators (DBAs) or external consultants. Quite some time is needed to gather this information through reverse engineering of the application: development, on the other hand, has that information anyway.
事实证明，开发者唯一需要学习的就是**如何建立索引**。实际上，数据库索引是一项**开发**任务。这是因为，对于正确的索引而言，最重要的信息并非存储系统配置或硬件设置，而是应用程序如何**查询**数据。关于「访问路径」的知识，对于数据库管理员（DBA）或外部顾问来说并不容易获取。通过逆向工程来推导这些信息需要耗费大量时间；而另一方面，**开发人员**本身就已经掌握了这些信息。

This book covers everything developers need to know about indexes—and nothing more. To be more precise, the book covers the most important index type only: the B-tree index.
本书涵盖了开发者需要了解的关于索引的**一切**——且仅此而已。确切地说，本书只涵盖最重要的一种索引类型：**B-树**索引。

If you like this page, you might also like …
… to subscribe my mailing lists, get free stickers, buy my book or join a training.
如果您喜欢这个页面，您可能也会喜欢……
……订阅我的邮件列表，获取免费贴纸，购买我的书或参加培训。

The B-tree index works almost identically in many databases. The book primarily uses the terminology of the Oracle® database, but refers to the corresponding terms of other database where appropriate. Side notes provide more information about MySQL, PostgreSQL and SQL Server®.
**B-树**索引在许多数据库中的工作原理几乎完全相同。本书主要使用 Oracle® 数据库的术语，但在适当的地方会提及其他数据库的对应术语。旁注提供了关于 MySQL、PostgreSQL 和 SQL Server® 的更多信息。

The structure of the book is tailor-made for developers; most chapters correspond to a particular part of an SQL statement.
本书的结构是为开发者量身定制的；大多数章节都对应于 SQL 语句的特定部分。

[CHAPTER 1 - Anatomy of an Index](todo)
[第一章 - 索引的解剖结构](todo)

The first chapter is the only one that doesn’t cover SQL specifically; it is about the fundamental structure of an index. An understanding of the index structure is essential to following the later chapters—don’t skip this!
第一章是唯一不专门讲解 SQL 的章节；它是关于索引的**基础结构**。理解索引结构对于跟上后续章节的内容至关重要——**不要**跳过这一章！

Although the chapter is rather short—only about eight pages—after working through the chapter you will already understand the phenomenon of slow indexes.
尽管这一章很短——只有大约八页——但在学完本章后，您将已经能够理解**慢索引**的现象。

[CHAPTER 2 - The Where Clause](todo)
[第二章 - Where 子句](todo)

This is where we pull out all the stops. This chapter explains all aspects of the where clause, from very simple single column lookups to complex clauses for ranges and special cases such as LIKE.
这是我们**火力全开**的部分。本章解释了 Where 子句的所有方面，从非常简单的单列查找到用于范围查询的复杂子句，以及像 `LIKE` 这样的特殊情况。

This chapter makes up the main body of the book. Once you learn to use these techniques, you will write much faster SQL.
本章构成了本书的主体。一旦您学会使用这些技术，您写出的 SQL 运行速度将快得多。

[CHAPTER 3 - Performance and Scalability](todo)
[第三章 - 性能和可扩展性](todo)

This chapter is a little digression about performance measurements and database scalability. See why adding hardware is not the best solution to slow queries.
本章稍微离题，讨论性能测量和数据库可扩展性。了解为什么增加硬件并非解决慢查询的最佳方案。

[CHAPTER 4 - The Join Operation](todo)
[第四章 - 连接操作](todo)

Back to SQL: here you will find an explanation of how to use indexes to perform a fast table join.
回到 SQL：在这里您将找到关于如何使用索引来执行快速**表连接**的解释。

[CHAPTER 5 - Clustering Data](todo)
[第五章 - 聚簇数据](todo)

Have you ever wondered if there is any difference between selecting a single column or all columns? Here is the answer—along with a trick to get even better performance.
您是否曾想过，选择**单列**与选择**所有列**之间是否有区别？这里有答案——还附带一个能获得更好性能的技巧。

[CHAPTER 6 - Sorting and Grouping](todo)
[第六章 - 排序和分组](todo)

Even order by and group by can use indexes.
即使 `order by` 和 `group by` 也能使用索引。

[CHAPTER 7 - Partial Results](todo)
[第七章 - 部分结果](todo)

This chapter explains how to benefit from a “pipelined” execution if you don’t need the full result set.
如果您不需要完整的结果集，本章将解释如何从「流水线」执行中获益。

[CHAPTER 8 - Insert, Delete and Update](todo)
[第八章 - 插入、删除和更新](todo)

How do indexes affect write performance? Indexes don’t come for free—use them wisely!
索引如何影响写入性能？索引不是免费午餐——请明智地使用它们！

[APPENDIX A - Execution Plans](todo)
[附录 A - 执行计划](todo)

Asking the database how it executes a statement.
询问数据库它是如何执行一条语句的。

[APPENDIX B - Myth Directory](todo)
[附录 B - 迷思目录](todo)

Lists some common myth and explains the truth. Will be extended as the book grows.
列出一些常见的迷思并揭示真相。将随着书籍内容的增加而扩展。

[APPENDIX C - Example Schema](todo)
[附录 C - 示例模式](todo)

All create and insert statements for the tables from the book.
书中表格的所有创建和插入语句。

Tip
提示

Slides of my talk “[Indexes: The neglected performance all-rounder](todo)”
我的演讲幻灯片「[索引：被忽视的性能全能选手](todo)」

[Next page](todo)
[下一页](todo)