# Measure index performance in Azure Cosmos DB SQL API

Measure the performance of an indexing policy in Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Tune an indexing policy for specific queries
* Measure the cost for a query or operation

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

As you evaluate your indexing policy, it makes sense to track both the index’s metrics and the number of RU/s charged per query or operation. In this module, you will use request option classes in the .NET SDK to track metadata about your queries and operations.

After completing this module, you'll be able to:

* Tune an indexing policy for specific queries
* Measure the cost for a query or operation

---

# Enable indexing metrics

You may often wonder how your index and indexing policy specifically impact a SQL query you authored. Azure Cosmos DB SQL API includes opt-in indexing metrics that illuminate how the current state of the index affects your query filters. The indexing metrics will go even further and recommend single or composite indexes to improve future query performance.

Using the Azure Cosmos DB SQL API SDK for .NET, you can build a typical query request to select all items from your container.

```cs
Container container = client.GetContainer("cosmicworks", "products");

string sql = "SELECT * FROM products p";

QueryDefinition query = new(sql);

FeedIterator<Product> iterator = container.GetItemQueryIterator<Product>(query);

while(iterator.HasMoreResults)
{
    FeedResponse<Product> response = await iterator.ReadNextAsync();
    foreach(Product product in response)
    {
        Console.WriteLine($"[{product.id}]\t{product.name,35}\t{product.price,15:C}");       
    }    
}
```

This sample code creates a **QueryDefinition** variable with the SQL query and then passes in that variable to the **GetItemQueryIterator<>** method of the **Container** class. Once the iterator is available, the code uses a combination of a while loop and a foreach loop to iterator over pages of results as long as there is another page available.

To enable index metrics, you must first create an object of type `QueryRequestOptions` and set the `PopulateIndexMetrics` property to true.

> :grey_exlcamation: Note
>
> By default, `PopulateIndexMetrics` is disabled. You should only enable this if you are troubleshooting query performance or are unsure how to modify your indexing policy.

```cs
QueryRequestOptions options = new()
{
    PopulateIndexMetrics = true
};
```

Once created, you can pass in the options variable as an extra parameter to the `GetItemQueryIterator<>` method of the `Container` class.

```cs
FeedIterator<Product> iterator = container.GetItemQueryIterator<Product>(query, requestOptions: options);
```

Next, you can iterate over results like normal using a combination of a while and foreach loop. However, the `FeedResponse<>` object within the while loop will contain an `IndexMetrics` string property with information about the query’s performance relative to the current index.

In this example, the content of the `IndexMetrics` property is sent to the console output.

```C#
while(iterator.HasMoreResults)
{
    FeedResponse<Product> response = await iterator.ReadNextAsync();
    foreach(Product product in response)
    {
        // Do something with each product
    }

    Console.WriteLine(response.IndexMetrics);    
}
```

---

# Analyze indexing metrics results

Let's review a few sample queries to understand better how you can use the information in the indexing metrics.

This first SQL query returns all items in the container with a price property whose value is greater than 500.

```sql
SELECT 
    * 
FROM 
    products p 
WHERE 
    p.price > 500
```

All queries in this example assume that we are using the default indexing policy that:

* Uses the `consistent` indexing mode and automatically indexes items
* Includes all property paths
* Expressly excludes the `eTag` property path
* Does not contain any composite indexes

If you run this query and output the indexing metrics, you will find that the metrics indicate the `/price/?` field in the index is impacted significantly by this query. The default indexing policy includes this field, so the indexing metrics do not recommended to add this field to the index.

```bash
Index Utilization Information
  Utilized Single Indexes
    Index Spec: /price/?
    Index Impact Score: High
    ---
  Potential Single Indexes
  Utilized Composite Indexes
  Potential Composite Indexes
```

The filter is slightly more complex for the following query by using a built-in string function to find items whose `name` property starts with the term `Touring`.

```sql
SELECT 
    * 
FROM 
    products p 
WHERE 
    p.price > 500 AND 
    startsWith(p.name, 'Touring')
```

The indexing metrics for this query indicate that both the `/price/?` and `/name/?` property paths have a high impact. The default indexing policy covers both of these fields, so no extra indexes are recommended at this time.

```bash
Index Utilization Information
  Utilized Single Indexes
    Index Spec: /price/?
    Index Impact Score: High
    ---
    Index Spec: /name/?
    Index Impact Score: High
    ---
  Potential Single Indexes
  Utilized Composite Indexes
  Potential Composite Indexes
```

The final sample query uses an equality filter based on the value of the `categoryName` field. In this example, the field must equal `Bikes`, `Touring Bikes` per the filter expression.

```sql
SELECT 
    * 
FROM 
    products p 
WHERE 
    p.price > 500 AND 
    p.categoryName = 'Bikes, Touring Bikes'
```

Now, the indexing metrics will make a stronger recommendation. As expected, the impact of the `/price/?` and `/categoryName/?` property paths is high. However, the indexing metrics now recommend that we create a composite index with both property paths to improve the performance of future queries.

```bash
Index Utilization Information
  Utilized Single Indexes
    Index Spec: /price/?
    Index Impact Score: High
    ---
    Index Spec: /categoryName/?
    Index Impact Score: High
    ---
  Potential Single Indexes
  Utilized Composite Indexes
  Potential Composite Indexes
    Index Spec: /categoryName ASC, /price ASC
    Index Impact Score: High
    ---
```

You can create a new indexing policy with a composite index that includes both the `price` and `categoryName` property paths in ascending order to implement this recommendation.

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {
      "path": "/*"
    }
  ],
  "excludedPaths": [],
  "compositeIndexes": [
    [
      {
        "path": "/categoryName",
        "order": "ascending"
      },
      {
        "path": "/price",
        "order": "ascending"
      }
    ]
  ]
}
```

After allowing the index to update, you can then rerun the query in .NET to observe the latest indexing metrics. The new metrics metadata indicates that the composite index was utilized in the latest execution of the query.

```bash
Index Utilization Information
  Utilized Single Indexes
  Potential Single Indexes
  Utilized Composite Indexes
    Index Spec: /categoryName ASC, /price ASC
    Index Impact Score: High
    ---
  Potential Composite Indexes
```

---

# Measure query cost

The `QueryRequestOptions` class is also helpful in measuring the cost of a query in request units per second (or RU/s).

To start, set the `MaxItemCount` property of the `QueryRequestOptions` class to the number of items you would like to return in each result page. Invoke the `GetItemQueryIterator<>` method passing in the SQL query and the options variable.

```cs
QueryRequestOptions options = new()
{
    MaxItemCount = 25
};

FeedIterator<Product> iterator = container.GetItemQueryIterator<Product>(query, requestOptions: options);
```

Next, iterate over your results as usual with two minor changes:

1. Within each iteration of the while loop. Print the current RU/s charge for the result page to the console.
2. Aggregate all of the RU/s charges for each page of results, and then print the total RU/s consumed outside of the while loop.

```cs
double totalRUs = 0;
while(iterator.HasMoreResults)
{
    FeedResponse<Product> response = await iterator.ReadNextAsync();
    foreach(Product product in response)
    {
        // Do something with each product
    }

    Console.WriteLine($"RUs:\t\t{response.RequestCharge:0.00}");

    totalRUs += response.RequestCharge;
}

Console.WriteLine($"Total RUs:\t{totalRUs:0.00}");
```

The console output should render a RU/s charge for each page of results and then a final RU/s charge that is an aggregate (sum) of all previous values.

```bash
RUs:            2.82
RUs:            2.82
RUs:            2.83
RUs:            2.84
RUs:            2.25
Total RUs:      13.56
```

> :grey_exclamation: Note
>
> This sample is using a `SELECT * FROM products` query against a demo database generated using the **cosmicworks** command-line tool.

---

# Measure point operation cost

You can also use the .NET SDK to measure the cost, in RU/s, of individual operations. Since this is already included in a response header, there is no need to use a custom options object.

Here is an example block of C# code that gets a container, creates a new object, and then uses the `CreateItemAsync` method of the `Container` class to execute a create operation and store the results in a variable of type `ItemResponse<>`.

```cs
Container container = client.GetContainer("cosmicworks", "products");

Product item = new(
    $"{Guid.NewGuid()}",
    $"{Guid.NewGuid()}",
    "Road Bike",
    500,
    "rd-bk-500"
);

ItemResponse<Product> response = await container.CreateItemAsync<Product>(item);
```

The `ItemResponse<>` variable is helpful for many things, but this example only focuses on two uses.

First, the variable contains a `Resource` property that will output a deserialized instance of your item in the specified generic type. This resource is the item that was recently created server-side in Azure Cosmos DB SQL API.

Second, the variable contains a `RequestCharge` property that returns a value of type `double`, indicating the number of request units consumed by this operation.

```cs
Product createdItem = response.Resource;

Console.WriteLine($"RUs:\t{response.RequestCharge:0.00}");
```

Once you run the code, the output window will show a request unit charge for creating the item in a single operation.

```bash
RUs:    7.05
```

---

# Exercise - Optimize an Azure Cosmos DB SQL API container's index policy for a specific query

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/measure-index-azure-cosmos-db-sql-api/6-exercise-optimize-containers-index-policy-for-specific-query)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge Check

1. You're creating a SQL query that will run in the .NET SDK. Which property of the QueryRequestOptions class should you configure to enable viewing recommendations on single and composite indexes along with the impact of utilized indexes?

- [ ] MaxItemCount
- [x] PopulateIndexMetrics
> That's correct. The PopulateIndexMetrics property will return metadata about utilized and recommended indexes.
- [ ] EnableScanInQuery

2. You've created a SQL query in the .NET SDK and received the response in an object of type FeedResponse<>. Which property of the feed response should you use to determine how many request units were used per page of results?

- [ ] IndexMetrics
- [x] RequestCharge
> That's correct. This property returns a numeric value with the number of request units consumed per page or operation.
- [ ] Diagnostics

---

# Summary

In this module, you used the .NET SDK for Azure Cosmos DB SQL API to monitor index metrics and RU charges for queries and operations.

Now that you have completed this module, you can:

* Adjust an indexing policy based on your most used queries
* Measure the cost, in request units, for a specific query or operation