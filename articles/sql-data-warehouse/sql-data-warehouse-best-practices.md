<properties
   pageTitle="Best Practices for Azure SQL Data Warehouse | Microsoft Azure"
   description="Recommendations and best practices you should know as you develop solutions for Azure SQL Data Warehouse. These will help you be successful."
   services="sql-data-warehouse"
   documentationCenter="NA"
   authors="sonyam"
   manager="barbkess"
   editor=""/>

<tags
   ms.service="sql-data-warehouse"
   ms.devlang="NA"
   ms.topic="article"
   ms.tgt_pltfrm="NA"
   ms.workload="data-services"
   ms.date="05/02/2016"
   ms.author="sonyama;barbkess"/>

# SQL Data Warehouse Best Practices

This article is a collection of many best practices which will allow you to achieve the most optimal price/performance from your Azure SQL Data Warehouse.  Some of the concepts in this article are very basic and easy to explain, other concepts are more advanced and we just scratch the surface in this article.  The purpose of this article is to give you some basic guidance and to raise awareness of important things to consider as you build your data warehouse.  Each section will introduce you to a concept and then point you to more detailed articles which cover the concept in more depth.

If you are just getting started with Azure SQL Data Warehouse, do not let this article overwhelm you.  The sequence of the topics is mostly in the order of importance.  If you just focus on the first three at first, you'll be in good shape.  As you get more familiar and comfortable with using SQL Date Warehouse, come back and look at a few more topics.  It won't take long for everything to make sense.  It may seem overwhelming at first, but over time and with experience these topics will become basic to you.

## Reduce Cost with Pause and Scale
A key feature of SQL Data Warehouse is the ability to pause when you are not using it, which stops the billing of compute resources.  Another key feature is the ability to scale resources.  Pausing and Scaling can be done via the Azure Portal or through PowerShell commands.  Become familiar with these features as these features can greatly reduce the cost of your data warehouse when it is not in use.  If you always want your data warehouse accessible, you may want to consider scaling it down to the smallest size, a DW100 rather than pausing.

See also [Pause compute resources][], [Resume compute resources][], [Scale compute resources][]


## Drain Transactions Before Pausing or Scaling 
When you pause or scale your SQL Data Warehouse, behind the scenes your database instance is stopped.  This means that all in-flight queries will be canceled.  Canceling a simple SELECT query is a quick operation and has almost no impact to the time it takes to  pause or scale your instance.  However, transactional queries, which modify your data or the structure of the data, may not be able to stop quickly.  **Transactional queries must either complete in their entirety or rollback their changes.**  Rolling back the work completed by a transactional query can take as long, or even longer, than the original change the query was applying.  For example, if you cancel a query which was deleting rows and has already been running for an hour, it could take the system an hour to insert back the rows which were deleted.  If you run pause or scaling while transactions are in flight, your pause or scaling may seem to take a long time because pausing and scaling has to wait for the rollback to complete before it can proceed.

See also [Transactions in SQL Data Warehouse][], [Optimizing transactions for SQL Data Warehouse][]

## Maintain Statistics
Unlike SQL Server, which automatically detects and creates or updates statistics on columns which would benefit, SQL Data Warehouse requires manual maintenance of statistics.  While we do plan to change this in the future, for now you will want to maintain your statistics to ensure that the SQL Data Warehouse plans are optimized.  The plans created by the optimizer are only as good as the available statistics.  **Creating sampled statistics on every column is an easy way to get started with statistics.**  It's equally important to update statistics as significant changes happen to your data.  A conservative approach may be to update your statistics daily or after each load.  There are always trade-offs between performance and the cost to create and update statistics. If you find it is taking too long to maintain all of your statistics, you may want to try to be more selective about which columns have statistics or which columns need frequent updating.  For example, you might want to update date columns, where new values may be added, daily. **You will gain the most benefit by having statistics on columns involved in joins, columns used in the WHERE clause and columns found in GROUP BY.**

See also [Manage statistics in SQL Data Warehouse][], [CREATE STATISTICS (Transact-SQL)][], [UPDATE STATISTICS (Transact-SQL)][]

## Group INSERT statements into batches
A one-time load to a small table with an INSERT statement or even a periodic reload of a look-up may perform just fine for your needs with a statement like `INSERT INTO MyLookup VALUES (1, 'Type 1')`.  However, if you need to load thousands or millions of rows throughout the day, you might find that singleton INSERTS just can't keep up.  Instead, develop your processes so that they write to a file and and another process periodically comes along and loads this file.

See also [Insert (Transact-SQL)][]
 
## Use PolyBase to Load and Export Data Quickly
SQL Data Warehouse supports loading and exporting data through several tools including Azure Data Factory, PolyBase, and BCP.  For small amounts of data where performance isn't critical, any tool may be sufficient for your needs.   However, when you are loading or exporting large volumes of data or fast performance is needed, PolyBase is the best choice.  PolyBase is designed to leverage the MPP (Massively Parallel Processing) architecture of SQL Data Warehouse and will therefore load and export data magnitudes faster than any other tool.  PolyBase loads can be run using CTAS or INSERT INTO.  **Using CTAS will minimize transaction logging and the fastest way to load your data.**  Azure Data Factory also supports PolyBase loads.  PolyBase supports a variety of file formats including Gzip files.  **To maximize throughput when using gzip text files, break files up into 60 or more files to maximize parallelism of your load.**  For faster total throughput, consider loading data concurrently.

See also [Load data into SQL Data Warehouse][], [Guide for using PolyBase in SQL Data Warehouse][], [Azure SQL Data Warehouse loading patterns and strategies][], [Load Data with Azure Data Factory][], [Move data to and from Azure SQL Data Warehouse using Azure Data Factory][], [CREATE EXTERNAL FILE FORMAT (Transact-SQL)][], [Create Table As Select (CTAS) in SQL Data Warehouse][]

## Hash Distribute Large Tables
By default, tables are Round Robin distributed.  This makes it easy for users to get started creating tables without having to decide how their tables should be distributed.  Round Robin tables may perform well for some workloads, but often selecting a distribution column will perform much better.  The most common example of when a table distributed by a column will far outperform a Round Robin table is when two large fact tables are joined.  For example, if you have an orders table, which is distributed by order_id, and a transactions table, which is also distributed by order_id, when you join your orders table to your transactions table on order_id, this query becomes a pass-through query, which means we eliminate data movement operations.  Less steps mean a faster query.  Less data movement also makes for faster queries.  This explanation just scratches the surface. When loading a distributed table, be sure that your incoming data is not sorted on the distribution key as this will slow down your loads.  See the below links for much more details on how selecting a distribution column can improve performance as well as how to define a distributed table in the WITH clause of your CREATE TABLES statement.

See also [Hash distribution and its effect on query performance in SQL Data Warehouse][], [Choosing hash distributed table vs. round-robin distributed table][], [CREATE TABLE (Azure SQL Data Warehouse, Parallel Data Warehouse)][], [CREATE TABLE AS SELECT (Azure SQL Data Warehouse)][]

## Do Not Over Partition
While partitioning data can be very effective for maintaining your data through partition switching or optimizing scans by with partition elimination, having too many partitions can slow down your queries.  Often a high granularity partitioning strategy which may work well on SQL Server may not work well on SQL Data Warehouse.  Having too many partitions can also reduce the effectiveness of clustered columnstore indexes if each partition has fewer than 1 million rows.  Keep in mind that behind the scenes, SQL Data Warehouse partitions your data for you into 60 databases, so if you create a table with 100 partitions, this actually results in 6000 partitions.  Each workload is different so the best advice is to experiment with partitioning to see what works best for your workload.  Consider lower granularity than what may have worked for you in SQL Server.  For example, consider using weekly or monthly partitions rather than daily partitions.

See also [Table partitions in SQL Data Warehouse][]

## Minimize Transaction Sizes
INSERT, UPDATE, and DELETE statements run in a transaction and when they fail they must be rolled back.  To minimize the potential for a long rollback, minimize transaction sizes whenever possible.  This can be done by dividing INSERT, UPDATE, and DELETE statements into parts.  For example, if you have an INSERT which you expect to take 1 hour, if possible, break the INSERT up into 4 parts, which will each run in 15 minutes.  Leverage special Minimal Logging cases, like CTAS, TRUNCATE, DROP TABLE or INSERT to empty tables, to reduce rollback risk.  Another way to eliminate rollbacks is to use Metadata Only operations like partition switching for data management.  For example, rather than execute a DELETE statement to delete all rows in a table where the order_date was in October of 2001, you could partition your data monthly and then switch out the partition with data for an empty partition from another table (see ALTER TABLE examples).  For unpartitioned tables consider using a CTAS to write the data you want to keep in a table rather than using DELETE.  If a CTAS takes the same amount of time, it is a much safer operation to run as it has very minimal transaction logging and can be canceled quickly if needed.

See also [Transactions in SQL Data Warehouse][], [Optimizing transactions for SQL Data Warehouse][], [Table partitions in SQL Data Warehouse][], [TRUNCATE TABLE (Transact-SQL)][], [ALTER TABLE (Transact-SQL)][], [Create Table As Select (CTAS) in SQL Data Warehouse][]

## Use the Smallest Possible Column Size
When defining your DDL, using the smallest data type which will support your data will improve query performance.  This is especially important for CHAR and VARCHAR columns.  If the longest value in a column is 25 characters, then define your column as VARCHAR(25).  Avoid defining all character columns to a large default length.  In addition, define columns as VARCHAR when that is all that is needed rather than use NVARCHAR.

See also [CREATE TABLE (Azure SQL Data Warehouse, Parallel Data Warehouse)][]

## Use Temporary Heap Tables for Transient Data
When you are temporarily landing data on SQL Data Warehouse, you may find that using a heap table will make the overall process faster.  If you are loading data only to stage it before running more transformations, loading the table to heap table will be much faster than loading the data to a clustered columnstore table.  In addition, loading data to a temp table will also load much faster than loading a table to permanent storage.  Temporary tables start with a "#" and are only accessible by the session which created it, so they may not work in some scenarios.   Heap tables are defined in the WITH clause of a CREATE TABLE.  If you do use a temporary table, remember to create statistics on that temporary table too.

See also [Temporary tables in SQL Data Warehouse][], [CREATE TABLE (Azure SQL Data Warehouse, Parallel Data Warehouse)][], [CREATE TABLE AS SELECT (Azure SQL Data Warehouse)][]

## Optimize Clustered Columnstore Tables
Clustered columnstore indexes are one of the most efficient ways you can store your data in Azure SQL Data Warehouse.  By default, tables in SQL Data Warehouse are created as Clustered ColumnStore.  To get the best performance for queries on columnstore tables, having good segment quality is important.  When rows are written to columnstore tables under memory pressure, columnstore segment quality may suffer.  Segment quality can be measured by number of rows in a compressed Row Group.  See the section **Clustered Columnstore Segment Quality** in the [Troubleshooting][] for step by step instructions on detecting and improving segment quality for clustered columnstore tables.  Because getting good quality columnstore segments is fairly important, it's generally a good idea to create a special users ids just for loading which utilize a medium or large resource class.  The less DWUs you use, the larger the resource class you will want to assign to your loading user. 

Since columnstore tables generally won't push data into a compressed columnstore segment until there are more than 1 million rows per table and each SQL Data Warehouse table is partitioned into 60 tables, as a rule of thumb, columnstore tables won't benefit a query unless the table has more than 60 million rows.  For table with less than 60 million rows, it may not make any sense to have a columnstore index.  It also may not hurt.  Furthermore, if you partition your data, then you will want to consider that each partition will need to have 1 million rows to benefit from a clustered columnstore index.  If a table has 100 partitions, then it will need to have at least 6 billion rows to benefit from a clustered columns store (60 distributions * 100 partitions * 1 million rows).  If your table does not have 6 billion rows in this example, either reduce the number of partitions or consider using a heap table insted.  It also may be worth experimenting to see if better performance can be gained with a heap table with secondary indexes rather than a columnstore table.  Columnstore tables do not yet support secondary indexes.

When querying a columnstore table, queries will run faster if you select only the columns you need.  

See also [Troubleshooting][], [Manage columnstore indexes in Azure SQL Data Warehouse][], [Columnstore Indexes Guide][]

## Use Larger Resource Class to Improve Query Performance
SQL Data Warehouse uses resource groups as a way to allocate memory to queries.  Out of the box, all users are assigned to the small resource class which grants 100 MB of memory per distribution.  Since there is always 60 distributions and each distribution is given a minimum of 100 MB, system wide the total memory allocation is 6,000 MB, or just under 6 GB.  Certain queries, like large joins or loads to clustered columnstore tables, will benefit from larger memory allocations.  Some queries, like pure scans, will see no benefit.  On the flip side, utilizing larger resource classes impacts concurrency, so you will want to take this into consideration before moving all of your users to a large resource class.
 
See also [Concurrency and workload management in SQL Data Warehouse][]

## Use Smaller Resource Class to Increase Concurrency
If you are noticing that user queries seem to have a long delay, it could be that your users are running in larger resource classes and are consuming a lot of concurrency slots causing other queries to queue up.  To see if users queries are queued, run `SELECT * FROM sys.dm_pdw_waits` to see if any rows are returned.

See also [Concurrency and workload management in SQL Data Warehouse][], [sys.dm_pdw_waits (Transact-SQL)][]

## Use DMVs to Monitor and Optimize your Queries
SQL Data Warehouse has several DMVs which can be used to monitor query execution.  The monitoring article below walks through step-by-step instructions on how to look at the details of an executing query.  To quickly find queries in these DMVs, using the LABEL option with your queries can help.

See also [Monitor your workload using DMVs][], [Use labels to instrument queries in SQL Data Warehouse][], [OPTION Clause (Transact-SQL)][], [sys.dm_exec_sessions (Transact-SQL)][], [sys.dm_pdw_exec_requests (Transact-SQL)][], [sys.dm_pdw_request_steps (Transact-SQL)][], [sys.dm_pdw_sql_requests (Transact-SQL)][], [sys.dm_pdw_dms_workers (Transact-SQL)], [DBCC PDW_SHOWEXECUTIONPLAN (Transact-SQL)][], [sys.dm_pdw_waits (Transact-SQL)][]

## Other Resources
There are many places to look for information about how to use Azure SQL Data Warehouse.  This article is part of the Azure documentation and has lots of links to other Azure articles as well as MSDN articles.  We monitor all of these articles for your feedback and do frequent updates.  If you find an article useful, please let us know by answering the question "Was this page helpful?".  You can provide comments whether you answer Yes or No.  If you find an article helpful, but have feedback, please click Yes and add your comments on anything we can do to make the article better.  If you are not prompted with this question, it can always be found at the end of every Azure article and for MSDN articles use the "Any suggestions?" link in the top right corner of every MSDN page.  We value your feedback and do take action on most feedback.

If you have **feature suggestions** for SQL Data Warehouse, please use the [Azure SQL Data Warehouse Feedback][] page.  Adding your requests or up-voting other requests helps us prioritize features.

The [Azure SQL Data Warehouse MSDN Forum][] was create as a place for you to ask questions to other users and to the SQL Data Warehouse Product Group.  We actively monitor this forum to ensure that your question are answered either by another user or one of us.  If you prefer to ask your questions on Stack Overflow, we also have a [Azure SQL Data Warehouse Stack Overflow Forum][].

<!--Image references-->

<!--Article references-->
[create a support ticket]: sql-data-warehouse-get-started-create-support-ticket.md
[Concurrency and workload management in SQL Data Warehouse]: sql-data-warehouse-develop-concurrency.md
[Create Table As Select (CTAS) in SQL Data Warehouse]: sql-data-warehouse-develop-ctas.md
[Guide for using PolyBase in SQL Data Warehouse]: sql-data-warehouse-load-polybase-guide.md
[Hash distribution and its effect on query performance in SQL Data Warehouse]: sql-data-warehouse-develop-hash-distribution-key.md
[Load data into SQL Data Warehouse]: sql-data-warehouse-overview-load.md
[Load data with Azure Data Factory]: sql-data-warehouse-get-started-load-with-azure-data-factory.md
[Load data with bcp]: sql-data-warehouse-load-with-bcp.md
[Load data with PolyBase in SQL Data Warehouse]: sql-data-warehouse-get-started-load-with-polybase.md
[Manage columnstore indexes in Azure SQL Data Warehouse]: sql-data-warehouse-manage-columnstore-indexes.md
[Manage statistics in SQL Data Warehouse]: sql-data-warehouse-develop-statistics.md
[Monitor your workload using DMVs]: sql-data-warehouse-manage-monitor.md
[Move data to and from Azure SQL Data Warehouse using Azure Data Factory]: ../data-factory/data-factory-azure-sql-data-warehouse-connector.md
[Optimizing transactions for SQL Data Warehouse]: sql-data-warehouse-develop-best-practices-transactions.md
[Pause compute resources]: sql-data-warehouse-overview-scalability.md#pause-compute-bk
[Resume compute resources]: sql-data-warehouse-overview-scalability.md#resume-compute-bk
[Scale compute resources]: sql-data-warehouse-overview-scalability.md#scale-performance-bk
[Table design in SQL Data Warehouse]: sql-data-warehouse-develop-table-design.md
[Table partitions in SQL Data Warehouse]: sql-data-warehouse-develop-table-partitions.md
[Temporary tables in SQL Data Warehouse]: sql-data-warehouse-develop-temporary-tables.md
[Transactions in SQL Data Warehouse]: sql-data-warehouse-develop-transactions.md
[Troubleshooting]: sql-data-warehouse-troubleshoot.md
[Use labels to instrument queries in SQL Data Warehouse]: sql-data-warehouse-develop-label.md

<!--MSDN references-->
[ALTER TABLE (Transact-SQL)]: https://msdn.microsoft.com/library/ms190273.aspx
[Columnstore Indexes Guide]: https://msdn.microsoft.com/library/gg492088.aspx
[CREATE TABLE (Azure SQL Data Warehouse, Parallel Data Warehouse)]: https://msdn.microsoft.com/library/mt203953.aspx
[CREATE EXTERNAL FILE FORMAT (Transact-SQL)]: https://msdn.microsoft.com/library/dn935026.aspx
[CREATE TABLE AS SELECT (Azure SQL Data Warehouse)]: https://msdn.microsoft.com/library/mt204041.aspx
[CREATE STATISTICS (Transact-SQL)]: https://msdn.microsoft.com/library/ms188038.aspx
[DBCC PDW_SHOWEXECUTIONPLAN (Transact-SQL)]: https://msdn.microsoft.com/library/mt204017.aspx
[Insert (Transact-SQL)]: https://msdn.microsoft.com/library/ms174335.aspx
[OPTION Clause (Transact-SQL)]: https://msdn.microsoft.com/library/ms190322.aspx
[sys.dm_exec_sessions (Transact-SQL)]: https://msdn.microsoft.com/library/ms176013.aspx
[sys.dm_pdw_exec_requests (Transact-SQL)]: https://msdn.microsoft.com/library/mt203887.aspx
[sys.dm_pdw_request_steps (Transact-SQL)]: https://msdn.microsoft.com/library/mt203913.aspx
[sys.dm_pdw_sql_requests (Transact-SQL)]: https://msdn.microsoft.com/library/mt203889.aspx
[sys.dm_pdw_dms_workers (Transact-SQL)]: https://msdn.microsoft.com/library/mt203878.aspx
[sys.dm_pdw_waits (Transact-SQL)]: https://msdn.microsoft.com/library/mt203893.aspx
[TRUNCATE TABLE (Transact-SQL)]: https://msdn.microsoft.com/library/ms177570.aspx
[UPDATE STATISTICS (Transact-SQL)]: https://msdn.microsoft.com/library/ms187348.aspx

<!--Other Web references-->
[Choosing hash distributed table vs. round-robin distributed table]: https://blogs.msdn.microsoft.com/sqlcat/2015/08/11/choosing-hash-distributed-table-vs-round-robin-distributed-table-in-azure-sql-dw-service/
[Azure SQL Data Warehouse Feedback]: https://feedback.azure.com/forums/307516-sql-data-warehouse
[Azure SQL Data Warehouse MSDN Forum]: https://social.msdn.microsoft.com/Forums/sqlserver/home?forum=AzureSQLDataWarehouse
[Azure SQL Data Warehouse Stack Overflow Forum]:  http://stackoverflow.com/questions/tagged/azure-sqldw
[Azure SQL Data Warehouse loading patterns and strategies]: https://blogs.msdn.microsoft.com/sqlcat/2016/02/06/azure-sql-data-warehouse-loading-patterns-and-strategies
