# Author complex queries with the Azure Cosmos DB SQL API

Create SQL queries for Azure Cosmos DB SQL API that uses subqueries or cross-products.

**Learning objectives**

After completing this module, you'll be able to:

* Implement a correlated subquery
* Create a cross-product query

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

It's not uncommon for JSON documents to include child documents and arrays. These more complex documents will require SQL queries with more complex expressions than ones that are available in the typical SQL language used with typical relational databases. In this module, you will learn how to author complex queries using cross-products and correlated subqueries.

After completing this module, you'll be able to:

* Implement a correlated subquery
* Create a cross-product query

---

# Create cross-product queries

A JOIN in Azure Cosmos DB SQL API is different from a JOIN in a relational database as its only scope is a single item. A **JOIN** creates a cross-product between different sections of a single item.

Let’s take this example JSON object, which has a name property and an array with three objects that each have their own group property:

```json
{
    "id": "E08E4507-9666-411B-AAC4-519C00596B0A",
    "name": "Men's Bib-Shorts",
    "groups": [
        {
            "group": "accessories"
        },
        {
            "group": "new"
        },
        {
            "group": "sale"
        }
    ]
}
```

If you create a cross-product of the **name** and **group** properties, you will create a JSON array with permutations of possible combinations of names and groups, making it easier for your applications to iterate over items in the array:

```json
[
    {
        "name": "Men's Bib-Shorts",
        "group": "accessories"
    },
    {
        "name": "Men's Bib-Shorts",
        "group": "new"
    },
    {
        "name": "Men's Bib-Shorts",
        "group": "sale"
    }
]
```

So, how can you create this type of cross-product in a SQL query? The `JOIN` keyword in Azure Cosmos DB SQL API returns all possible combinations of values within two sets. Let’s use a different example JSON object with a more complex group of tags:

```json
{
    "id": "80D3630F-B661-4FD6-A296-CD03BB7A4A0C",
    "categoryId": "629A8F3C-CFB0-4347-8DCC-505A4789876B",
    "categoryName": "Clothing, Vests",
    "sku": "VE-C304-L",
    "name": "Classic Vest, L",
    "description": "A worn brown classic vest that was a trade-in apparel item",
    "price": 32.4,
    "tags": [
        {
            "id": "2CE9DADE-DCAC-436C-9D69-B7C886A01B77",
            "name": "apparel",
            "class": "group"
        },
        {
            "id": "CA170AAD-A5F6-42FF-B115-146FADD87298",
            "name": "worn",
            "class": "trade-in"
        },
        {
            "id": "CA170AAD-A5F6-42FF-B115-146FADD87298",
            "name": "no-damaged",
            "class": "trade-in"
        }
    ]
}
```

The corresponding query for this is structured like most `SELECT FROM` query, but also includes the `JOIN` keyword, which references the **tags** property and aliases it with the letter t. Then, we add the **t.name** to the list of projected fields in the query results:

```sql
SELECT 
    p.id,
    p.name,
    t.name AS tag
FROM 
    products p
JOIN
    t IN p.tags
```

The result of this query is a JSON array that includes three objects for the single JSON item in the container:

```json
[
    {
        "id": "80D3630F-B661-4FD6-A296-CD03BB7A4A0C",
        "name": "Classic Vest, L",
        "tag": "apparel"
    },
    {
        "id": "80D3630F-B661-4FD6-A296-CD03BB7A4A0C",
        "name": "Classic Vest, L",
        "tag": "worn"
    },
    {
        "id": "80D3630F-B661-4FD6-A296-CD03BB7A4A0C",
        "name": "Classic Vest, L",
        "tag": "no-damaged"
    }
]
```

---

# Implement correlated subqueries

We can optimize JOIN expressions further by writing subqueries to filter the number of array items we want to include in the cross-product set.

Let’s examine the example JSON object again:

```json
{
    "id": "80D3630F-B661-4FD6-A296-CD03BB7A4A0C",
    "categoryId": "629A8F3C-CFB0-4347-8DCC-505A4789876B",
    "categoryName": "Clothing, Vests",
    "sku": "VE-C304-L",
    "name": "Classic Vest, L",
    "description": "A worn brown classic vest that was a trade-in apparel item",
    "price": 32.4,
    "tags": [
        {
            "id": "2CE9DADE-DCAC-436C-9D69-B7C886A01B77",
            "name": "apparel",
            "class": "group"
        },
        {
            "id": "CA170AAD-A5F6-42FF-B115-146FADD87298",
            "name": "worn",
            "class": "trade-in"
        },
        {
            "id": "CA170AAD-A5F6-42FF-B115-146FADD87298",
            "name": "no-damaged",
            "class": "trade-in"
        }
    ]
}
```

In this example, we include tags that are in both classes **trade-in** and **group**. What if we want to filter out the **group** tags?

We can rewrite our JOIN expression by writing a subquery to filter out the group tags using a subquery:

```sql
SELECT VALUE t FROM t IN p.tags WHERE t.class = 'trade-in'
```

If we add this subquery to the entire all-up query, it will total up to this:

```sql
SELECT 
    p.id,
    p.name,
    t.name AS tag
FROM 
    products p
JOIN
    (SELECT VALUE t FROM t IN p.tags WHERE t.class = 'trade-in') AS t
```

Our final JSON result array would then be this with one less result in the set:

```json
[
    {
        "id": "80D3630F-B661-4FD6-A296-CD03BB7A4A0C",
        "name": "Classic Vest, L",
        "tag": "worn"
    },
    {
        "id": "80D3630F-B661-4FD6-A296-CD03BB7A4A0C",
        "name": "Classic Vest, L",
        "tag": "no-damaged"
    }
]
```

---

# Implement variables in queries

We can implement many common cross-product queries on the SDK side and may want to add filters to prevent the queries from exploding in result size and complexity. Using the QueryDefinition class and the fluent SDK, we can add query parameters to quickly adjust the values in a WHERE filter for a SQL query.

Let’s look at an example SQL query that uses a `JOIN` and a `WHERE` filter:

```sql
SELECT 
    p.name,
    t.name AS tag
FROM 
    products p
JOIN
    t IN p.tags
WHERE
    p.price > 500
```

In C#, we would typically create a query definition using the following syntax with the value of 500 hard-coded in a string value:

```cs
string sql = "SELECT p.name, t.name AS tag FROM products p JOIN t IN p.tags WHERE p.price > 500"
QueryDefinition query = new (sql);
```

However, using the **.WithParameter(string, string)** fluent method, you can add parameters to the query making it easier to configure parameters in the query:

```cs
string sql = "SELECT p.name, t.name AS tag FROM products p JOIN t IN p.tags WHERE p.price > @lower"
QueryDefinition query = new (sql)
    .WithParameter("@lower", 500);
```

You can even use multiple parameters in more complex queries:

```cs
string sql = "SELECT p.name, t.name AS tag FROM products p JOIN t IN p.tags WHERE p.price >= @lower AND p.price <= @upper"
QueryDefinition query = new (sql)
    .WithParameter("@lower", 500)
    .WithParameter("@upper", 1000);
```

---

# Paginate query results

The **Microsoft.Azure.Cosmos.Container** class supports asynchronous streams for the simplest and easiest way to iterate over multiple pages of results. In scenarios where you wish to paginate between results manually, you can retrieve the feed iterator and read each page of results.

First, define a SQL query string that you wish to execute, and then use it as a constructor parameter for a variable of type **QueryDefinition**.

```cs
string sql = "SELECT * FROM products WHERE p.price > 500";
QueryDefinition query = new (sql);
```

Then, build an object of type **QueryRequestOptions** using the **MaxItemCount** property to specify how many items you wish to return for each page of results.

```cs
QueryRequestOptions options = new()
{
    MaxItemCount = 100
};
```

Finally, create a new **FeedIterator<>** using your generic type and the **GetItemQueryIterator** method.

```cs
FeedIterator<Product> iterator = container.GetItemQueryIterator<Product>(query, requestOptions: options);
```

The feed iterator class contains a **HasMoreResults** boolean property that indicates more pages to return server-side. This property is ideal to use within a while loop. The iterator also has a **ReadNextAsync** method that gets the next set of items into an enumerable collection that can be iterated over with a foreach loop.

```cs
while(iterator.HasMoreResults)
{
    foreach(Product product in await iterator.ReadNextAsync())
    {
        // Handle individual items
    }
}
```

---

# Exercise: Paginate cross-product query results with the Azure Cosmos DB SQL API SDK

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/author-complex-queries-azure-cosmos-db-sql-api/6-exercise-paginate-cross-product-query-results-sdk)

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

1. You have a container of products with JSON documents that contain a string property named name. The JSON documents also all contain a property named tags that's an array of string values. You are tasked with authoring a SQL query that will create a cross-product of the document names and tags. Which SQL query should you use?

- [ ] SELECT p.name, t.tags[0] FROM products p
- [ ] SELECT p.name, t.tags FROM products p
- [x] SELECT p.name, t FROM products p JOIN t IN p.tags
> That's correct. This query will create a cross-product of name and tag values.

2. Which method of the Microsoft.Azure.Cosmos.Container class takes in a SQL query as a string parameter and returns an iterator that can be used to iterate over the query results as deserialized C# objects?

- [ ] GetItemQueryStreamIterator<>
- [x] GetItemQueryIterator<>
> That's correct. This method will use a SQL string to construct the query and then deserialize the results into C# objects.
- [ ]  GetItemLinqQueryable<>

---

# Summary

In this module, you learned how to author cross-product queries in SQL and then later implement the same queries in the .NET SDK.

Now that you have completed this module, you can:

* Author a cross-product query using the JOIN keyword
* Create a correlated subquery within a cross-product query
