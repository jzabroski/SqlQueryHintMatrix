# SqlQueryHintMatrix
Ontological framework for use by ORM Provider APIs like EntityFrameworkCore

> I looked for a query hint matrix online thinking someone must have done this before, but couldn't find much, especially for lock control. And after looking across the database engines with providers in EF Core, I came away thinking it's not really possible to put a useful query hint feature into EF Core at the "Relational" level. Even NOLOCK isn't universally supported.

# PostgreSQL
The PostgreSQL people seem to think [very differently about query tuning](https://www.postgresql.org/message-id/flat/14821.1149104747@sss.pgh.pa.us) than SQL Server engineers. The PostgreSQL people do not hint at the "relational level", but rather believe in tuning 
[Planner Cost Constants](https://www.postgresql.org/docs/9.3/static/runtime-config-query.html#RUNTIME-CONFIG-QUERY-CONSTANTS). In effect, their approach is a lot like financial engineering and making a decision based on a "mark-to-market hurdle adjustment".  They state a model factor, such as cpu_index_tuple_cost, and bump it up until it disables a decision tree in the Cost-Based Optimizer/Query Planner.  For example, setting random_page_cost and the seq_page_cost both to 0 emulates a query where all the table data is already mmap'd (pinned to memory). In this sense, PostgreSQL simply thinks about locking differently than SQL Server:

1. In the negative sense, all these *_cost parameters are ultimately a function of page size. SQL Server DBA's use relation-level hints effectively to describe a formula: table_cost = table_pages * page_cost where page_cost = Tuple(num_pages, read_write_skew, read_write_volatility) . This does produce interesting idioms where you can set random_page_cost and the seq_page_cost both to 0 when you are certain something should be hot in the cache, vs. not setting those if it's cold.
2. In the positive sense, most places I've worked people just slap NOLOCK everywhere, so PostgreSQL is nicer in that it doesn't let engineers pollute the actual SQL with tall fescue shading the core business logic; most Microsoft engineers I've met don't know they can simply SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED for the same effect.

# MySQL
Here is a link to MySQL's Optimization documentation, including specific links for [8.11.2 Table Locking Issues](https://dev.mysql.com/doc/refman/8.0/en/table-locking.html) and [8.9 Controlling the Query Optimizer](https://dev.mysql.com/doc/refman/8.0/en/controlling-optimizer.html). MySQL seems fairly similar to SQL Server, especially [8.9.4 Index Hints](https://dev.mysql.com/doc/refman/8.0/en/index-hints.html):

> ```
> tbl_name [[AS] alias] [index_hint_list]
> 
> index_hint_list:
>     index_hint [index_hint] ...
> 
> index_hint:
>     USE {INDEX|KEY}
>       [FOR {JOIN|ORDER BY|GROUP BY}] ([index_list])
>   | IGNORE {INDEX|KEY}
>       [FOR {JOIN|ORDER BY|GROUP BY}] (index_list)
>   | FORCE {INDEX|KEY}
>       [FOR {JOIN|ORDER BY|GROUP BY}] (index_list)
> 
> index_list:
>     index_name [, index_name] ...
> ```

# Oracle
The latest documentation is available here: [Oracle Release 18c: Influencing the Optimizer](https://docs.oracle.com/en/database/oracle/oracle-database/18/tgsql/influencing-the-optimizer.html)

Does anyone know how far back Oracle releases are supported right now?

Here is a table from Donald Burleson (famous Oracle guy), summarizing the various hints:

## Oracle SQL Hints Tuning
Hints are expressed in special comments that start with +.
```/*+ <hint_expression> */```
and
```--+ <hint_expression>```
are both valid ways to express a hint.

### General hints:
| Hint | Meaning |
| ----- | --------- |
|ALL_ROWS | Use the cost based approach for best throughput.|
|CHOOSE | Default, if statistics are available will use cost, if not, rule.|
|FIRST_ROWS | Use the cost based approach for best response time.|
|RULE | Use rules based approach; this cancels any other hints specified for this statement.|

### Access Method Oracle Hints:
| Hint | Meaning |
| ----- | --------- |
|CLUSTER(table) | This tells Oracle to do a cluster scan to access the table.|
|FULL(table) | This tells the optimizer to do a full scan of the specified table.|
|HASH(table) | Tells Oracle to explicitly choose the hash access method for the table.|
|HASH_AJ(table) | Transforms a NOT IN subquery to a hash anti-join.|
|ROWID(table) | Forces a rowid scan of the specified table.|
|INDEX(table [index]) | Forces an index scan of the specified table using the specified index(s). If a list of indexes is specified, the optimizer chooses the one with the lowest cost. If no index is specified then the optimizer chooses the available index for the table with the lowest cost.|
|INDEX_ASC (table [index]) | Same as INDEX only performs an ascending search of the index chosen, this is functionally identical to the INDEX statement.|
|INDEX_DESC(table [index]) | Same as INDEX except performs a descending search. If more than one table is accessed, this is ignored.|
|INDEX_COMBINE(table index) | Combines the bitmapped indexes on the table if the cost shows that to do so would give better performance.|
|INDEX_FFS(table index) | Perform a fast full index scan rather than a table scan.|
|MERGE_AJ (table) | Transforms a NOT IN subquery into a merge anti-join.|
|AND_EQUAL(table index index [index index index]) | This hint causes a merge on several single column indexes. Two must be specified, five can be.|
|NL_AJ | Transforms a NOT IN subquery into a NL anti-join (nested loop).|
|HASH_SJ(t1, t2) | Inserted into the EXISTS subquery; This converts the subquery into a special type of hash join between t1 and t2 that preserves the semantics of the subquery. That is, even if there is more than one matching row in t2 for a row in t1, the row in t1 is returned only once.|
|MERGE_SJ (t1, t2) | Inserted into the EXISTS subquery; This converts the subquery into a special type of merge join between t1 and t2 that preserves the semantics of the subquery. That is, even if there is more than one matching row in t2 for a row in t1, the row in t1 is returned only once.|
|NL_SJ | Inserted into the EXISTS subquery; This converts the subquery into a special type of nested loop join between t1 and t2 that preserves the semantics of the subquery. That is, even if there is more than one matching row in t2 for a row in t1, the row in t1 is returned only once.|

### Oracle Hints for join orders and transformations:
| Hint | Meaning |
| ----- | --------- |
|ORDERED | This hint forces tables to be joined in the order specified. If you know table X has fewer rows, then ordering it first may speed execution in a join.|
|STAR | Forces the largest table to be joined last using a nested loops join on the index.|
|STAR_TRANSFORMATION | Makes the optimizer use the best plan in which a start transformation is used.|
|FACT(table) | When performing a star transformation use the specified table as a fact table.|
|NO_FACT(table) | When performing a star transformation do not use the specified table as a fact table.|
|PUSH_SUBQ | This causes nonmerged subqueries to be evaluated at the earliest possible point in the execution plan.|
|REWRITE(mview) | If possible forces the query to use the specified materialized view, if no materialized view is specified, the system chooses what it calculates is the appropriate view.
|NOREWRITE | Turns off query rewrite for the statement, use it for when data returned must be concurrent and can't come from a materialized view.|
|USE_CONCAT | Forces combined OR conditions and IN processing in the WHERE clause to be transformed into a compound query using the UNION ALL set operator.|
|NO_MERGE (table) | This causes Oracle to join each specified table with another row source without a sort-merge join.|
|NO_EXPAND | Prevents OR and IN processing expansion.
Oracle Hints for Join Operations: |  
|USE_HASH (table) | This causes Oracle to join each specified table with another row source with a hash join.|
|USE_NL(table) | This operation forces a nested loop using the specified table as the controlling table.|
|USE_MERGE(table,[table, - ]) | This operation forces a sort-merge-join operation of the specified tables.|
|DRIVING_SITE | The hint forces query execution to be done at a different site than that selected by Oracle. This hint can be used with either rule-based or cost-based optimization.|
|LEADING(table) | The hint causes Oracle to use the specified table as the first table in the join order.
Oracle Hints for Parallel Operations: |  
|[NO]APPEND | This specifies that data is to be or not to be appended to the end of a file rather than into existing free space. Use only with INSERT commands.|
|NOPARALLEL (table | This specifies the operation is not to be done in parallel.|
|PARALLEL(table, instances) | This specifies the operation is to be done in parallel.|
|PARALLEL_INDEX | Allows parallelization of a fast full index scan on any index.|

### Other Oracle Hints
| Hint | Meaning |
| ----- | --------- |
|CACHE | Specifies that the blocks retrieved for the table in the hint are placed at the most recently used end of the LRU list when the table is full table scanned.|
|NOCACHE | Specifies that the blocks retrieved for the table in the hint are placed at the least recently used end of the LRU list when the table is full table scanned.|
|[NO]APPEND | For insert operations will append (or not append) data at the HWM of table.|
|UNNEST | Turns on the UNNEST_SUBQUERY option for statement if UNNEST_SUBQUERY parameter is set to FALSE.|
|NO_UNNEST | Turns off the UNNEST_SUBQUERY option for statement if UNNEST_SUBQUERY parameter is set to TRUE.|
|PUSH_PRED | Pushes the join predicate into the view.|

## Sybase Adaptive Enterprise 12.1
[Adaptive Server Enterprise 12.5.1 > Performance and Tuning: Optimizer and Abstract Plans](http://infocenter.sybase.com/help/index.jsp?topic=/com.sybase.dc20023_1251/html/optimizer/optimizer1.htm)

## CosmosDB
[Performance tips for Azure Cosmos DB and .NET > SDK Usage > Tuning parallel queries for partitioned collections](https://docs.microsoft.com/en-us/azure/cosmos-db/performance-tips#sdk-usage)

## DB2
This doesn't come from IBM documentation but from a Gold-level partner named Craig S. Mullins. I chose this over the docs because the docs are extremely full of IBM-speak and I don't speak IBM-speak.
lol @ how complex/obtuse this is (ho-lee cow!):
[Influencing the DB2 Optimizer: Part 1](https://db2portal.blogspot.com/2015/07/influencing-db2-optimizer-part-1.html)
[Influencing the DB2 Optimizer: Part 2 - Standard Methods](https://db2portal.blogspot.com/2015/07/influencing-db2-optimizer-part-2.html)
[Influencing the DB2 Optimizer: Part 3 - Tweaking SQL Statements](https://db2portal.blogspot.com/2015/07/influencing-db2-optimizer-part-3.html)
lol @ this, too:
> At times, you may need to disable a specific index from being considered by the optimizer. One method of achieving this is to append OR 0 = 1 to the predicate. For example, consider a query against the EMP table on which two indexes exist: one on EMPNO and one on WORKDEPT. Appending OR 0 = 1 (as shown next) to the WORKDEPT predicate will cause DB2 to avoid using an index on WORKDEPT.
> 
> ```
> SELECT  EMPNO, WORKDEPT, EDLEVEL, SALARY
> FROM    EMP
> WHERE   EMPNO BETWEEN ‘000020’ AND ‘000350’
> AND     (WORKDEPT > ‘A01’ OR 0 = 1);
> ```
> The `OR 0 = 1` clause does not change the results of the query, but it can change the access path chosen. 
> 
> There are other tweaks that can be used to dissuade the DB2 optimizer from choosing a specific index, such as multiplying or dividing by 1, adding or subtracting 0, or appending an empty string to a character column. 

[Influencing the DB2 Optimizer: Part 4 - OPTIMIZE FOR n ROWS](https://db2portal.blogspot.com/2015/07/influencing-db2-optimizer-part-4.html)
[Influencing the DB2 Optimizer: Part 5 - Changing DB2 Catalog Statistics](https://db2portal.blogspot.com/2015/08/influencing-db2-optimizer-part-5.html)
[Influencing the DB2 Optimizer: Part 6 - Using Optimization Hints](https://db2portal.blogspot.com/2015/08/influencing-db2-optimizer-part-6-using.html)
