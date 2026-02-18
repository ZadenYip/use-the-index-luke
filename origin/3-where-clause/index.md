# The Where Clause

## The Where Clause

> Source: https://use-the-index-luke.com/sql/where-clause

The [previous chapter](https://use-the-index-luke.com/sql/anatomy) described the structure of indexes and explained the cause of poor index performance. In the next step we learn how to spot and avoid these problems in SQL statements. We start by looking at the `where` clause.

The `where` clause defines the search condition of an SQL statement, and it thus falls into the core functional domain of an index: finding data quickly. Although the `where` clause has a huge impact on performance, it is often phrased carelessly so that the database has to scan a large part of the index. The result: a poorly written `where` clause is the first ingredient of a slow query.

This chapter explains how different operators affect index usage and how to make sure that an index is usable for as many queries as possible. The last section shows common anti-patterns and presents alternatives that deliver better performance.

## The Equals Operator

> Source: https://use-the-index-luke.com/sql/where-clause/the-equals-operator

The equality operator is both the most trivial and the most frequently used SQL operator. Indexing mistakes that affect performance are still very common and `where` clauses that combine multiple conditions are particularly vulnerable.

This section shows how to verify index usage and explains how concatenated indexes can optimize combined conditions. To aid understanding, we will analyze a slow query to see the real world impact of the causes explained in [Chapter 1](https://use-the-index-luke.com/sql/anatomy).

## Primary Keys

> Source: https://use-the-index-luke.com/sql/where-clause/the-equals-operator/primary-keys

We start with the simplest yet most common `where` clause: the primary key lookup. For the examples throughout this chapter we use the `EMPLOYEES` table defined as follows:

```
CREATE TABLE employees (
   employee_id   NUMBER        NOT NULL,
   first_name    VARCHAR(1000) NOT NULL,
   last_name     VARCHAR(1000) NOT NULL,
   date_of_birth DATE          NOT NULL,
   phone_number  VARCHAR(1000) NOT NULL,
   CONSTRAINT employees_pk PRIMARY KEY (employee_id)
)
```

The database automatically creates an index for the primary key. That means there is an index on the `EMPLOYEE_ID` column, even though there is no `create index` statement.

#### Tip

[Appendix C*Example Schema*](https://use-the-index-luke.com/sql/example-schema) contains scripts to populate the `EMPLOYEES` table with sample data. You can use it to test the examples in your own environment.

To follow the text, it is enough to know that the table contains 1000 rows.

The following query uses the primary key to retrieve an employee’s name:

```
SELECT first_name, last_name
  FROM employees
 WHERE employee_id = 123
```

The `where` clause cannot match multiple rows because the primary key constraint ensures uniqueness of the `EMPLOYEE_ID` values. The database does not need to follow the index leaf nodes—it is enough to traverse the index tree. We can use the so-called *execution plan* for verification:

Db2 (LUW)
:   The following execution plan was gathered with the [`last_explained` view](https://use-the-index-luke.com/sql/explain-plan/db2/getting-an-execution-plan#apa-db2-last_explained) available from the [appendix](https://use-the-index-luke.com/sql/explain-plan/db2/getting-an-execution-plan#apa-db2-last_explained).

    ```
    Explain Plan
    -------------------------------------------------------
    ID | Operation             |                Rows | Cost
     1 | RETURN                |                     |   13
     2 |  FETCH EMPLOYEES      |    1 of 1 (100.00%) |   13
     3 |   IXSCAN EMPLOYEES_PK | 1 of 1000 (   .10%) |    6

    Predicate Information
     3 - START (Q1.EMPLOYEE_ID = +00123.)
          STOP (Q1.EMPLOYEE_ID = +00123.)
    ```

    The Operation `IXSCAN` is similar to Oracle’s `INDEX [RANGE|UNIQUE] SCAN`. From this output, we cannot decided if it is a unique or range scan. The `FETCH` operation corresponds to Oracle’s `TABLE ACCESS BY INDEX ROWID`.

MySQL
:   ```
    +----+-----------+-------+---------+---------+------+-------+
    | id | table     | type  | key     | key_len | rows | Extra |
    +----+-----------+-------+---------+---------+------+-------+
    |  1 | employees | const | PRIMARY | 5       |    1 |       |
    +----+-----------+-------+---------+---------+------+-------+
    ```

    Type `const` is MySQL’s equivalent of Oracle’s `INDEX UNIQUE SCAN`.

Oracle
:   ```
    ---------------------------------------------------------------
    |Id |Operation                   | Name         | Rows | Cost |
    ---------------------------------------------------------------
    | 0 |SELECT STATEMENT            |              |    1 |    2 |
    | 1 | TABLE ACCESS BY INDEX ROWID| EMPLOYEES    |    1 |    2 |
    |*2 |  INDEX UNIQUE SCAN         | EMPLOYEES_PK |    1 |    1 |
    ---------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------
       2 - access("EMPLOYEE_ID"=123)
    ```

PostgreSQL
:   ```
                    QUERY PLAN
    -------------------------------------------
     Index Scan using employees_pk on employees 
       (cost=0.00..8.27 rows=1 width=14)
       Index Cond: (employee_id = 123::numeric)
    ```

    The PostgreSQL operation `Index Scan` combines the `INDEX [UNIQUE/RANGE] SCAN` and `TABLE ACCES BY INDEX ROWID` operations from the Oracle Database. It is not visible from the execution plan if the index access might potentially return more than one row.

SQL Server
:   ```
    |--Nested Loops(Inner Join)
       |--Index Seek(OBJECT:employees_pk,
       |               SEEK:employees.employee_id=@1
       |            ORDERED FORWARD)
       |--RID Lookup(OBJECT:employees,
                       SEEK:Bmk1000=Bmk1000
                     LOOKUP ORDERED FORWARD)
    ```

    The SQL Server operation `INDEX SEEK` and `RID Lookup` correspond to Oracle’s `INDEX RANGE SCAN` and `TABLE ACCESS BY ROWID` respectively. Unlike the Oracle Database, SQL Server explicitly shows the `Nested Loops` join to combine the index and table data.

The Oracle execution plan shows an `INDEX UNIQUE SCAN`—the operation that only traverses the index tree. It fully utilizes the logarithmic scalability of the index to find the entry very quickly—almost independent of the table size.

#### Tip

The *execution plan* (sometimes *explain plan* or *query plan*) shows the steps the database takes to execute an SQL statement. [Appendix A](https://use-the-index-luke.com/sql/explain-plan) explains how to retrieve and read execution plans with other databases.

After accessing the index, the database must do one more step to fetch the queried data (`FIRST_NAME`, `LAST_NAME`) from the table storage: the `TABLE ACCESS BY INDEX ROWID` operation. This operation can become a performance bottleneck—as explained in [“*Slow Indexes, Part I*”](https://use-the-index-luke.com/sql/anatomy/slow-indexes)—but there is no such risk in connection with an `INDEX UNIQUE SCAN`. This operation cannot deliver more than one entry so it cannot trigger more than one table access. That means that the ingredients of a slow query are not present with an `INDEX UNIQUE SCAN`.

## Primary Keys without Unique Index

A primary key does not necessarily need a unique index—you can use a non-unique index as well. In that case the Oracle database does not use an `INDEX UNIQUE SCAN` but instead the `INDEX RANGE SCAN` operation. Nonetheless, the constraint still maintains the uniqueness of keys so that the index lookup delivers at most one entry.

One of the reasons for using non-unique indexes for a primary keys are *deferrable constraints*. As opposed to regular constraints, which are validated during statement execution, the database postpones the validation of deferrable constraints until the transaction is committed. Deferred constraints are required for inserting data into tables with circular dependencies.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-surrogate&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

## Concatenated Keys

> Source: https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys

Even though the database creates the index for the primary key automatically, there is still room for manual refinements if the key consists of multiple columns. In that case the database creates an index on all primary key columns—a so-called *concatenated* index (also known as *multi-column*, *composite* or *combined* index). Note that the column order of a concatenated index has great impact on its usability so it must be chosen carefully.

For the sake of demonstration, let’s assume there is a company merger. The employees of the other company are added to our `EMPLOYEES` table so it becomes ten times as large. There is only one problem: the `EMPLOYEE_ID` is not unique across both companies. We need to extend the primary key by an extra identifier—e.g., a subsidiary ID. Thus the new primary key has two columns: the `EMPLOYEE_ID` as before and the `SUBSIDIARY_ID` to reestablish uniqueness.

The index for the new primary key is therefore defined in the following way:

```
CREATE UNIQUE INDEX employees_pk
    ON employees (employee_id, subsidiary_id)
```

A query for a particular employee has to take the full primary key into account—that is, the `SUBSIDIARY_ID` column also has to be used:

```
SELECT first_name, last_name
  FROM employees
 WHERE employee_id   = 123
   AND subsidiary_id = 30
```

Whenever a query uses the complete primary key, the database can use an `INDEX UNIQUE SCAN`—no matter how many columns the index has. But what happens when using only one of the key columns, for example, when searching all employees of a subsidiary?

```
SELECT first_name, last_name
  FROM employees
 WHERE subsidiary_id = 20
```

The execution plan reveals that the database does not use the index. Instead it performs a `TABLE ACCESS FULL`. As a result the database reads the entire table and evaluates every row against the `where` clause. The execution time grows with the table size: if the table grows tenfold, the `TABLE ACCESS FULL` takes ten times as long. The danger of this operation is that it is often fast enough in a small development environment, but it causes serious performance problems in production.

## Full Table Scan

The operation `TABLE ACCESS FULL`, also known as *full table scan*, can be the most efficient operation in some cases anyway, in particular when retrieving a large part of the table.

This is partly due to the overhead for the index lookup itself, which does not happen for a `TABLE ACCESS FULL` operation. This is mostly because an index lookup reads one block after the other as the database does not know which block to read next until the current block has been processed. A `FULL TABLE SCAN` must get the entire table anyway so that the database can read larger chunks at a time (*multi block read*). Although the database reads more data, it might need to execute fewer read operations.

The database does not use the index because it cannot use single columns from a concatenated index arbitrarily. A closer look at the index structure makes this clear.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-concatenated&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

A concatenated index is just a B-tree index like any other that keeps the indexed data in a sorted list. The database considers each column according to its position in the index definition to sort the index entries. The first column is the primary sort criterion and the second column determines the order only if two entries have the same value in the first column and so on.

#### Important

A concatenated index is *one index across multiple columns*.

The ordering of a two-column index is therefore like the ordering of a telephone directory: it is first sorted by surname, then by first name. That means that a two-column index does not support searching on the second column alone; that would be like searching a telephone directory by first name.

## Figure 2.1 Concatenated Index

123123123202127ROWIDROWIDROWID123124125302011ROWIDROWIDROWID12312312512618271119121126131251911Index-TreeEMPLOYEE\_IDSUBSIDIARY\_IDEMPLOYEE\_IDSUBSIDIARY\_IDEMPLOYEE\_IDSUBSIDIARY\_ID

The index excerpt in [Figure 2.1](#fig-concat-key) shows that the entries for subsidiary 20 are not stored next to each other. It is also apparent that there are no entries with `SUBSIDIARY_ID = 20` in the tree, although they exist in the leaf nodes. The tree is therefore useless for this query.

#### Tip

Visualizing an index helps in understanding what queries the index supports. You can query the database to retrieve the entries in index order (SQL:2008 syntax, see [syntax of top-n queries](https://use-the-index-luke.com/sql/partial-results/top-n-queries#overview_top_n) for proprietary solutions using `LIMIT`, `TOP` or `ROWNUM`):

```
SELECT <INDEX COLUMN LIST> 
  FROM <TABLE>  
 ORDER BY <INDEX COLUMN LIST>
 FETCH FIRST 100 ROWS ONLY
```

If you put the index definition and table name into the query, you will get a sample from the index. Ask yourself if the requested rows are clustered in a central place. If not, the index tree cannot help find that place.

We could, of course, add another index on `SUBSIDIARY_ID` to improve query speed. There is however a better solution—at least if we assume that searching on `EMPLOYEE_ID` alone does not make sense.

We can take advantage of the fact that the first index column is always usable for searching. Again, it is like a telephone directory: you don’t need to know the first name to search by last name. The trick is to reverse the index column order so that the `SUBSIDIARY_ID` is in the first position:

```
CREATE UNIQUE INDEX EMPLOYEES_PK 
    ON EMPLOYEES (SUBSIDIARY_ID, EMPLOYEE_ID)
```

Both columns together are still unique so queries with the full primary key can still use an `INDEX UNIQUE SCAN` but the sequence of index entries is entirely different. The `SUBSIDIARY_ID` has become the primary sort criterion. That means that all entries for a subsidiary are in the index consecutively so the database can use the B-tree to find their location.

#### Important

The most important consideration when defining a concatenated index is how to choose the column order so it can be used as often as possible.

The execution plan confirms that the database uses the “reversed” index. The `SUBSIDIARY_ID` alone is not unique anymore so the database must follow the leaf nodes in order to find all matching entries: it is therefore using the `INDEX RANGE SCAN` operation.

Db2 (LUW)
:   ```
    Explain Plan
    -------------------------------------------------------------
    ID | Operation               |                    Rows | Cost
     1 | RETURN                  |                         |  128
     2 |  FETCH EMPLOYEES        |  1195 of 1195 (100.00%) |  128
     3 |   RIDSCN                |  1195 of 1195 (100.00%) |   43
     4 |    SORT (UNIQUE)        |  1195 of 1195 (100.00%) |   43
     5 |     IXSCAN EMPLOYEES_PK | 1195 of 10000 ( 11.95%) |   43

    Predicate Information
     2 - SARG (Q1.SUBSIDIARY_ID = +00002.)
     5 - START (Q1.SUBSIDIARY_ID = +00002.)
          STOP (Q1.SUBSIDIARY_ID = +00002.)
    ```

    This execution plan looks more complex than the execution plan that was using the index before. They key operations are still there, however: the `IXSCAN` representing the index range scan and the `FETCH` for the table access. In between these operations there is an unexpected `SORT` and `RIDSCN` operation: the `SORT` operation sorts the entries fetched from the index according to the rows physical storage location in the heap table. The `RIDSCAN` then prefetches all the affected database pages (collapsing multiple adjacent blocks into a single IO operation).

MySQL
:   ```
    +----+-----------+------+---------+---------+------+-------+
    | id | table     | type | key     | key_len | rows | Extra |
    +----+-----------+------+---------+---------+------+-------+
    |  1 | employees | ref  | PRIMARY | 5       |  123 |       |
    +----+-----------+------+---------+---------+------+-------+
    ```

    The MySQL access type `ref` is the equivalent of `INDEX RANGE SCAN` in the Oracle database.

Oracle
:   ```
    ---------------------------------------------------------------
    |Id |Operation                   | Name         | Rows | Cost |
    ---------------------------------------------------------------
    | 0 |SELECT STATEMENT            |              |  106 |   75 |
    | 1 | TABLE ACCESS BY INDEX ROWID| EMPLOYEES    |  106 |   75 |
    |*2 |  INDEX RANGE SCAN          | EMPLOYEES_PK |  106 |    2 |
    ---------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------
       2 - access("SUBSIDIARY_ID"=20)
    ```

PostgreSQL
:   ```
                     QUERY PLAN
    ----------------------------------------------
     Bitmap Heap Scan on employees
     (cost=24.63..1529.17 rows=1080 width=13)
       Recheck Cond: (subsidiary_id = 2::numeric)
       -> Bitmap Index Scan on employees_pk
          (cost=0.00..24.36 rows=1080 width=0)
          Index Cond: (subsidiary_id = 2::numeric)
    ```

    The PostgreSQL database uses two operations in this case: a `Bitmap Index Scan` followed by a `Bitmap Heap Scan`. They roughly correspond to Oracle’s `INDEX RANGE SCAN` and `TABLE ACCESS BY INDEX ROWID` with one important difference: it first fetches all results from the index (`Bitmap Index Scan`), then sorts the rows according to the physical storage location of the rows in the heap table and than fetches all rows from the table (`Bitmap Heap Scan`). This method reduces the number of random access IOs on the table.

SQL Server
:   ```
    |--Nested Loops(Inner Join)
       |--Index Seek(OBJECT:employees_pk,
       |               SEEK:subsidiary_id=20
       |            ORDERED FORWARD)
       |--RID Lookup(OBJECT:employees,
                       SEEK:Bmk1000=Bmk1000
                     LOOKUP ORDERED FORWARD)
    ```

In general, a database can use a concatenated index when searching with the leading (leftmost) columns. An index with three columns can be used when searching for the first column, when searching with the first two columns together, and when searching using all columns.

Even though the two-index solution delivers very good `select` performance as well, the single-index solution is preferable. It not only saves storage space, but also the maintenance overhead for the second index. The fewer indexes a table has, the better the `insert`, `delete` and `update` performance.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-concatenated&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

To define an optimal index you must understand more than just how indexes work—you must also know how the application queries the data. This means you have to know the column combinations that appear in the `where` clause.

Defining an optimal index is therefore very difficult for external consultants because they don’t have an overview of the application’s access paths. Consultants can usually consider one query only. They do not exploit the extra benefit the index could bring for other queries. Database administrators are in a similar position as they might know the database schema but do not have deep insight into the access paths.

The only place where the technical database knowledge meets the functional knowledge of the business domain is the development department. Developers have a feeling for the data and know the access path. They can properly index to get the best benefit for the overall application without much effort.

## Slow Indexes, Part II

> Source: https://use-the-index-luke.com/sql/where-clause/the-equals-operator/slow-indexes-part-ii

The [previous section](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys) explained how to gain additional benefits from an existing index by changing its column order, but the example considered only two SQL statements. Changing an index, however, may affect all queries on the indexed table. This section explains the way databases pick an index and demonstrates the possible side effects when changing existing indexes.

The adopted `EMPLOYEES_PK` index improves the performance of all queries that search by subsidiary only. It is however usable for all queries that search by `SUBSIDIARY_ID`—regardless of whether there are any additional search criteria. That means the index becomes usable for queries that used to use another index with another part of the `where` clause. In that case, if there are multiple access paths available it is the optimizer’s job to choose the best one.

## The Query Optimizer

The query optimizer, or query planner, is the database component that transforms an SQL statement into an execution plan. This process is also called *compiling* or *parsing*. There are two distinct optimizer types.

*Cost-based optimizers* (CBO) generate many execution plan variations and calculate a *cost* value for each plan. The cost calculation is based on the operations in use and the estimated row numbers. In the end the cost value serves as the benchmark for picking the “best” execution plan.

*Rule-based optimizers* (RBO) generate the execution plan using a hard-coded rule set. Rule based optimizers are less flexible and are seldom used today.

Changing an index might have unpleasant side effects as well. In our example, it is the internal telephone directory application that has become very slow since the merger. The first analysis identified the following query as the cause for the slowdown:

```
SELECT first_name, last_name, subsidiary_id, phone_number
  FROM employees
 WHERE last_name  = 'WINAND'
   AND subsidiary_id = 30
```

The execution plan is:

## Example 2.1 Execution Plan with Revised Primary Key Index

```
---------------------------------------------------------------
|Id |Operation                   | Name         | Rows | Cost |
---------------------------------------------------------------
| 0 |SELECT STATEMENT            |              |    1 |   30 |
|*1 | TABLE ACCESS BY INDEX ROWID| EMPLOYEES    |    1 |   30 |
|*2 |  INDEX RANGE SCAN          | EMPLOYEES_PK |   40 |    2 |
---------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  1 - filter("LAST_NAME"='WINAND')
  2 - access("SUBSIDIARY_ID"=30)
```

The execution plan uses an index and has an overall cost value of 30. So far, so good. It is however suspicious that it uses the index we just changed—that is enough reason to suspect that our index change caused the performance problem, especially when bearing the old index definition in mind—it started with the `EMPLOYEE_ID` column which is not part of the `where` clause at all. The query could not use that index before.

For further analysis, it would be nice to compare the execution plan before and after the change. To get the original execution plan, we could just deploy the old index definition again, however most databases offer a simpler method to prevent using an index for a specific query. The following example uses an Oracle *optimizer hint* for that purpose.

```
SELECT /*+ NO_INDEX(EMPLOYEES EMPLOYEES_PK) */ 
       first_name, last_name, subsidiary_id, phone_number
  FROM employees
 WHERE last_name  = 'WINAND'
   AND subsidiary_id = 30
```

The execution plan that was presumably used before the index change did not use an index at all:

```
----------------------------------------------------
| Id | Operation         | Name      | Rows | Cost |
----------------------------------------------------
|  0 | SELECT STATEMENT  |           |    1 |  477 |
|* 1 |  TABLE ACCESS FULL| EMPLOYEES |    1 |  477 |
----------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter("LAST_NAME"='WINAND' AND "SUBSIDIARY_ID"=30)
```

Even though the `TABLE ACCESS FULL` must read and process the entire table, it seems to be faster than using the index in this case. That is particularly unusual because the query matches one row only. Using an index to find a single row should be much faster than a full table scan, but in this case it is not. The index seems to be slow.

In such cases it is best to go through each step of the troublesome execution plan. The first step is the `INDEX RANGE SCAN` on the `EMPLOYEES_PK` index. That index does not cover the `LAST_NAME` column—the `INDEX RANGE SCAN` can consider the `SUBSIDIARY_ID` filter only; the Oracle database shows this in the “Predicate Information” area—entry “2” of the execution plan. There you can see the conditions that are applied for each operation.

#### Tip

[Appendix A, “*Execution Plans*”](https://use-the-index-luke.com/sql/explain-plan), explains how to find the “Predicate Information” for other databases.

The `INDEX RANGE SCAN` with operation ID 2 ([Example 2.1](#ex-plan-revised-pk)) applies only the `SUBSIDIARY_ID=30` filter. That means that it traverses the index tree to find the first entry for `SUBSIDIARY_ID` 30. Next it follows the leaf node chain to find all other entries for that subsidiary. The result of the `INDEX RANGE SCAN` is a list of `ROWIDs` that fulfill the `SUBSIDIARY_ID` condition: depending on the subsidiary size, there might be just a few ones or there could be many hundreds.

The next step is the `TABLE ACCESS BY INDEX ROWID` operation. It uses the `ROWIDs` from the previous step to fetch the rows—all columns—from the table. Once the `LAST_NAME` column is available, the database can evaluate the remaining part of the `where` clause. That means the database has to fetch all rows for `SUBSIDIARY_ID=30` before it can apply the `LAST_NAME` filter.

The statement’s response time does not depend on the result set size but on the number of employees in the particular subsidiary. If the subsidiary has just a few members, the `INDEX RANGE SCAN` provides better performance. Nonetheless a `TABLE ACCESS FULL` can be faster for a huge subsidiary because it can read large parts from the table in one shot (see [“*Full Table Scan*”](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys#sb-full-table-scan)).

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-slow-index-ii&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

The query is slow because the index lookup returns many `ROWIDs`—one for each employee of the original company—and the database must fetch them individually. It is the perfect combination of the two ingredients that make an index slow: the database reads a wide index range and has to fetch many rows individually.

Choosing the best execution plan depends on the table’s data distribution as well so the optimizer uses statistics about the contents of the database. In our example, a histogram containing the distribution of employees over subsidiaries is used. This allows the optimizer to estimate the number of rows returned from the index lookup—the result is used for the cost calculation.

## Statistics

A cost-based optimizer uses statistics about tables, columns, and indexes. Most statistics are collected on the column level: the number of distinct values, the smallest and largest values (data range), the number of `NULL` occurrences and the column histogram (data distribution). The most important statistical value for a table is its size (in rows and blocks).

The most important index statistics are the tree depth, the number of leaf nodes, the number of distinct keys and the clustering factor (see [Chapter 5, “*Clustering Data: The Second Power of Indexing*”](https://use-the-index-luke.com/sql/clustering)).

The optimizer uses these values to estimate the selectivity of the `where` clause predicates.

If there are no statistics available—for example because they were deleted—the optimizer uses default values. The default statistics of the Oracle database suggest a small index with medium selectivity. They lead to the estimate that the `INDEX RANGE SCAN` will return 40 rows. The execution plan shows this estimation in the Rows column (again, see [Example 2.1](#ex-plan-revised-pk)). Obviously this is a gross underestimate, as there are 1000 employees working for this subsidiary.

If we provide correct statistics, the optimizer does a better job. The following execution plan shows the new estimation: 1000 rows for the `INDEX RANGE SCAN`. Consequently it calculated a higher cost value for the subsequent table access.

```
---------------------------------------------------------------
|Id |Operation                   | Name         | Rows | Cost |
---------------------------------------------------------------
| 0 |SELECT STATEMENT            |              |    1 |  680 |
|*1 | TABLE ACCESS BY INDEX ROWID| EMPLOYEES    |    1 |  680 |
|*2 |  INDEX RANGE SCAN          | EMPLOYEES_PK | 1000 |    4 |
---------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  1 - filter("LAST_NAME"='WINAND')
  2 - access("SUBSIDIARY_ID"=30)
```

The cost value of 680 is even higher than the cost value for the execution plan using the `FULL TABLE SCAN` (477). The optimizer will therefore automatically prefer the `FULL TABLE SCAN`.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-slow-index-ii&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

This example of a slow index should not hide the fact that proper indexing is the best solution. Of course searching on last name is best supported by an index on `LAST_NAME`:

```
CREATE INDEX emp_name ON employees (last_name)
```

Using the new index, the optimizer calculates a cost value of 3:

## Example 2.2 Execution Plan with Dedicated Index

```
--------------------------------------------------------------
| Id | Operation                   | Name      | Rows | Cost |
--------------------------------------------------------------
|  0 | SELECT STATEMENT            |           |    1 |    3 |
|* 1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES |    1 |    3 |
|* 2 |   INDEX RANGE SCAN          | EMP_NAME  |    1 |    1 |
--------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter("SUBSIDIARY_ID"=30)
   2 - access("LAST_NAME"='WINAND')
```

The index access delivers—according to the optimizer’s estimation—one row only. The database thus has to fetch only that row from the table: this is definitely faster than a `FULL TABLE SCAN`. A properly defined index is still better than the original full table scan.

The two execution plans from [Example 2.1](#ex-plan-revised-pk) and [Example 2.2](#fig-plan-dedicted-name-index) are almost identical. The database performs the same operations and the optimizer calculated similar cost values, nevertheless the second plan performs much better. The efficiency of an `INDEX RANGE SCAN` may vary over a wide range—especially when followed by a table access. Using an index does not automatically mean a statement is executed in the best way possible.

## Functions

> Source: https://use-the-index-luke.com/sql/where-clause/functions

The index on `LAST_NAME` has improved the performance considerably, but it requires you to search using the same case (upper/lower) as is stored in the database. This section explains how to lift this restriction without a decrease in performance.

Db2 (LUW)
:   Db2 supports function based indexes on [zOS](https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=statements-create-index) for a while, but only [since version 10.5 on LUW](https://www.ibm.com/docs/en/db2/11.5.x?topic=statements-create-index#sdx-synid_key-expression). The use of user-defined functions in indexes is not allowed.

    The backup solution is to create a real column in the table that holds the result of the function or expression. The column must be maintained by a trigger or by the application layer—whatever is more appropriate. The new column can be indexed. The `where` clause must use the new column (without the expression).

MySQL
:   MySQL is case-insensitive by default, but that can be [controlled on column level](https://dev.mysql.com/doc/refman/8.0/en/case-sensitivity.html). Starting with version 5.7 MySQL can create indexes on [generated columns](https://dev.mysql.com/doc/refman/8.0/en/generated-column-index-optimizations.html).

    The backup solution for older versions is to create a real column in the table that holds the result of the function or expression. The column must be maintained by a trigger or by the application layer—whatever is more appropriate. The new column can be indexed. The `where` clause must use the new column (without the expression).

Oracle
:   The Oracle database supports function-based indexes since release 8*i*. Virtual columns were additionally added with 11*g*.

PostgreSQL
:   PostgreSQL fully supports [Indexes on Expressions](https://www.postgresql.org/docs/current/indexes-expressional.html) since release 7.4 (partially supported since 7.2)

SQL Server
:   SQL Server supports [Computed Columns](https://learn.microsoft.com/en-us/sql/relational-databases/tables/specify-computed-columns-in-a-table?view=sql-server-ver16) that can be indexed since release 2000.

## Case-Insensitive Search

> Source: https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search

Ignoring the case in a `where` clause is very simple. You can, for example, convert both sides of the comparison to all caps notation:

```
SELECT first_name, last_name, phone_number
  FROM employees
 WHERE UPPER(last_name) = UPPER('winand')
```

Regardless of the capitalization used for the search term or the `LAST_NAME` column, the `UPPER` function makes them match as desired.

#### Note

Another way for case-insensitive matching is to use a different “collation”. The default collations used by SQL Server and MySQL do not distinguish between upper and lower case letters—they are case-insensitive by default.

The logic of this query is perfectly reasonable but the execution plan is not:

Db2 (LUW)
:   ```
    Explain Plan
    ------------------------------------------------------
    ID | Operation         |                   Rows | Cost
     1 | RETURN            |                        |  690
     2 |  TBSCAN EMPLOYEES | 400 of 10000 (  4.00%) |  690

    Predicate Information
     2 - SARG ( UPPER(Q1.LAST_NAME) = 'WINAND')
    ```

Oracle
:   ```
    ----------------------------------------------------
    | Id | Operation         | Name      | Rows | Cost |
    ----------------------------------------------------
    |  0 | SELECT STATEMENT  |           |   10 |  477 |
    |* 1 |  TABLE ACCESS FULL| EMPLOYEES |   10 |  477 |
    ----------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------
       1 - filter(UPPER("LAST_NAME")='WINAND')
    ```

PostgreSQL
:   ```
                         QUERY PLAN
    ------------------------------------------------------
     Seq Scan on employees
       (cost=0.00..1722.00 rows=50 width=17)
       Filter: (upper((last_name)::text) = 'WINAND'::text)
    ```

It is a return of our old friend the full table scan. Although there is an index on `LAST_NAME`, it is unusable—because the search is *not* on `LAST_NAME` but on `UPPER(LAST_NAME)`. From the database’s perspective, that’s something *entirely different*.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-insensitive&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

This is a trap we all might fall into. We recognize the relation between `LAST_NAME` and `UPPER(LAST_NAME)` instantly and expect the database to “see” it as well. In reality the optimizer’s view is more like this:

```
SELECT first_name, last_name, phone_number
  FROM employees
 WHERE BLACKBOX(...) = 'WINAND'
```

The `UPPER` function is just a black box. The parameters to the function are not relevant because there is no general relationship between the function’s parameters and the result.

#### Tip

Replace the function name with `BLACKBOX` to understand the optimizer’s point of view.

## Compile Time Evaluation

The optimizer can evaluate the expression on the right-hand side during “compile time” because it has all the input parameters. The Oracle execution plan (“Predicate Information” section) therefore only shows the upper case notation of the search term. This behavior is very similar to a compiler that evaluates constant expressions at compile time.

To support that query, we need an index that covers the actual search term. That means we do not need an index on `LAST_NAME` but on `UPPER(LAST_NAME)`:

```
CREATE INDEX emp_up_name 
    ON employees (UPPER(last_name))
```

An index whose definition contains functions or expressions is a so-called *function-based index (FBI)*. Instead of copying the column data directly into the index, a function-based index applies the function first and puts the result into the index. As a result, the index stores the names in all caps notation.

The database can use a function-based index if the *exact* expression of the index definition appears in an SQL statement—like in the example above. The execution plan confirms this:

Db2 (LUW)
:   ```
    Explain Plan
    -------------------------------------------------------
    ID | Operation            |                 Rows | Cost
     1 | RETURN               |                      |   13
     2 |  FETCH EMPLOYEES     |     1 of 1 (100.00%) |   13
     3 |   IXSCAN EMP_UP_NAME | 1 of 10000 (   .01%) |    6

    Predicate Information
     3 - START ( UPPER(Q1.LAST_NAME) = 'WINAND')
          STOP ( UPPER(Q1.LAST_NAME) = 'WINAND')
    ```

    The query was changed to `WHERE UPPER(last_name) = 'WINAND'` (no `UPPER` on the right hand side) to get the expected result. When using `UPPER('winand')`, the optimizer does a gross misestimation and expects 4% of the table rows to be selected. This causes the optimizer to ignore the index and do a `TBSCAN`. See [*Full Table Scan*](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys#sb-full-table-scan) to see why that might make sense.

Oracle
:   ```
    --------------------------------------------------------------
    |Id |Operation                   | Name        | Rows | Cost |
    --------------------------------------------------------------
    | 0 |SELECT STATEMENT            |             |  100 |   41 |
    | 1 | TABLE ACCESS BY INDEX ROWID| EMPLOYEES   |  100 |   41 |
    |*2 |  INDEX RANGE SCAN          | EMP_UP_NAME |   40 |    1 |
    --------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------
      2 - access(UPPER("LAST_NAME")='WINAND')
    ```

PostgreSQL
:   ```
                           QUERY PLAN
    ------------------------------------------------------------
    Bitmap Heap Scan on employees
      (cost=4.65..178.65 rows=50 width=17)
      Recheck Cond: (upper((last_name)::text) = 'WINAND'::text)
      -> Bitmap Index Scan on emp_up_name
         (cost=0.00..4.64 rows=50 width=0)
         Index Cond: (upper((last_name)::text) = 'WINAND'::text)
    ```

It is a regular `INDEX RANGE SCAN` as described in [Chapter 1](https://use-the-index-luke.com/sql/anatomy). The database traverses the B-tree and follows the leaf node chain. There are no dedicated operations or keywords for function-based indexes.

#### Warning

Sometimes ORM tools use `UPPER` and `LOWER` without the developer’s knowledge. Hibernate, for example, [injects an implicit `LOWER`](https://use-the-index-luke.com/sql/myth-directory/dynamic-sql-is-slow#myth-dynamic-sql-sample) for case-insensitive searches.

The execution plan is not yet the same as it was in the previous section without `UPPER`; the row count estimate is too high. It is particularly strange that the optimizer expects to fetch more rows from the table than the `INDEX RANGE SCAN` delivers in the first place. How can it fetch 100 rows from the table if the preceding index scan returned only 40 rows? The answer is that it can not. Contradicting estimates like this often indicate problems with the statistics. In this particular case it is because the Oracle database does not update the table statistics when creating a new index (see also [“*Oracle Statistics for Function-Based Indexes*”](#sb-collecting-statistics)).

## Oracle Statistics for Function-Based Indexes

The Oracle database maintains the information about the number of distinct column values as part of the table statistics. These figures are reused if a column is part of multiple indexes.

Statistics for a function-based index (FBI) are also kept on table level as *virtual columns*. Although the Oracle database collects the *index statistics* for new indexes automatically ([since release 10*g*](https://docs.oracle.com/cd/B14117_01/server.101/b10763/compat.htm#sthref320)), it does not update the *table statistics*. For this reason, the Oracle documentation recommends updating the table statistics after creating a function-based index:

> After creating a function-based index, collect statistics on both the index and its base table using the `DBMS_STATS` package. Such statistics will enable Oracle Database to correctly decide when to use the index.
>
> — [Oracle Database SQL Language Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-INDEX.html#GUID-1F89BBC0-825F-4215-AF71-7588E31D8BFE__I2100962)

My personal recommendation goes even further: after every index change, update the statistics for the base table and all its indexes. That might, however, also lead to unwanted side effects. Coordinate this activity with the database administrators (DBAs) and make a backup of the original statistics.

After updating the statistics, the optimizer calculates more accurate estimates:

Oracle
:   ```
    --------------------------------------------------------------
    |Id |Operation                   | Name        | Rows | Cost |
    --------------------------------------------------------------
    | 0 |SELECT STATEMENT            |             |    1 |    3 |
    | 1 | TABLE ACCESS BY INDEX ROWID| EMPLOYEES   |    1 |    3 |
    |*2 |  INDEX RANGE SCAN          | EMP_UP_NAME |    1 |    1 |
    --------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------
      2 - access(UPPER("LAST_NAME")='WINAND')
    ```

PostgreSQL
:   ```
                          QUERY PLAN
    ----------------------------------------------------------
     Index Scan using emp_up_name on employees
       (cost=0.00..8.28 rows=1 width=17)
       Index Cond: (upper((last_name)::text) = 'WINAND'::text)
    ```

    As the row count estimate has decreased—from 50 in the example above down to 1 in this execution plan—the query planner prefers to use the simpler `Index Scan` operation.

#### Note

[The so-called “extended statistics” on expressions and column groups](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/managing-extended-statistics.html#GUID-BD0F0B71-DD8B-44A0-888E-495830FC09A4) were introduced with Oracle release 11*g*.

Although the updated statistics do not improve execution performance in this case—the index was properly used anyway—it is always a good idea to check the optimizer’s estimates. The number of rows processed for each operation (cardinality estimate) is a particularly important figure that is also shown in SQL Server and PostgreSQL execution plans.

#### Tip

[Appendix A, “*Execution Plans*”](https://use-the-index-luke.com/sql/explain-plan), describes the row count estimates in the execution plans of other databases.

SQL Server and MySQL do not support function-based indexes as described but both offer a workaround via computed or generated columns. To make use of this, you have to first add a generated column to the table that can be indexed afterwards:

MySQL
:   Since MySQL 5.7 you [can index a generated columns](https://dev.mysql.com/doc/refman/8.0/en/create-table.html#create-table-secondary-indexes-virtual-columns) as follows:

    ```
    ALTER TABLE employees
      ADD COLUMN last_name_up VARCHAR(255) AS (UPPER(last_name));
    ```

    ```
    CREATE INDEX emp_up_name ON employees (last_name_up);
    ```

SQL Server
:   ```
    ALTER TABLE employees ADD last_name_up AS UPPER(last_name)
    ```

    ```
    CREATE INDEX emp_up_name ON employees (last_name_up)
    ```

SQL Server and MySQL are able to use this index whenever the indexed expression appears in the statement. In some simple cases, SQL Server and [MySQL](https://dev.mysql.com/doc/refman/8.0/en/generated-column-index-optimizations.html) can use this index even if the query remains unchanged. Sometimes, however, the query must be changed to refer to the name of the new columns in order to use the index. Always check the execution plan in case of doubt.

## User-Defined Functions

> Source: https://use-the-index-luke.com/sql/where-clause/functions/user-defined-functions

Function-based indexing is a very generic approach. Besides functions like `UPPER` you can also index expressions like `A + B` and even use user-defined functions in the index definition.

There is one important exception. It is, for example, not possible to refer to the current time in an index definition, neither directly nor indirectly, as in the following example.

```
CREATE FUNCTION get_age(date_of_birth DATE) 
RETURN NUMBER
AS
BEGIN
  RETURN 
    TRUNC(MONTHS_BETWEEN(SYSDATE, date_of_birth)/12);
END
```

The function `GET_AGE` uses the current date (`SYSDATE`) to calculate the age based on the supplied date of birth. You can use this function in all parts of an SQL query, for example in `select` and the `where` clauses:

```
SELECT first_name, last_name, get_age(date_of_birth)
  FROM employees
 WHERE get_age(date_of_birth) = 42
```

The query lists all 42-year-old employees. Using a function-based index is an obvious idea for optimizing this query, but you cannot use the function `GET_AGE` in an index definition because it is not *deterministic*. That means the result of the function call is not fully determined by its parameters. Only functions that always return the same result for the same parameters—functions that are deterministic—can be indexed.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-user-defined&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

The reason behind this limitation is simple. When inserting a new row, the database calls the function and stores the result in the index and there it stays, unchanged. There is no periodic process that updates the index. The database updates the indexed age only when the date of birth is changed by an `update` statement. After the next birthday, the age that is stored in the index will be wrong.

Besides *being* deterministic, PostgreSQL and the Oracle database require functions to be *declared* to be deterministic when used in an index so you have to use the keyword `DETERMINISTIC` (Oracle) or `IMMUTABLE` (PostgreSQL).

#### Caution

PostgreSQL and the Oracle database trust the `DETERMINISTIC` or `IMMUTABLE` declarations—that means they trust the developer.

You can declare the `GET_AGE` function to be deterministic and use it in an index definition. Regardless of the declaration, it will *not* work as intended because the age stored in the index will not increase as the years pass; the employees will not get older—at least not in the index.

Other examples for functions that cannot be “indexed” are random number generators and functions that depend on environment variables.

#### Note

Db2 (LUW) cannot use user-defined functions in indexes (not even if they are deterministic).

#### Think About It

How can you still use an index to optimize a query for all 42-year-old employees?

## Over-Indexing

> Source: https://use-the-index-luke.com/sql/where-clause/functions/over-indexing

If the concept of function-based indexing is new to you, you might be tempted to just index everything, but this is in fact the very last thing you should do. The reason is that every index causes ongoing maintenance. Function-based indexes are particularly troublesome because they make it very easy to create *redundant indexes*.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-over-indexing&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

The [case-insensitive search from above](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) could be implemented with the `LOWER` function as well:

```
SELECT first_name, last_name, phone_number
  FROM employees
 WHERE LOWER(last_name) = LOWER('winand')
```

A single index cannot support both methods of ignoring the case. We could, of course, create a second index on `LOWER(last_name)` for this query, but that would mean the database has to maintain two indexes for each `insert`, `update`, and `delete` statement (see also [Chapter 8, “*Modifying Data*”](https://use-the-index-luke.com/sql/dml)). To make one index suffice, you should consistently use the same function throughout your application.

#### Tip

Unify the access path so that one index can be used by several queries.

#### Warning

Sometimes ORM tools use `UPPER` and `LOWER` without the developer’s knowledge. Hibernate, for example, [injects an implicit `LOWER`](https://use-the-index-luke.com/sql/myth-directory/dynamic-sql-is-slow#myth-dynamic-sql-sample) for case-insensitive searches.

#### Tip

Always aim to index the original data as that is often the most useful information you can put into an index.

## Bind Variables

> Source: https://use-the-index-luke.com/sql/where-clause/bind-parameters

This section covers a topic that is skipped in most SQL textbooks: *parameterized queries* and *bind parameters*.

Bind parameters—also called dynamic parameters or bind variables—are an alternative way to pass data to the database. Instead of putting the values directly into the SQL statement, you just use a placeholder like `?`, `:name` or `@name` and provide the actual values using a separate API call.

There is nothing bad about writing values directly into ad-hoc statements; there are, however, two good reasons to use bind parameters in programs:

Security
:   Bind variables are the best way to prevent [SQL injection](https://en.wikipedia.org/wiki/SQL_injection).

Performance
:   Databases with an execution plan cache like SQL Server and the Oracle database can reuse an execution plan when executing the same statement multiple times. It saves effort in rebuilding the execution plan but works only if the SQL statement is *exactly* the same. If you put different values into the SQL statement, the database handles it like a different statement and recreates the execution plan.

    When using bind parameters you do not write the actual values but instead insert placeholders into the SQL statement. That way the statements do not change when executing them with different values.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-bind&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

Naturally there are exceptions, for example if the affected data volume depends on the actual values:

```
SELECT first_name, last_name
  FROM employees
 WHERE subsidiary_id = 20
```

```
99 rows selected.

----------------------------------------------------------------
|Id | Operation                   | Name         | Rows | Cost |
----------------------------------------------------------------
| 0 | SELECT STATEMENT            |              |   99 |   70 |
| 1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES    |   99 |   70 |
|*2 |   INDEX RANGE SCAN          | EMPLOYEES_PK |   99 |    2 |
----------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("SUBSIDIARY_ID"=20)
```

An index lookup delivers the best performance for small subsidiaries, but a `TABLE ACCESS FULL` can outperform the index for large subsidiaries:

```
SELECT first_name, last_name
  FROM employees
 WHERE subsidiary_id = 30
```

```
1000 rows selected.

----------------------------------------------------
| Id | Operation         | Name      | Rows | Cost |
----------------------------------------------------
|  0 | SELECT STATEMENT  |           | 1000 |  478 |
|* 1 |  TABLE ACCESS FULL| EMPLOYEES | 1000 |  478 |
----------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("SUBSIDIARY_ID"=30)
```

In this case, the histogram on `SUBSIDIARY_ID` fulfills its purpose. The optimizer uses it to determine the frequency of the subsidiary ID mentioned in the SQL query. Consequently it gets two different row count estimates for both queries.

The subsequent cost calculation will therefore result in two different cost values. When the optimizer finally selects an execution plan it takes the plan with the lowest cost value. For the smaller subsidiary, it is the one using the index.

The cost of the `TABLE ACCESS BY INDEX ROWID` operation is highly sensitive to the row count estimate. Selecting ten times as many rows will elevate the cost value by that factor. The overall cost using the index is then even higher than a full table scan. The optimizer will therefore select the other execution plan for the bigger subsidiary.

When using bind parameters, the optimizer has no concrete values available to determine their frequency. It then just assumes an equal distribution and always gets the same row count estimates and cost values. In the end, it will always select the same execution plan.

#### Tip

Column histograms are most useful if the values are not uniformly distributed.

For columns with uniform distribution, it is often sufficient to divide the number of distinct values by the number of rows in the table. This method also works when using bind parameters.

If we compare the optimizer to a compiler, bind variables are like program variables, but if you write the values directly into the statement they are more like constants. The database can use the values from the SQL statement during optimization just like a compiler can evaluate constant expressions during compilation. Bind parameters are, put simply, not visible to the optimizer just as the runtime values of variables are not known to the compiler.

From this perspective, it is a little bit paradoxical that bind parameters can improve performance if not using bind parameters enables the optimizer to always opt for the best execution plan. But the question is at what price? Generating and evaluating all execution plan variants is a huge effort that does not pay off if you get the same result in the end anyway.

#### Tip

Not using bind parameters is like recompiling a program every time.

Deciding to build a specialized or generic execution plan presents a dilemma for the database. Either effort is taken to evaluate all possible plan variants for each execution in order to always get the best execution plan or the optimization overhead is saved and a cached execution plan is used whenever possible—accepting the risk of using a suboptimal execution plan. The quandary is that the database does not know if the full optimization cycle delivers a different execution plan without actually doing the full optimization. Database vendors try to solve this dilemma with heuristic methods—but with very limited success.

As the developer, you can use bind parameters deliberately to help resolve this dilemma. That is, you should always use bind parameters except for values that *shall* influence the execution plan.

Unevenly distributed status codes like “todo” and “done” are a good example. The number of “done” entries often exceeds the “todo” records by an order of magnitude. Using an index only makes sense when searching for “todo” entries in that case. Partitioning is another example—that is, if you split tables and indexes across several storage areas. The actual values can then influence which partitions have to be scanned. The performance of `LIKE` queries can suffer from bind parameters as well as we will see in the [next section](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning).

#### Tip

In all reality, there are only a few cases in which the actual values affect the execution plan. You should therefore use bind parameters if in doubt—just to prevent SQL injections.

The following code snippets show how to use bind parameters in various programming languages.

C#
:   Without bind parameters:

    ```
    int subsidiary_id;
    SqlCommand cmd = new SqlCommand(
                       "select first_name, last_name" 
                     + "  from employees"
                     + " where subsidiary_id = " + subsidiary_id
                     , connection);
    ```

    Using a bind parameter:

    ```
    int subsidiary_id;
    SqlCommand cmd =
           new SqlCommand(
                          "select first_name, last_name" 
                        + "  from employees"
                        + " where subsidiary_id = @subsidiary_id
                        , connection);
    cmd.Parameters.AddWithValue("@subsidiary_id", subsidiary_id);
    ```

    See also: [`SqlParameterCollection`](https://learn.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlparametercollection?view=netframework-4.8.1&viewFallbackFrom=dotnet-plat-ext-5.0) class documentation.

Java
:   Without bind parameters:

    ```
    int subsidiary_id;
    Statement command = connection.createStatement(
                        "select first_name, last_name" 
                      + "  from employees"
                      + " where subsidiary_id = " + subsidiary_id
                      );
    ```

    Using a bind parameter:

    ```
    int subsidiary_id;
    PreparedStatement command = connection.prepareStatement(
                        "select first_name, last_name" 
                      + "  from employees"
                      + " where subsidiary_id = ?"
                      );
    command.setInt(1, subsidiary_id);
    ```

    See also: [`PreparedStatement`](https://docs.oracle.com/javase/8/docs/api/java/sql/PreparedStatement.html) class documentation.

Perl
:   Without bind parameters:

    ```
    my $subsidiary_id;
    my $sth = $dbh->prepare(
                      "select first_name, last_name" 
                    . "  from employees"
                    . " where subsidiary_id = $subsidiary_id"
                    );
    $sth->execute();
    ```

    Using a bind parameter:

    ```
    my $subsidiary_id;
    my $sth = $dbh->prepare(
                      "select first_name, last_name" 
                    . "  from employees"
                    . " where subsidiary_id = ?"
                    );
    $sth->execute($subsidiary_id);
    ```

    See: [Programming the Perl DBI](https://docstore.mik.ua/orelly/linux/dbi/ch05_03.htm).

PHP
:   Using MySQL, without bind parameters:

    ```
    $mysqli->query("select first_name, last_name" 
                 . "  from employees"
                 . " where subsidiary_id = " . $subsidiary_id);
    ```

    Using a bind parameter:

    ```
    if ($stmt = $mysqli->prepare("select first_name, last_name" 
                               . "  from employees"
                               . " where subsidiary_id = ?")) 
    {
       $stmt->bind_param("i", $subsidiary_id);
       $stmt->execute();
    } else {
      /* handle SQL error */
    }
    ```

    See also: [`mysqli_stmt::bind_param`](https://www.php.net/manual/en/mysqli-stmt.bind-param.php) class documentation and [“Prepared statements and stored procedures” in the PDO documentation](https://www.php.net/manual/en/pdo.prepared-statements.php).

Ruby
:   Without bind parameters:

    ```
    dbh.execute("select first_name, last_name" 
              + "  from employees"
              + " where subsidiary_id = #{subsidiary_id}");
    ```

    Using a bind parameter:

    ```
    dbh.prepare("select first_name, last_name" 
              + "  from employees"
              + " where subsidiary_id = ?");
    dbh.execute(subsidiary_id);
    ```

    See also: [“Quoting, Placeholders, and Parameter Binding” in the Ruby DBI Tutorial](https://web.archive.org/web/20190228233411/http://www.kitebird.com/articles/ruby-dbi.html#TOC_8).

#### See Also

[Examples using ORM Tools](https://use-the-index-luke.com/sql/myth-directory/dynamic-sql-is-slow#myth-dynamic-sql-sample)

The question mark (`?`) is the only placeholder character that the SQL standard defines. Question marks are positional parameters. That means the question marks are numbered from left to right. To bind a value to a particular question mark, you have to specify its number. That can, however, be very impractical because the numbering changes when adding or removing placeholders. Many databases offer a proprietary extension for named parameters to solve this problem—e.g., using an “at” symbol (`@name`) or a colon (`:name`).

#### Note

Bind parameters cannot change the structure of an SQL statement.

That means you cannot use bind parameters for table or column names. The following bind parameters do not work:

```
String sql = prepare("SELECT * FROM ? WHERE ?");

sql.execute('employees', 'employee_id = 1');
```

If you need to change the structure of an SQL statement during runtime, use [dynamic SQL](https://use-the-index-luke.com/sql/myth-directory/dynamic-sql-is-slow).

## Cursor Sharing and Forced Parameterization

The more complex the optimizer and the SQL query become, the more important execution plan caching becomes. The SQL Server and Oracle databases have features to automatically replace the literal values in a SQL string with bind parameters. These features are called `CURSOR_SHARING` (Oracle) or *forced parameterization* (SQL Server).

Both features are workarounds for applications that do not use bind parameters at all. Enabling these features prevents developers from intentionally using literal values.

#### See Also

- [*Smart Logic*](https://use-the-index-luke.com/sql/where-clause/obfuscation/smart-logic#smart_logic_affected) has more information on the execution plan caching capabilities of different databases.
- Article: [Planning for Execution Plan Reuse](https://use-the-index-luke.com/blog/2011-07-16/planning-for-reuse)

## Searching for Ranges

> Source: https://use-the-index-luke.com/sql/where-clause/searching-for-ranges

Inequality operators such as `<`, `>` and `between` can use indexes just like the equals operator [explained above](https://use-the-index-luke.com/sql/where-clause/the-equals-operator). Even a `LIKE` filter can—under certain circumstances—use an index just like range conditions do.

Using these operations limits the choice of the column order in multi-column indexes. This limitation can even rule out all optimal indexing options—there are queries where you simply cannot define a “correct” column order at all.

## Greater, Less and BETWEEN

> Source: https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates

The biggest performance risk of an `INDEX RANGE SCAN` is the [leaf node traversal](https://use-the-index-luke.com/sql/anatomy/the-leaf-nodes). It is therefore the golden rule of indexing to keep the scanned index range as small as possible. You can check that by asking yourself where an index scan starts and where it ends.

The question is easy to answer if the SQL statement mentions the start and stop conditions explicitly:

```
SELECT first_name, last_name, date_of_birth
  FROM employees
 WHERE date_of_birth >= TO_DATE(?, 'YYYY-MM-DD')
   AND date_of_birth <= TO_DATE(?, 'YYYY-MM-DD')
```

An index on `DATE_OF_BIRTH` is only scanned in the specified range. The scan starts at the first date and ends at the second. We cannot narrow the scanned index range any further.

The start and stop conditions are less obvious if a second column becomes involved:

```
SELECT first_name, last_name, date_of_birth
  FROM employees
 WHERE date_of_birth >= TO_DATE(?, 'YYYY-MM-DD')
   AND date_of_birth <= TO_DATE(?, 'YYYY-MM-DD')
   AND subsidiary_id  = ?
```

Of course an ideal index has to cover both columns, but the question is in which order?

The following figures show the effect of the column order on the scanned index range. For this illustration we search all employees of subsidiary 27 who were born between January 1st and January 9th 1971.

[Figure 2.2](#fig-range-bad) visualizes a detail of the index on `DATE_OF_BIRTH` and `SUBSIDIARY_ID`—in that order. Where will the database start to follow the leaf node chain, or to put it another way: where will the [tree traversal](https://use-the-index-luke.com/sql/anatomy/the-tree) end?

## Figure 2.2 Range Scan in `DATE_OF_BIRTH`, `SUBSIDIARY_ID` Index

Scanned index rangeDATE\_OF\_BIRTHSUBSIDIARY\_IDDATE\_OF\_BIRTHSUBSIDIARY\_ID28-DEC-7001-JAN-7101-JAN-71436ROWIDROWIDROWID02-JAN-7104-JAN-7105-JAN-71113ROWIDROWIDROWID06-JAN-7106-JAN-7108-JAN-714116ROWIDROWIDROWID08-JAN-7109-JAN-7109-JAN-71271017ROWIDROWIDROWID09-JAN-7109-JAN-7112-JAN-7117303ROWIDROWIDROWID27-DEC-7001-JAN-7105-JAN-71196308-JAN-7109-JAN-7112-JAN-716173

The index is ordered by birth dates first. Only if two employees were born on the same day is the `SUBSIDIARY_ID` used to sort these records. The query, however, covers a date *range*. The ordering of `SUBSIDIARY_ID` is therefore useless during tree traversal. That becomes obvious if you realize that there is no entry for subsidiary 27 in the branch nodes—although there is one in the leaf nodes. The filter on `DATE_OF_BIRTH` is therefore the only condition that limits the scanned index range. It starts at the first entry matching the date range and ends at the last one—all five leaf nodes shown in [Figure 2.2](#fig-range-bad).

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-greater-less-between&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

The picture looks entirely different when reversing the column order. [Figure 2.3](#fig-range-good) illustrates the scan if the index starts with the `SUBSIDIARY_ID` column.

## Figure 2.3 Range Scan in `SUBSIDIARY_ID`, `DATE_OF_BIRTH` Index

ROWIDROWIDROWIDScanned index rangeSUBSIDIARY\_IDDATE\_OF\_BIRTHSUBSIDIARY\_IDDATE\_OF\_BIRTH01-SEP-8323-NOV-6425-JUN-6926272723-SEP-6908-JAN-7126-SEP-7227272704-OCT-7318-DEC-7516-AUG-7627272723-AUG-7630-JUL-7814-SEP-8427272709-MAR-8808-OCT-9130-SEP-5327273012-SEP-6025-JUN-6926-SEP-7226272716-AUG-7614-SEP-8430-SEP-53272730

The difference is that the equals operator limits the first index column to a single value. Within the range for this value (`SUBSIDIARY_ID` 27) the index is sorted according to the second column—the date of birth—so there is no need to visit the first leaf node because the branch node already indicates that there is no employee for subsidiary 27 born after June 25th 1969 in the first leaf node.

The tree traversal directly leads to the second leaf node. In this case, all `where` clause conditions limit the scanned index range so that the scan terminates at the very same leaf node.

#### Tip

Rule of thumb: index for equality first—then for ranges.

The actual performance difference depends on the data and search criteria. The difference can be negligible if the filter on `DATE_OF_BIRTH` is very selective on its own. The bigger the date range becomes, the bigger the performance difference will be.

With this example, we can also falsify the myth that the most selective column should be at the leftmost index position. If we look at the figures and consider the selectivity of the first column only, we see that both conditions match 13 records. This is the case regardless whether we filter by `DATE_OF_BIRTH` only or by `SUBSIDIARY_ID` only. The selectivity is of no use here, but one column order is still better than the other.

To optimize performance, it is very important to know the scanned index range. With most databases you can even see this in the execution plan—you just have to know what to look for. The following execution plan from the Oracle database unambiguously indicates that the `EMP_TEST` index starts with the `DATE_OF_BIRTH` column.

Db2 (LUW)
:   ```
    Explain Plan
    ----------------------------------------------------
    ID | Operation         |                 Rows | Cost
     1 | RETURN            |                      |   26
     2 |  FETCH EMPLOYEES  |     3 of 3 (100.00%) |   26
     3 |   IXSCAN EMP_TEST | 3 of 10000 (   .03%) |    6

    Predicate Information
     3 - START ( TO_DATE(?, 'YYYY-MM-DD') <= Q1.DATE_OF_BIRTH)
         START (Q1.SUBSIDIARY_ID = ?)
          STOP (Q1.DATE_OF_BIRTH <= TO_DATE(?, 'YYYY-MM-DD'))
          STOP (Q1.SUBSIDIARY_ID = ?)
          SARG (Q1.SUBSIDIARY_ID = ?)
    ```

    In Db2 access predicates are labeled `START` and/or `STOP` while filter predicates are marked shown as `SARG`.

Oracle
:   ```
    --------------------------------------------------------------
    |Id | Operation                    | Name      | Rows | Cost |
    --------------------------------------------------------------
    | 0 | SELECT STATEMENT             |           |    1 |    4 |
    |*1 |  FILTER                      |           |      |      |
    | 2 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEES |    1 |    4 |
    |*3 |    INDEX RANGE SCAN          | EMP_TEST  |    2 |    2 |
    --------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------
    1 - filter(:END_DT >= :START_DT)
    3 - access(DATE_OF_BIRTH >= :START_DT 
           AND DATE_OF_BIRTH <= :END_DT)
        filter(SUBSIDIARY_ID  = :SUBS_ID)
    ```

PostgreSQL
:   ```
                                QUERY PLAN
    -------------------------------------------------------------------
    Index Scan using emp_test on employees
      (cost=0.01..8.59 rows=1 width=16)
      Index Cond: (date_of_birth >= to_date('1971-01-01','YYYY-MM-DD'))
              AND (date_of_birth <= to_date('1971-01-10','YYYY-MM-DD'))
              AND (subsidiary_id = 27::numeric)
    ```

    The PostgreSQL database does not indicate index access and filter predicates in the execution plan. However, the `Index Cond` section lists the columns in order of the index definition. In that case, we see the two `DATE_OF_BIRTH` predicates first, than the `SUBSIDIARY_ID`. Knowing that any predicates following a range condition cannot be an access predicate the `SUBSIDIARY_ID` must be a filter predicate. See [*Distinguishing Access and Filter-Predicates*](https://use-the-index-luke.com/sql/explain-plan/postgresql/filter-predicates) for more details.

SQL Server
:   ```
    |--Nested Loops(Inner Join)
       |--Index Seek(OBJECT:emp_test,
       |               SEEK:       (date_of_birth, subsidiary_id)
       |                        >= ('1971-01-01', 27)
       |                    AND    (date_of_birth, subsidiary_id)
       |                        <= ('1971-01-10', 27),
       |              WHERE:subsidiary_id=27
       |            ORDERED FORWARD)
       |--RID Lookup(OBJECT:employees,
                       SEEK:Bmk1000=Bmk1000
                     LOOKUP ORDERED FORWARD)
    ```

    SQL Server 2012 shows the seek predicates (=access predicates) using the [row-value syntax](https://use-the-index-luke.com/sql/partial-results/fetch-next-page#sb-row-values).

The *predicate information* for the `INDEX RANGE SCAN` gives the crucial hint. It identifies the conditions of the `where` clause either as *access* or as *filter* predicates. This is how the database tells us how it uses each condition.

#### Note

The execution plan was simplified for clarity. [The appendix](https://use-the-index-luke.com/sql/explain-plan/oracle/filter-predicates) explains the details of the “Predicate Information” section in an Oracle execution plan.

The conditions on the `DATE_OF_BIRTH` column are the only ones listed as access predicates; they limit the scanned index range. The `DATE_OF_BIRTH` is therefore the first column in the `EMP_TEST` index. The `SUBSIDIARY_ID` column is used only as a filter.

#### Important

The *access predicates* are the start and stop conditions for an index lookup. They define the scanned index range.

*Index filter predicates* are applied during the [leaf node traversal](https://use-the-index-luke.com/sql/anatomy/the-leaf-nodes) only. They do not narrow the scanned index range.

The appendix explains how to recognize access predicates in [MySQL](https://use-the-index-luke.com/sql/explain-plan/mysql/access-filter-predicates), [SQL Server](https://use-the-index-luke.com/sql/explain-plan/sql-server/filter-predicates) and [PostgreSQL](https://use-the-index-luke.com/sql/explain-plan/postgresql/filter-predicates).

The database can use all conditions as access predicates if we turn the index definition around:

Db2 (LUW)
:   ```
    -----------------------------------------------------
    ID | Operation          |                 Rows | Cost
     1 | RETURN             |                      |   13
     2 |  FETCH EMPLOYEES   |     3 of 3 (100.00%) |   13
     3 |   IXSCAN EMP_TEST2 | 3 of 10000 (   .03%) |    6

    Predicate Information
     3 - START (Q1.SUBSIDIARY_ID = ?)
         START ( TO_DATE(?, 'YYYY-MM-DD') <= Q1.DATE_OF_BIRTH)
          STOP (Q1.SUBSIDIARY_ID = ?)
          STOP (Q1.DATE_OF_BIRTH <= TO_DATE(?, 'YYYY-MM-DD'))
    ```

Oracle
:   ```
    ---------------------------------------------------------------
    | Id | Operation                    | Name      | Rows | Cost |
    ---------------------------------------------------------------
    |  0 | SELECT STATEMENT             |           |    1 |    3 |
    |* 1 |  FILTER                      |           |      |      |
    |  2 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEES |    1 |    3 |
    |* 3 |    INDEX RANGE SCAN          | EMP_TEST2 |    1 |    2 |
    ---------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------
    1 - filter(:END_DT >= :START_DT)
    3 - access(SUBSIDIARY_ID  = :SUBS_ID
           AND DATE_OF_BIRTH >= :START_DT
           AND DATE_OF_BIRTH <= :END_T)
    ```

PostgreSQL
:   ```
                                QUERY PLAN
    -------------------------------------------------------------------
    Index Scan using emp_test on employees
       (cost=0.01..8.29 rows=1 width=17)
       Index Cond: (subsidiary_id = 27::numeric)
               AND (date_of_birth >= to_date('1971-01-01', 'YYYY-MM-DD'))
               AND (date_of_birth <= to_date('1971-01-10', 'YYYY-MM-DD'))
    ```

    The PostgreSQL database does not indicate index access and filter predicates in the execution plan. However, the `Index Cond` section lists the columns in order of the index definition. In that case, we see the `SUBSIDIARY_ID` predicate first, than the two on `DATE_OF_BIRTH`. As there is no further column filtered after the range condition on `DATE_OF_BIRTH` we know that all predicates can be used as access predicate. See [*Distinguishing Access and Filter-Predicates*](https://use-the-index-luke.com/sql/explain-plan/postgresql/filter-predicates) for more details.

SQL Server
:   ```
    |--Nested Loops(Inner Join)
       |--Index Seek(OBJECT:emp_test,
       |               SEEK: subsidiary_id=27
       |                 AND date_of_birth >= '1971-01-01'
       |                 AND date_of_birth <= '1971-01-10'
       |            ORDERED FORWARD)
       |--RID Lookup(OBJECT:employees),
                       SEEK:Bmk1000=Bmk1000
                     LOOKUP ORDERED FORWARD)
    ```

Finally, there is the `between` operator. It allows you to specify the upper and lower bounds in a single condition:

```
DATE_OF_BIRTH BETWEEN '01-JAN-71'
                  AND '10-JAN-71'
```

Note that `between` always includes the specified values, just like using the less than or equal to (`<=`) and greater than or equal to (`>=`) operators:

```
    DATE_OF_BIRTH >= '01-JAN-71' 
AND DATE_OF_BIRTH <= '10-JAN-71'
```

## Indexing SQL LIKE Filters

> Source: https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning

The SQL `LIKE` operator very often causes unexpected performance behavior because some search terms prevent efficient index usage. That means that there are search terms that can be indexed very well, but others can not. It is the position of the wild card characters that makes all the difference.

The following example uses the `%` wild card in the middle of the search term:

```
SELECT first_name, last_name, date_of_birth
  FROM employees
 WHERE UPPER(last_name) LIKE 'WIN%D'
```

Db2 (LUW)
:   ```
    Explain Plan
    ----------------------------------------------------
    ID | Operation         |                 Rows | Cost
     1 | RETURN            |                      |   13
     2 |  FETCH EMPLOYEES  |     1 of 1 (100.00%) |   13
     3 |   IXSCAN EMP_NAME | 1 of 10000 (   .01%) |    6

    Predicate Information
     3 - START ('WIN....................................
          STOP (Q1.LAST_NAME <= 'WIN....................
          SARG (Q1.LAST_NAME LIKE 'WIN%D')
    ```

    For this example, the query was changed to read `WHERE last_name LIKE 'WIN%D'` (no `UPPER`). It seems like Db2 (LUW) 10.5 cannot use an access predicates from `LIKE` on a function-based index (does a full index scan at best).

    Otherwise, Db2 shines here: it clearly shows the `START` and `STOP` conditions, which consist of the part before the first wild card, but also shows that the full pattern is applied as filter predicate.

MySQL
:   ```
    +----+-----------+-------+----------+---------+------+-------------+
    | id | table     | type  | key      | key_len | rows | Extra       |
    +----+-----------+-------+----------+---------+------+-------------+
    |  1 | employees | range | emp_name | 767     |    2 | Using where |
    +----+-----------+-------+----------+---------+------+-------------+
    ```

Oracle
:   ```
    ---------------------------------------------------------------
    |Id | Operation                   | Name        | Rows | Cost |
    ---------------------------------------------------------------
    | 0 | SELECT STATEMENT            |             |    1 |    4 |
    | 1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES   |    1 |    4 |
    |*2 |   INDEX RANGE SCAN          | EMP_UP_NAME |    1 |    2 |
    ---------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------
       2 - access(UPPER("LAST_NAME") LIKE 'WIN%D')
           filter(UPPER("LAST_NAME") LIKE 'WIN%D')
    ```

PostgreSQL
:   ```
                           QUERY PLAN
    ----------------------------------------------------------
    Index Scan using emp_up_name on employees
       (cost=0.01..8.29 rows=1 width=17)
       Index Cond: (upper((last_name)::text) ~>=~ 'WIN'::text)
               AND (upper((last_name)::text) ~<~  'WIO'::text)
           Filter: (upper((last_name)::text) ~~ 'WIN%D'::text)
    ```

`LIKE` filters can only use the characters *before the first wild card* during tree traversal. The remaining characters are just filter predicates that do not narrow the scanned index range. A single `LIKE` expression can therefore contain two predicate types: (1) the part before the first wild card as an access predicate; (2) the other characters as a filter predicate.

#### Caution

The `LIKE` operator works on a character-by-character basis while collations can treat multiple characters as a single sorting item. Thus some collations prevent using indexes for `LIKE`. Read [Indexing “LIKE” in PostgreSQL and Oracle](https://www.cybertec-postgresql.com/en/indexing-like-postgresql-oracle/) by Laurenz Albe for further details.

The more selective the prefix before the first wild card is, the smaller the scanned index range becomes. That, in turn, makes the index lookup faster. [Figure 2.4](#fig-like) illustrates this relationship using three different `LIKE` expressions. All three select the same row, but the scanned index range—and thus the performance—is very different.

## Figure 2.4 Various `LIKE` Searches

LIKE 'WI%ND'LIKE 'WIN%D'LIKE 'WINA%'WIAWWIBLQQNPUAWIBYHSNZWIFMDWUQMBWIGLZXWIHWIHTFVZNLCWIJYAXPPWINANDWINBKYDSKWWIPOJWISRGPKWITJIVQJWIWWIWGPJMQGGWIWKHLBJWIYETHNWIYJWIAWWIBLQQNPUAWIBYHSNZWIFMDWUQMBWIGLZXWIHWIHTFVZNLCWIJYAXPPWINANDWINBKYDSKWWIPOJWISRGPKWITJIVQJWIWWIWGPJMQGGWIWKHLBJWIYETHNWIYJWIAWWIBLQQNPUAWIBYHSNZWIFMDWUQMBWIGLZXWIHWIHTFVZNLCWIJYAXPPWINANDWINBKYDSKWWIPOJWISRGPKWITJIVQJWIWWIWGPJMQGGWIWKHLBJWIYETHNWIYJ

The first expression has two characters before the wild card. They limit the scanned index range to 18 rows. Only one of them matches the entire `LIKE` expression—the other 17 are fetched but discarded. The second expression has a longer prefix that narrows the scanned index range down to two rows. With this expression, the database just reads one extra row that is not relevant for the result. The last expression does not have a filter predicate at all: the database just reads the entry that matches the entire `LIKE` expression.

#### Important

Only the part before the first wild card serves as an access predicate.

The remaining characters do not narrow the scanned index range—non-matching entries are just left out of the result.

The opposite case is also possible: a `LIKE` expression that starts with a wild card. Such a `LIKE` expression cannot serve as an access predicate. The database has to scan the entire table if there are no other conditions that provide access predicates.

#### Tip

Avoid `LIKE` expressions with leading wildcards (e.g., `'%TERM'`).

The position of the wild card characters affects index usage—at least in theory. In reality the optimizer creates a generic execution plan when the search term is supplied via [bind parameters](https://use-the-index-luke.com/sql/where-clause/bind-parameters). In that case, the optimizer has to guess whether or not the majority of executions will have a leading wild card.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-like&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

Most databases just assume that there is no leading wild card when optimizing a `LIKE` condition with bind parameter, but this assumption is wrong if the `LIKE` expression is used for a full-text search. There is, unfortunately, no direct way to tag a `LIKE` condition as full-text search. The box [“*Labeling Full-Text `LIKE` Expressions*”](#sb-static-like-prefix) shows an attempt that does not work. Specifying the search term without bind parameter is the most obvious solution, but that increases the optimization overhead and opens an SQL injection vulnerability. An effective but still secure and portable solution is to intentionally obfuscate the `LIKE` condition. [“*Combining Columns*”](https://use-the-index-luke.com/sql/where-clause/obfuscation/concatenation) explains this in detail.

## Labeling Full-Text `LIKE` Expressions

When using the `LIKE` operator for a full-text search, we could separate the wildcards from the search term:

```
WHERE text_column LIKE '%' || ? || '%'
```

For the PostgreSQL database, the problem is different because PostgreSQL assumes there *is* a leading wild card when using bind parameters for a `LIKE` expression. PostgreSQL just does not use an index in that case. The only way to get an index access for a `LIKE` expression is to make the actual search term visible to the optimizer. If you do not use a bind parameter but put the search term directly into the SQL statement, you must take other precautions against SQL injection attacks!

Even if the database optimizes the execution plan for a leading wild card, it can still deliver insufficient performance. You can use another part of the `where` clause to access the data efficiently in that case—see also [“*Index Filter Predicates Used Intentionally*”](https://use-the-index-luke.com/sql/clustering/index-filter-predicates). If there is no other access path, you might use one of the following proprietary full-text index solutions.

Db2 (LUW)
:   Db2 supports the `contains` keyword. See “[Search functions for Db2 Text Search](https://www.ibm.com/docs/en/db2/11.5.x?topic=indexes-search-functions)“.

MySQL
:   MySQL offers the `match` and `against` keywords for full-text searching. Starting with MySQL 5.6, you can create full-text indexes for InnoDB tables as well—previously, this was only possible with MyISAM tables. See “[Full-Text Search Functions](https://dev.mysql.com/doc/refman/8.0/en/fulltext-search.html)” in the MySQL documentation.

Oracle
:   The Oracle database offers the `contains` keyword. See the “[Oracle Text Application Developer’s Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/ccapp/#Oracle%C2%AE-Text).”

PostgreSQL
:   PostgreSQL offers the `@@` operator to implement full-text searches. See “[Full Text Search](https://www.postgresql.org/docs/current/textsearch.html)” in the PostgreSQL documentation.

    Another option is to use the [WildSpeed](http://www.sai.msu.su/~megera/wiki/wildspeed) extension to optimize `LIKE` expressions directly. The extension stores the text in all possible rotations so that each character is at the beginning once. That means that the indexed text is not only stored once but instead as many times as there are characters in the string—thus it needs a lot of space.

SQL Server
:   SQL Server offers the `contains` keyword. See “[Full-Text Search](https://learn.microsoft.com/en-us/sql/relational-databases/search/full-text-search?view=sql-server-ver16)” in the SQL Server documentation.

#### Think About It

How can you index a `LIKE` search that has only one wild card at the beginning of the search term (`'%TERM'`)?

## Index Combine

> Source: https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/index-merge-performance

It is one of the most common question about indexing: is it better to create one index for each column or a single index for all columns of a `where` clause? The answer is very simple in most cases: one index with multiple columns is better—that is, a concatenated or compound index. [“*Concatenated Indexes*”](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys) explains them in detail.

Nevertheless there are queries where a single index cannot do a perfect job, no matter how you define the index; e.g., queries with two or more independent range conditions as in the following example:

```
SELECT first_name, last_name, date_of_birth 
  FROM employees
 WHERE UPPER(last_name) < ? 
   AND date_of_birth    < ?
```

It is impossible to define a B-tree index that would support this query without filter predicates. For an explanation, you just need to remember that an [index is a linked list](https://use-the-index-luke.com/sql/anatomy).

If you define the index as `UPPER(LAST_NAME)`, `DATE_OF_BIRTH` (in that order), the list begins with A and ends with Z. The date of birth is considered only when there are two employees with the same name. If you define the index the other way around, it will start with the eldest employees and end with the youngest. In that case, the names only have a minor impact on the sort order.

No matter how you twist and turn the index definition, the entries are always arranged along a chain. At one end, you have the small entries and at the other end the big ones. An index can therefore only support one range condition as an access predicate. Supporting two independent range conditions requires a second axis, for example like a chessboard. The query above would then match all entries from one corner of the chessboard, but an index is not like a chessboard—it is like a chain. There is no corner.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-index-combine&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

You can of course accept the filter predicate and use a multi-column index nevertheless. That is the best solution in many cases anyway. The index definition should then mention the more selective column first so it can be used with an access predicate. That might be the origin of the “[most selective first](https://use-the-index-luke.com/sql/myth-directory/most-selective-first)” myth but this rule only holds true if you cannot avoid a filter predicate.

The other option is to use two separate indexes, one for each column. Then the database must scan both indexes first and then combine the results. The duplicate index lookup alone already involves more effort because the database has to traverse two index trees. Additionally, the database needs a lot of memory and CPU time to combine the intermediate results.

#### Note

One index scan is faster than two.

Databases use two methods to combine indexes. Firstly there is the index join. [Chapter 4*The Join Operation*](https://use-the-index-luke.com/sql/join) explains the related algorithms in detail. The second approach makes use of functionality from the data warehouse world.

The [data warehouse](https://en.wikipedia.org/wiki/Data_warehouse) is the mother of all ad-hoc queries. It just needs a few clicks to combine arbitrary conditions into the query of your choice. It is impossible to predict the column combinations that might appear in the `where` clause and that makes indexing, as explained so far, almost impossible.

Data warehouses use a special purpose index type to solve that problem: the so-called *bitmap index*. The advantage of bitmap indexes is that they can be combined rather easily. That means you get decent performance when indexing each column individually. Conversely if you know the query in advance, so that you can create a tailored multi-column B-tree index, it will still be faster than combining multiple bitmap indexes.

By far the greatest weakness of bitmap indexes is the ridiculous `insert`, `update` and `delete` scalability. Concurrent write operations are virtually impossible. That is no problem in a data warehouse because the load processes are scheduled one after another. In online applications, bitmap indexes are mostly useless.

#### Important

Bitmap indexes are almost unusable for online transaction pro­cessing (OLTP).

Many database products offer a hybrid solution between B-tree and bitmap indexes. In the absence of a better access path, they convert the results of several B-tree scans into in-memory bitmap structures. Those can be combined efficiently. The bitmap structures are not stored persistently but discarded after statement execution, thus bypassing the problem of the poor write scalability. The downside is that it needs a lot of memory and CPU time. This method is, after all, an optimizer’s act of desperation.

## Partial Indexes

> Source: https://use-the-index-luke.com/sql/where-clause/partial-and-filtered-indexes

So far we have only discussed which *columns* to add to an index. With *partial* (PostgreSQL) or *filtered* (SQL Server) indexes you can also specify the *rows* that are indexed.

#### Caution

The Oracle database has a unique approach to partial indexing. The [next section](https://use-the-index-luke.com/sql/where-clause/null) explains it while building upon this section.

Db2 (LUW) does not support partial indexes, but the can be [emulated like in the Oracle database](https://use-the-index-luke.com/blog/2014-11/seven-surprising-findings-about-DB2#blog-db2intro-partialidx) when using the `EXCLUDE NULL KEYS` feature.

A partial index is useful for commonly used `where` conditions that use constant values—like the status code in the following example:

```
SELECT message
  FROM messages
 WHERE processed = 'N'
   AND receiver  = ?
```

Queries like this are very common in queuing systems. The query fetches all unprocessed messages for a specific recipient. Messages that were already processed are rarely needed. If they are needed, they are usually accessed by a more specific criteria like the primary key.

We can optimize this query with a two-column index. Considering this query only, the column order does not matter because there is no range condition.

```
CREATE INDEX messages_todo
          ON messages (receiver, processed)
```

The index fulfills its purpose, but it includes many rows that are never searched, namely all the messages that were already processed. Due to the logarithmic scalability the index nevertheless makes the query very fast even though it wastes a lot of disk space.

With partial indexing you can limit the index to include only the unprocessed messages. The syntax for this is surprisingly simple: a `where` clause.

```
CREATE INDEX messages_todo
          ON messages (receiver)
       WHERE processed = 'N'
```

The index only contains the rows that satisfy the `where` clause. In this particular case, we can even remove the `PROCESSED` column because it is always `'N'` anyway. That means the index reduces its size in two dimensions: vertically, because it contains fewer rows; horizontally, due to the removed column.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-partial-indexes&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

The index is therefore very small. For a queue, it can even mean that the index size remains unchanged although the table grows without bounds. The index does not contain all messages, just the unprocessed ones.

The `where` clause of a partial index can become arbitrarily complex. The only fundamental limitation is about functions: you can only use deterministic functions as is the case everywhere in an index definition. SQL Server has, however, more restrictive rules and [neither allow functions nor the `OR` operator in index predicates](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-index-transact-sql?view=sql-server-ver16).

A database can use a partial index whenever the `where` clause appears in a query.

#### Think About It

What peculiarity has the smallest possible index for the following query:

```
SELECT message
  FROM messages
 WHERE processed = 'N'
```

## NULL in the Oracle Database

> Source: https://use-the-index-luke.com/sql/where-clause/null

SQL’s `NULL` frequently causes confusion. Although the basic idea of `NULL`—[to represent missing data](https://en.wikipedia.org/wiki/Null_%28SQL%29)—is rather simple, there are some peculiarities. You have to use `IS NULL` instead of `= NULL`, for example. Moreover the Oracle database has additional `NULL` oddities, on the one hand because it does not always handle `NULL` as required by the standard and on the other hand because it has a very “special” handling of `NULL` in indexes.

The SQL standard does not define `NULL` as a value but rather as a placeholder for a missing or unknown value. Consequently, no value can be `NULL`. Instead the Oracle database treats an empty string as `NULL`:

```
   SELECT     '0 IS NULL???' AS "what is NULL?" FROM dual
    WHERE      0 IS NULL
UNION ALL
   SELECT    '0 is not null' FROM dual
    WHERE     0 IS NOT NULL
UNION ALL
   SELECT ''''' IS NULL???'  FROM dual
    WHERE    '' IS NULL
UNION ALL
   SELECT ''''' is not null' FROM dual 
    WHERE    '' IS NOT NULL
```

To add to the confusion, there is even a case when the Oracle database treats `NULL` as empty string:

```
SELECT dummy
     , dummy || ''
     , dummy || NULL
  FROM dual
```

Concatenating the `DUMMY` column (always containing `'X'`) with `NULL` should return `NULL`.

The concept of `NULL` is used in many programming languages. No matter where you look, an empty string is never `NULL`…except in the Oracle database. It is, in fact, impossible to store an empty string in a `VARCHAR2` field. If you try, the Oracle database just stores `NULL`.

This peculiarity is not only strange; it is also dangerous. Additionally the Oracle database’s `NULL` oddity does not stop here—it continues with indexing.

## NULL in Indexes

> Source: https://use-the-index-luke.com/sql/where-clause/null/index

The Oracle database does not include rows in an index if all indexed columns are `NULL`. That means that every index is a [partial index](https://use-the-index-luke.com/sql/where-clause/partial-and-filtered-indexes)—like having a `where` clause:

```
CREATE INDEX idx
          ON tbl (A, B, C, ...)
       WHERE A IS NOT NULL
          OR B IS NOT NULL
          OR C IS NOT NULL
             ...
```

Consider the `EMP_DOB` index. It has only one column: the `DATE_OF_BIRTH`. A row that does not have a `DATE_OF_BIRTH` value is not added to this index.

```
INSERT INTO employees ( subsidiary_id, employee_id
                      , first_name   , last_name
                      , phone_number)
               VALUES ( ?, ?, ?, ?, ? )
```

The `insert` statement does not set the `DATE_OF_BIRTH` so it defaults to `NULL`—hence, the record is not added to the `EMP_DOB` index. As a consequence, the index cannot support a query for records where `DATE_OF_BIRTH` `IS NULL`:

```
SELECT first_name, last_name
  FROM employees
 WHERE date_of_birth IS NULL
```

Nevertheless, the record is inserted into a concatenated index if at least one index column is not `NULL`:

```
CREATE INDEX demo_null
          ON employees (subsidiary_id, date_of_birth)
```

The above created row is added to the index because the `SUBSIDIARY_ID` is not `NULL`. This index can thus support a query for all employees of a specific subsidiary that have no `DATE_OF_BIRTH` value:

```
SELECT first_name, last_name
  FROM employees
 WHERE subsidiary_id = ?
   AND date_of_birth IS NULL
```

Please note that the index covers the entire `where` clause; all filters are used as access predicates during the `INDEX RANGE SCAN`.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-indexingnull&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

We can extend this concept for the original query to find all records where `DATE_OF_BIRTH` `IS NULL`. For that, the `DATE_OF_BIRTH` column has to be the leftmost column in the index so that it can be used as access predicate. Although we do not need a second index column for the query itself, we add another column that can never be `NULL` to make sure the index has all rows. We can use any column that has a `NOT NULL` constraint, like `SUBSIDIARY_ID`, for that purpose.

Alternatively, we can use a constant expression that can never be `NULL`. That makes sure the index has all rows—even if `DATE_OF_BIRTH` is `NULL`.

```
DROP   INDEX emp_dob
```

```
CREATE INDEX emp_dob ON employees (date_of_birth, 'X')
```

Technically, this index is a [function-based index](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search). This example also dis­proves the myth that the Oracle database cannot index `NULL`.

#### Tip

Add a column that cannot be `NULL` to index `NULL` like any value.

## NOT NULL Constraints

> Source: https://use-the-index-luke.com/sql/where-clause/null/not-null-constraint

To index an `IS NULL` condition in the Oracle database, the index must have a column that can never be `NULL`.

That said, it is not enough that there are no `NULL` entries. The database has to be sure there can never be a `NULL` entry, otherwise the database must assume that the table has rows that are not in the index.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-knowingnotnull&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

The following index supports the query only if the column `LAST_NAME` has a `NOT NULL` constraint:

```
DROP INDEX emp_dob
```

```
CREATE INDEX emp_dob_name
          ON employees (date_of_birth, last_name)
```

```
SELECT *
  FROM employees
 WHERE date_of_birth IS NULL
```

Removing the `NOT NULL` constraint renders the index unusable for this query:

```
ALTER TABLE employees MODIFY last_name NULL
```

```
SELECT *
  FROM employees
 WHERE date_of_birth IS NULL
```

#### Tip

A missing `NOT NULL` constraint can prevent index usage in an Oracle database—especially for `count(*)` queries.

Besides `NOT NULL` constraints, the database also knows that constant expressions like in the [previous section](https://use-the-index-luke.com/sql/where-clause/null/index) cannot become `NULL`.

An index on a user-defined function, however, does not impose a `NOT NULL` constraint on the index expression:

```
CREATE OR REPLACE FUNCTION blackbox(id IN NUMBER) RETURN NUMBER
DETERMINISTIC
IS BEGIN
   RETURN id;
END
```

```
DROP INDEX emp_dob_name
```

```
CREATE INDEX emp_dob_bb 
    ON employees (date_of_birth, blackbox(employee_id))
```

```
SELECT *
  FROM employees
 WHERE date_of_birth IS NULL
```

```
----------------------------------------------------
| Id | Operation         | Name      | Rows | Cost |
----------------------------------------------------
|  0 | SELECT STATEMENT  |           |    1 |  477 |
|* 1 |  TABLE ACCESS FULL| EMPLOYEES |    1 |  477 |
----------------------------------------------------
```

The function name `BLACKBOX` emphasizes the fact that the optimizer has no idea what the function does (see [“*Case-Insensitive Search Using `UPPER` or `LOWER`*”](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search)). We can see that the function passes the input value straight through, but for the database it is just a function that returns a number. The `NOT NULL` property of the parameter is lost. Although the index must have all rows, the database does not know that so it cannot use the index for the query.

If *you know* that the function never returns `NULL`, as in this example, you can change the query to reflect that:

```
SELECT *
  FROM employees
 WHERE date_of_birth IS NULL
   AND blackbox(employee_id) IS NOT NULL
```

```
-------------------------------------------------------------
|Id |Operation                   | Name       | Rows | Cost |
-------------------------------------------------------------
| 0 |SELECT STATEMENT            |            |    1 |    3 |
| 1 | TABLE ACCESS BY INDEX ROWID| EMPLOYEES  |    1 |    3 |
|*2 |  INDEX RANGE SCAN          | EMP_DOB_BB |    1 |    2 |
-------------------------------------------------------------
```

The extra condition in the `where` clause is always true and therefore does not change the result. Nevertheless the Oracle database recognizes that you only query rows that must be in the index per definition.

There is, unfortunately, no way to tag a function that never returns `NULL` but you can move the function call to a [virtual column](https://modern-sql.com/caniuse/generated-always-as) (since 11*g*) and put a `NOT NULL` constraint on this column.

```
ALTER TABLE employees ADD bb_expression
      GENERATED ALWAYS AS (blackbox(employee_id)) NOT NULL
```

```
DROP   INDEX emp_dob_bb
```

```
CREATE INDEX emp_dob_bb 
    ON employees (date_of_birth, bb_expression)
```

```
SELECT *
  FROM employees
 WHERE date_of_birth IS NULL
   AND blackbox(employee_id) IS NOT NULL
```

```
-------------------------------------------------------------
|Id |Operation                   | Name       | Rows | Cost |
-------------------------------------------------------------
| 0 |SELECT STATEMENT            |            |    1 |    3 |
| 1 | TABLE ACCESS BY INDEX ROWID| EMPLOYEES  |    1 |    3 |
|*2 |  INDEX RANGE SCAN          | EMP_DOB_BB |    1 |    2 |
-------------------------------------------------------------
```

The Oracle database knows that some internal functions only return `NULL` if `NULL` is provided as input.

```
DROP INDEX emp_dob_bb
```

```
CREATE INDEX emp_dob_upname 
    ON employees (date_of_birth, upper(last_name))
```

```
SELECT *
  FROM employees
 WHERE date_of_birth IS NULL
```

```
----------------------------------------------------------
|Id |Operation                   | Name           | Cost |
----------------------------------------------------------
| 0 |SELECT STATEMENT            |                |    3 |
| 1 | TABLE ACCESS BY INDEX ROWID| EMPLOYEES      |    3 |
|*2 |  INDEX RANGE SCAN          | EMP_DOB_UPNAME |    2 |
----------------------------------------------------------
```

The `UPPER` function preserves the `NOT NULL` property of the `LAST_NAME` column. Removing the constraint, however, renders the index unusable:

```
ALTER TABLE employees MODIFY last_name NULL
```

```
SELECT *
  FROM employees
 WHERE date_of_birth IS NULL
```

```
----------------------------------------------------
| Id | Operation         | Name      | Rows | Cost |
----------------------------------------------------
|  0 | SELECT STATEMENT  |           |    1 |  477 |
|* 1 |  TABLE ACCESS FULL| EMPLOYEES |    1 |  477 |
----------------------------------------------------
```

## Emulating Partial Indexes

> Source: https://use-the-index-luke.com/sql/where-clause/null/partial-index

The strange way the Oracle database handles `NULL` in indexes can be used to emulate partial indexes. For that, we just have to use `NULL` for rows that should not be indexed.

To demonstrate, we emulate the following partial index:

```
CREATE INDEX messages_todo
          ON messages (receiver)
       WHERE processed = 'N'
```

First, we need a function that returns the `RECEIVER` value only if the `PROCESSED` value is `'N'`.

```
CREATE OR REPLACE
FUNCTION pi_processed(processed CHAR, receiver NUMBER)
RETURN NUMBER
DETERMINISTIC
AS BEGIN
   IF processed IN ('N') THEN
      RETURN receiver;
   ELSE
      RETURN NULL;
   END IF;
END
```

The function must be [deterministic so it can be used in an index definition](https://use-the-index-luke.com/sql/where-clause/functions/user-defined-functions).

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-partial-indexes-oracle&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

Now we can create an index that contains only the rows having `PROCESSED='N'`.

```
CREATE INDEX messages_todo
          ON messages (pi_processed(processed, receiver))
```

To use the index, you must use the indexed expression in the query:

```
SELECT message
  FROM messages
 WHERE pi_processed(processed, receiver) = ?
```

```
----------------------------------------------------------
|Id | Operation                   | Name          | Cost |
----------------------------------------------------------
| 0 | SELECT STATEMENT            |               | 5330 |
| 1 |  TABLE ACCESS BY INDEX ROWID| MESSAGES      | 5330 |
|*2 |   INDEX RANGE SCAN          | MESSAGES_TODO | 5303 |
----------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("PI_PROCESSED"("PROCESSED","RECEIVER")=:X)
```

## Partial Indexes, Part II

As of release 11*g*, there is a second—equally scary—approach to emulating partial indexes in the Oracle database by using an intentionally broken index partition and the [`SKIP_UNUSABLE_INDEXES`](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/SKIP_UNUSABLE_INDEXES.html) parameter.

## Obfuscated Conditions

> Source: https://use-the-index-luke.com/sql/where-clause/obfuscation

The following sections demonstrate some popular methods for obfuscating conditions. Obfuscated conditions are `where` clauses that are phrased in a way that prevents proper index usage. This section is a collection of anti-patterns every developer should know about and avoid.

## Dates

> Source: https://use-the-index-luke.com/sql/where-clause/obfuscation/dates

Most obfuscations involve `DATE` types. The Oracle database is particularly vulnerable in this respect because it has only one `DATE` type that always includes a time component as well.

It has become common practice to use the `TRUNC` function to remove the time component. In truth, it does not remove the time but instead sets it to midnight because the Oracle database has no pure `DATE` type. To disregard the time component for a search you can use the `TRUNC` function on both sides of the comparison—e.g., to search for yesterday’s sales:

```
SELECT ...
  FROM sales
 WHERE TRUNC(sale_date) = TRUNC(sysdate - INTERVAL '1' DAY)
```

It is a perfectly valid and correct statement but it cannot properly make use of an index on `SALE_DATE`. It is as explained in [“*Case-Insensitive Search Using `UPPER` or `LOWER`*”](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search); `TRUNC(sale_date)` is something entirely different from `SALE_DATE`—functions are black boxes to the database.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-obfuscated-dates&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

There is a rather simple solution for this problem: a [function-based index](https://use-the-index-luke.com/sql/where-clause/functions).

```
CREATE INDEX index_name
          ON sales (TRUNC(sale_date))
```

But then you must always use `TRUNC(sale_date)` in the `where` clause. If you use it inconsistently—sometimes with, sometimes without `TRUNC`—then you need two indexes!

The problem also occurs with databases that have a pure date type if you search for a longer period as shown in the following MySQL query:

```
SELECT ...
  FROM sales
 WHERE DATE_FORMAT(sale_date, "%Y-%M")
     = DATE_FORMAT(now()    , "%Y-%M")
```

The query uses a date format that only contains year and month: again, this is an absolutely correct query that has the same problem as before. However the solution from above does not apply to MySQL prior to version 5.7, because MySQL didn’t support function-based indexing before that version.

The alternative is to use an explicit range condition. This is a generic solution that works for all databases:

```
SELECT ...
  FROM sales
 WHERE sale_date BETWEEN quarter_begin(?) 
                     AND quarter_end(?)
```

If you have done your homework, you probably recognize the pattern from the [exercise about all employees who are 42 years old](https://use-the-index-luke.com/sql/where-clause/functions/user-defined-functions#think-age-index).

A straight index on `SALE_DATE` is enough to optimize this query. The functions `QUARTER_BEGIN` and `QUARTER_END` compute the boundary dates. The calculation can become a little complex because the [`between` operator always includes the boundary values](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates#para-between). The `QUARTER_END` function must therefore return a time stamp just before the first day of the next quarter if the `SALE_DATE` has a time component. This logic can be hidden in the function.

The following examples show implementations of the functions `QUARTER_BEGIN` and `QUARTER_END` for various databases.

Db2 (LUW)
:   ```
    CREATE FUNCTION quarter_begin(dt TIMESTAMP)
    RETURNS TIMESTAMP
    RETURN TRUNC(dt, 'Q')
    ```

    ```
    CREATE FUNCTION quarter_end(dt TIMESTAMP)
    RETURNS TIMESTAMP
    RETURN TRUNC(dt, 'Q') + 3 MONTHS - 1 SECOND
    ```

MySQL
:   ```
    CREATE FUNCTION quarter_begin(dt DATETIME)
    RETURNS DATETIME DETERMINISTIC
    RETURN CONVERT
           (
             CONCAT
             ( CONVERT(YEAR(dt),CHAR(4))
             , '-'
             , CONVERT(QUARTER(dt)*3-2,CHAR(2))
             , '-01'
             )
           , datetime
           )
    ```

    ```
    CREATE FUNCTION quarter_end(dt DATETIME)
    RETURNS DATETIME DETERMINISTIC
    RETURN DATE_ADD
           ( DATE_ADD ( quarter_begin(dt), INTERVAL 3 MONTH )
           , INTERVAL -1 MICROSECOND)
    ```

Oracle
:   ```
    CREATE FUNCTION quarter_begin(dt IN DATE) 
    RETURN DATE
    AS
    BEGIN
       RETURN TRUNC(dt, 'Q');
    END
    ```

    ```
    CREATE FUNCTION quarter_end(dt IN DATE) 
    RETURN DATE
    AS
    BEGIN
       -- the Oracle DATE type has seconds resolution
       -- subtract one second from the first 
       -- day of the following quarter
       RETURN TRUNC(ADD_MONTHS(dt, +3), 'Q') 
            - (1/(24*60*60));
    END
    ```

PostgreSQL
:   ```
    CREATE FUNCTION quarter_begin(dt timestamp with time zone)
    RETURNS timestamp with time zone AS $$
    BEGIN
        RETURN date_trunc('quarter', dt);
    END;
    $$ LANGUAGE plpgsql
    ```

    ```
    CREATE FUNCTION quarter_end(dt timestamp with time zone)
    RETURNS timestamp with time zone AS $$
    BEGIN
       RETURN   date_trunc('quarter', dt) 
              + interval '3 month'
              - interval '1 microsecond';
    END;
    $$ LANGUAGE plpgsql
    ```

SQL Server
:   ```
    CREATE FUNCTION quarter_begin (@dt DATETIME )
    RETURNS DATETIME
    BEGIN
      RETURN DATEADD (qq, DATEDIFF (qq, 0, @dt), 0)  
    END
    ```

    ```
    CREATE FUNCTION quarter_end (@dt DATETIME )
    RETURNS DATETIME
    BEGIN
      RETURN DATEADD
             ( ms
             , -3 
             , DATEADD(mm, 3, dbo.quarter_begin(@dt))
             );
    END
    ```

You can use similar auxiliary functions for other periods—most of them will be less complex than the examples above, especially when using greater than or equal to (`>=`) and less than (`<`) conditions instead of the `between` operator. Of course you could calculate the boundary dates in your application if you wish.

#### Tip

Write queries for continuous periods as explicit range condition. Do this even for a single day—e.g., for the Oracle database:

```
    sale_date >= TRUNC(sysdate)
AND sale_date <  TRUNC(sysdate + INTERVAL '1' DAY)
```

Another common obfuscation is to compare dates as strings as shown in the following PostgreSQL example:

```
SELECT ...
  FROM sales
 WHERE TO_CHAR(sale_date, 'YYYY-MM-DD') = '1970-01-01'
```

The problem is, again, converting `SALE_DATE`. Such conditions are often created in the belief that you cannot pass different types than numbers and strings to the database. [Bind parameters](https://use-the-index-luke.com/sql/where-clause/bind-parameters), however, support all data types. That means you can for example use a `java.util.Date` object as bind parameter. This is yet another benefit of bind parameters.

If you cannot do that, you just have to convert the search term instead of the table column:

```
SELECT ...
  FROM sales
 WHERE sale_date = TO_DATE('1970-01-01', 'YYYY-MM-DD')
```

This query can use a straight index on `SALE_DATE`. Moreover it converts the input string only once. The previous statement must convert all dates stored in the table before it can compare them against the search term.

Whatever change you make—using a bind parameter or converting the other side of the comparison—you can easily introduce a bug if `SALE_DATE` has a time component. You must use an explicit range condition in that case:

```
SELECT ...
  FROM sales
 WHERE sale_date >= TO_DATE('1970-01-01', 'YYYY-MM-DD') 
   AND sale_date <  TO_DATE('1970-01-01', 'YYYY-MM-DD') 
                  + INTERVAL '1' DAY
```

Always consider using an explicit range condition when comparing dates.

## `LIKE` on Date Types

The following obfuscation is particularly tricky:

```
sale_date LIKE SYSDATE
```

It does not look like an obfuscation at first glance because it does not use any functions.

The `LIKE` operator, however, enforces a string comparison. Depending on the database, that might yield an error or cause an implicit type conversion on both sides. The “Predicate Information” section of the execution plan shows what the Oracle database does:

```
filter( INTERNAL_FUNCTION(SALE_DATE)
   LIKE TO_CHAR(SYSDATE@!))
```

The function [`INTERNAL_FUNCTION`](https://tanelpoder.com/2013/01/16/what-the-heck-is-the-internal_function-in-execution-plan-predicate-section/) converts the type of the `SALE_DATE` column. As a side effect it also prevents using a straight index on `DATE_COLUMN` *just as any other function would*.

## Numeric Strings

> Source: https://use-the-index-luke.com/sql/where-clause/obfuscation/numeric-strings

Numeric strings are numbers that are stored in text columns. Although it is a very bad practice, it does not automatically render an index useless if you consistently treat it as string:

```
SELECT ...
  FROM ...
 WHERE numeric_string = '42'
```

Of course this statement can use an index on `NUMERIC_STRING`. If you compare it using a number, however, the database can no longer use this condition as an [access predicate](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates#imp-index-predicate-types).

```
SELECT ...
  FROM ...
 WHERE numeric_string = 42
```

Note the missing quotes. Although some database yield an error (e.g. PostgreSQL) many databases just add an implicit type conversion.

```
SELECT ...
  FROM ...
 WHERE CAST(numeric_string AS INT) = 42
```

It is the same problem as before. An index on `NUMERIC_STRING` cannot be used due to the function call. The solution is also the same as before: do not convert the table column, instead convert the search term.

```
SELECT ...
  FROM ...
 WHERE numeric_string = CAST(42 AS VARCHAR(10))
```

You might wonder why the database does not do it this way automatically? I think it is because converting a string to a number always gives an unambiguous result. This is not true the other way around. A number, formatted as text, can contain spaces, punctation, and leading zeros. A single value can be written in many ways:

```
42
042
0042
00042
...
```

The database cannot know the number format used in the `NUMERIC_STRING` column so it does it the other way around: the database converts the strings to numbers—this is an unambiguous transformation.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-numeric-strings&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

The `CAST AS VARCHAR` expression returns only one string representation of the number. It will therefore only match the first of above listed strings. If we use `CAST AS INT`, it matches all of them. That means there is not only a performance difference between the two variants but also a semantic difference!

BigQuery 2024-12-18Db2 (LUW) 12.1.2MariaDB 12.0.2MySQL 9.6.0Oracle DB 23.26.1PostgreSQL 18SQL Server 2025SQLite 3.50.0[No implicit conversion: syntax error](https://use-the-index-luke.com/sql/where-clause/obfuscation/numeric-strings/quiz.numeric-strings.syntax-error.html)[CAST(numeric\_string AS INT) = 42](https://use-the-index-luke.com/sql/where-clause/obfuscation/numeric-strings/quiz.numeric-strings.fs.html)[numeric\_string = CAST(42 AS VARCHAR…)](https://use-the-index-luke.com/sql/where-clause/obfuscation/numeric-strings/quiz.numeric-strings.irs.html)

Using numeric strings is generally troublesome: most importantly it causes performance problems due to the implicit conversion and also introduces a risk of running into conversion errors due to invalid numbers. Even the most trivial query that does not use any functions in the `where` clause can cause an abort with a conversion error if there is just one invalid number stored in the table.

#### Tip

Use numeric types to store numbers.

Note that the problem does not exist the other way around:

```
SELECT ...
  FROM ...
 WHERE numeric_number = '42'
```

The database will consistently transform the string into a number. It does not apply a function on the potentially indexed column: a regular index will therefore work. Nevertheless it is possible to do a manual conversion the wrong way:

```
SELECT ...
  FROM ...
 WHERE TO_CHAR(numeric_number) = '42'
```

## Combining Columns

> Source: https://use-the-index-luke.com/sql/where-clause/obfuscation/concatenation

This section is about a popular obfuscation that affects [concatenated indexes](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys).

The first example is again about [date and time](https://use-the-index-luke.com/sql/where-clause/obfuscation/dates) types but the other way around. The following MySQL query combines a date and a time column to apply a range filter on both of them.

```
SELECT ...
  FROM ...
 WHERE ADDTIME(date_column, time_column)
     > DATE_ADD(now(), INTERVAL -1 DAY)
```

It selects all records from the last 24 hours. The query cannot use a concatenated index on (`DATE_COLUMN`, `TIME_COLUMN`) properly because the search is not done on the indexed columns but on derived data.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-obf-concat&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

You can avoid this problem by using a data type that has both a date and time component (e.g., MySQL `DATETIME`). You can then use this column without a function call:

```
SELECT ...
  FROM ...
 WHERE datetime_column
     > DATE_ADD(now(), INTERVAL -1 DAY)
```

Unfortunately it is often not possible to change the table when facing this problem.

The next option is a [function-based index](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) if the database supports it—although this has all the drawbacks [discussed before](https://use-the-index-luke.com/sql/where-clause/obfuscation/dates). When using MySQL, function-based indexes are not an option anyway.

It is still possible to write the query so that the database can use a concatenated index on `DATE_COLUMN`, `TIME_COLUMN` with an [access predicate](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates#imp-index-predicate-types)—at least partially. For that, we add an extra condition on the `DATE_COLUMN`.

```
 WHERE ADDTIME(date_column, time_column)
     > DATE_ADD(now(), INTERVAL -1 DAY)
   AND date_column
    >= DATE(DATE_ADD(now(), INTERVAL -1 DAY))
```

The new condition is absolutely redundant but it is a straight filter on `DATE_COLUMN` that can be used as access predicate. Even though this technique is not perfect, it is usually a good enough approximation.

#### Tip

Use a redundant condition on the most significant column when a range condition combines multiple columns.

For PostgreSQL, it’s preferable to use the [row values syntax](https://use-the-index-luke.com/sql/partial-results/fetch-next-page#ch07-paging-row-values-example).

You can also use this technique when storing date and time in text columns, but you have to use date and time formats that yields a chronological order when sorted lexically—e.g., as suggested by [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) (`YYYY-MM-DD HH:MM:SS`). The following example uses the Oracle database’s `TO_CHAR` function for that purpose:

```
SELECT ...
  FROM ...
 WHERE date_string || time_string
     > TO_CHAR(sysdate - 1, 'YYYY-MM-DD HH24:MI:SS')
   AND date_string
    >= TO_CHAR(sysdate - 1, 'YYYY-MM-DD')
```

We will face the problem of applying a range condition over multiple columns again in the section entitled [“*Paging Through Results*”](https://use-the-index-luke.com/sql/partial-results/fetch-next-page). We’ll also use the same approximation method to mitigate it.

Sometimes we have the reverse case and might want to obfuscate a condition intentionally so it cannot be used anymore as access predicate. We already looked at that problem when discussing the effects of [bind parameters](https://use-the-index-luke.com/sql/where-clause/bind-parameters) on `LIKE` conditions. Consider the following example:

```
SELECT last_name, first_name, employee_id
  FROM employees
 WHERE subsidiary_id = ?
   AND last_name LIKE ?
```

Assuming there is an index on `SUBSIDIARY_ID` and another one on `LAST_NAME`, which one is better for this query?

Without knowing the wildcard’s position in the search term, it is impossible to give a qualified answer. The optimizer has no other choice than to “guess”. If *you know* that there is always a leading wild card, you can obfuscate the `LIKE` condition intentionally so that the optimizer can no longer consider the index on `LAST_NAME`.

```
SELECT last_name, first_name, employee_id
  FROM employees
 WHERE subsidiary_id = ?
   AND last_name || '' LIKE ?
```

It is enough to append an empty string to the `LAST_NAME` column. This is, however, an option of last resort. Only do it when absolutely necessary.

## Smart Logic

> Source: https://use-the-index-luke.com/sql/where-clause/obfuscation/smart-logic

One of the key features of SQL databases is their support for ad-hoc queries: new queries can be executed at any time. This is only possible because the [query optimizer](https://use-the-index-luke.com/sql/where-clause/the-equals-operator/slow-indexes-part-ii#sb-optimizer) (query planner) works at runtime; it analyzes each statement when received and generates a reasonable execution plan immediately. The overhead introduced by runtime optimization can be minimized with [bind parameters](https://use-the-index-luke.com/sql/where-clause/bind-parameters).

The gist of that recap is that databases are optimized for dynamic SQL—so use it if you need it.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-smart-logic&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

Nevertheless there is a widely used practice that avoids dynamic SQL in favor of static SQL—often because of the “[dynamic SQL is slow](https://use-the-index-luke.com/sql/myth-directory/dynamic-sql-is-slow)” myth. This practice does more harm than good if the database uses a shared execution plan cache like Db2 (LUW), the Oracle database, or SQL Server.

For the sake of demonstration, imagine an application that queries the `EMPLOYEES` table. The application allows searching for subsidiary id, employee id and last name (case-insensitive) in any combination. It is still possible to write a single query that covers all cases by using “smart” logic.

```
SELECT first_name, last_name, subsidiary_id, employee_id
  FROM employees
 WHERE ( subsidiary_id    = :sub_id OR :sub_id IS NULL )
   AND ( employee_id      = :emp_id OR :emp_id IS NULL )
   AND ( UPPER(last_name) = :name   OR :name   IS NULL )
```

The query uses [named bind variables](https://use-the-index-luke.com/sql/where-clause/bind-parameters) for better readability. All possible filter expressions are statically coded in the statement. Whenever a filter isn’t needed, you just use `NULL` instead of a search term: it disables the condition via the `OR` logic.

It is a perfectly reasonable SQL statement. The use of `NULL` is even in line with its [definition according to the three-valued logic of SQL](https://use-the-index-luke.com/sql/where-clause/null). Nevertheless it is one of the *worst performance anti-patterns* of all.

The database cannot optimize the execution plan for a particular filter because any of them could be canceled out at runtime. The database needs to prepare for the worst case—if all filters are disabled:

```
----------------------------------------------------
| Id | Operation         | Name      | Rows | Cost |
----------------------------------------------------
|  0 | SELECT STATEMENT  |           |    2 |  478 |
|* 1 |  TABLE ACCESS FULL| EMPLOYEES |    2 |  478 |
----------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
1 - filter((:NAME   IS NULL OR UPPER("LAST_NAME")=:NAME) 
       AND (:EMP_ID IS NULL OR "EMPLOYEE_ID"=:EMP_ID) 
       AND (:SUB_ID IS NULL OR "SUBSIDIARY_ID"=:SUB_ID))
```

As a consequence, the database uses a full table scan *even if there is an index for each column*.

It is not that the database cannot resolve the “smart” logic. It creates the generic execution plan due to the use of bind parameters so it can be cached and re-used with other values later on. If we do not use [bind parameters](https://use-the-index-luke.com/sql/where-clause/bind-parameters) but write the actual values in the SQL statement, the optimizer selects the proper index for the active filter:

```
SELECT first_name, last_name, subsidiary_id, employee_id
  FROM employees
 WHERE( subsidiary_id    = NULL     OR NULL IS NULL )
   AND( employee_id      = NULL     OR NULL IS NULL )
   AND( UPPER(last_name) = 'WINAND' OR 'WINAND' IS NULL )
```

```
---------------------------------------------------------------
|Id | Operation                   | Name        | Rows | Cost |
---------------------------------------------------------------
| 0 | SELECT STATEMENT            |             |    1 |    2 |
| 1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES   |    1 |    2 |
|*2 |   INDEX RANGE SCAN          | EMP_UP_NAME |    1 |    1 |
---------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  2 - access(UPPER("LAST_NAME")='WINAND')
```

This, however, is no solution. It just proves that the database can resolve these conditions.

#### Warning

Using literal values makes your application vulnerable to [SQL injection](https://en.wikipedia.org/wiki/SQL_injection) attacks and can cause performance problems due to increased optimization overhead.

The obvious solution for dynamic queries is dynamic SQL. According to the [KISS principle](https://en.wikipedia.org/wiki/KISS_principle), just tell the database what you need right now—and nothing else.

```
SELECT first_name, last_name, subsidiary_id, employee_id
  FROM employees
 WHERE UPPER(last_name) = :name
```

Note that the query uses a bind parameter.

#### Tip

Use dynamic SQL if you need dynamic `where` clauses.

Still use bind parameters when generating dynamic SQL—otherwise the “[dynamic SQL is slow](https://use-the-index-luke.com/sql/myth-directory/dynamic-sql-is-slow)” myth comes true.

The problem described in this section is widespread. All databases that use a shared execution plan cache have a feature to cope with it—often introducing new problems and bugs.

Db2 (LUW)
:   Db2 uses a shared execution plan cache and is fully exposed to the problem described in this section.

    Db2 allows to specify the [re-optimization approach](https://www.ibm.com/docs/en/db2/11.5.x?topic=commands-bind) using the `REOPT` hint. The default is `NONE`, which produces a generic execution plan and suffers from the problem described above. `REOPT(ALWAYS)` will tell the optimizer to always peek the actual bind variables to produce the best plan for each execution. That is effectively turning off execution plan caching for that statement.

    The last option is `REOPT(ONCE)` which will peek the bind parameters for the first execution only. The problem with this approach is its nondeterministic behavior: the values from the first execution affect all executions. The execution plan can change whenever the database is restarted or, less predictably, the cached plan expires and the optimizer recreates it using different values the next time the statement is executed.

MySQL
:   MySQL does not suffer from this particular problem because it has no execution plan cache at all . A [feature request from 2009](https://bugs.mysql.com/bug.php?id=42808) discusses the impact of execution plan caching. It seems that MySQL’s optimizer is simple enough so that execution plan caching does not pay off.

Oracle
:   The Oracle database uses a shared execution plan cache (“SQL area”) and is fully exposed to the problem described in this section.

    Oracle introduced the so-called *bind peeking* with release 9*i*. Bind peeking enables the optimizer to use the actual bind values of the first execution when preparing an execution plan. The problem with this approach is its nondeterministic behavior: the values from the first execution affect all executions. The execution plan can change whenever the database is restarted or, less predictably, the cached plan expires and the optimizer recreates it using different values the next time the statement is executed.

    Release 11*g* introduced *adaptive cursor sharing* to further improve the situation. This feature allows the database to cache multiple execution plans for the same SQL statement. Further, the optimizer peeks the bind parameters and stores their estimated selectivity along with the execution plan. When the cache is subsequently accessed, the selectivity of the current bind values must fall within the selectivity ranges of a cached execution plan to be reused. Otherwise the optimizer creates a new execution plan and compares it against the already cached execution plans for this query. If there is already such an execution plan, the database replaces it with a new execution plan that also covers the selectivity estimates of the current bind values. If not, it caches a new execution plan variant for this query — along with the selectivity estimates, of course.

PostgreSQL
:   The PostgreSQL query plan cache works for open statements only—that is as long as you keep the `PreparedStatement` open. The above described problem occurs only when re-using a statement handle. Note that PostgreSQL’s JDBC driver enables the cache after the fifth execution only. See also: [Planning with Actual Bind Values](https://use-the-index-luke.com/sql/explain-plan/postgres/concrete-planning).

SQL Server
:   SQL Server uses so-called *parameter sniffing*. Parameter sniffing enables the optimizer to use the actual bind values of the first execution during parsing. The problem with this approach is its nondeterministic behavior: the values from the first execution affect all executions. The execution plan can change whenever the database is restarted or, less predictably, the cached plan expires and the optimizer recreates it using different values the next time the statement is executed.

    SQL Server provides a query hints to gain more control over parameter sniffing and recompiling. The [query hint](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-query?view=sql-server-ver16) `RECOMPILE` bypasses the plan cache for a selected statement. `OPTIMIZE FOR` allows the specification of actual parameter values that are used for optimization only. Finally, you can provide an entire execution plan with the `USE PLAN` hint.

    However, these features come a long way as there were several bugs and surprising behavior in special cases. The description of these is way beyond the scope of this book but luckily [Erland Sommarskog maintains all the relevant information up to SQL Server 2022](https://www.sommarskog.se/dyn-search-2008.html).

Although heuristic methods can improve the “smart logic” problem to a certain extent, they were actually built to deal with the problems of bind parameter in connection with column histograms and `LIKE` expressions.

The most reliable method for arriving at the best execution plan is to avoid unnecessary filters in the SQL statement.

#### See Also

[Using Bind-Variables - Examples](https://use-the-index-luke.com/sql/where-clause/bind-parameters#samples_bind_parameters)

[Building DynamicSQL using ORM Tools - Examples](https://use-the-index-luke.com/sql/myth-directory/dynamic-sql-is-slow#myth-dynamic-sql-sample)

## Math

> Source: https://use-the-index-luke.com/sql/where-clause/obfuscation/math

There is one more class of obfuscations that is smart and prevents proper index usage. Instead of using logic expressions it is using a calculation.

Consider the following statement. Can it use an index on `NUMERIC_NUMBER`?

```
SELECT numeric_number
  FROM table_name
 WHERE numeric_number - 1000 > ?
```

Similarly, can the following statement use an index on `A` and `B`—you choose the order?

```
SELECT a, b
  FROM table_name
 WHERE 3*a + 5 = b
```

Let’s put these questions into a different perspective; if you were developing an SQL database, would you add an equation solver? Most database vendors just say “No!” and thus, neither of the two examples uses the index.

#### If you like this page, you might also like …

… to [subscribe my **mailing lists**](https://winand.at/lists), [get **free stickers**](https://use-the-index-luke.com/shop), [buy **my book**](https://sql-performance-explained.com/?utm_source=use-the-index-luke.com&utm_campaign=sec-obf-math&utm_medium=web) or [join a **training**](https://winand.at/sql-training/open-online-class).

You can even use math to obfuscate a condition intentionally—[as we did it previously for the full text `LIKE` search](https://use-the-index-luke.com/sql/where-clause/obfuscation/concatenation). It is enough to add zero, for example:

```
SELECT numeric_number
  FROM table_name
 WHERE numeric_number + 0 = ?
```

Nevertheless we can index these expressions with a [function-based index](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) if we use calculations in a smart way and transform the `where` clause like an equation:

```
SELECT a, b
  FROM table_name
 WHERE 3*a - b = -5
```

We just moved the table references to the one side and the constants to the other. We can then create a function-based index for the left hand side of the equation:

```
CREATE INDEX math ON table_name (3*a - b)
```
