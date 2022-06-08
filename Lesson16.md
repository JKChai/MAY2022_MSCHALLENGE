# Handle events with Azure Functions and Azure Cosmos DB SQL API change feed

Use Azure Functions bindings to integrate a function with Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Create an Azure Function using the Azure Cosmos DB trigger

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Azure Functions contains a suite of triggers that are used to start the processing of function code. Specifically, the Azure Cosmos DB trigger for Azure Functions uses the change feed to trigger the execution of a function when there is a batch of changes ready to process. In this module, you will explore the Azure Cosmos DB trigger for Azure Functions and how to process batches of changes using a function.

After completing this module, you'll be able to:

* Create an Azure Function using the Azure Cosmos DB trigger

---

# Understand Azure Function bindings for Azure Cosmos DB SQL API

**Azure Functions** is a service that offers serverless blocks of code that can run logic on-demand. Functions are often the building blocks of more complex solutions and are, therefore, reactive in their nature. Typically, some external action invokes a function, the function code performs a small task, and the function returns a response. A complex cloud-native solution could run with as few or as many functions as needed, as they are building blocks that can be used in repeatable and scale-out scenarios.

Each function starts with some external event, called a **trigger** that indicates the function should start. Many triggers also include some payload for the function to process. Azure Functions has triggers for various cloud services to ease the complexity of integrating Azure Functions across your entire solution.

A function can also contain an **input** binding that provides more data after the function has already been triggered. Functions, in addition, can include an **output** binding indicating where the function should send its response.

![Diagram illustrating an Azure Function with a generic trigger, input, and output.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/handle-events-azure-functions-azure-cosmos-db-sql-api-change-feed/media/2-bindings.png)

Azure Cosmos DB has support for all three types of bindings in Azure Functions:

| Binding | Description |
|-----|-----|
| Trigger | Start a function’s execution when there is a batch of documents ready from the change feed |
| Input | Read one or more items using either a point read or a SQL query |
| Output | Write one or more items to a container |

> :grey_exclamation: Note
>
> The bindings referenced here are only supported by the SQL API.

![Diagram illustrating an Azure Function with an Azure Cosmos DB trigger, generic input, and generic output.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/handle-events-azure-functions-azure-cosmos-db-sql-api-change-feed/media/2-bindings-cosmos-db.png)

---

# Configure function bindings

To configure an Azure Function to use an Azure Cosmos DB SQL API binding, you should first create an app setting in the function instance with the connection string of the Azure Cosmos DB account. If you are using the Azure portal, this can be done automatically on your behalf. Once you have the app setting with the connection string, you can leverage the setting in bindings for your Azure Function.

> :grey_exclamation: Note
>
> The following examples assume that an app setting named **cosmosdbsqlconnstr** is already configured in the function instance with the connection string of the Azure Cosmos DB account.

The function.json file is the configuration file for all bindings within a function. The file contains a JSON object with a property named bindings. The bindings object is an array of trigger, input, and output bindings for that particular function.

```json
{
  "bindings": []
}
```

## Trigger function on changes in the change feed

Configuring the Azure Cosmos DB SQL API trigger requires a JSON object within the bindings array. This object contains various properties that you can configure to change the behavior of the trigger. These properties include, but are not limited to:

| Property | Description |
|-----|-----|
| type | This is statically set to cosmosDBTrigger |
| name | This is the name used for the method parameter that will map to this binding in code |
| direction | For a trigger, this will be set to in |
| connectionStringSetting | This is the name of the connection string in the function’s app settings |
| databaseName | The name of the database, which contains the container to monitor |
| collectionName | The name of the container to monitor | 
| leaseCollectionName | The name of the container used to manage change feed leases |
| createLeaseCollectionIfNotExists | A boolean value to indicate if the Azure Functions runtime should create the lease container on your behalf if it does not already exist |

An example of a trigger that monitors changes in the **cosmicworks** database and **products** container is included here. This trigger will use the change feed to monitor if new items are created or if existing items are updated.

```json
{
  "type": "cosmosDBTrigger",
  "name": "changes",
  "direction": "in",
  "connectionStringSetting": "cosmosdbsqlconnstr",
  "databaseName": "cosmicworks",
  "collectionName": "products",
  "leaseCollectionName": "productslease",
  "createLeaseCollectionIfNotExists": false
}
```

This trigger will start the function when there is a new batch of items to process from the change feed.

## Bind the function's input parameter to an item or query

The bindings array can optionally have multiple input bindings within the bindings array. There are two types of input bindings; input bindings that perform a point read and lookup a single item, and input bindings that perform a SQL query and return multiple items.

### Point read input binding

A point read input bindings uses an item's unique identifier and partition key value to perform a quick read operation. This configuration object only differs from the trigger with a few changes to the properties:

| Property | Description |
|-----|-----|
| type | This input binding has a static type of cosmosDB |
| direction | Input bindings will be set to in |
| id | Unique identifier for the target item |
| partitionKey | Partition key value for the target item |

> :grey_exclamation: Note
>
> Duplicate properties, such as **databaseName** and **collectionName**, are excluded from this table as they were described earlier in this unit.

An example of an input binding that reads an item with an **id** of **91AA100C-D092-4190-92A7-7C02410F04EA** and a **partition key** of **F3FBB167-11D8-41E4-84B4-5AAA92B1E737** is included here.

```json
{
  "type": "cosmosDB",
  "name": "item",
  "direction": "in",
  "connectionStringSetting": "cosmosdbsqlconnstr",
  "databaseName": "cosmicworks",
  "collectionName": "products",
  "id": "91AA100C-D092-4190-92A7-7C02410F04EA",
  "partitionKey": "F3FBB167-11D8-41E4-84B4-5AAA92B1E737"
}
```

This input binding, as configured, will include a single item as an input value to the function.

### SQL query input binding

A SQL query input binding uses a SQL query to look up multiple items and provide them to the function. The configuration object for this type of binding only includes one extra field:

| Property | Description |
|-----|-----|
| sqlQuery | SQL query used to look up multiple items |

Included here is an example of this type of input bindings that performs a SQL query to return a subset of items from the container with only a few fields included in the results.

```json
{
  "type": "cosmosDB",
  "name": "items",
  "direction": "in",
  "connectionStringSetting": "cosmosdbsqlconnstr",
  "databaseName": "cosmicworks",
  "collectionName": "products",
  "sqlQuery": "SELECT p.id, p.name, p.categoryId FROM products p WHERE p.price > 500"
}
```

## Output items from the function

Finally, the bindings array includes an output binding to configure pipelines to send data to other application components or cloud services. There are primarily two ways to configure the output binding; use the output binding to write a single item to a container, or use the output binding to write multiple items to a container.

In both examples, the output binding is configured by manipulating only a few properties:

| Property | Description |
|-----|-----|
| type | This output binding has a static type of cosmosDB |
| direction | Output bindings will be set to out |

An example of an output binding that writes one or more items to the **cosmicworks** container is included here.

```json
{
  "type": "cosmosDB",
  "name": "output",
  "direction": "out",
  "connectionStringSetting": "cosmosdbsqlconnstr",
  "databaseName": "cosmicworks",
  "collectionName": "products"
}
```

---

# Develop function

The code to use various bindings in a function is primarily focused on the parameters of the function’s method. Once the parameters are configured, the data associated with each binding can be used and modified in the method much like any other C# variable.

## Trigger function on changes in the change feed

The code for a change feed trigger includes a method parameter of type **IReadOnlyList<Document>** that has an enumerated list of the current batch of changes. The name of the method parameter must match the value provided in the binding configuration for the name property. In this example method signature, the **name** in the binding is set to **changes**. Within the function code, your logic can use a foreach loop to iterate over all changes in the current batch.

```cs
public static void Run(IReadOnlyList<Document> changes)
{
    foreach(Document doc in changes ?? Enumerable.Empty<Document>())
    {
        //Do something with each item
    }
}
```

> :bulb: Tip
>
> This sample code uses a null coalescing operator to ensure that the function doesn't throw an exception if the list of changes is null.

## Bind the function's input parameter to an item or query

An input binding for a function can include either a single item or multiple items. Input bindings also support the ability to bind to a C# type.

> :grey_exclamation: Note
> 
> The remaining examples for input/output bindings will assume that the function is triggered using a HTTP request.

### Point read input binding

For a point read, the input binding’s parameter should be set to a simple type. In this first example, the binding is named **item** and is set to an object of type **Document**. The Document class is a special class that can represent an item in Azure Cosmos DB SQL API if you don't want to define your own type.

```cs
public static void Run(HttpRequest request, Document item)
{
    //Do something with this item
}
```

As an alternative, you can define a type in C# to represent the data in your Azure Cosmos DB SQL API container.

```cs
public class Product
{
    public string id { get; set; }

    public string categoryId { get; set; }

    public string name { get; set; }
}
```

This type, once defined, can be used with the method parameter. The Azure Functions runtime will automatically bind the input item to the specified type.

```cs
public static void Run(HttpRequest request, Product item)
{
    //Do something with this item
}
```

### SQL query input binding

Input bindings with a SQL query require a method parameter of type **IEnumerable<>**. The type used as a generic parameter can be **Document** or a type you created in C#.

In this first example, the input binding is named **items** and is bound to a collection of type **Document**.

```cs
public static async Task Run(HttpRequest request, IEnumerable<Document> items)
{
    //Do something with these items
}
```

The second example illustrates the ability to use your own custom C# types instead of the **Document** type.

```cs
public static async Task Run(HttpRequest request, IEnumerable<Product> items)
{
    //Do something with these items
}
```

## Output items from the function

Whether an output binding will update one or multiple items is made by setting the input parameter to one of two types.

First, to write only a single item to the container, the output binding’s corresponding parameter should be configured as an **out** parameter and can be any C# type.

In this example, the output binding is named **output**, and the out parameter is of type **Product**. This example also illustrates

```cs
public static void Run(HttpRequest request, out Product output)
{    
    var product = new Product()
    {
        id = "FDEF01CB-5067-414F-B0A3-07FF8A4B80DD",
        categoryId = "14A1AD5D-59EA-4B63-A189-67B077783B0E",
        name = "Sport-100 Helmet, Red"
    };
    output = product;
}
```

This following example illustrates writing multiple items to a container. The primary changes here include:

* The method is now configured for async calls
* The input parameter is of type **IAsyncCollector<>** using our C# type as a generic parameter
* The items are added to the container asynchronously using the **AddAsync** method

```cs
public static async Task Run(HttpRequest request, IAsyncCollector<Product> output)
{    
    var firstProduct = new Product()
    {
        id = "7236DDB5-CFE0-4D3D-8FE5-799B398396B1",
        categoryId = "AE48F0AA-4F65-4734-A4CF-D48B8F82267F",
        name = "Road-650 Black, 48"
    };

    var secondProduct = new Product()
    {
        id = "878C50F0-7E29-4D0D-A52E-6D8B063673E3",
        categoryId = "AE48F0AA-4F65-4734-A4CF-D48B8F82267F",
        name = "Road-250 Red, 58"
    };

    await output.AddAsync(firstProduct); 
    await output.AddAsync(secondProduct);
}
```

---

# Exercise: Process Azure Cosmos DB SQL API data using Azure Functions

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.


[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/handle-events-azure-functions-azure-cosmos-db-sql-api-change-feed/5-exercise-archive-data-using-azure-functions)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge check

1. You're creating an Azure Function to monitor a container for new items and changes to existing items. Which type of binding should you use to start executing the function where there are changes to items?

- [ ] Input binding
- [x] Trigger binding
> Correct. A trigger binding will start executing a function when the change feed has a new batch of changes to process.
- [ ] Output binding

2. You have an output binding configured for Azure Cosmos DB SQL API named items. You would like to add multiple items to a container using this binding. Which function method parameter should you use to add more than one item to a container?

- [ ] out Product items
- [x] IAsyncCollector<Product> items
> Correct. The IAsyncCollector class contains the AddAsync method to add items to a container.
- [ ] out IEnumerable<Product> items

3. When configuring an Azure Cosmos DB SQL API trigger, which of the following settings would be valid?

- [ ] Assign the type to AzureCosmosDBTrigger
- [x] Set the direction to in.
> Correct. For a trigger, this will be set to in.
- [ ] Set the leaseCollectionName to the name of the container that will contain all the new leased documents

---

# Summary

In this module, you used an Azure Function to process changes, in batches, from the change feed. You used a C# script function to iterate over the changes in the batch, and perform an operation.

Now that you have completed this module, you can:

* Use the Azure Cosmos DB trigger in Azure Functions to process changes from the change feed