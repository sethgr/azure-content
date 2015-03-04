#Connecting Azure SQL Database to Azure Search Using Indexers#

Azure Search service makes it easy to provide a great search experience, but before you can search, you need to index the data. If the data lives in an Azure SQL database, the new **Azure Search indexer for Azure SQL Database** (or **Azure SQL indexer** for short) feature in Azure Search service can automate the indexing process. This means you have less code to write and less infrastructure to maintain.

Currently, indexers only work with Azure SQL Database, SQL Server on Azure VMs, and Azure DocumentDB. In this article, we’ll focus on indexers that work with Azure SQL Database. If you would like to see support for additional data sources, please provide your feedback on the [Azure Search feedback forum](http://feedback.azure.com/forums/263029-azure-search).

This article will cover the mechanics of using indexers, but we’ll also drill down into features and behaviors that are only available with SQL databases (for example, integrated change tracking).

## Indexers and Data Sources ##

To set up and configure an Azure SQL indexer, you can call the [Azure Search REST API](http://go.microsoft.com/fwlink/p/?LinkID=528173) to create and manage **indexers** and **data sources**. In the future, this functionality will also be available in Azure management portal and in our client SDKs. 

A **data source** specifies which data to index, credentials needed to access the data, and policies that enable Azure Search to efficiently identify changes in the data (new, modified or deleted rows). It's defined as an independent resource so that it can be used by multiple indexers.

An **indexer** is a resource that connects data sources with target search indexes. An indexer is used in the following ways:
 
- Perform a one-time copy of the data to populate an index.
- Sync an index with changes in the data source on a schedule.
- Run on-demand to update an index as needed. 

## When to Use Azure SQL Indexer ##

Depending on several factors relating to your data, the use of Azure SQL indexer may or may not be appropriate. If your data fits the following requirements, you can use Azure SQL indexer: 

- All the data comes from a single table or view
	- If the data is scattered across multiple tables, you can create a view and use that view with the indexer. However, be aware that if you use a view, you won’t be able to use SQL Server integrated change detection. See this section for more details. 
- The data types used in the data source are supported by the indexer. Most but not all of the SQL types are supported. For details, see [Mapping data types in Azure Search](http://go.microsoft.com/fwlink/p/?LinkID=528105). 
- You don’t need near real-time updates to the index when a row changes. 
	- The indexer can re-index your table at most every 5 minutes. If your data changes frequently and the changes need to be reflected in the index within seconds or single minutes, we recommend using Azure Search indexing API directly. 
- If you have a large data set and plan to run the indexer on a schedule, your schema allows to us to efficiently identify changed (and deleted, if applicable) rows. For more details, see Capturing Changed and Deleted Rows section below. 
- The size of the indexed fields in a row doesn’t exceed the maximum size of Azure Search indexing request, which is 16 MB. 

## Create and Use an Azure SQL Indexer ##

First, create the data source: 

	POST https://myservice.search.windows.net/datasources?api-version=2015-02-28 
	Content-Type: application/json
	api-key: admin-key
	
	{ 
	    "name" : "myazuresqldatasource",
	    "type" : "azuresql",
	    "credentials" : { "connectionString" : "connection string" },
	    "container" : { "name" : "table or view name" }
	}

Then, create the target Azure Search index if you don’t have one already. You can do this from the [portal UI](https://portal.azure.com) or by using the [Create Index API](https://msdn.microsoft.com/en-us/library/azure/dn798941.aspx).  Ensure that the schema of your target index is compatible with the schema of the source table. For details, see [Mapping data types in Azure Search](http://go.microsoft.com/fwlink/p/?LinkID=528105).

Finally, create the indexer by giving it a name and referencing the data source and target index:

	POST https://myservice.search.windows.net/indexers?api-version=2015-02-28 
	Content-Type: application/json
	api-key: admin-key
	
	{ 
	    "name" : "myindexer",
	    "dataSourceName" : "myazuresqldatasource",
	    "targetIndexName" : "target index name"
	}

An indexer created in this way doesn’t have a schedule. It automatically runs once as soon as it’s created. You can run it again at any time using a **run indexer** request:

	POST https://myservice.search.windows.net/indexers/myindexer/run?api-version=2015-02-28 
	api-key: admin-key
 
You may need to allow Azure services to connect to your database. See [Connecting From Azure](https://msdn.microsoft.com/en-us/library/azure/ee621782.aspx#ConnectingFromAzure) for instructions on how to do that.

To monitor the indexer status and execution history (number of items indexed, failures, etc.), use an **indexer status** request: 

	GET https://myservice.search.windows.net/indexers/myindexer/status?api-version=2015-02-28 
	api-key: admin-key

If there are no errors, the response should look similar to the following: 

TODO

Execution history contains up to 50 of the most recent completed executions, which are sorted in the reverse chronological order (so that the latest execution comes first in the response). 
Additional information about the response can be found in [Get Indexer Status](http://go.microsoft.com/fwlink/p/?LinkId=528198)

## Run Indexers on a Schedule ##

You can also arrange the indexer to run periodically on a schedule. To do this, just add the **schedule** property when creating the indexer: 

	{ 
	    "name" : "myindexer",
	    "dataSourceName" : "myazuresqldatasource",
	    "targetIndexName" : "target index name",
	    "schedule" : { "interval" : "PT10M", "startTime" : "2015-01-01T00:00:00Z" }
	}

The **interval** can range from 5 minutes (`PT5M`) to 1 day (`P1D`). The optional **startTime** indicates when the scheduled executions should commence; if it is omitted, the current time will be used. This time can be in the past – in which case the first execution will be scheduled as if the indexer has been running continuously since the startTime.  

The interval refers to the time between the start of two consecutive indexer executions. Only one indexer execution can run at a time. If an indexer is already executing when the next one is supposed to start, the execution is skipped and resumes at the next interval, assuming no other indexer job is running.

Let’s consider an example to make this more concrete. Suppose we the following hourly schedule configured: 

	"schedule" : { "interval" : "PT1H", "startTime" : "2015-03-01T00:00:00Z" }

Here’s what happens: 

1. The first indexer execution starts at or around March 1, 2015 12:00 a.m. UTC.
1. Assume this execution takes 20 minutes (or any time less than 1 hour).
1. The second execution starts at or around March 1, 2015 1:00 a.m. 
1. Now suppose that this execution takes more than an hour (this would require a huge number of documents for this to actually occur, but it’s a useful illustration) – for example, 70 minutes – so that it completes around 2:10 a.m.
1. It’s now 2:00 a.m., time for the third execution to start. However, because the second execution from 1 a.m. is still running, the third execution is skipped. The third execution starts at 3 a.m.

You can add, change, or delete a schedule for an existing indexer by using a **PUT indexer** request. 

## Capturing New, Changed and Deleted Rows ##

If you’re using a schedule and your table contains a non-trivial number of rows, you should use a data change detection policy, so that the indexer can efficiently retrieve only the new or changed rows without having to re-index the entire table.

### SQL Integrated Change Tracking Policy ###

If your SQL database supports [change tracking](https://msdn.microsoft.com/en-us/library/bb933875.aspx), we recommend using **SQL Integrated Change Tracking Policy**. This policy enables the most efficient change tracking, and it also allows Azure Search to identify deleted rows without you having to add an explicit "soft delete" column to your table.

Integrated change tracking is supported starting with the following SQL Server database versions:
 
- SQL Server 2008 R2 and later, if you're using SQL Server on Azure VMs. 
- Azure SQL Database V12, if you're using Azure SQL Database.

When using SQL integrated change tracking policy, do not specify a separate data deletion detection policy - this policy has built-in support for identifying deleted rows.

This policy can only be used with tables; it cannot be used with views. You need to enable change tracking for the table you're using before you can use this policy. See [Enable and disable change tracking](https://msdn.microsoft.com/en-us/library/bb964713.aspx) for instructions. 

To use this policy, create or update your data source like this:
 
	{ 
	    "name" : "myazuresqldatasource",
	    "type" : "azuresql",
	    "credentials" : { "connectionString" : "connection string" },
	    "container" : { "name" : "table or view name" }, 
	    "dataChangeDetectionPolicy" : {
	       "@odata.type" : "#Microsoft.Azure.Search.SqlIntegratedChangeTrackingPolicy" 
	  }
	}

### High Water Mark Change Detection Policy ###

While the SQL Integrated Change Tracking policy is recommended, you won’t be able to use it if your data is in a view, or you’re using an older version of Azure SQL database. In such a case, consider using the high water mark policy. This policy can be used if your table contains a column that meets the following criteria:

- All inserts specify a value for the column. 
- All updates to an item also change the value of the column. 
- The value of this column increases with each change.
- Queries that use a `WHERE` clause similar to `WHERE [High Water Mark Column] > [Current High Water Mark Value]` can be executed efficiently.

For example, an indexed **rowversion** column is an ideal candidate for the high water mark column. 
To use this policy, create or update your data source like this: 

	{ 
	    "name" : "myazuresqldatasource",
	    "type" : "azuresql",
	    "credentials" : { "connectionString" : "connection string" },
	    "container" : { "name" : "table or view name" }, 
	    "dataChangeDetectionPolicy" : {
	       "@odata.type" : "#Microsoft.Azure.Search.HighWaterMarkChangeDetectionPolicy",
	       "highWaterMarkColumnName" : "[a row version or last_updated column name]" 
	  }
	}

### Soft Delete Column Deletion Detection Policy ###

When rows are deleted from the source table, you probably want to delete those rows from the search index as well. If you had used the SQL integrated change tracking policy, this would have been taken care of for you. However, the high water mark change tracking policy doesn’t help you with deleted rows. What to do? 

If the rows are physically removed from the table, you’re out of luck – there’s no way to infer the presence of records that no longer exist.  However, you can use the “soft-delete” technique to mark rows as logically deleted without removing them from the table by adding a column and marking rows as deleted using a marker value in that column.

When using the soft-delete technique, you can use the soft delete policy as follows: 

	{ 
	    …, 
	    "dataChangeDetectionPolicy" : {
	       "@odata.type" : "#Microsoft.Azure.Search.HighWaterMarkChangeDetectionPolicy",
	       "highWaterMarkColumnName" : "[a row version or last_updated column name]" 
	  },
	    "dataDeletionDetectionPolicy" : { 
	       "@odata.type" : "#Microsoft.Azure.Search.SoftDeleteColumnDeletionDetectionPolicy",
	       "softDeleteColumnName" : "[a column name]", 
	       "softDeleteMarkerValue" : "[the value that indicates that a row is deleted]" 
	   }
	}

Note that the **softDeleteMarkerValue** must be a string – use the string representation of your actual value. For example, if you have an integer column where deleted rows are marked with the value 1, use `"1"`; if you have a BIT column where deleted rows are marked with the Boolean true value, use `"True"`. 

## Customize Azure SQL Indexer ##
 
You can customize certain aspects of indexer behavior (for example, batch size, how many documents can be skipped before an indexer execution will be failed, etc.). For more details, see [Indexer API documentation](http://go.microsoft.com/fwlink/p/?LinkId=528173). 

## Frequently Asked Questions ##

**Q:** Can I use Azure SQL indexer with SQL databases running on IaaS VMs in Azure?

A: Yes, as long as you allow Azure services to connect to your database by opening appropriate ports.

**Q:** Can I use Azure SQL indexer with SQL databases running on-premises? 

A: We do not recommend or support this, as doing this would require you to open your databases to Internet traffic. 

**Q:** Can I use Azure SQL indexer with databases other than SQL Server running in IaaS on Azure? 

A: We don’t support this scenario, because we haven’t tested the indexer with any databases other than SQL Server.  

**Q:** Can I create multiple indexers running on a schedule? 

A: Yes. However, only one indexer can be running on one node at one time. If you need multiple indexers running concurrently, consider scaling up your search service to more than one search unit. 

**Q:** Does running an indexer affect my query workload? 

A: Yes. Indexer runs on one of the nodes in your search service, and that node’s resources are shared between indexing and serving query traffic and other API requests. If you run intensive indexing and query workloads and encounter a high rate of 503 errors or increasing response times, consider scaling up your search service. 


