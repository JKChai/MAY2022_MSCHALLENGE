# Monitor responses and events in Azure Cosmos DB SQL API

We will learn to use a rich set of REST response codes returned by Azure Cosmos DB request to help you analyze potential issues.

**Learning objectives**

* Review common response codes
* Understand transit errors
* Review rate-limiting errors
* Configure Alerts
* Audit Security

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

To help you monitor the current state of your request, Azure Cosmos DB offers a rich set of HTTP status codes. These codes are returned by the REST operations. In this module, we'll review some of the most common codes returned by Azure Cosmos DB.

After completing this module, you'll be able to:

* Review common response codes
* Understand transit errors
* Review rate-limiting errors
* Configure Alerts
* Audit Security

---

# Review common response codes

Most common request operations using the Azure Cosmos DB SQL API, will be to create, query, or manage container documents. Each request will return an HTTP status code on the status of the operation. This code might let us know if the operation was successful. Or the code will let us know the request was unsuccessful and provides us with some insight of what might have gone wrong. In this section we'll review some of the most common HTTP status codes returned by the following types of request:

* Create a Document
* List Documents
* Get a Document
* Replace a Document
* Patch a Document
* Delete a Document
* Query Documents

## Common Status codes for all types of operations

While some status codes like 400, 403 and 404 are share among different operation types, their description varies slightly and wont be listed in this table.

| Status Code | Name | Operation Type | Description |
|-----|-----|-----|-----|
| 200 | OK | List, Get, Replace, Patch, Query | The operation was successful. |

## Create a Document

The Create Document operation creates a new document in a collection. Its status codes are:

| Status Code | Operation Type | Description |
|-----|-----|-----|
| 201 | Created | The operation was successful. |
| 400 | Bad Request | The JSON body is invalid. |
| 403 | Forbidden | The operation couldn't be completed because the storage limit of the partition has been reached. |
| 409 | Conflict | The id provided for the new document has been taken by an existing document. |
| 413 | Entity Too Large | The document size in the request exceeded the allowable document size. |

## List documents under the collection using ReadFeed

ReadFeed can be used to retrieve all documents, or just the incremental changes to documents within the collection. Its status codes are:

| Status Code | Operation Type | Description |
|-----|-----|-----|
| 400 | Bad Request | The override set in x-ms-consistency-level is stronger than the one set during account creation. For example, if the consistency level is Session, the override can't be Strong or Bounded. |

## Get a Document

The Get Document operation retrieves a document by its partition key and document key. Its status codes are:

| Status Code | Operation Type | Description |
|-----|-----|-----|
| 304 | Not Modified | The document requested wasn't modified since the specified eTag value in the If-Match header. The service returns an empty response body. |
| 400 | Bad Request | The override set in the x-ms-consistency-level header is stronger than the one set during account creation. For example, if the consistency level is Session, the override can't be Strong or Bounded. |
| 404 | Not Found | The document is no longer a resource, that is, the document was deleted. |

## Replace a Document

The Replace Document operation replaces the entire content of a document. Its status codes are:

| Status Code | Operation Type | Description |
|-----|-----|-----|
| 400 | Bad Request | The JSON body is invalid. Check for missing curly brackets or quotes. |
| 404 | Not Found | The document no longer exists, that is, the document was deleted. |
| 409 | Conflict | The id provided for the new document has been taken by an existing document. |
| 413 | Entity Too Large | The document size in the request exceeded the allowable document size in a request. |

## Patch a Document

The Patch Document operation does path-level updates to specific files/properties in a single document. Its status codes are:

| Status Code | Operation Type | Description |
|-----|-----|-----|
| 400 | Bad Request | The JSON body is invalid. |
| 412 | Precondition Failed | The specified pre-condition isn't met. |

## Delete Document

The Delete Document operation deletes an existing document in a collection. Its status codes are:

| Status Code | Operation Type | Description |
|-----|-----|-----|
| 204 | No Content | The delete operation was successful. |
| 404 | Not Found | The document is not found. |

## Query Documents

You can query the collection documents using Azure Cosmos DB SQL queries. Its status codes are:

| Status Code | Operation Type | Description |
|-----|-----|-----|
| 400 | Bad Request | The specified request was specified with an incorrect SQL syntax, or missing required headers. |

## Other important status codes Azure Cosmos DB request could return

Some failed status codes are also references as exceptions. We'll discuss a couple of these status codes in more detail in the next sections, but here are a few more common status codes to review:

| Status Code | Operation Type | Description |
|-----|-----|-----|
| 408 | Request timeout | The operation did not complete within the allotted amount of time. This code is returned when a stored procedure, trigger, or UDF (within a query) does not complete execution within the maximum execution time. |
| 429 | Too many requests | The collection has exceeded the provisioned throughput limit. Retry the request after the server specified retry after duration. For more information, see request units. |
| 500 | Internal Server Error | The operation failed because of an unexpected service error. Contact support. |
| 503 | Service Unavailable | The operation couldn't be completed because the service was unavailable. This situation could happen because of network connectivity or service availability issues. It's safe to retry the operation. If the issue persists, contact support. |

---

# Understand transient errors

In this section, we'll diagnose and troubleshoot Azure Cosmos DB service unavailable exceptions. We usually can identify this exception when our request return status code 503. It means that the operations couldn't complete because the service was unavailable. There are several reasons why this exception could be raised. The status code could be returned because of network connectivity or service availability issues. In most cases, it's safe to retry the operation and the issue might have been resolved. If the issue persists, you will need to contact Azure support. Let's evaluate the three main cases when this status code would be returned.

## Required ports are blocked

Verify that the following ports are enabled for the SQL API.

| Connection mode | Supported protocol | Supported SDKs | API/Service port |
|-----|-----|-----|
| Gateway | HTTPS | All SDKs | SQL (443) |
| Direct | TCP | .NET SDK, Java SDK | When using public/service endpoints: ports in the 10000 through 20000 range. When using private endpoints: ports in the 0 through 65535 range |

## Client-side transient connectivity issues

This exception can happen when there are transient connectivity problems that are causing timeouts. The stack trace related to this scenario will contain a `TransportException` error. This error could look like:

```cs
TransportException: A client transport error occurred: The request timed out while waiting for a server response. 
(Time: xxx, activity ID: xxx, error code: ReceiveTimeout [0x0010], base error: HRESULT 0x80131500
```

This error should be troubleshot like a request timeout error (status code 408).

## Service Outage

Check the [Azure status](https://status.azure.com/status) page to see if there's an ongoing issue.

---

# Review rate-limiting errors

Request return status code **429** for the exception **request rate too large** status code. This status code indicates that your requests against Azure Cosmos DB are being rate-limited.

When provisioned throughput is used, the request units per second (RU/s) is set for the workload. Operations (read, writes, queries) against the service consume request units(RUs). If, in any given second, the operations consume more RUs than the provisioned RU/s, Azure Cosmos DB will return a 429 exception. Let's review the three different reasons why this exception is encountered.

## Request rate is large

Request rate is large is the most common reason for a 429 exception. Review occurrences of this exception in the Azure Cosmos DB Insight report **Total Request by Status Code** under the Request tab. Review what is the percentage of 429 exceptions occur vs. successful requests against your database.

While you could see 429 exceptions in your charts, notes that many applications are designed to retry one or more times if this type of exception is encountered. So it's possible your applications could be managing the throttling without returning any errors.

No action should be required, if your analysis of the chart determines that 1-5% of the workload requests are generating a 429 exception, and end-to-end latency is acceptable. This small percentage of exceptions is a healthy sign of RU/s utilization.

However, if the percentage of 429 exceptions is higher than 5%, it's possible the exceptions are caused by a hot partition. If a relative small number of logical partition keys consume a much larger amount of the total request units per second, that can create a hot partition. Hot partitions can cause 429 exception by not distributing the throughput across multiple partitions better.

To determine if we have a hot partition, review the Azure Cosmos DB insight report **Normalized RU Consumption (%) By PartitionKeyRangeID** under the *Throughput* tab. In this report, the *PartitionKeyRangeID* identify each physical partition. Any physical partition that is identified with a significant higher consumption on this chart, could be a hot partition. It's specially true if that physical partition stays at 100% constantly, and while the other physical partitions remain at much lower percentages all most of the time.

To determine which request types are causing the 429 exceptions, running a query under **Azure Diagnostic Logs** can return the RUs consumed by request type. The sample query below returns the average RUs per minute per operation for those operations with 429 exceptions.

```kusto
AzureDiagnostics
| where TimeGenerated >= ago(24h)
| where Category == "DataPlaneRequests"
| summarize throttledOperations = dcountif(activityId_g, statusCode_s == 429), totalOperations = dcount(activityId_g), totalConsumedRUPerMinute = sum(todouble(requestCharge_s)) by databaseName_s, collectionName_s, OperationName, requestResourceType_s, bin(TimeGenerated, 1min)
| extend averageRUPerOperation = 1.0 * totalConsumedRUPerMinute / totalOperations 
| extend fractionOf429s = 1.0 * throttledOperations / totalOperations
| order by fractionOf429s desc
```

Some possible solutions to this type of 429 exceptions:

* If it's determined that the 429 exceptions occur because of a hot partition, consider changing the partition key.
* If the exceptions aren't caused by a hot partition, increasing the RU/s on the container might be the solution.
* If the exceptions occur on query document requests, troubleshoot the queries with high RU charge.

## Rate-limiting on metadata requests

A high volume of metadata operations can cause 429 exceptions. Metadata operations are those operations who list, create, modify, or delete database or containers. They could also be operations like querying the current provisioned throughput.

Review occurrences of this type of 429 exception in the Azure Cosmos DB Insight report **Metadata Requests That Exceeded Capacity (429s)** under the System tab.

If this type of request causes 429 exceptions, increasing the provisioned RU/s isn't recommended. Increasing the provisioned RU/s won't have any impact on the occurrence of the exceptions. There's a system-reserve RU limit for metadata request.

Possible solutions for 429 exceptions caused by metadata request:

* Consider implementing a backoff policy to perform the metadata requests at a lower rate.
* Use a single DocumentClient instance for the lifetime of your application
* Cache the names of the databases and containers.

## Rate-limiting due to transient service error

If this type of request causes 429 exceptions, increasing the provisioned RU/s isn't recommended. Just increasing the provisioned RU/s won't have any impact on the occurrence of the exceptions. Retrying the request is the only recommended solution, if the exception persists, open a support ticket from the Azure portal. Transient service errors might also be reported around the same time you're getting 429 errors.

---

# Configure Alerts

Azure Cosmos DB uses the Azure Monitor Service to set up and send alerts. Alerts monitor the availability and responsiveness of Azure Cosmos DB resources and send notification when monitored metrics hit specified thresholds. Alerts can take the form of emails or even execute Azure Functions when they're triggered. Alerts also monitor the activity log events of your Azure Cosmos DB account.

Alerts can be set up from either your Azure Cosmos DB account page or from Azure Monitor. From both places, you'll set up the alerts in similar fashions.

## Setting up an alert

Let's take a look at an example of setting alerts when over one thousand **429** exceptions are triggered within 15 minutes. The alert should check every 5 minutes for the condition. Finally it should send an email to admins@contoso.com when the condition is met.

1. In your Azure Cosmos DB account page, under the Monitoring section choose Alerts.
2. Select `+ Create` and select `Alert rule` to create a new alert. You should see your current Azure Cosmos DB account, subscription, and Resource Group already selected.
3. Select `Add condition`. This condition will define the trigger for this alert.
    a. Time to pick the Signal type. Signals are either Metrics or Activity Logs. Since 429 exceptions can occur when requests are made, search for the signal name `Total Request Units`. We should see a graph that shows us the total request units in the last 6 hours.
    b. Currently if you add an Alert Logic, it will be measured against all the request units for this account. What you need is only to create a condition against the Requests that returned a status code of 429. To create that filter, under `Split by dimension` choose:
        i. Select `StatusCode` under the Dimension name pulldown.
        ii. Select `=` under Operator.
        iii. If a `429` exception has occurred within the last 6 hours, you could see it under the Dimension values options. If 429 isn't an option under Dimension values, Select `Add custom value` and add the value `429`. You could add extra filters like database, collection, region, or operation type if you needed an even more precise filter.
    c. Set the Alert Logic `Threshold` value to `1000`.
    d. Under Evaluated based on, set the `Aggregation granularity (Period)` to `15 minutes` and the `Frequency of Evaluation` to `5 minutes`.
    e. Select `Done` to complete the Condition setup.
4. The alert needs to know what to do when the condition is met. Let's send out the email. Under Actions, select `Add action group`.
    a. If you already had some action create, you could reuse it. We'll create a new Action, select + Create action group.
    b. Under the `Basic` tab:
        i. Give the Action group a name.
        ii. If needed, change the Display name.
    c. Under the Notification Tab:
        i. Choose **Email/SMS message/Push/Voice** under Notification type.
        ii. Give the Notification a Name.
        iii. Select the **pencil icon** to add the notification recipient.
            i. Select the Email checkbox.
            ii. Set the Email to admins@contoso.com and select OK.
    d. Select the Review + create button and then the Create button
5. Finally we need to fill out the alert's general information in the **Alert Rule details**. You can change any of the preselected options as needed, but you need to at least set the **Alert rule name**, so give the alert a name.
6. Select the **Create alert rule** button to create the alert.

Once the alert is created, it can take up to 10 minutes to activate.

## Common alerting scenarios

The following are some scenarios where you can use alerts:

* When the keys of an Azure Cosmos account are updated.
* When the data or index usage of a container, database, or a region exceeds a certain number of bytes.
* When the normalized RU/s consumption is greater than certain percentage.
* When a region is added, removed, or if it goes offline.
* When a database or a container is created, deleted, or updated.
* When the throughput of your database or the container is changed.

---

# Audit Security

There are several options we have to audit security on your Azure Cosmos DB Account.

## Activity logs

By using audit logging and activity logs, you can monitor your account for normal and abnormal activity. You can view what operations were done on your resources, who started the operation, when the operation occurred, the status of the operation.

Activity logs, which are automatically available, contain all write operations (PUT, POST, DELETE) for your Cosmos DB resources except read operations (GET). Activity logs can be used to find an error when troubleshooting or to monitor how a user in your organization modified a resource.

To view the Activity log, on the Azure Cosmos DB account page, select **Activity log**.

![Diagram that shows the Azure Cosmos DB account activity log.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/monitor-responses-events-azure-cosmos-db-sql-api/media/6-activity-log.png)

## Azure Resource Logs

Enable Azure resource logs for Cosmos DB. You can use Microsoft Defender for Cloud and Azure Policy to enable resource logs and log data collecting. These logs can be critical for investigating security incidents and doing forensic exercises.

Enable the auditing control plane under Diagnostics settings. You want to get an alert when the firewall rules for your Azure Cosmos account are modified. The alert is required to find unauthorized modifications to rules that govern the network security of your Azure Cosmos account and take quick action.

---

# Exercise: Troubleshoot an application using the Azure Cosmos DB SQL API SDK

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/monitor-responses-events-azure-cosmos-db-sql-api/7-exercise-troubleshoot-application-using-sdk)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge check

1. What is the status code for a transient error?

- [ ] 408
- [ ] 429
- [x] 503
> That's correct. Status code 503 is return on a transient error.

2. Which of the following tools will help you audit Azure Cosmos DB account security?

- [ ] Azure Cosmos DB Explorer.
- [ ] Insights.
- [x] Activity logs.
> That's correct. By using audit logging and activity logs, you can monitor your account for normal and abnormal activity.

3. What should you do if your request are being rate limited because of transient service errors?

- [ ] Increasing the provisioned RU/s.
- [x] Retry the request.
> That's correct. Retrying the request is the only recommended solution, if the exception persists, open a support ticket from the Azure portal.
- [ ] Cache the names of the databases and containers.

---

# Summary

In this module, you learned how to monitor and alert on response events using the Azure Cosmos DB SQL API.

Now that you have completed this module, you can:

* Review common response codes
* Understand transit errors
* Review rate-limiting errors
* Configure Alerts
* Audit Security
