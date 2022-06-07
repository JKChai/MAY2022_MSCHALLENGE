# Customize indexes in Azure Cosmos DB SQL API

Customize indexing policies for a container in Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Implement a correlated subquery
* Create a cross-product query

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

As your application matures, you can customize your indexing policy to better match the needs of your solution. In this module, you will learn how to create a custom indexing policy.

After completing this module, you'll be able to:

* Customize an indexing policy in Azure Cosmos DB SQL API
* Evaluate whether a composite index is appropriate for a solution

---

# Customize the Index Policy

In a solution with a large amount of throughput, it’s not uncommon to selectively optimize the number of paths indexed to reduce both the latency and RU charge of individual operations that create or modify an item. To accomplish this, you would need to create a container using a custom indexing policy.

Let’s start with a simple example JSON document. This document has multiple properties, but the goal is to only index the **categoryName** and name **properties**.

```json
{
  "id": "8B363B8B-378E-402A-9E68-A935302000B8",
  "name": "HL Touring Frame - Yellow, 46",
  "categoryId": "F3FBB167-11D8-41E4-84B4-5AAA92B1E737",
  "categoryName": "Components, Touring Frames",
  "sku": "FR-T98Y-46",
  "price": 1003.91
}
```

In raw JSON, this indexing policy would start by excluding all possible paths, and then opt-in to only including the `/name/?` and `/categoryName/?` paths.

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {
      "path": "/name/?"
    },
    {
      "path": "/categoryName/?"
    }
  ],
  "excludedPaths": [
    {
      "path": "/*"
    }
  ]
}
```

> :grey_exclamation: Note
>
> We are excluding all paths here for demonstration purposes. In general, it's much better to include all paths by default and only exclude specific paths.

The .NET SDK ships with a **Microsoft.Azure.Cosmos.IndexingPolicy** class that is a representation of the typical JSON policy object. When creating a new instance of the class, you can immediately set the **IndexingMode** and **Automatic** properties much like their JSON counterparts. In this example, the indexing mode is set to consistent and automatic indexing is enabled.

```cs
IndexingPolicy policy = new ()
{
    IndexingMode = IndexingMode.Consistent,
    Automatic = true
};
```

The class also includes an **ExcludedPaths** collection with an **Add** method to add new object of type **ExcludedPath**. In this example, the `\*` path is added to the list of excluded paths.

```cs
policy.ExcludedPaths.Add(
    new ExcludedPath{ Path = "/*" }
);
```

Similarly, the class includes an **IncludedPaths** collection. This example illustrates the /name/? and /categoryName/? paths being added to the list of included paths.

```cs
policy.IncludedPaths.Add(
    new IncludedPath{ Path = "/name/?" }
);
policy.IncludedPaths.Add(
    new IncludedPath{ Path = "/categoryName/?" }
);
```

Once the indexing policy is configured, the **Microsoft.Azure.Cosmos.ContainerProperties** class is used to configure properties of a container such as a name, partition key path, and indexing policy. This class instance is then passed into the **CreateContainerIfNotExistsAsync** method as the first parameter to create a new container with the custom indexing policy.

```cs
ContainerProperties options = new ()
{
    Id = "products",
    PartitionKeyPath = "/categoryId",
    IndexingPolicy = policy
};
Container container = await database.CreateContainerIfNotExistsAsync(options, throughput: 400);
```

---

# Evaluate Composite Index

It’s not uncommon to have a query sort or filter on multiple properties. In these scenarios, customizing the indexing policy in small ways could reap benefits in the performance of those queries.

For example, if you are writing SQL queries that filters on multiple properties simultaneously, you could benefit from a particular type of index called a **composite index** that combines two paths in a specific order.

Let’s look at an example query:

```sql
SELECT * FROM products p WHERE p.name = "Road Saddle" AND p.price > 50
```

This query includes two filters:

* An equality filter that checks the value of the **name** property for equivalency to the string `Road Saddle`.
* A range filter that checks to see if the value of the **price** property is greater than the number `50`.

If the query is able to use a composite index, that includes both the **name** and **price** properties. That composite index could be:

* `(name ASC, price ASC)`
* `(name DESC, price ASC)`

> :grey_exclamation: Note
>
> In this example, the range filter appeared last. This is a best practice for queries with multiple filters that leverage a composite index.

Going even deeper, queries that order the results using multiple properties must include a composite index.

Let’s look at another example query:

```sql
SELECT * FROM products p ORDER BY p.price ASC, p.name ASC
```

The composite index that will support this query must exactly match the sequence of the properties in the ORDER BY clause. A composite index of `(name ASC, price ASC)` would not work in this example. Composite indexes such as these will work in this scenario:

* `(price ASC, name ASC)`
* `(price DESC, name DESC)`

> :bulb: Tip
>
> You can also use composite indexes with queries that have different permutations of filters and order by clauses.

To create a raw JSON indexing policy with a composite index, you should include the optional **compositeIndexes** array property. This external array consists of a series of internal arrays for each composite index definition.

For example, to create a composite index of `(name ASC, price DESC)`, you can define a JSON object with this structure:

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {
      "path": "/*"
    }
  ],
  "excludedPaths": [
    {
      "path": "/_etag/?"
    }
  ],
  "compositeIndexes": [
    [
      {
        "path": "/name",
        "order": "ascending"
      },
      {
        "path": "/price",
        "order": "descending"
      }
    ]
  ]
}
```

> :bulb: Tip
>
> Remember, you can define multiple composite indexes within your indexing policy for various important queries that your application[s] require.

---

# Exercise: Configure an Azure Cosmos DB SQL API container's index policy with the portal

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/customize-indexes-azure-cosmos-db-sql-api/4-exercise-configure-containers-index-policy-portal)

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

1. You are writing a C# application that interfaces with Azure Cosmos DB SQL API and are not tasked with creating a custom indexing policy in code. Which class should you use to author your new indexing policy and customize the indexing mode, enable automatic updates, and set the include/exclude paths?

- [x] IndexingPolicy
> That's correct. The IndexingPolicy class is used to define a custom indexing policy in code.
- [ ] ContainerProperties
- [ ] IndexingMode

2. The most used query in your Azure Comsos DB SQL API application is SELECT * FROM products p ORDER BY p.name ASC, p.price ASC. You would like to define a composite index to make the query more efficient and consume fewer RU/s. Which composite index should you use to support this query?

- [ ] (price ASC, name ASC)
- [ ] (price DESC, name ASC)
- [x] (name ASC, price ASC)
> That's correct. This composite index will support the SQL query.

---

# Summary

In this module, you explored the various ways you can customize an indexing policy while exploring the options to manage indexing policies.

Now that you have completed this module, you can:

* Customize the contents of an indexing policy
