# Consume an Azure Cosmos DB SQL API change feed using the SDK

Process change feed events using the change feed processor in the Azure Cosmos DB SQL API .NET SDK.

**Learning objectives**

After completing this module, you'll be able to:

* Create a change feed processor in the .NET SDK
* Author a delegate to handle a batch of changes in a client-side application

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

The change feed processor in the .NET SDK for Azure Cosmos DB SQL API is simple to implement, designed for scale-out, and can be hosted in various .NET environments. In this module, you will implement a change feed processor using a delegate in C# and read changes from a container.

After completing this module, you'll be able to:

* Create a change feed processor in the .NET SDK
* Author a delegate to handle a batch of changes in a client-side application

---

# Understand change feed features in the SDK

The .NET SDK for Azure Cosmos DB SQL API ships with a change feed processor that simplifies the task of reading changes from the feed. The change feed processor also natively supports distributed scenarios where event processing responsibilities are shared across multiple consumer client applications in an efficient manner.

The change feed processor includes four core components:

| Component | Description |
|-----|-----|
| Monitored container | This container is monitored for any insert or update operations. These changes are then reflected in the feed. |
| Lease container | The lease container serves as a storage mechanism to manage state across multiple change feed consumers (clients). |
| Host | The host is a client application instance that listens for and reacts to changes from the change feed. | 
| Delegate | The delegate is code within the client application that will implement business logic for each batch of changes. |

Prior to using the change feed processor, you should create a **lease** container that you will reference when configuring the processor.

---

# Implement a delegate for the change feed processor

In C#, a delegate is a special type of variable or member that references a method with a specific parameter list and return type.

For the change feed processor, the library expects a delegate of type **ChangesHandler<>** that takes in a generic type to represent your serialized individual items. The delegate includes two parameters; a read-only list of changes and a cancellation token.

You could create a method named **HandleChangesAsync** with the same method signature as the delegate in a verbose syntax.

```cs
static async Task HandleChangesAsync(
    IReadOnlyCollection<Product> changes,
    CancellationToken cancellationToken
) 
{
    // Do something with the batch of changes
}
```

Once that is created, you can create a new variable of type **ChangesHandler<>**, and then reference the earlier method.

```cs
ChangesHandler<Product> changeHandlerDelegate = HandleChangesAsync;
```

An even more concise syntax that accomplishes the same thing would use an anonymous function instead of a named method member.

```cs
ChangesHandler<Product> changeHandlerDelegate = async (
    IReadOnlyCollection<Product> changes,
    CancellationToken cancellationToken
) => {
    // Do something with the batch of changes
};
```

Within the delegate, you can iterate over the list of changes and then implement any business logic that makes sense for your application. In this example, you can think of each change as a “snapshot” of the item at some point in time that is delivered at-least-once to the host client application.

A foreach loop is used to iterate through the current batch of changes, and then each item is printed to the console window

```cs
ChangesHandler<Product> changeHandlerDelegate = async (
    IReadOnlyCollection<Product> changes,
    CancellationToken cancellationToken
) => {
    foreach(Product product in changes)
    {
        await Console.Out.WriteLineAsync($"Detected Operation:\t[{product.id}]\t{product.name}");
        // Do something with each change
    }
};
```

---

# Implement the change feed processor

The change feed processor is created in a few steps:

1. Get the processor builder from the monitored (source) container variable
2. Use the builder to build-out the processor by specifying the delegate, processor name, lease container, and host instance name
3. Start the processor

First, you should create an instance of **Microsoft.Azure.Cosmos.Container** for both the source and lease container.

```cs
Container sourceContainer = client.GetContainer("cosmicworks", "products");

Container leaseContainer = client.GetContainer("cosmicworks", "productslease");
```

Next, you can use the **GetChangeFeedProcessorBuilder** method from a container instance to create a builder. At this point, you should specify the name of the processor and the delegate to handle each batch of changes.

```cs
var builder = sourceContainer.GetChangeFeedProcessorBuilder<Product>(
    processorName: "productItemProcessor",
    onChangesDelegate: changeHandlerDelegate
);
```

The builder ships with a series of fluent methods to configure the processor that includes, but is not limited to:

| Method | Description |
|-----|-----|
| WithInstanceName | Name of host instance |
| WithStartTime | Set the pointer (in time) to start looking for changes after |
| WithLeaseContainer | Configures the lease container |
| WithErrorNotification | Assigns a delegate to handle errors during execution |
| WithMaxItems | Quantifies the max number of items in each batch |
| WithPollInterval | Sets the delay when the processor will poll the change feed for new changes |

A simple example would be to configure the instance name using **WithInstanceName**, the lease container using **WithLeaseContainer**, and then **Build** the change feed processor.

```cs
ChangeFeedProcessor processor = builder
    .WithInstanceName("desktopApplication")
    .WithLeaseContainer(leaseContainer)
    .Build();
```

The resulting change feed processor includes both **StartAsync** and **StopAsync** methods to run and terminate the processor asynchronously.

```cs
await processor.StartAsync();

// Wait while processor handles items

await processor.StopAsync();
```

---

# Implement the change feed estimator

As the change feed processor handles changes, it functions as a time-based pointer. The pointer moves forward in time across the change feed and sends batches of changes to the delegate to run business logic.

The change feed processor can potentially be constrained by the physical resources of the host application. If so, it’s almost immediately assumed that the change feed processor should be scaled out across multiple hosts, all reading from the change feed concurrently.

However, identifying if your change feed solution needs to scale out can be difficult and requires an estimator feature. The change feed estimator is a sidecar feature to the processor that measures the number of changes that are pending to be read by the processor at any point in time.

To implement the estimator, you must first analyze the processor. In this example, a processor is created with a specific lease container and a delegate to handle changes.

```cs
Container sourceContainer = client.GetContainer("cosmicworks", "products");

Container leaseContainer = client.GetContainer("cosmicworks", "productslease");

ChangeFeedProcessor processor = sourceContainer.GetChangeFeedProcessorBuilder<Product>(
    processorName: "productItemProcessor",
    onChangesDelegate: changeHandlerDelegate)
    .WithInstanceName("desktopApplication")
    .WithLeaseContainer(leaseContainer)
    .Build();
```

Next, you should implement a delegate to using the type **ChangesEstimationHandler** to handle each time the estimator polls the change feed to see how many changes have not been processed yet.

```cs
ChangesEstimationHandler changeEstimationDelegate = async (
    long estimation, 
    CancellationToken cancellationToken
) => {
    // Do something with the estimation
};
```

Finally, you build the estimator in a manner similar to the processor reusing the same lease container.

```cs
ChangeFeedProcessor estimator = sourceContainer.GetChangeFeedEstimatorBuilder(
    processorName: "productItemEstimator",
    estimationDelegate: changeEstimationDelegate)
    .WithLeaseContainer(leaseContainer)
    .Build();
```

---

# Exercise: Process change feed events using the Azure Cosmos DB SQL API SDK

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/consume-azure-cosmos-db-sql-api-change-feed-use-sdk/6-exercise-process-change-feed-events-use-sdk)

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

# Knowledge check

---

# Summary
