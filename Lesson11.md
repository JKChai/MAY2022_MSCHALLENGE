# Query the Azure Cosmos DB SQL API

Author queries for Azure Cosmos DB SQL API using the SQL query language.

**Learning objectives**

After completing this module, you'll be able to:

* Create and execute a SQL query
* Project query results
* Use built-in functions in a query

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

The Azure Cosmos DB SQL API supports Structured Query Language (SQL) as a JSON query language. In this module, you will learn how to create efficient queries using the SQL query language.

After completing this module, you'll be able to:

* Create and execute a SQL query
* Project query results
* Use built-in functions in a query

---

# Understand SQL query language

Azure Cosmos DB SQL API uses the already popular Structured Query Language (SQL) syntax to perform queries over semi-structured data. If you have performed queries in database platforms like MySQL or SQL, then you may already have some of the tools necessary to write queries in Azure Cosmos DB SQL API.

For this module, we will focus on a fictional container of **products** with the following structure:

| Property | Value |
|-----|-----|
| id | String \| unique identifier |
| categoryId | String \| partition key |
| categoryName | String |
| sku | String |
| description | String |
| price | Number |
| tags | Array \| [ String id, String name ] |

Here is an example of a JSON object that would be in this container:

```json
{
    "id": "86FD9250-4BD5-42D2-B941-1C1865A6A65E",
    "categoryId": "F3FBB167-11D8-41E4-84B4-5AAA92B1E737",
    "categoryName": "Components, Touring Frames",
    "sku": "FR-T67U-58",
    "name": "LL Touring Frame - Blue, 58",
    "description": "The product called \"LL Touring Frame - Blue, 58\"",
    "price": 333.42,
    "tags": [
        {
            "id": "764C1CC8-2E5F-4EF5-83F6-8FF7441290B3",
            "name": "Tag-190"
        },
        {
            "id": "765EF7D7-331C-42C0-BF23-A3022A723BF7",
            "name": "Tag-191"
        }
}
```

---

# Create queries with SQL

A basic SQL query in Azure Cosmos DB SQL API would be similar to the same query in any other database platform; it would be composed of a few essential components:

* The `SELECT` keyword
* Either an asterisk to indicate all possible fields or an inclusive list of fields
* The `FROM` keyword followed by the data source (container)

Here is a basic query that returns all fields from a container:

```sql
SELECT * FROM products
```

Here is another query that returns only a few fields from a container:

```sql
SELECT 
    products.id, 
    products.name, 
    products.price, 
    products.categoryName 
FROM 
    products
```

One interesting caveat here is that it doesn’t matter what name is used here for the source, as this will reference the source moving forward. You can think of this as a variable. It’s not uncommon to use a single letter from the container name:

```sql
SELECT
    p.name, 
    p.price
FROM 
    p
```

You can use any word or phrase like you would in developer code:

```sql
SELECT
    supercalifragilisticexpialidocious.id,
    supercalifragilisticexpialidocious.categoryId
FROM 
    supercalifragilisticexpialidocious
```

Alternatively, you can alias the data source and use the alias if that’s your preference:

```sql
SELECT 
    alternativealias.id, 
    alternativealias.name 
FROM 
    reallyinterestingdatasource alternativealias
```

We can also filter our queries using the `WHERE` keyword. In this example, we filter the list of products to those that have a price that is between $50 and $100:

```sql
SELECT
    p.name, 
    p.categoryName,
    p.price
FROM 
    products p
WHERE
    p.price >= 50 AND
    p.price <= 100
```

---

# Project query results

When developing middle-tier and API applications, there is a tendency to build highly complex solutions to translate database results to something that the business application can understand and use. This workaround often occurs because the database platform is inflexible and must store the data in some fixed schema that can never be changed.

One of the great things about JSON is that it’s compatible with various developer platforms making it highly flexible. Azure Cosmos DB SQL API extends the SQL query language by adding functionality to manipulate the JSON results of your query so you can change the query result to map to the schema and shape that your developer team needs.

Let’s look at an example.

In the previous unit, you ran this query:

```sql
SELECT
    p.name, 
    p.categoryName,
    p.price
FROM 
    products p
WHERE
    p.price >= 50 AND
    p.price <= 100
```

And here is an example result:

```json
{
    "name": "LL Bottom Bracket",
    "categoryName": "Components, Bottom Brackets",
    "price": 53.99
}
```

While this result is acceptable, your dev team needs this result mapped to this C# object and does not want to write extra code to accomplish this task.

```cs
public class ProductAdvertisement
{
    public string Name { get; set; }
    
    public string Category { get; set; }

    public class ScannerData
    {
        public decimal Price { get; set; }
    }
}
```

> :grey_exclamation: Note
>
> For the purposes of this exercises, you can ignore casing. The JSON parser will properly handle converting between camel and pascal casing.

The first change that could be made is to use a SQL alias to change the **categoryName** property to **category**. This change is accomplished by adding an `AS` keyword to the existing query:

```sql
SELECT
    p.name, 
    p.categoryName AS category,
    p.price
FROM 
    products p
WHERE
    p.price >= 50 AND
    p.price <= 100
```

This new query will result in this JSON output:

``` sql
{
    "name": "LL Bottom Bracket",
    "category": "Components, Bottom Brackets",
    "price": 53.99
}
```

The following change will require us to think about how we want to change the structure of our JSON output. Before changing the query, we need to think about how our JSON object should change. We, essentially, need to create a child JSON object. In this example, we have a child `scannerData` object with a property for `price`:

```json
{
    "name": "LL Bottom Bracket",
    "category": "Components, Bottom Brackets",
    "scannerData": {
        "price": 53.99
    }
}
```

How does this affect the query? We need to create a field that defines a JSON object with a single property named price that references the **p.price** property and an alias of **scannerData**. This expression would look like this:

```sql
{ "price": p.price } AS scannerData
```

Altogether, the entire query looks like this:

```sql
SELECT
    p.name, 
    p.categoryName AS category,
    { "price": p.price } AS scannerData
FROM 
    products p
WHERE
    p.price >= 50 AND
    p.price <= 100
```

## Reviewing specific properties in query results

Sometimes you want to shape your query results to drill down to specific properties. Two keywords are useful in these scenarios.

First, consider a scenario where you would like to find all of the category names in your container. You could use this query to get all of the container names for every item:

```sql
SELECT
    p.categoryName
FROM
    products p
```

This would return a JSON result set

Unfortunately, there would be repeated values within the result set:

```json
[
    {
        "categoryName": "Components, Road Frames"
    },
    {
        "categoryName": "Components, Touring Frames"
    },
    {
        "categoryName": "Bikes, Touring Bikes"
    },
    {
        "categoryName": "Clothing, Vests"
    },
    {
        "categoryName": "Accessories, Locks"
    },
    {
        "categoryName": "Components, Pedals"
    },
    {
        "categoryName": "Components, Touring Frames"
    },
...
```

Instead, you can use the `DISTINCT` keyword only to return unique values in the result set.

```sql
SELECT DISTINCT
    p.categoryName
FROM
    products p
```

Let’s consider another scenario. If your .NET developers wanted to consume this list of category names, they would need to create a C# wrapper class to consume this list:

```cs
public class CategoryReader
{
    public string CategoryName { get; set; }
}

// Developers read this as List<CategoryReader>
```

This extra step is both needless and unnecessary. It can quickly become cumbersome as you will need to do this multiple times for multiple types in your container[s]. But, if you have a query that returns an object with only a single property, you can use the `VALUE` keyword to flatten the result set to an array of a simple type.

```sql
SELECT DISTINCT VALUE
    p.categoryName
FROM
    products p
```

```json
[
    "Components, Road Frames",
    "Components, Touring Frames",
    "Bikes, Touring Bikes",
    "Clothing, Vests",
    "Accessories, Locks",
    "Components, Pedals",
...
```

```cs
// Developers read this as List<string>
```

The `VALUE` keyword can even be used on its own without the `DISTINCT` keyword:

```sql
SELECT VALUE
    p.name
FROM
    products p
```

```json
[
    "LL Road Frame - Red, 60",
    "LL Touring Frame - Blue, 58",
    "Touring-1000 Yellow, 54",
    "Classic Vest, L",
    "Cable Lock",
    "ML Road Pedal",
    "LL Touring Frame - Yellow, 62",
...
```

---

# Implement type-checking in queries

One of Azure Cosmos DB SQL API’s advantages as a data store is its flexibility to store data with varying structures and shapes. As the developer crafting queries for this data, the responsibility for type checking will often fall on your queries. The SQL query language for the SQL API includes a suite of built-in functions to make it possible for you to check the types of properties or expressions on the fly when they are variable or unknown.

Up until now, we have had a sample data structure that is well known and understood. But let’s consider some possible exceptions.

Each **product** item in the container has a property named **tags**. The tags property is an array of objects with **id** and **name** properties. The assumption, until now, is that the tags array always exists for every product in the container. But if we remove that baseline assumption, we could have a situation where a new product item is inserted into the container without a tag property such as this example:

```json
{
    "id": "6374995F-9A78-43CD-AE0D-5F6041078140",
    "categoryid": "3E4CEACD-D007-46EB-82D7-31F6141752B2",
    "sku": "FR-R38R-60",
    "name": "LL Road Frame - Red, 60",
    "price": 337.22
}
```

First, we can use the `IS_DEFINED` built-in function to check if the tags property exists at all in this item:

```sql
SELECT
    IS_DEFINED(p.tags) AS tags_exist
FROM
    products p
```

```json
[
    {
        "tags_exist": false
    }
]
```

Let’s say that the tags property does exist, but it’s not an array; it’s another type of property:

```json
{
    "id": "6374995F-9A78-43CD-AE0D-5F6041078140",
    "categoryid": "3E4CEACD-D007-46EB-82D7-31F6141752B2",
    "sku": "FR-R38R-60",
    "name": "LL Road Frame - Red, 60",
    "price": 337.22,
    "tags": "fun, sporty, rad"
}
```

We can use the `IS_ARRAY` built-in function to check if the tags property is an array:

```sql
SELECT
    IS_ARRAY(p.tags) AS tags_is_array
FROM
    products p
```

We can also check if the tags property is *null* or not using the `IS_NULL` built-in function:

```sql
SELECT
    IS_NULL(p.tags) AS tags_is_null
FROM
    products p
```

There are even more built-in functions for different scenarios involving other data types.

For example, consider a situation where different data stores persist pricing information inconsistently. Some persist pricing information using string data, while others may store pricing information using numbers. The built-in `IS_NUMBER` function could be used in a WHERE expression of our queries:

```sql
SELECT
    p.id,
    p.price, 
    (p.price * 1.25) AS priceWithTax
FROM
    products p
WHERE
    IS_NUMBER(p.price)
```

We could also use the built-in `IS_STRING` function to see if our price is a string and not apply any formatting:

```sql
SELECT
    p.id,
    p.price
FROM
    products p
WHERE
    IS_STRING(p.price)
```

There are other built-in type checking functions including `IS_OBJECT` and `IS_BOOLEAN`.

---

# Use built-in functions

The SQL query language for the Azure Cosmos DB SQL API ships with built-in functions for common tasks in a query. In this unit, we will walk through a brief set of examples of those functions.

Let’s start with an example where the name and the category are concatenated in the query result. For this example, the CONCAT built-in string function is used to concatenate these two fields together with a single vertical bar in the middle:

```sql
SELECT VALUE
    CONCAT(p.name, ' | ', p.categoryName)
FROM
    products p
```

For the next example, the query returns a flattened array with a single field, sku. Unfortunately, the sku may, or may not, be in lowercase. To solve for this, the LOWER built-in function is used to manipulate the string to all lowercase characters.

```sql
SELECT VALUE 
    LOWER(p.sku) 
FROM 
    products p
```

For this last example, the query is intended to filter out products that shouldn’t be retired yet by using the GetCurrentDateTime built-in function in a WHERE expression:

```sql
SELECT 
    *
FROM
    products p
WHERE
    p.retirementDate >= GetCurrentDateTime()
```

> :bulb: Tip
> 
> This is not a comprehensive list of built-in functions for the Azure Cosmos DB SQL API query language.

---

# Execute queries in the SDK

The Microsoft.Azure.Cosmos.Container class in the SDK has a series of built-in classes to create a query, issue the query to Azure Cosmos DB SQL API, set up an asynchronous stream in C#, and return items efficiently back to the client.

To start, let’s use a straightforward SQL command that returns all products:

```sql
SELECT * FROM products p
```

The equivalent command definition in C# would use the **QueryDefinition** class:

```cs
QueryDefinition query = new ("SELECT * FROM products p");
```

First, define a C# type to represent the type of item you will query, for this example, a simple C# **Product** class will suffice:

```cs
public class Product
{
    public string id { get; set; }

    public string name { get; set; }

    public string price { get; set; }
}
```

Next, use the **GetItemQueryIterator** generic method with the C# **Product** type to in the **await foreach** loop. The asynchronous stream structure will automatically handle the looping and pagination to go to the server and get each subsequent page of results. Within the foreach loop, add your code to handle each item; in this example, each item’s **id**, **name**, and **price** is output to the console:

```cs
using (FeedIterator<Product> feedIterator = this.Container.GetItemQueryIterator<Product>(
    query,
    null,
    new QueryRequestOptions() { }))
{
    while (feedIterator.HasMoreResults)
    {
        foreach(var item in await feedIterator.ReadNextAsync())
        {
            Console.WriteLine($"[{item.productid}]\t{item.name,35}\t{item.price,15:C}");
        }
    }
}
```

---

# Exercise: Execute a query with the Azure Cosmos DB SQL API SDK

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/query-azure-cosmos-db-sql-api/8-exercise-execute-sdk)

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

1. You are tasked with writing the shortest possible SQL query to select all items and include all fields from your container. Which of these queries is both valid and would accomplish this task?

- [ ] SELECT FROM
- [ ] SELECT FROM p
- [x] SELECT * FROM p
> That's correct. This query includes the data source and the specification for which fields should be returned.

2. Which SQL keyword is used to flatten your query results into an array of a specific field value for each item?

- [ ] FROM
- [x] VALUE
> That's correct. The VALUE keyword specifies that the JSON value should be retrieved for each item instead of the complete object.
- [ ] DISTINCT

3. You have a container named products. What must the data source be named after the FROM keyword in the query?

- [ ] p
- [ ] products
- [x] Any of these
> That's correct. The data source can have a unique name that's distinct from the container's name.

---

# Summary

In this module, you created your first queries using the SQL query language in Azure Cosmos DB SQL API. You learned how similar how to query and project fields from items into JSON results.

Now that you have completed this module, you can:

* Project fields from items into JSON results
* Use built-in functions in your query filters and fields