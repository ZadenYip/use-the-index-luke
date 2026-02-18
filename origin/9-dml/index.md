# Insert, Delete and Update

## Insert, Delete and Update

> Source: https://use-the-index-luke.com/sql/dml

So far we have only discussed query performance, but SQL is not only about queries. It supports data manipulation as well. The respective commands—`insert`, `delete`, and `update`—form the so-called “data manipulation language” (DML)—a section of the SQL standard. The performance of these commands is for the most part negatively influenced by indexes.

An index is pure redundancy. It contains only data that is also stored in the table. During write operations, the database must keep those redundancies consistent. Specifically, it means that `insert`, `delete` and `update` not only affect the table but also the indexes that hold a copy of the affected data.

## Insert

> Source: https://use-the-index-luke.com/sql/dml/insert

The number of indexes on a table is the most dominant factor for `insert` performance. The more indexes a table has, the slower the execution becomes. The `insert` statement is the only operation that cannot directly benefit from indexing because it has no `where` clause.

Adding a new row to a table involves several steps. First, the database must find a place to store the row. For a regular heap table—which has no particular row order—the database can take any table block that has enough free space. This is a very simple and quick process, mostly executed in main memory. All the database has to do afterwards is to add the new entry to the respective data block.

If there are indexes on the table, the database must make sure the new entry is also found via these indexes. For this reason it has to add the new entry to each and every index on that table. The number of indexes is therefore a multiplier for the cost of an `insert` statement.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-insert&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

Moreover, adding an entry to an index is much more expensive than inserting one into a heap structure because the database has to keep the index order and tree balance. That means the new entry cannot be written to any block—it belongs to a specific [leaf node](https://use-the-index-luke.com/sql/anatomy/the-leaf-nodes). Although the database uses the [index tree](https://use-the-index-luke.com/sql/anatomy/the-tree) itself to find the correct leaf node, it still has to read a few index blocks for the tree traversal.

Once the correct leaf node has been identified, the database confirms that there is enough free space left in this node. If not, the database splits the leaf node and distributes the entries between the old and a new node. This process also affects the reference in the corresponding branch node as that must be duplicated as well. Needless to say, the branch node can run out of space as well so it might have to be split too. In the worst case, the database has to split all nodes up to the root node. This is the only case in which the tree gains an additional layer and grows in depth.

The index maintenance is, after all, the most expensive part of the `insert` operation. That is also visible in [Figure 8.1*Insert Performance by Number of Indexes*](#fig_insert): the execution time is hardly visible if the table does not have any indexes. Nevertheless, adding a single index is enough to increase the execute time by a factor of a hundred. Each additional index slows the execution down further.

## Figure 8.1 Insert Performance by Number of Indexes

0.000.020.040.060.080.100123450.000.020.040.060.080.10Indexes0.0003sExecution time [sec]Execution time [sec]insert

#### Note

The first index makes the greatest difference.

To optimize `insert` performance, it is very important to keep the number of indexes small.

#### Tip

Use indexes deliberately and sparingly, and avoid redundant indexes whenever possible. This is also beneficial for `delete` and `update` statements.

Considering `insert` statements only, it would be best to avoid indexes entirely—this yields by far the best `insert` performance. However tables without indexes are rather unrealistic in real world applications. You usually want to retrieve the stored data again so that you need indexes to improve query speed. Even write-only log tables often have a primary key and a respective index.

Nevertheless, the performance without indexes is so good that it can make sense to temporarily drop all indexes while loading large amounts of data—provided the indexes are not needed by any other SQL statements in the meantime. This can unleash a dramatic speed-up which is visible in the chart and is, in fact, a common practice in data warehouses.

#### Think About It

How would [Figure 8.1](#fig_insert) change when using an [index organized table](https://use-the-index-luke.com/sql/clustering/index-organized-clustered-index)  or [clustered index](https://use-the-index-luke.com/sql/clustering/index-organized-clustered-index)?

Is there any indirect way an `insert` statement could possibly benefit from indexing? That is, could an additional index make an `insert` statement faster?

## Delete

> Source: https://use-the-index-luke.com/sql/dml/delete

Unlike the `insert` statement, the `delete` statement has a `where` clause that can use all the methods described in [Chapter 2, “*The Where Clause*”](https://use-the-index-luke.com/sql/where-clause), to benefit directly from indexes. In fact, the `delete` statement works like a `select` that is followed by an extra step to delete the identified rows.

The actual deletion of a row is a similar process to inserting a new one—especially the removal of the references from the indexes and the activities to keep the index trees in balance. The performance chart shown in [Figure 8.2](#fig_delete) is therefore very similar to the one [shown for `insert`](https://use-the-index-luke.com/sql/dml/insert).

## Figure 8.2 Delete Performance by Number of Indexes

0.000.020.040.060.080.100.12123450.000.020.040.060.080.100.12IndexesExecution time [sec]Execution time [sec]insert

In theory, we would expect the best `delete` performance for a table without any indexes—as it is for `insert`. If there is no index, however, the database must read the full table to find the rows to be deleted. That means deleting the row would be fast but finding would be very slow. This case is therefore not shown in [Figure 8.2](#fig_delete).

Nevertheless it can make sense to execute a `delete` statement without an index just as it can make sense to execute a `select` statement without an index if it returns a large part of the table.

#### Tip

Even `delete` and `update` statements have an [execution plan](https://use-the-index-luke.com/sql/explain-plan).

A `delete` statement without `where` clause is an obvious example in which the database cannot use an index, although this is a special case that has its own SQL command: `truncate table`. This command has the same effect as `delete` without `where` except that it deletes all rows in one shot. It is very fast but has two important side effects: (1) it does an implicit `commit` (exceptions: [PostgreSQL](https://www.postgresql.org/docs/current/sql-truncate.html) and SQL Server); (2) it does not execute any triggers.

## Side Effects of MVCC

Multiversion concurrency control (MVCC) is a database mechanism that enables non-blocking concurrent data access and a consistent transaction view. The implementations, however, differ from database to database and might even have considerable effects on performance.

The PostgreSQL database, for example, only keeps the version information (=visibility information) on the table level: deleting a row just sets the “deleted” flag in the table block. PostgreSQL’s delete performance therefore *does not* depend on the number of indexes on the table. The physical deletion of the table row and the related index maintenance is carried out only during the [VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html) process.

## Update

> Source: https://use-the-index-luke.com/sql/dml/update

An `update` statement must relocate the changed index entries to maintain the index order. For that, the database must remove the old entry and add the new one at the new location. The response time is basically the same as for the respective `delete` and `insert` statements together.

The `update` performance, just like `insert` and `delete`, also depends on the number of indexes on the table. The only difference is that `update` statements do not necessarily affect all columns because they often modify only a few selected columns. Consequently, an `update` statement does not necessarily affect all indexes on the table but only those that contain updated columns.

[Figure 8.3](#fig-update) shows the response time for two `update` statements: one that sets all columns and affects all indexes and then a second one that updates a single column so it affects only one index.

## Figure 8.3 Update Performance by Indexes and Column Count

0.000.050.100.150.20123450.000.050.100.150.20Index CountExecution time [sec]Execution time [sec] all columns all columns one column one column

The `update` on all columns shows the same pattern we have already observed in the [previous sections](https://use-the-index-luke.com/sql/dml/insert): the response time grows with each additional index. The response time of the `update` statement that affects only one index does not increase so much because it leaves most indexes unchanged.

To optimize `update` performance, you must take care to only update those columns that were changed. This is obvious if you write the `update` statement manually. ORM tools, however, might generate `update` statements that set all columns every time. Hibernate, for example, does this when disabling the [*dynamic-update mode*](https://docs.hibernate.org/orm/6.2/userguide/html_single/#pc-managed-state-dynamic-update). Since version 4.0, this mode is enabled by default.

When using ORM tools, it is a good practice to occasionally enable query logging in a development environment to verify the generated SQL statements. The tip entitled [“*Enabling SQL Logging*”](https://use-the-index-luke.com/sql/join/nested-loops-join-n1-problem#tip-orm-log-sql) has a short overview of how to enable SQL logging in some widely used ORM tools.

#### Think About It

Can you think of a case where `insert` or `delete` statements do not affect all indexes of a table?
