# Perform cross-document transactional operations with the Azure Cosmos DB SQL API

Perform operations on multiple items in single logical units of work.

**Learning objectives**

After completing this module, you'll be able to:

* Create a transactional batch and review results
* Implement optimistic concurrency control for an operation

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Writing multi-document transactions in the programming language of your choice, rather than JavaScript, gives you power and flexibility. You can manage, version, and optimize your code using your team’s practices in the way you want without having to adapt to a language you may or may not want to use.

After completing this module, you'll be able to:

* Create a transactional batch and review results
* Implement optimistic concurrency control for an operation

---

# Create a transactional batch with the SDK

Consider a fictional scenario where two used bicycle accessories will be created in a container, a **worn saddle**, and a **rusty handlebar**. For simplicity's sake, they have short unique identifiers and category identifiers.

```cs
Product saddle = new("0120", "Worn Saddle", "accessories-used");
Product handlebar = new("012A", "Rusty Handlebar", "accessories-used");
```

For this example, we have also opted to use a public record definition.

```cs
public record Product(string id, string name, string categoryId);
```

The **Microsoft.Azure.Cosmos.Container** class has a **CreateTransactionalBatch** member method that will create a new instance of type **TransactionalBatch** that supports the fluent syntax. This batch and the fluent **CreateItem** methods can be used to build a two-step transaction that inserts two items within the same partition key value.

```cs
PartitionKey partitionKey = new ("accessories-used");
TransactionalBatch batch = container.CreateTransactionalBatch(partitionKey)
    .CreateItem<Product>(saddle)
    .CreateItem<Product>(handlebar);
```

To execute the batch, asynchronously invoke the **ExecuteAsync** method. Add a using statement to immediately ensure the object is disposed of correctly after the current application scope.

```cs
using TransactionalBatchResponse response = await batch.ExecuteAsync();
```

The transactional batch supports operations with the **same logical partition key**. Operations with different logical partition keys will fail. In the example below, the transactional batch will fail with a bad request due to having a different logical partition key.

```cs
Product saddle = new("0120", "Worn Saddle", "accessories-used");
Product handlebar = new("012C", "Pristine Handlebar", "accessories-new");
PartitionKey partitionKey = new ("accessories-used");
TransactionalBatch batch = container.CreateTransactionalBatch(partitionKey)
    .CreateItem<Product>(saddle)
    .CreateItem<Product>(handlebar);
```

Transactional batch also supports a wide variety of operations using the fluent syntax including, but not limited to:

| Method | Description |
|-----|-----|
| `CreateItemStream()` | Create item from existing stream |
| `DeleteItem()` | Delete an item |
| `ReadItem()` | Read an item |
| `ReplaceItem()` & `ReplaceItemStream()` | Update an existing item or stream |
| `UpsertItem()` & `UpsertItemStream()` | Create or update an existing item or stream based on the item's unique identifier |

---

# Review batch operation results with the SDK

The **TransactionalBatchResponse** class contains multiple members to interrogate the results of the batch operation. The **StatusCode** property, specifically, is of type HttpStatusCode and will return an **HTTP status** code giving easy to identify error message for common transaction failures.

```cs
response.StatusCode
```

The **IsSuccessStatusCode** property is a boolean property that you can use directly in your application code before extracting results.

```cs
if (batchResponse.IsSuccessStatusCode)
{
}
```

The **GetOperationResultAtIndex<>** generic method returns the individual deserialized item at the index you specify.

```cs
TransactionalBatchOperationResult<Product> result = response.GetOperationResultAtIndex<Product>(0);
Product firstProductResult = result.Resource;

TransactionalBatchOperationResult<Product> result = response.GetOperationResultAtIndex<Product>(1);
Product secondProductResult = result.Resource;
```

---

# Exercise: Batch multiple point operations together with the Azure Cosmos DB SQL API SDK

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/perform-cross-document-transactional-operations-azure-cosmos-db-sql-api/4-exercise-batch-multiple-point-operations-together-sdk)

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

# Implement optimistic concurrency control

Using the SDK to read an item and then update the same item in a subsequent operation carries some inherent risk. Another operation could potentially come in from a separate client and change the underlying document before the first client’s update operation is finalized. This conflict could create a “lost update” situation. Let’s illustrate this conflict with an example.

Here is a typical C# code example with a separate read and update operation.

```cs
string categoryId = "9603ca6c-9e28-4a02-9194-51cdb7fea816";
PartitionKey partitionKey = new (categoryId);

Product product = await container.ReadItemAsync<Product>("01AC0", partitionKey);

product.price = 50d;

await container.UpsertItemAsync<Product>(product, partitionKey);
```

Since read and write in this example are distinct operations, there is a latency between these operations. This latency is represented in this diagram as *n*.

![N latency between read and update](https://docs.microsoft.com/en-us/learn/wwl-data-ai/perform-cross-document-transactional-operations-azure-cosmos-db-sql-api/media/5-latency.png)

This latency can be as short as milliseconds or seconds in computer code but could still be catastrophic enough to lose potential updates. Some user-facing applications, where user input causes a longer latency between a read and update operation, can cause a much longer *n* value and a higher potential for lost updates. This issue can be resolved by implementing **optimistic concurrency control**.

```cs
ItemResponse<Product> response = await container.ReadItemAsync<Product>("01AC0", partitionKey);
```

Each item has an **ETag** value. This value is updated when the item is updated. You can retrieve the ETag value of the item by observing the response from the read operation of your request.

The **ItemResponse** class has an **ETag** property that contains the corresponding string value.

```cs
Product product = response.Resource;
string eTag = response.ETag;
```

To prevent lost updates, you can use the if-match rule to see if the **ETag** still matches the current **ETag** header of the item server-side as part of your update request.

```cs
ItemRequestOptions options = new ItemRequestOptions { IfMatchEtag = eTag };
await container.UpsertItemAsync<Product>(product, partitionKey, requestOptions: options);
```

The updates to the C# code only required minor changes to implement optimistic concurrency control to ensure that your update operations did not lose changes previously saved server-side by competing clients.

```cs
string categoryId = "9603ca6c-9e28-4a02-9194-51cdb7fea816";
PartitionKey partitionKey = new (categoryId);

ItemResponse<Product> response = await container.ReadItemAsync<Product>("01AC0", partitionKey);
Product product = response.Resource;
string eTag = response.ETag;

product.price = 50d;

ItemRequestOptions options = new ItemRequestOptions { IfMatchEtag = eTag };
await container.UpsertItemAsync<Product>(product, partitionKey, requestOptions: options);
```

---

# Summary

1. What method parameters are required to create a transaction batch using container.CreateTransactionBatch()?

- [x] Partition Key
> That's correct. The CreateTransactionalBatch() method has a single parameter of a partition key.
- [ ] Partition Key, List of Operations
- [ ] None

2. You are using the ItemRequestOptions class to configure a request to enable optimistic concurrency control. Which header should you use to check for an exact value of an _etag?

- [ ] SessionToken
- [ ] IfNoneMatchEtag
- [x] IfMatchETag
> That's correct. This header is used to match the exact value of an ETag.

3. Which property of the TransactionalBatchResponse returns the HTTP status code indicating success or failure of the transaction?

- [ ] ErrorMessage
- [x] StatusCode
> That's correct. This returns the HTTP status code of type HTTPStatusCode from the underlying communication infrastructure with specific information about why the operation may have failed.
- [ ] IsSuccessStatusCode

---

# Summary

In this module, you explored how to create a transactional batch using client-side code and perform multiple operations before committing the entire batch as a single logical unit of work.

Now that you have completed this module, you can:

* Create a transactional batch and review results
* Implement optimistic concurrency control in the Azure Cosmos DB SQL API
