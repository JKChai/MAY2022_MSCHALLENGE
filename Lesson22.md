# Implement integrated cache in Azure Cosmos DB SQL API

Implement, configure, and monitor integrated cache in Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Implement the integrated cache
* Configure integrated cache options

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Read-heavy workloads can run into situations where small request unit charges can add up in the aggregate and cause the developer team to write custom code to cache the response of read operations and queries. Implementing a client-side cache can become cumbersome quickly, and the integrated cache feature of Azure Cosmos DB is a solution to enable caching without having to write custom code. In this module, you will learn about the integrated cache, its configuration, and strategies to monitor it.

After completing this module, you'll be able to:

* Implement the integrated cache
* Configure integrated cache options

---

# Review workloads that benefit from the cache

Point operations and queries consume request units (RU/s) when they are performed. In a scenario with a read-heavy workload, you can find your application performing the same point-reads and issuing queries with identical filters many times. Typically, you would multiply the normalized RU cost of each operation and query by the number of times performed. With a sizeable read-heavy workload, this can aggregate to a high cost quickly.

As a developer, it can be tempting to write a custom cache client in code, but you have to consider multiple things:

* First, you must route all requests through your custom cache. You would be responsible for scaling out your cache compute to levels that can keep up with Azure Cosmos DB’s scale
* You will need to handle cache invalidation when items or updated or deleted
* You will also need to increase the complexity of operations that create one or more new items

For all of these reasons, an integrated in-memory cache in Azure Cosmos DB is a viable solution. As a developer, you will get the benefits of caching without the complexities of implementing the cache yourself.

For some workloads in Azure Cosmos DB, an integrated cache comes at a great benefit. These workloads include, but are not limited to:

* Workloads with far more read operations and queries than write operations
* Workloads that read large individual items multiple times
* Workloads that execute queries multiple times with a large amount of RU/s
* Workloads that have hot partition key[s] for read operations and queries

Workloads that consistently perform the same point read and query operations are ideal to use with the integrated cache. When using the integrated cache, you will only consume request units in the first operation or query. Subsequent requests, as long as the item is not stale, will not consume any request units if the data is retrieved from the cache.

---

# Enable integrated cache

Enabling the integrated cache is done in two primary steps:

* Creating a dedicated gateway in your Azure Cosmos DB SQL API account
* Updating your SDK code to use the gateway for requests

## Create a dedicated gateway

First, you must provision a dedicated gateway in your account. This action can be done using the portal and the **Dedicated Gateway** pane.

![Dedicated Gateway navigation open in Azure Cosmos DB blade](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-integrated-cache/media/3-dedicated-gateway.png)

As part of the provisioning process, you will be asked to configure the number of gateway instances and an SKU. These settings determine the number of nodes and the compute and memory size for each gateway node. The number of nodes and SKU can be modified later as the amount of data you need to cache increases.

![Dedicated Gateway configuration options for SKU and number of nodes](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-integrated-cache/media/3-dedicated-gateway-config-full.png#lightbox)

Once the new gateway is provisioned, you can get the connection string for the gateway.

> :grey_exclamation: Note
>
> The gateway connection string is a distinct connection string from the one used typically with an Azure Cosmos DB SQL API client.

![Keys pane with multiple connection strings both using and not using the dedicated gateway](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-integrated-cache/media/3-connection-string-full.png)

## Update .NET SDK code

For the .NET SDK client to use the integrated cache, you must make sure that three things are true:

* The client uses the dedicated gateway connection string instead of the typical connection string
* The client is configured to use `Gateway` mode instead of the default `Direct` connectivity mode
* The client’s consistency level must be set to `session` or `eventual`

First, ensure that the connection string is set to the dedicated gateway’s connection string. Typically, Azure Cosmos DB SQL API connection strings are in the format of `<cosmos-account-name>.documents.azure.com`. For the dedicated gateway, the connection string is in the structure of `<cosmos-account-name>.sqlx.cosmos.azure.com`.

```cs
string connectionString = "AccountEndpoint=https://<cosmos-account-name>.sqlx.cosmos.azure.com/;AccountKey=<cosmos-key>;";
```

> :grey_exclamation: Note
>
> For example, if your account name is **dp420** and your key is: **fDR2ci9QgkdkvERTQ==**, then the connection string would be: `AccountEndpoint=https://dp420.sqlx.cosmos.azure.com/;AccountKey=fDR2ci9QgkdkvERTQ==;`

Next, the .NET `CosmosClient` class must be configured using a `CosmosClientOptions` instance. By default, the .NET SDK uses the `Direct` connectivity mode. The client options object sets the connectivity mode to `Gateway`.

```cs
CosmosClientOptions options = new()
{
    ConnectionMode = ConnectionMode.Gateway
};

CosmosClient client = new (connectionString, options);
```

## Configure point read operations

To configure a point read operation to use the integrated cache, you must create an object of type `ItemRequestOptions`. In this object, you can manually set the `ConsistencyLevel` property to either `ConsistencyLevel.Session` or `ConsistencyLevel.Eventual`. You can then use the options variable in the `ReadItemAsync` method invocation.

```C#
string id = "9DB28F2B-ADC8-40A2-A677-B0AAFC32CAC8";
PartitionKey partitionKey = new ("56400CF3-446D-4C3F-B9B2-68286DA3BB99");

ItemResponse<Product> response = await container.ReadItemAsync<Product>(id, partitionKey, requestOptions: operationOptions);
```

To observe the RU usage, use the `RequestCharge` property of the response variable. The first invocation of this read operation will use the expected number of request units. In this example, that would be one RU for a point read operation. Subsequent requests will not use any request units as the data will be pulled from the cache until it expires.

```cs
Console.WriteLine($"Request charge:\t{response.RequestCharge:0.00} RU/s");
```

## Configure Queries

To configure a query to use the integrated cache, create an object of type `QueryRequestOptions`. In this object, you should also manually change the consistency level. Then, pass in the options variable to the `GetItemQueryIterator` method invocation.

```cs
string sql = "SELECT * FROM products";
QueryDefinition query = new(sql);

QueryRequestOptions queryOptions = new()
{
    ConsistencyLevel = ConsistencyLevel.Eventual
};

FeedIterator<Product> iterator = container.GetItemQueryIterator<Product>(query, requestOptions: queryOptions);
```

You can also observe the RU usage by getting the `RequestCharge` property of each `FeedResponse` object associated with each page of results. If you aggregate the request charges, you will get the total request charge for the entire query. Much like with point reads, the first query will use the typical number of request units. Any extra queries will use no request units until the data expires in the cache.

```cs
double totalRequestCharge = 0;
while(iterator.HasMoreResults)
{
    FeedResponse<Product> response = await iterator.ReadNextAsync();
    totalRequestCharge += response.RequestCharge;
    Console.WriteLine($"Request charge:\t\t{response.RequestCharge:0.00} RU/s");
}

Console.WriteLine($"Total request charge:\t{totalRequestCharge:0.00} RU/s");
```

---

# Configure cache staleness

By default, the cache will keep data for five minutes. This staleness window can be configured using the `MaxIntegratedCacheStaleness` property in the SDK.

For point read operations, set the `DedicatedGatewayRequestOptions` property of the `ItemRequestOptions` class to a new instance of the `DedicatedGatewayRequestOptions` class with the `MaxIntegratedCacheStaleness` property set to an appropriate timespan for your application. In this example, the staleness is configured to 15 minutes.

```cs
ItemRequestOptions operationOptions = new()
{
    ConsistencyLevel = ConsistencyLevel.Eventual,
    DedicatedGatewayRequestOptions = new() 
    { 
        MaxIntegratedCacheStaleness = TimeSpan.FromMinutes(15) 
    }
};
```

For query operations, perform the same configuration tasks in the `QueryRequestOptions` class instead. In this example, the cache staleness is only set to **120** seconds or **2** minutes.

```cs
QueryRequestOptions queryOptions = new()
{
    ConsistencyLevel = ConsistencyLevel.Eventual,
    DedicatedGatewayRequestOptions = new() 
    { 
        MaxIntegratedCacheStaleness = TimeSpan.FromSeconds(120) 
    }
};
```

---

# Knowledge Check

1. In the .NET SDK for Azure Cosmos DB SQL API, how should you configure the CosmosClientOptions.ConnectionMode property so that your application can use the integrated cache?

- [ ] Configure the connectivity mode to ConnectionMode.Direct
- [x] Configure the connectivity mode to ConnectionMode.Gateway
> That's correct. The gateway connectivity mode will correctly use the dedicated gateway and is part of an overall solution to enable the integrated cache.
- [ ] Leave the connectivity mode to the default value

2. Which property of the ItemRequestOptions class should you configure to manually adjust how long items will remain in the cache?

- [ ] `SessionToken`
- [ ] `ConsistencyLevel`
- [x] `DedicatedGatewayRequestOptions.MaxIntegratedCacheStaleness`
> That's correct. The MaxIntegratedCacheStaleness property will configure a TimeSpan that is used to determine how long items will remain in the cache.

---

# Summary

In this module, you implemented and configured the intergated cache feature of Azure Cosmos DB.

Now that you have completed this module, you can:

* Create a new dedicated gateway and integrated cache
* Configure the integrated cache


