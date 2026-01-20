# SQL 索引的剖析（Anatomy of an SQL Index）

“An index makes the query fast” is the most basic explanation of an index I have ever seen. Although it describes the most important aspect of an index very well, it is—unfortunately—not sufficient for this book. This chapter describes the index structure in a less superficial way but doesn’t dive too deeply into details. It provides just enough insight for one to understand the SQL performance aspects discussed throughout the book.

「索引让查询变快」是我见过对索引的解释当中最简单的一个。虽然这个解释很好地描述了索引最重要的点，但不幸的是，这一解释对于本书来说这还不够。本章将以一种不那么流于表面的方式去描述索引结构，但也不会深究太多的细节，但提供的见解足以让人理解本书中讨论的 SQL 性能问题。

An index is a distinct structure in the database that is built using the create index statement. It requires its own disk space and holds a copy of the indexed table data. That means that an index is pure redundancy. Creating an index does not change the table data; it just creates a new data structure that refers to the table. A database index is, after all, very much like the index at the end of a book: it occupies its own space, it is highly redundant, and it refers to the actual information stored in a different place.

索引是数据库中一个独特的结构，是使用 `create index` 语句去创建的。索引需要占用额外的磁盘空间，并持有被索引表数据的副本。这意味着索引是纯粹冗余（pure redundancy）。创建索引不会改变表数据；它只是创建了一个指向该表的新数据结构。归根结底，数据库索引非常像书末尾的索引（译：有的书末尾会有额外的索引页）：它占有自己的空间，是高度冗余的，并且指向存储在别处的实际信息。

---
**聚簇索引 (SQL Server, MySQL/InnoDB)**（**Clustered Indexes (SQL Server, MySQL/InnoDB)**

SQL Server and MySQL (using InnoDB) take a broader view of what “index” means. They refer to tables that consist of the index structure only as clustered indexes. These tables are called Index-Organized Tables (IOT) in the Oracle database.

SQL Server 和 MySQL（使用 InnoDB）对「索引」的定义有更广泛的看法。它们将只由索引结构组成的表称为**聚簇索引**（Clustered Indexes）。这些表在 Oracle 数据库中被称为**索引组织表**（Index-Organized Tables, IOT）。

Chapter 5, “Clustering Data: The Second Power of Indexing”, describes them in more detail and explains their advantages and disadvantages.

第五章「[聚簇数据：索引的第二种力量](https://use-the-index-luke.com/sql/clustering)」将更详细地描述它们，并解释它们的优缺点。

---

Searching in a database index is like searching in a printed telephone directory. The key concept is that all entries are arranged in a well-defined order. Finding data in an ordered data set is fast and easy because the sort order determines each entry’s position.

在数据库索引中搜索就像在纸质的电话簿中搜索一样。关键点是所有条目都按照着一个明确定义的顺序排列的。在有序数据集中去查找数据既快又简单，因为排序顺序确定好了每个条目的位置。

A database index is, however, more complex than a printed directory because it undergoes constant change. Updating a printed directory for every change is impossible for the simple reason that there is no space between existing entries to add new ones. A printed directory bypasses this problem by only handling the accumulated updates with the next printing. An SQL database cannot wait that long. It must process insert, delete and update statements immediately, keeping the index order without moving large amounts of data.

然而，数据库索引比纸质电话簿更复杂，因为数据库索引会不断地变化。对于每次信息变更都去更新电话簿是不可能的，原因很简单：现有条目之间没有空间插入一条新条目。纸质电话簿只能通过在下次印刷时集中处理这些条目的变更来绕过这个问题。但 SQL 数据库不能等那么久，数据库必须立即处理 insert、delete 和 update 语句，要在不移动大量数据的情况下保持索引顺序。

The database combines two data structures to meet the challenge: a doubly linked list and a search tree. These two structures explain most of the database’s performance characteristics.

数据库结合了两种数据结构来应对这一挑战：双向链表和搜索树。这两种结构决定了数据库的大部分的性能特征。

**Contents**
**目录**
---
1. The Leaf Nodes — A doubly linked list
2. The B-Tree — It’s a balanced tree
3. Slow Indexes, Part I — Two ingredients make the index slow

1. 叶节点 —— 双向链表
2. B-树 —— 一个平衡树 
3. 慢索引，第一部分 —— 导致索引缓慢的两个因素

## The Index Leaf Nodes（索引叶子节点） // TODO

The primary purpose of an index is to provide an ordered representation of the indexed data. It is, however, not possible to store the data sequentially because an insert statement would need to move the following entries to make room for the new one. Moving large amounts of data is very time-consuming so the insert statement would be very slow. The solution to the problem is to establish a logical order that is independent of physical order in memory.
索引的主要目的是提供被索引数据的**有序**表示。然而，按顺序存储数据是不可能的，因为 insert 语句需要移动随后的条目以为新条目腾出空间。移动大量数据非常耗时，因此 insert 语句会非常慢。这个问题的解决方案是建立一个独立于内存中物理顺序的**逻辑顺序**。

If you like this page, you might also like …
… to subscribe my mailing lists, get free stickers, buy my book or join a training.
如果您喜欢这个页面，您可能也会喜欢……
……订阅我的邮件列表，获取免费贴纸，购买我的书或参加培训。

The logical order is established via a doubly linked list. Every node has links to two neighboring entries, very much like a chain. New nodes are inserted between two existing nodes by updating their links to refer to the new node. The physical location of the new node doesn’t matter because the doubly linked list maintains the logical order.
逻辑顺序是通过**双向链表**建立的。每个节点都有指向两个相邻条目的链接，非常像一条链子。通过更新链接以指向新节点，新节点被插入到两个现有节点之间。新节点的物理位置并不重要，因为双向链表维护了逻辑顺序。

The data structure is called a doubly linked list because each node refers to the preceding and the following node. It enables the database to read the index forwards or backwards as needed. It is thus possible to insert new entries without moving large amounts of data—it just needs to change some pointers.
这种数据结构被称为双向链表，是因为每个节点都引用了**前一个**和**后一个**节点。这使得数据库能够根据需要向前或向后读取索引。因此，插入新条目无需移动大量数据——它只需要改变一些指针。

Doubly linked lists are also used for collections (containers) in many programming languages.
双向链表也用于许多编程语言的集合（容器）中。

| Programming Language | Name                                      |
|----------------------|-------------------------------------------|
| Java                 | `java.util.LinkedList`                    |
| .NET Framework       | `System.Collections.Generic.LinkedList`  |
| C++                  | `std::list`                               |

| 编程语言 | 名称                                      |
|----------------------|-------------------------------------------|
| Java                 | `java.util.LinkedList`                    |
| .NET Framework       | `System.Collections.Generic.LinkedList`  |
| C++                  | `std::list`                               |

Databases use doubly linked lists to connect the so-called index leaf nodes. Each leaf node is stored in a database block or page; that is, the database’s smallest storage unit. All index blocks are of the same size—typically a few kilobytes. The database uses the space in each block to the extent possible and stores as many index entries as possible in each block. That means that the index order is maintained on two different levels: the index entries within each leaf node, and the leaf nodes among each other using a doubly linked list.
数据库使用双向链表来连接所谓的**索引叶子节点**。每个叶子节点存储在一个数据库块或页中；那是数据库最小的存储单元。所有索引块的大小都是相同的——通常是几千字节。数据库尽可能利用每个块中的空间，并在每个块中存储尽可能多的索引条目。这意味着索引顺序在两个不同的层面上被维护：每个叶子节点内的索引条目，以及使用双向链表连接的叶子节点之间。

Figure 1.1 Index Leaf Nodes and Corresponding
图 1.1 索引叶子节点和对应的数据
![Figure 1.1 Index Leaf Nodes and Corresponding Table Data](images/figure-1.1.png)

Figure 1.1 illustrates the index leaf nodes and their connection to the table data. Each index entry consists of the indexed columns (the key, column 2) and refers to the corresponding table row (via ROWID or RID). Unlike the index, the table data is stored in a heap structure and is not sorted at all. There is neither a relationship between the rows stored in the same table block nor is there any connection between the blocks.
图 1.1 展示了索引叶子节点及其与表数据的连接。每个索引条目由被索引的列（键，第 2 列）组成，并引用相应的表行（通过 ROWID 或 RID）。与索引不同，表数据存储在**堆结构**中，根本没有排序。存储在同一个表块中的行之间既没有关系，块之间也没有任何连接。

## The Search Tree (B-Tree) Makes the Index Fast
## 搜索树 (B-树) 使索引变快

The index leaf nodes are stored in an arbitrary order—the position on the disk does not correspond to the logical position according to the index order. It is like a telephone directory with shuffled pages. If you search for “Smith” but first open the directory at “Robinson”, it is by no means granted that Smith follows Robinson. A database needs a second structure to find the entry among the shuffled pages quickly: a balanced search tree—in short: the B-tree.
索引叶子节点以任意顺序存储——磁盘上的位置不对应于索引顺序的逻辑位置。这就像一本页码被打乱的电话簿。如果你搜索「Smith」但首先翻开的是「Robinson」，绝不能保证 Smith 就在 Robinson 后面。数据库需要第二种结构来在打乱的页面中快速找到条目：**平衡搜索树**——简称：**B-树**。

**Figure 1.2 B-tree Structure**
**图 1.2 B-树结构**
![B-tree Structure](images/figure-1.2.png)

Figure 1.2 shows an example index with 30 entries. The doubly linked list establishes the logical order between the leaf nodes. The root and branch nodes support quick searching among the leaf nodes.
图 1.2 显示了一个包含 30 个条目的示例索引。双向链表建立了叶子节点之间的逻辑顺序。根节点和分支节点支持在叶子节点之间进行快速搜索。

The figure highlights a branch node and the leaf nodes it refers to. Each branch node entry corresponds to the biggest value in the respective leaf node. Take the first leaf node as an example: the biggest value in this node is 46, which is thus stored in the corresponding branch node entry. The same is true for the other leaf nodes so that in the end the branch node has the values 46, 53, 57 and 83. According to this scheme, a branch layer is built up until all the leaf nodes are covered by a branch node.
该图高亮显示了一个分支节点及其引用的叶子节点。每个分支节点条目对应于相应叶子节点中的**最大值**。以第一个叶子节点为例：该节点中的最大值是 46，因此存储在相应的分支节点条目中。其他叶子节点也是如此，所以最终该分支节点拥有值 46、53、57 和 83。按照这个方案，建立起一个分支层，直到所有叶子节点都被一个分支节点覆盖。

If you like this page, you might also like …
… to subscribe my mailing lists, get free stickers, buy my book or join a training.
如果您喜欢这个页面，您可能也会喜欢……
……订阅我的邮件列表，获取免费贴纸，购买我的书或参加培训。

The next layer is built similarly, but on top of the first branch node level. The procedure repeats until all keys fit into a single node, the root node. The structure is a balanced search tree because the tree depth is equal at every position; the distance between root node and leaf nodes is the same everywhere.
下一层以类似的方式构建，但是在第一个分支节点层之上。该过程重复进行，直到所有键都能放入单个节点，即**根节点**。该结构是一个平衡搜索树，因为树的深度在每个位置都是相等的；根节点和叶子节点之间的距离在任何地方都是一样的。

Note
注意

A B-tree is a balanced tree—not a binary tree.
B-树是**平衡**树（Balanced Tree）——不是**二叉**树（Binary Tree）。

Once created, the database maintains the index automatically. It applies every insert, delete and update to the index and keeps the tree in balance, thus causing maintenance overhead for write operations. [Chapter 8, “Modifying Data”](TODO), explains this in more detail.
一旦创建，数据库就会自动维护索引。它将每个 insert、delete 和 update 应用于索引并保持树的平衡，从而导致写操作的维护开销。[第八章「修改数据」](TODO)将更详细地解释这一点。

Figure 1.3 B-Tree Traversal
图 1.3 B-树遍历
![B-Tree Traversal](images/figure-1.3.png)

Figure 1.3 shows an index fragment to illustrate a search for the key “57”. The tree traversal starts at the root node on the left-hand side. Each entry is processed in ascending order until a value is greater than or equal to (>=) the search term (57). In the figure it is the entry 83. The database follows the reference to the corresponding branch node and repeats the procedure until the tree traversal reaches a leaf node.
图 1.3 展示了一个索引片段，用以说明对键「57」的搜索。树的遍历从左侧的根节点开始。每个条目按升序处理，直到有一个值大于或等于（>=）搜索词（57）。在图中，这个条目是 83。数据库跟随引用指向相应的分支节点，并重复该过程，直到树的遍历到达一个叶子节点。

Important
重要

The B-tree enables the database to find a leaf node quickly.
B-树使数据库能够快速找到一个**叶子节点**。

The tree traversal is a very efficient operation—so efficient that I refer to it as the first power of indexing. It works almost instantly—even on a huge data set. That is primarily because of the tree balance, which allows accessing all elements with the same number of steps, and secondly because of the logarithmic growth of the tree depth. That means that the tree depth grows very slowly compared to the number of leaf nodes. Real world indexes with millions of records have a tree depth of four or five. A tree depth of six is hardly ever seen. The box “Logarithmic Scalability” describes this in more detail.
树的遍历是一个非常高效的操作——如此高效，以至于我将其称为**索引的第一种力量**。它几乎瞬间完成——即使是在巨大的数据集上。这主要是因为树的平衡性允许用相同数量的步骤访问所有元素，其次是因为树深度的**对数**增长。这意味着与叶子节点的数量相比，树的深度增长非常缓慢。拥有数百万条记录的现实世界索引，其树深度通常为四或五。六层的树深度几乎从未见过。方框「对数扩展性」更详细地描述了这一点。

Logarithmic Scalability
对数扩展性

In mathematics, the logarithm of a number to a given base is the power or exponent to which the base must be raised in order to produce the number [Wikipedia](https://en.wikipedia.org/wiki/Logarithm).
在数学中，一个数对给定底数的对数是该底数必须被提升到的幂或指数，以产生该数 [Wikipedia](https://en.wikipedia.org/wiki/Logarithm)。

In a search tree the base corresponds to the number of entries per branch node and the exponent to the tree depth. The example index in Figure 1.2 holds up to four entries per node and has a tree depth of three. That means that the index can hold up to 64 (43) entries. If it grows by one level, it can already hold 256 entries (44). Each time a level is added, the maximum number of index entries quadruples. The logarithm reverses this function. The tree depth is therefore log4(number-of-index-entries).
在搜索树中，底数对应于每个分支节点的条目数，指数对应于树的深度。图 1.2 中的示例索引每个节点最多容纳四个条目，树深度为三。这意味着该索引最多可以容纳 64 (4^3) 个条目。如果它增加一层，它就已经可以容纳 256 个条目 (4^4)。每增加一层，最大索引条目数就会翻两番。对数反转了这个函数。因此树深度是 log4(索引条目数)。

| Tree Depth | Index Entries |
|------------|---------------|
| 3          | 64            |
| 4          | 256           |
| 5          | 1,024         |
| 6          | 4,096         |
| 7          | 16,384        |
| 8          | 65,536        |
| 9          | 262,144       |
| 10         | 1,048,576     |

| 树深度 | 索引条目 |
|------------|---------------|
| 3          | 64            |
| 4          | 256           |
| 5          | 1,024         |
| 6          | 4,096         |
| 7          | 16,384        |
| 8          | 65,536        |
| 9          | 262,144       |
| 10         | 1,048,576     |

The logarithmic growth enables the example index to search a million records with ten tree levels, but a real world index is even more efficient. The main factor that affects the tree depth, and therefore the lookup perfor­mance, is the number of entries in each tree node. This number corresponds to—mathematically speaking—the basis of the loga­rithm. The higher the basis, the shallower the tree, the faster the traversal.
对数增长使得示例索引能够用十个树层级搜索一百万条记录，但现实世界的索引甚至更高效。影响树深度以及查找性能的主要因素是每个树节点中的条目数。这个数字对应于——数学上讲——对数的**底数**。底数越大，树越浅，遍历越快。

Databases exploit this concept to a maximum extent and put as many entries as possible into each node—often hundreds. That means that every new index level supports a hundred times more entries.
数据库最大程度地利用了这一概念，在每个节点中放入尽可能多的条目——通常是数百个。这意味着每一个新的索引层级都支持多一百倍的条目。

Links
[B+tree simulator](TODO)
链接
[B+树模拟器](TODO)

## Slow Indexes, Part I
## 慢索引，第一部分

Despite the efficiency of the tree traversal, there are still cases where an index lookup doesn’t work as fast as expected. This contradiction has fueled the myth of the “degenerated index” for a long time. The myth proclaims an index rebuild as the miracle solution. Appendix B, “Myth Directory” covers this and other myths in detail. For now, you can take it for granted that rebuilding an index does not improve performance on the long run. The real reason trivial statements can be slow—even when using an index—can be explained on the basis of the previous sections.
尽管树的遍历效率很高，但仍有一些情况索引查找不像预期的那样快。这种矛盾长期以来助长了「退化索引」的迷思。该迷思宣称重建索引是奇迹般的解决方案。附录 B「[迷思目录](TODO)」详细涵盖了这一点和其他迷思。现在，你可以认为重建索引在长期内不会提高性能是理所当然的。即使在使用索引时，简单的语句也可能很慢，其实际原因可以根据前面的章节来解释。

The first ingredient for a slow index lookup is the leaf node chain. Consider the search for “57” in Figure 1.3 again. There are obviously two matching entries in the index. At least two entries are the same, to be more precise: the next leaf node could have further entries for “57”. The database must read the next leaf node to see if there are any more matching entries. That means that an index lookup not only needs to perform the tree traversal, it also needs to follow the leaf node chain.
慢索引查找的第一个因素是**叶子节点链**。再次考虑图 1.3 中对「57」的搜索。索引中显然有两个匹配的条目。至少有两个条目是相同的，更准确地说：下一个叶子节点可能还有更多「57」的条目。数据库必须读取下一个叶子节点以查看是否有更多匹配的条目。这意味着索引查找不仅需要执行树的遍历，还需要跟随叶子节点链。

If you like this page, you might also like …
… to subscribe my mailing lists, get free stickers, buy my book or join a training.
如果您喜欢这个页面，您可能也会喜欢……
……订阅我的邮件列表，获取免费贴纸，购买我的书或参加培训。

The second ingredient for a slow index lookup is accessing the table. Even a single leaf node might contain many hits—often hundreds. The corresponding table data is usually scattered across many table blocks (see [Figure 1.1, “Index Leaf Nodes and Corresponding Table Data”](TODO)). That means that there is an additional table access for each hit.
慢索引查找的第二个因素是**访问表**。即使是单个叶子节点也可能包含许多命中——通常是数百个。相应的表数据通常分散在许多表块中（参见 [图 1.1，「索引叶子节点和对应的数据」](TODO)）。这意味着对于每个命中都有一次额外的表访问。

An index lookup requires three steps: (1) the tree traversal; (2) following the leaf node chain; (3) fetching the table data. The tree traversal is the only step that has an upper bound for the number of accessed blocks—the index depth. The other two steps might need to access many blocks—they cause a slow index lookup.
一个索引查找需要三个步骤：(1) 树的遍历；(2) 跟随叶子节点链；(3) 获取表数据。树的遍历是唯一一个对访问块的数量有上限的步骤——即索引深度。其他两个步骤可能需要访问许多块——它们会导致索引查找变慢。

The origin of the “slow indexes” myth is the misbelief that an index lookup just traverses the tree, hence the idea that a slow index must be caused by a “broken” or “unbalanced” tree. The truth is that you can actually ask most databases how they use an index. The Oracle database is rather verbose in this respect and has three distinct operations that describe a basic index lookup:
「慢索引」迷思的根源在于一种误解，即认为索引查找只是遍历树，因此认为慢索引一定是由「损坏」或「不平衡」的树引起的。事实是，实际上你可以询问大多数数据库它们是如何使用索引的。Oracle 数据库在这方面相当详细，并有三个不同的操作来描述基本的索引查找：

INDEX UNIQUE SCAN
INDEX UNIQUE SCAN

The INDEX UNIQUE SCAN performs the tree traversal only. The Oracle database uses this operation if a unique constraint ensures that the search criteria will match no more than one entry.
`INDEX UNIQUE SCAN` 仅执行树的遍历。如果唯一约束确保搜索条件匹配不超过一个条目，Oracle 数据库将使用此操作。

INDEX RANGE SCAN
INDEX RANGE SCAN

The INDEX RANGE SCAN performs the tree traversal and follows the leaf node chain to find all matching entries. This is the fall­back operation if multiple entries could possibly match the search criteria.
`INDEX RANGE SCAN` 执行树的遍历并跟随叶子节点链以找到所有匹配的条目。如果多个条目可能匹配搜索条件，这是**后备**操作。

TABLE ACCESS BY INDEX ROWID
TABLE ACCESS BY INDEX ROWID

The TABLE ACCESS BY INDEX ROWID operation retrieves the row from the table. This operation is (often) performed for every matched record from a preceding index scan operation.
`TABLE ACCESS BY INDEX ROWID` 操作从表中检索行。通常对前一个索引扫描操作匹配的每条记录执行此操作。

The important point is that an INDEX RANGE SCAN can potentially read a large part of an index. If there is one more table access for each row, the query can become slow even when using an index.
重要的一点是，`INDEX RANGE SCAN` 可能会读取索引的很大一部分。如果每一行还需要一次表访问，那么即使使用了索引，查询也可能变得很慢。