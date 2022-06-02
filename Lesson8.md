# Implement Azure Cosmos DB SQL API point operations

Write code to create, read, update, and delete items in Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Perform CRUD operations using the SDK
* Configure TTL for a specific item

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

The SQL API SDK for Azure Cosmos DB is used to perform various point operations, perform transactions, and to process bulk data. In this module, you will use the SDK to manipulate items individually using common Create, Read, Update, and Delete (CRUD) operations.

After completing this module, you'll be able to:

* Perform CRUD operations using the SDK
* Configure TTL for a specific item

---

# Understand point operations

The **Microsoft.Azure.Cosmos** library includes first-class support for generics in the C# language, making it vital for you, as a developer, to think about how you want to represent what you are interacting with within your container.

At the most foundational level, you can create a C# class that represents an item in your container that, at a minimum, contains two members:

* a string property named id with a public getter and setter
* a string property with the same name as your partition key path with a public getter and setter

```c#
public class item
{
    public string id { get; set; }

    public string partitionKey { get; set; }
}
```

You can include a rich collection of other members of other types. You can even have other members of different complex types, such as other classes.

```cs
public decimal money { get; set;}

public bool boolean { get; set; }

public string[] set { get; set; }

public double numbers { get; set; }

public int morenumbers { get; set; }

public ComplexClass sophisticated { get; set;}

public List<ComplexType> onetomany { get; set; }
```

Let's establish a fictional scenario for the remainder of this module. We have a **Product** class with five members for the unique **id**, the product's **name**, the unique **category** identifier, the **price**, and a collection of **tags**. The **category** identifier is the **partition key** path for the container.

```cs
public class Product
{
    public string id { get; set; }
    
    public string name { get; set; }

    public string categoryId { get; set; }

    public double price { get; set; }

    public string[] tags { get; set; }
}
```

This implementation is an incredibly versatile C# class that any developer can pick and use immediately. Suppose, for any reason; you need to change the name of properties to fit you business needs. In that case, you can use property attributes to disassociate the name of the property you use in the C# code from the name of the property used in JSON and, in effect, in Azure Cosmos DB SQL API. In this example, you can use the name **InternalId** in C# code and still use the identifier **id** in JSON and Azure Cosmos DB SQL API.

```cs
[JsonProperty(PropertyName = "id")]
public string InternalId { get; set; }
```

> :bulb: Tip
>
> If you have an existing application with C# member names that you cannot change, property attributes is a way to reuse types without incurring the risk of changing your existing code in significant ways or incurring technical debt.

---

# Create Documents

To create a new item, we must first create a new variable in C# code of the **Product** type.

```cs
Product saddle = new()
{
    id = "027D0B9A-F9D9-4C96-8213-C8546C4AAE71",
    categoryId = "26C74104-40BC-4541-8EF5-9892F7F03D72",
    name = "LL Road Seat/Saddle",
    price = 27.12d,
    tags = new string[] 
    {
        "brown",
        "weathered"
    }
};
```

Let’s infer there’s already a variable of type **Microsoft.Azure.Cosmos.Container** named **container**.

We can asynchronously invoke the **CreateItemAsync<>** method passing in the generic Product type and the new item variable into the constructor.

```cs
await container.CreateItemAsync<Product>(saddle);
```

This invocation of the method will create the new item, but you will not have any metadata about the result of the operation. Alternatively, you can store the result of the operation in a variable of type **ItemResponse<>**.

```cs
ItemResponse<Product> response = await container.CreateItemAsync<Product>(saddle);

HttpStatusCode status = response.StatusCode;
double requestUnits = response.RequestCharge;

Product item = response.Resource;
```

If you are using a try-catch block, you can handle the **CosmosException** type, which includes a **StatusCode** property for HTTP status code values. There are a few common HTTP status codes that you should consider in your application code:

| Code | Title | Reason |
|-----|-----|-----|
| 400 | Bad Request | Something was wrong with the item in the body of the request | 
| 403 | Forbidden | Container was likely full |
| 409 | Conflict | Item in container likely already had a matching id |
| 413 | RequestEntityTooLarge | Item exceeds max entity size |
| 429 | TooManyRequests | Current request exceeds the maximum RU/s provisioned for the container |

In this example

```cs
try
{
    await container.CreateItemAsync<Product>(saddle);
}
catch(CosmosException ex) when (ex.StatusCode == HttpStatusCode.Conflict)
{
    // Add logic to handle conflicting ids
}
catch(CosmosException ex) 
{
    // Add general exception handling logic
}
```

---

# Read a document

To do a point read of an existing item from the container, we need two things.

First, we need the unique id of the item. Here, we store that id in a variable of the same name.

```cs
string id = "027D0B9A-F9D9-4C96-8213-C8546C4AAE71";
```

Second, we need to create a variable of type **PartitionKey** with the string value at the partition key path for the item we are seeking.

```cs
string categoryId = "26C74104-40BC-4541-8EF5-9892F7F03D72";
PartitionKey partitionKey = new (categoryId);
```

Once we both item, we can invoke the asynchronous and generic **ReadItemAsync<>** method, which will return an item of the given generic type, **Product**, in this example.

```cs
Product saddle = await container.ReadItemAsync<Product>(id, partitionKey);
```

At this point, we can access properties of the **saddle** variable and print them to the console much like any local variable.

```cs
string formattedName = $"New Product [${saddle}]";
Console.WriteLine(formattedName);
```

---

# Update documents

We can also modify the properties of the **saddle** variable.

In this example, we change the price of the variable from the original **$27.12** to **$35.00**.

```cs
saddle.price = 35.00d;
```

To persist this change, we can invoke the asynchronous **UpsertItemAsync<>** method passing in only the update item.

```cs
await container.UpsertItemAsync<Product>(saddle);
```

We can also continue to make new changes. In this example, we replace the array of tags with a new array of tags with more accurate descriptions of the saddle.

```cs
saddle.tags = new string[] { "brown", "new", "crisp" };
```

Even though we upserted the document already, we don’t have to read a new item before upserting the item again.

```cs
await container.UpsertItemAsync<Product>(saddle);
```

---

# Configure time-to-live (TTL) value for a specific document

To implement time-to-live (TTL) on an individual item, you can use the same strategy as you use to upsert an item.

First, let’s take a look at the **Product** class. We can define a new **TimeToLive** property that will only set the ttl property on the JSON if it’s not null. This technique is accomplished by configuring the JsonProperty header to ignore null values and configuring the member as a nullable int.

```cs
[JsonProperty(PropertyName = "ttl", NullValueHandling = NullValueHandling.Ignore)]
public int? ttl { get; set; }
```

From there, you can update the **saddle** variable by setting the **TimeToLive** value to an integer to indicate how long, in seconds, you want the item to last before it’s automatically purged beyond its last modified time.

```cs
saddle.ttl = 1000;
```

Update the item using the **UpsertItemAsync<>** method.

```cs
await container.UpsertItemAsync<Product>(saddle);
```

> :grey_exclamation: Note
>
> Remember, this will not work if the **DefaultTimeToLive** property is not configured at the container level.

---

# Delete documents

Deleting a document is similar, in process, to reading an item. You will need the id and the value of the partition key path.

```cs
string id = "027D0B9A-F9D9-4C96-8213-C8546C4AAE71";
string categoryId = "26C74104-40BC-4541-8EF5-9892F7F03D72";
PartitionKey partitionKey = new (categoryId);
```

Once you have both of those values, you invoke the asynchronous **DeleteItemAsync<>** method in a manner similar to the **ReadItemAsync<>** method.

```cs
await container.DeleteItemAsync<Product>(id, partitionKey);
```

---

# Exercise: Create and update documents with the Azure Cosmos DB SQL API SDK

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/implement-azure-cosmos-db-sql-api-point-operations/8-exercise-create-update-documents-sdk)

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

1. What parameter[s] are required to read an item in Azure Cosmos DB SQL API?

- [x] Both an id and partition key value
> That's correct. You can perform a point read with both the id and partition key values.
- [ ] Only an id
- [ ] Only a partition key value

2. Which method should you use to update an item in Azure Cosmos DB SQL API using the .NET SDK?

- [ ] PatchItemAsync<>
- [ ] UpdateItemAsync<>
- [x] UpsertItemAsync<>
> That's correct. TheUpsertItemAsync<> method takes in an item as the method parameter and updates the item.

3. Which class contains the methods for Create, Read, Update, and Delete point operations on items in Azure Cosmos DB SQL API?

- [ ] Database
- [x] Container
> That's correct. The Microsoft.Azure.Cosmos.Container class contains methods for CRUD operations on items.
- [ ] CosmosClient

---

# Summary

In this module, you used the .NET SDK to perform some of the most common operations that your applications will perform on individual items. These operations will often be the individual building blocks of bigger bulk and transactional operations that you may build in the future.

Now that you have completed this module, you can:

* Perform Create, Read, Update, and Delete operations on a specific item using the SDK
* Update the TTL value of a specific item using the SDK
