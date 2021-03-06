---
title: Best practices for SQL on-demand (preview) in Azure Synapse Analytics
description: Recommendations and best practices you should know as you work with SQL on-demand (preview). 
services: synapse-analytics
author: filippopovic
manager: craigg
ms.service: synapse-analytics
ms.topic: conceptual
ms.subservice:
ms.date: 05/01/2020
ms.author: fipopovi
ms.reviewer: jrasnick
---

# Best practices for SQL on-demand (preview) in Azure Synapse Analytics

In this article, you'll find a collection of best practices for using SQL on-demand (preview). SQL on-demand is an additional resource within Azure Synapse Analytics.

## General considerations

SQL on-demand allows you to query files in your Azure storage accounts. It doesn't have local storage or ingestion capabilities. As such, all files that the query targets are external to SQL on-demand. Everything related to reading files from storage might have an impact on query performance.

## Colocate Azure Storage account and SQL on-demand

To minimize latency, colocate your Azure storage account and your SQL on-demand endpoint. Storage accounts and endpoints provisioned during workspace creation are located in the same region.

For optimal performance, if you access other storage accounts with SQL on-demand, make sure they are in the same region. If they aren't in the same region, there will be increased latency for the data's network transfer between the remote and endpoint's regions.

## Azure Storage throttling

Multiple applications and services may access your storage account. Storage throttling occurs when the combined IOPS or throughput generated by applications, services, and SQL on-demand workload exceed the limits of the storage account. As a result, you'll experience a significant negative effect on query performance.

Once throttling is detected, SQL on-demand has built-in handling of this scenario. SQL on-demand will make requests to storage at a slower pace until throttling is resolved.

> [!TIP]
> For optimal query execution, you shouldn't stress the storage account with other workloads during query execution.

## Prepare files for querying

If possible, you can prepare files for better performance:

- Convert CSV to Parquet - Parquet is columnar format. Since it's compressed, its file sizes are smaller than CSV files with the same data. SQL on-demand will need less time and storage requests to read it.
- If a query targets a single large file, you'll benefit from splitting it into multiple smaller files.
- Try keeping your CSV file size below 10 GB.
- It's better to have equally sized files for a single OPENROWSET path or an external table LOCATION.
- Partition your data by storing partitions to different folders or file names - check [use filename and filepath functions to target specific partitions](#use-fileinfo-and-filepath-functions-to-target-specific-partitions).

## Push wildcards to lower levels in path

You can use wildcards in your path to [query multiple files and folders](develop-storage-files-overview.md#query-multiple-files-or-folders). SQL on-demand lists files in your storage account starting from first * using storage API and eliminates files that do not match specified path. Reducing initial list of files can improve performance if there are many files that match specified path up to first wildcard.

## Use appropriate data types

Data types used in your query affects performance. You can get better performance if you: 

- Use the smallest data size that will accommodate the largest possible value.
  - If maximum character value length is 30 characters, use character data type of length 30.
  - If all character column values are of fixed size, use char or nchar. Otherwise, use varchar or nvarchar.
  - If maximum integer column value is 500, use smallint as it is smallest data type that can accommodate this value. You can find integer data type ranges [here](https://docs.microsoft.com/sql/t-sql/data-types/int-bigint-smallint-and-tinyint-transact-sql?view=sql-server-ver15).
- If possible, use varchar and char instead of nvarchar and nchar.
- Use integer-based data types if possible. Sort, join and group by operations are performed faster on integers than on characters data.
- If you are using schema inference, [check inferred data type](#check-inferred-data-types).

## Check inferred data types

[Schema inference](query-parquet-files.md#automatic-schema-inference) helps you quickly write queries and explore data without knowing file schema. This comfort comes at expense of inferred data types being larger than they actually are. It happens when there is not enough information in source files to make sure appropriate data type is used. For example, Parquet files do not contain metadata about maximum character column length and SQL on-demand infers it as varchar(8000). 

You can check resulting data types of your query using [sp_describe_first_results_set](https://docs.microsoft.com/sql/relational-databases/system-stored-procedures/sp-describe-first-result-set-transact-sql?view=sql-server-ver15).

The following example shows how you can optimize inferred data types. Procedure is used to show inferred data types. 
```sql  
EXEC sp_describe_first_result_set N'
	SELECT
        vendor_id, pickup_datetime, passenger_count
	FROM 
		OPENROWSET(
        	BULK ''https://sqlondemandstorage.blob.core.windows.net/parquet/taxi/*/*/*'',
	        FORMAT=''PARQUET''
    	) AS nyc';
```

Here is the result set.

|is_hidden|column_ordinal|name|system_type_name|max_length|
|----------------|---------------------|----------|--------------------|-------------------||
|0|1|vendor_id|varchar(8000)|8000|
|0|2|pickup_datetime|datetime2(7)|8|
|0|3|passenger_count|int|4|

Once we know inferred data types for query we can specify appropriate data types:

```sql  
SELECT
    vendor_id, pickup_datetime, passenger_count
FROM 
	OPENROWSET(
		BULK 'https://sqlondemandstorage.blob.core.windows.net/parquet/taxi/*/*/*',
		FORMAT='PARQUET'
    ) 
	WITH (
		vendor_id varchar(4), -- we used length of 4 instead of inferred 8000
		pickup_datetime datetime2,
		passenger_count int
	) AS nyc;
```

## Use fileinfo and filepath functions to target specific partitions

Data is often organized in partitions. You can instruct SQL on-demand to query particular folders and files. This function will reduce the number of files and amount of data the query needs to read and process. An added bonus is that you'll achieve better performance.

For more information, check [filename](develop-storage-files-overview.md#filename-function) and [filepath](develop-storage-files-overview.md#filepath-function) functions and examples on how to [query specific files](query-specific-files.md).

> [!TIP]
> Always cast result of filepath and fileinfo functions to appropriate data types. If you use character data types, make sure appropriate length is used.

If your stored data isn't partitioned, consider partitioning it so you can use these functions to optimize queries targeting those files. When [querying partitioned Spark tables](develop-storage-files-spark-tables.md) from SQL on-demand, the query will automatically target only the files needed.

## Use CETAS to enhance query performance and joins

[CETAS](develop-tables-cetas.md) is one of the most important features available in SQL on-demand. CETAS is a parallel operation that creates external table metadata and exports the SELECT query results to a set of files in your storage account.

You can use CETAS to store frequently used parts of queries, like joined reference tables, to a new set of files. Next, you can join to this single external table instead of repeating common joins in multiple queries.

As CETAS generates Parquet files, statistics will be automatically created when the first query targets this external table, resulting in improved performance.

## Next steps

Review the [Troubleshooting](../sql-data-warehouse/sql-data-warehouse-troubleshoot.md?toc=/azure/synapse-analytics/toc.json&bc=/azure/synapse-analytics/breadcrumb/toc.json) article for common issues and solutions. If you're working with SQL pool rather than SQL on-demand, please see the [Best Practices for SQL pool](best-practices-sql-pool.md) article for specific guidance.
