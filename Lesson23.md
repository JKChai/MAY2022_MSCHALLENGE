# Measure performance in Azure Cosmos DB SQL API

We will learn to use Azure Monitor to create and analyze monitoring data for Azure Cosmos DB.

**Learning objectives**

After completing this module, you’ll be able to:

* Understand how Azure Cosmos DB uses Azure Monitor to monitor server-side metrics
* Measure Cosmos DB's throughput
* Observe rate-limiting events
* Query telemetry logs
* Measure cross-partition storage distribution throughput

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Azure Monitor is a full stack monitoring service in Azure that provides a complete set of features to monitor Azure resources. Azure Cosmos DB creates monitoring data using Azure Monitor. Azure Monitor captures Cosmos DB's metrics and telemetry data.

After completing this module, you'll be able to:

* Understand how Azure Cosmos DB uses Azure Monitor to monitor server-side metrics
* Measure Cosmos DB's throughput
* Observe rate-limiting events
* Query telemetry logs
* Measure cross-partition storage distribution throughput

---

# Understand Azure Monitor

Most applications using an Azure resource will create availability, performance, and operations metrics on both the application and Azure resource side. Azure Monitor is used to monitor the Azure resource availability, performance, and operations metrics.

Cosmos DB monitors its server-side counters using:

* Azure Monitor to monitor metrics: Azure Monitor collects Cosmos DB metrics by default. Metrics are collected every minute. The default retention period is 30 days. The collection includes throughput, storage availability, latency, consistency, and system level metrics. The dimension values for the metrics such as container name are case-insensitive.
* Azure Monitor to monitor diagnostic logs: Telemetries like events and traces are stored as logs. For example, changing the throughput properties of a container will be a logged event. Queries can then be run against these logs to analyze the data collected.
* The Azure Cosmos DB portal: The throughput, storage availability, latency, consistency, and system level metrics can be found under the Metrics tab of the Azure Cosmos DB account. The default retention period for these metrics is seven days.
* The Cosmos DB SQL API SDKs to programmatically monitor the account: Use the .NET, Java, Python, Node.js SDKs, and the headers in REST API to programmatically monitor a Cosmos DB account.

![Diagram that shows the options available to monitor Azure Cosmos DB.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/measure-performance-azure-cosmos-db-sql-api/media/2-monitor-cosmos-db.png)

In this module, the lesson will be limited to how Cosmos DB applies its monitoring using the Azure Monitor. Azure Monitor for Cosmos DB can be used to:

* Monitor data
* Collection and routing
* Analyze metrics
* Analyze logs
* Create Alerts
* Monitor Azure Cosmos DB Programmatically

## Monitor data

The Overview page in the Azure portal for each Azure Cosmos database includes a brief view including the requests and hourly billing for the database. This summary is just a small set of the metrics being collected by the Azure Monitor. Besides the hourly billing and request metrics that the Cosmos Database Overview page displays, Azure Monitor collects other request metrics plus request units, storage, latency, availability, and Casandra API metrics.

## Collection and routing

By default Azure Monitor collects and stores Cosmos DB metrics automatically. Azure Monitor can also route those metrics to other locations by using a diagnostic setting. Unlike metrics, Resource Logs aren't collected and stored without first creating a diagnostic setting to route them.

## Analyze metrics

To analyze Cosmos DB metrics, use the metrics explorer by opening Metrics from the Azure Monitor menu in the Azure portal. To filter out the Cosmos DB metrics, pick Cosmos DB standard metrics from the Metric Namespace pulldown. Other filters can be added for the collection name, database name, operation type, region, and status code dimensions.

## Analyze logs

Azure Monitor Logs data is stored into tables. Queries can be run against these tables to analyze their data. Azure Cosmos DB stores log data into the AzureDiagnostics and AzureActivity tables. To search the AzureDiagnostics table for Azure Cosmos DB entries, include a filter with the resourceprovider field equals to MICROSOFT.DOCUMENTDB in your queries. Additionally, Azure Cosmos DB also logs data to several resource-specific tables.

## Alerts

Azure Monitor can trigger alerts based on defined conditions. These alerts can be set on metrics, logs, and the activity log. For example, you can get an alert when a container or a database has exceeded the provisioned throughput limit.

## Monitor Azure Cosmos DB programmatically

The SQL APIs don't include account level metrics like storage usage and total requests. The SQL APIs however, provide collection level metrics either using the REST API or the .NET SDK.

---

# Measure throughput

Azure Monitor for Azure Cosmos DB provides the **Total Request Units** metric that can be used to analyze the request units consumed by the different Azure Cosmos DB operations. This metric can then be used to analyze those operations with the highest throughput.

Monitoring this metric, allows us to:

* Identify operations that are consuming more request units than others.
* Identify operations that are taking more cumulative request units in a given interval of time.

By identifying the operations with higher throughput, we can for example:

* Determine if these operations are insert and upserts, their index definition can be reviewed for over or under indexing-specific fields. We can then determine if we should include or exclude paths in their indexing policy.
* Modify the query to use and index with a filter clause.
* Use partition keys that will minimize the fan out of query into different partitions.
* If possible, evaluate if a smaller result set would meet the query needs.

## View the Total Request Unit metrics

To view the **Total Request Units** metric, under Azure Monitor's Metrics

1. Select the Resource Type **Azure Cosmos DB account** and **Apply** in the scope dialog.
2. Select the correct **Azure Cosmos DB account** from the drop-down list.
3. Under Metrics, select **Total Request Units** and the type of aggregation you need.
4. If needed, refine the **Time range** and **Time granularity** of the metric.

![Diagram that shows the options to monitor Total Request Units in Azure Cosmos DB.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/measure-performance-azure-cosmos-db-sql-api/media/3-monitor-total-request-units.png#lightbox)

## Filter the Total Request Units further

By default, Azure Monitor will display the overall throughput of all Azure Cosmos DB operations the selected account does. To better analyze the throughput, more granular filtering will be needed to find aggregate usage of the individual operation types or to further compare the usage of multiple operation types at the same time. Using the **Add filter** and **Apply splitting** options will help us with those analyses.

Azure Monitor allows us to filter further by specific **CollectionName**, **DatabaseName**, **OperationType**, **Region**, **Status**, and **StatusCode**. For example, we could add a filter by operation type to see the usage of our different Azure Cosmos DB operations.

![Diagram that shows the options to filter the monitoring of Total Request Units in Azure Cosmos DB.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/measure-performance-azure-cosmos-db-sql-api/media/3-monitor-total-request-units-filter.png)

---

# Observe rate-limiting events

In this unit, we'll discuss methods of how to identify rate-limiting events.

While using the Azure Cosmos DB SQL API, Azure Cosmos DB might return a **429** status code error. This error code, indicates that a Request rate too large exception has occurred. This exception means that Azure Cosmos DB requests are being rate limited.

When provisioned throughput is used, the request units per second (RU/s) is set for the workload. Operations (read, writes, queries) against the service consume request units(RUs). If in any given second, the operations consume more RUs than the provisioned RU/s, Azure Cosmos DB will return a **429** exception.

There are three main reasons why we get a **429** exception:

* Request rate is large.
* The request did not complete due to a high rate of metadata requests.
* The request did not complete due to a transient service error.

We'll look at each of these reasons and how to identify them.

## Request rate is large

Out of the three reasons for this exception, this reason is the most common one. Azure Cosmos DB returns this exception when the RUs by operations on data exceed the provisioned RU/s.

The first step to research a **429** exception, is that in the Azure portal under the Azure Cosmos DB account, under `Insights->Request`, review the Total Request by Status Code charts for occurrences of the exception. Further filter the charts by Time Range and Database to your desired time span and database. For most applications, it's normal to have less than 5% of the request with **429** exceptions.

![Diagram that shows the charts by Status code and Throttled Request (429 exceptions).](https://docs.microsoft.com/en-us/learn/wwl-data-ai/measure-performance-azure-cosmos-db-sql-api/media/4-monitor-429-exception.png)

If the percentage of **429** exceptions is higher than 5%, it's possible that the exceptions are caused by a hot partition.

To verify if the database access is coming across a hot partition, in the Azure portal under the Azure Cosmos DB account, under Insights->Throughput, review the Normalized RU Consumption (%) By PartitionKeyRangeID charts.

![Diagram that shows the charts by throughput of a hot partition.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/measure-performance-azure-cosmos-db-sql-api/media/4-monitor-hot-partition.png)

## Rate limiting on metadata requests

A **429** exception can occur when doing a high volume of the following metadata operations:

* Create, read, update, or delete a container or database
* List databases or containers in a Cosmos account
* Query the current provisioned throughput

To investigate if **429** exceptions are caused by Metadata requests, in the Azure portal under the Azure Cosmos DB account, under Insights->System, review the Metadata Requests That Exceeded Capacity (`429s`) charts.

![Diagram that shows the charts for metadata access.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/measure-performance-azure-cosmos-db-sql-api/media/4-monitor-metadata.png)

## Rate limiting due to transient service error

Around the time, you're noticing **429** exceptions happening, you might also notice **503** errors being reported in the `Insights->Request` Total Request by Status Code charts. These exceptions could indicate that the **429** exceptions are happening because of transient service errors.

---

# Query logs

Azure resources produce Azure Diagnostic Logs, which provide detailed operational data of those resources. Diagnostics settings are use to collect those resource logs.

While some logs like activity and platform metrics are collected automatically, diagnostic settings must be created to collect resource logs. These logs can be forwarded outside of Azure Monitor. Enabling diagnostics settings in Azure Cosmos DB accounts can be forwarded to Log Analytics workspaces, Event hubs, and Storage Accounts.

Forwarding data to Log Analytics workspaces, writes the logs into tables that can be queried using the `Kusto Query Language` (KQL). So, to use the diagnostic data stored in these tables, knowledge on reading and writing Kusto queries is essential. These tables can be a generic legacy table called `Azure Diagnostics`, or the recommended `Resource-specific` tables.

## Create Azure Cosmos DB diagnostics settings

There are multiple ways to create the diagnostics settings, the Azure portal, via REST API, PowerShell or via Azure CLI.

To create the diagnostics settings using the Azure portal, navigate to the Azure Cosmos DB account, and under the `Montoring` section, choose `Diagnostic settings`. Either edit an existing diagnostic setting or choose `+ Add diagnostic setting` and choose the logs you wish to collect and the destinations to forward these logs to.

![Diagram that shows the diagnostics settings options for Azure Cosmos DB.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/measure-performance-azure-cosmos-db-sql-api/media/5-monitor-diagnostics-settings.png)

The SQL API log tables are:

* `DataPlaneRequests` - This table logs back-end requests for operations that execute create, update, delete, or retrieve data.
* `QueryRuntimeStatistics` - This table logs query operations against the SQL API account.
* `PartitionKeyStatistics` - This table logs logical partition key statistics in estimated KB. It's helpful when troubleshooting skew storage.
* `PartitionKeyRUConsumption` - This table logs every second aggregated RU/s consumption of partition keys. It's helpful when troubleshooting hot partitions.
* `ControlPlaneRequests` - This table logs Azure Cosmos DB account control data, for example adding or removing regions in the replication settings.

## Troubleshoot issues with diagnostics queries

When Azure Cosmos DB diagnostics data is sent to Log Analytics, it's sent to either the `AzureDiagnostics` table or to `Resource-specific` tables. The preferred mode is to send the data to `Resource-specific` tables, as such, each log chosen under the diagnostics settings options will have its own table. Choosing this mode makes it easier to work with the diagnostic data, easier to discover the schemas used, and improve performance in latency and query times.

### AzureDiagnostics queries

If the legacy mode is chosen, the diagnostics data will be stored in the `AzureDiagnostics` table, so all `kusto` queries will be executed against that table. Since multiple Azure resources could also be populating this table, include the filter `ResourceProvider=="MICROSOFT.DOCUMENTDB"` in your `where` clause to only return Azure Cosmos DB entries. Additionally, to differentiate between the different logs you picked under diagnostic settings, add a filter on the `Category` column. For example, to return documents for the `QueryRuntimeStatistics` log, include the where clause `| where ResourceProvider=="MICROSOFT.DOCUMENTDB"` and `Category=="QueryRuntimeStatistics"`. `Kusto` is case-sensitive so make sure your column names are the right case. Let's review a couple of `Kusto` query examples using the `AzureDiagnostics` table.

* Query that returns the count and the total request charged of the different Azure Cosmos DB operation types in the last hour.

```kusto
AzureDiagnostics 
| where TimeGenerated >= ago(1h)
| where ResourceProvider=="MICROSOFT.DOCUMENTDB" and Category=="DataPlaneRequests" 
| summarize OperationCount = count(), TotalRequestCharged=sum(todouble(requestCharge_s)) by OperationName
| order by TotalRequestCharged desc
```

* Create a query that returns a timechart graph for all successful (status 200) and rate limited (status 429) request in the last hour. The requests will be aggregated every 10 minutes.

```kusto
AzureDiagnostics 
| where TimeGenerated >= ago(1h)
| where ResourceProvider=="MICROSOFT.DOCUMENTDB" and Category=="DataPlaneRequests" 
| summarize requestcount=count() by statusCode_s, bin(TimeGenerated, 10m)
| render timechart
```

## Resource-specific Queries

Unlike the `AzureDiagnostic` queries, the resource-specific queries will be run against the different tables that were created for each log category chosen in the diagnostic setting dialog. To use these tables, prefix the table names in the list above with the string `CDB`. Let's review aa couple of examples.

* Query that returns the count and the total request charged of the different Azure Cosmos DB operation types in the last hour.

```kusto
CDBDataPlaneRequests
| where TimeGenerated >= ago(1h)
| summarize OperationCount = count(), TotalRequestCharged=sum(todouble(RequestCharge)) by OperationName
| order by TotalRequestCharged desc
```

* Create a query that returns a timechart graph for all successful (status 200) and rate limited (status 429) request in the last hour.

```kusto
CDBDataPlaneRequests 
| where TimeGenerated >= ago(2h)
| summarize requestcount=count() by StatusCode, bin(TimeGenerated, 10m)
| render timechart
```

---

# Exercise: Use Azure Monitor to analyze an Azure Cosmos DB SQL API account

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/measure-performance-azure-cosmos-db-sql-api/6-exercise-use-azure-monitor-to-analyze-account)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge Check

1. Which Insight report will help you identify that your requests are getting request rate to large exceptions and that this condition is causing a problem to the databases or containers?

- [ ] Request->Total Requests by Operation Type
- [x] Request->Total Requests by Status Code
> That's correct. This report will allow us to compare the number of successful requests (200) against the number of rate-limiting ones (429). If relatively you're finding 1-5% of the request hit a 429 status, and high number of successful requests that might be normal for your application, but if we discover much then 5% of the requests are hitting 429 conditions, there might be a problem.
- [ ] Throughput->Normalized RU Consumption By

2. How would you query Azure Cosmos DB Log Analytics diagnostic logs tables using the Azure portal?

- [ ] Under Azure Cosmos DB account Data Explorer, run SQL Queries to query the tables.
- [ ] Under the Azure Cosmos DB account diagnostic setting defined Log Analytic workspace, use the Logs page, and run SQL Queries to query the tables.
- [x] Under the Azure Cosmos DB account diagnostic setting defined Log Analytic workspace, use the Logs page, and run KQL Queries to query the tables.
> That's correct. You would use the diagnostic setting defined Log Analytic workspace's Logs page, and you would run Kusto Query Language (KQL) queries to query the Log Analytics tables.

3. Which of the following Log Analytics tables will be the most helpful when troubleshooting hot partitions?

- [ ] PartitionKeyStatistics.
- [x] PartitionKeyRUConsumption.
> That's correct. This table logs every second aggregated RU/s consumption of partition keys. It's helpful when troubleshooting hot partitions.
- [ ] DataPlaneRequests.

---

# Summary

In this module, you learned how to use Azure Monitor to monitor metrics and telemetry data for Cosmos DB.

Now that you have completed this module, you can:

* Understand how Azure Cosmos DB uses Azure Monitor to monitor server-side metrics
* Measure Cosmos DB's throughput
* Observe rate-limiting events
* Query telemetry logs
* Measure cross-partition storage distribution throughput