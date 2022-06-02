# Process bulk data in Azure Cosmos DB SQL API

Perform bulk operations on Azure Cosmos DB in bulk from code using the SDK.

**Learning objectives**

After completing this module, you'll be able to:

* Use C# task asynchronous programming model and "bulk" support in the Azure Cosmos DB SQL API .NET SDK

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

If you’re using a high throughput database, it’s not uncommon for you to want to send a high volume of data to the database with as much throughput as possible. What if you wanted to send millions of overnight logs to your Azure Cosmos DB SQL API account? In this module, you will learn to use the bulk functionality in the .NET SDK to process multiple operations concurrently using classes you are already familiar with in C#.

After completing this module, you'll be able to:

* Use C# task asynchronous programming model and "bulk" support in the Azure Cosmos DB SQL API .NET SDK

---

# Create bulk operations with the SDK

Bulk execution must be enabled by creating a new instance of the **CosmosClientOptions** class and setting the **AllowBulkExecution** property of that instance to true.

```cs
CosmosClientOptions options = new () 
{ 
    AllowBulkExecution = true 
};
```

You can pass in this options instance as the last parameter to the **CosmosClient** constructor parameter. This options parameter can be used if you are using an endpoint and key pair.

```cs
CosmosClient client = new (endpoint, key, options);
```

This options parameter can also be used if you are using a connection string constructor.

```cs
CosmosClient client = new (connectionString, options);
```

For context, how do we usually perform a single “Create Item” operation? Here, we invoke the **CreateItemAsync** method, which returns a **Task**, which is in turn immediately invoked using the **await** keyword, so we never really handle the **Task** object. It’s just a syntax shortcut to make our code easier to read.

```cs
await container.CreateItemAsync<Product>(product, partitionKey);
```

Under the hood, we could handle the task objects and even add them to lists. Here is an example where we create two tasks for two “Create Item” operations that would create two products. We don’t start the tasks yet; we add them to a list to start them later.

```cs
List<Task> concurrentTasks = new List<Task>();

PartitionKey firstPartitionKey = new("some-value");
Task<ItemResponse<Product>> firstTask = container.CreateItemAsync<Product>(firstProduct, firstPartitionKey);
concurrentTasks.Add(firstTask);

PartitionKey secondPartitionKey = new("some-value");
Task<ItemResponse<Product>> secondTask = container.CreateItemAsync<Product>(secondProduct, secondPartitionKey);
concurrentTasks.Add(secondTask);
```

This code is ugly; we could make it relatively cleaner. Here, we have a method called **GetOurProductsFromSomeWhere** that generates **250,000** products. We then create a list of tasks. We can then use a clean C# foreach loop that most developers can understand quickly.

```cs
List<Product> productsToInsert = GetOurProductsFromSomeWhere();

List<Task> concurrentTasks = new List<Task>();

foreach(Product product in productsToInsert)
{
    concurrentTasks.Add(
        container.CreateItemAsync<Product>(
            product, 
            new PartitionKey(product.partitionKeyValue))
    );
}
```

For each product in the products list, add a task to create an item in our Azure Cosmos DB SQL API container. How much simpler can it get? Remember, nothing has happened yet. Even better, we haven’t written any Azure Cosmos DB-specific code yet other than the **container.CreateItemAsync** part. This is all C# code.

When we invoke **Task.WhenAll**, the SDK will kick in to create batches to group our operations by physical partition, then distribute the requests to run concurrently. Grouping operations greatly improves efficiency by reducing the number of back-end requests, and allowing batches to be dispatched to different physical partitions in parallel. It also reduces thread count on the client making it easier to consume more throughput that you could if done as individual operations using individual threads.

```cs
Task.WhenAll(concurrentTasks);
```

Once each batch is done, the SDK will translate the batches back to the results for the client-side application. This is seemless and transparent to the developer.

---

# Review bulk operation caveats

There are some caveats to consider when developing for bulk operations that you are different than designing for typical Azure Cosmos DB SQL API applications.

## Throughput consumption

The provisioned throughput in request units per second (RU/s) is higher than if the operations were executed individually. This increase should be considered as you evaluate total throughput requirements when measured against other operations that will happen concurrently.

## Latency impact

When the SDK is attempting to fill a batch and doesn’t quite have enough items, it will wait 100 milliseconds for more items. This wait can effect overall latency.

## Document size

The SDK automatically creates batches for optimization with a maximum of 2 Mb (or 100 operations). Smaller items can take advantage of this optimization, with oversized items having an inverse effect.

---

# Implement bulk best practices

While it is straightforward to add bulk support to your applications, there are a couple of best practices.

## Configure the partition key

You are not required to provide the partition key for many of the operations on the Container class; the SDK will determine it automatically from your class. However, this will add to your overhead in a bulk scenario and could create needless complexity. It’s a good practice to provide the partition key to the operation if you already have it.

## Use stream API in serialize-deserialize scenarios

If you are building an API, avoid unnecessary serialization and deserialization. For example, you are sometimes forced to deserialize and serialize going to and from some database platforms. With Azure Cosmos DB SQL API, you can use the Stream variants of common item operations to avoid unnecessary performance overhead. This is especially true when using the bulk features of the SDK.

## Configure worker task per partition key

If your items are already separated into logical partition keys, you can create a list of worker tasks per partition key. Each worker task in that list can then spawn child tasks for each operation within that logical partition key. This setup would de facto create a hierarchy of tasks which coordination of per item operations.

---

# Exercise: Move multiple documents in bulk with the Azure Cosmos DB SQL API SDK

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/process-bulk-data-azure-cosmos-db-sql-api/5-exercise-move-multiple-documents-bulk-sdk)

* Create new Azure Cosmos DB account using the SQL API
* Use Data Explorer to create a database, a container, and 2 items
* Query Database for the items created

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge Check

1. Which property of the CosmosClientOptions class enables the ability to run multiple operations concurrently and transparently at scale?

- [x] AllowBulkExecution
> That's correct. This property enables optimistic batching of requests for non-latency sensitive scenarios.
- [ ] ConnectionMode
- [ ] LimitToEndpoint

2. Which one of these statements is considered a best practice when working with the bulk features of the SDK?

- [ ] Stream APIs are not compatible with the bulk features of the SDK.
- [ ] Assume that bulk will save you on actual throughput costs versus individual operations.
- [x] Configure the partition key for each operation configured in a set of tasks.
> That's correct. Configuring the partition key for individual operations will save the SDK from having to compute the partition key on a per operation basis.

---

# Summary

In this module, you learned to implement parallel execution of multiple operations at massive scale for bulk operations of hundreds of thousands or even millions of concurrent operations. You can use this knowledge to perform jobs that can scale up to massive workloads that wouldn't normally be possible with typical means.

Now that you have completed this module, you can:

* Use the bulk SDK features integrated with the Task classes already implemented in the C# language
