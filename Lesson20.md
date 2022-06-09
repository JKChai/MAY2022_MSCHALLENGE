# Customize an indexing policy in Azure Cosmos DB SQL API

Tune the indexing policy based on your SQL queries in Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Customize an indexing policy for read-heavy workloads
* Customize an indexing policy for write-heavy workloads

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Azure Cosmos DB SQL API is schema-agnostic. Out of the box, the default indexing policy includes all possible paths and values for a good balance between the thoroughness of the index and performance. You can tune the indexing policy to better match your application’s requirements as you build more complex solutions. In this module, you will explore a few common patterns for adjusting an indexing policy and the resultant index.

After completing this module, you'll be able to:

* Customize an indexing policy for read-heavy workloads
* Customize an indexing policy for write-heavy workloads

---

# Index Usage

The query engine evaluates query filters and then traverses the index of your container. The query engine will automatically try to use the most efficient of the following three methods of evaluating filters:

| Method | Description | RU implication |
|-----|-----|-----|
| Index seek | The query engine will seek an exact match on a field’s value by traversing directly to that value and looking up how many items match. Once the matched items are determined, the query engine will return the items as the query result. | The RU charge is constant for the lookup. The RU charge for loading and returning items is linear based on the number of items. |
| Index scan | The query engine will find all possible values for a field and then perform various comparisons only on the values. Once matches are found, the query engine will load and return the items as the query result. | The RU charge is still constant for the lookup, with a slight increase over the index seek based on the cardinality of the indexed properties. The RU charge for loading and returning items is still linear based on the number of items returned. |
| Full scan | The query engine will load the items, in their entirety, to the transactional store to evaluate the filters. | This type of scan does not use the index; however, the RU charge for loading items is based on the number of items in the entire container. |

> :grey_exclamation: Note
> 
> An index scan can range in complexity from an efficient and precise index scan, to a more involved expanded index scan, and finally the most complex full index scan.

As a query developer, it’s essential to understand which queries will require a seek vs. a scan. It’s also important to understand which queries cannot use the index and will require a full scan. Specifically, you should optimize your queries to use filter predicates that use the most efficient lookup method.

Let’s illustrate how properties and queries can influence the lookup method. In this example, a fictional container has three items with unique identifiers, a **name** string property, and a **price** number property.

```json
[
  {
    "id": "1",
    "name": "Touring-1000 Blue",
    "price": 675.55
  },
  {
    "id": "2",
    "name": "Mountain-400-W Silver",
    "price": 1215.40
  },
  {
    "id": "3",
    "name": "Road-200 Red",
    "price": 405.85
  }
]
```

Each of these items could be visualized as a tree.

For the first item, the tree representation would include a name node with `Touring-1000 Blue` and a `price` node with a child value node of `675.55`.

![Tree for JSON item with a a name of "Touring-1000 Blue" and a price of $675.55](https://docs.microsoft.com/en-us/learn/wwl-data-ai/choose-indexes-azure-cosmos-db-sql-api/media/2-document-1.png)

The tree for the second item would have the same nodes with the values `Mountain-400-W Silver` and `1215.40`, respectively.

![Tree for JSON item with a a name of "Mountain-400-W Silver" and a price of $1,215.40](https://docs.microsoft.com/en-us/learn/wwl-data-ai/choose-indexes-azure-cosmos-db-sql-api/media/2-document-2.png)

The final item’s tree is similar with the values `Road-200 Red` and `405.85`.

![Tree for JSON item with a a name of "Road-200 Red" and a price of $405.85](https://docs.microsoft.com/en-us/learn/wwl-data-ai/choose-indexes-azure-cosmos-db-sql-api/media/2-document-3.png)

An inverted tree that includes all three items would have a root node that matches all three items. You can think of traversing the root node as similar, in concept, to a query with no filter. The tree contains a price node with three child nodes for each distinctive value. The price node matches all three items because every item includes a price field. However, each individual price node only matches a single item since the price for each item is distinct. The tree also contains a name node that matches all three items with individual child nodes for each distinct value.

![Inverted tree for all three JSON items](https://docs.microsoft.com/en-us/learn/wwl-data-ai/choose-indexes-azure-cosmos-db-sql-api/media/2-inverted-tree-alt.png)

To traverse the tree, a simple SQL query is written with a filter to only match items where the `name` is equivalent to the value of `Touring-1000 Blue`.

```sql
SELECT 
    *
FROM
    products p
WHERE
    p.name = 'Touring-1000 Blue'
```

The query engine could then traverse the tree in the following order:

1. The engine will start at the root. Right now, all items are still potential matches.
2. The engine will traverse the `name` node. Still, all items match.
3. Finally, the engine will traverse the exact match `Touring-1000 Blue` node. Only item `#2` matches at this point
4. The query engine will then load the entire JSON content for item `#2` and return that in the result set.

This tree diagram illustrates the traversal process down to the `Touring-1000 Blue` node.

![Search of inverted tree for an exact match on a field's value](https://docs.microsoft.com/en-us/learn/wwl-data-ai/choose-indexes-azure-cosmos-db-sql-api/media/2-search-name-equality.png)

This traversal is an example of the **index seek** lookup method in action. The actual matching of an exact value is a flat charge in RU/s since the query engine uses the index instead of searching in each item’s JSON content. Once the matched items are found, the query engine will load the JSON content to return to the client application.

If the query filter doesn’t match any known value, no items will be returned in the result set. If multiple items have the same value for the field, the tree will direct the query engine to return multiple items.

Another example of an index seek would be a SQL query that uses the **IN** filter to perform an equality match on multiple possible fields.

```sql
SELECT 
    *
FROM
    products p
WHERE
    p.name IN ('Road-200 Red', 'Mountain-400-W Silver')
```

The query engine then traverses the tree in the following order:

1. The engine will start at the root. Right now, all items are still potential matches.
2. The engine will traverse the `name` node. Still, all items match.
3. Finally, the engine will traverse the exact matches. This includes the `Road-200 Red` and `Mountain-400-W Silver` nodes. Only items `#1` and `#3` matches at this point
4. The query engine will then load the entire JSON content for both items `#1` and `#3` and then return that in the result set.

This tree diagram illustrates the traversal process for the matching child values in the `name` node.

![Search of an inverted tree for multiple matches on a field's value](https://docs.microsoft.com/en-us/learn/wwl-data-ai/choose-indexes-azure-cosmos-db-sql-api/media/2-search-name.png)

Some queries use other operators that necessitate the use of a more complex index lookup method. In this example SQL query, items are filtered based on two range comparisons. In plain language, the query looks for items whose price is between **$500** and **$1,000**.

```sql
SELECT
    *
FROM
    products p
WHERE
    p.price >= 500 AND
    p.price <= 1000
```

The query engine will traverse the tree in the following order:

1. The engine will start at the root. Right now, all items are still potential matches.
2. The engine will traverse the `price` node. Still, all items match.
3. The query engine will perform a binary search on all possible values to indicate whether they match the filter or not.
4. Finally, the engine will map the matches values to their items. Only the `675.55` node match. Then, only item `#1` matches at this point.
5. The query engine will then load the entire JSON content for item `#1` and then return that in the result set.

![Search of an inverted tree for all values for a specific field](https://docs.microsoft.com/en-us/learn/wwl-data-ai/choose-indexes-azure-cosmos-db-sql-api/media/2-search-price-range.png)

This binary search moved the query, in complexity, from an index seek to a precise index scan. The query engine scans all possible values in the index, but this is still far more efficient than a full scan of all JSON content. Queries that use range comparisons and string functions often will require the use of an index scan.

There are edge cases where the query engine cannot use the index to evaluate a filter. These cases will require the query engine to load the JSON content of all items into the transactional store before evaluating the filter. Full scans can potentially have significant request unit charges as the charge scales linearly with the total number of items in the container. While full scans are rare, it is essential to know that they are possible when using specific built-in functions in query filters.

---

# Review read-heavy index patterns

Some applications' workloads are read-centric, requiring SQL queries that filter on many different fields in each item. These read-centric workloads benefit from having an inverted index that includes as many fields as possible to maximize query performance and minimize request unit charges.

By default, Azure Cosmos DB SQL API creates containers that include an indexing policy that includes the root path. This strategy effectively includes all possible JSON properties in the index. The default indexing policy also excludes the eTag property path by default. In the inverted index, the result will be an index that indexes all properties except eTag.

This is the complete JSON object for the default indexing policy:

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
      "path": "/\"_etag\"/?"
    }
  ]
}
```

Let’s consider a JSON object for a product with a metadata object with fields never used in any queries and a description property used in query results but never included in a filter due to its sheer size.

```json
{
  "id": "3324789",
  "name": "Road-200 Green",
  "price": 510.55,
  "description": "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Cras faucibus, turpis ut pulvinar bibendum, sapien mauris fermentum magna, a tincidunt magna diam tincidunt enim. Fusce convallis justo nulla, at tristique diam tempus vel. Suspendisse potenti. Curabitur rhoncus neque vel elit condimentum finibus. Nullam porta lorem vitae enim tincidunt elementum. Vestibulum id felis sit amet neque commodo scelerisque. Suspendisse euismod ex ut hendrerit eleifend. Quisque euismod consectetur vulputate.",
  "metadata": {
    "created_by": "sdfuouu",
    "created_on": "2020-05-05T19:21:27.0000000Z",
    "department": "cycling",
    "sku": "RD200-G"
  }
}
```

When designing an indexing policy, you should consider the needs of your SQL queries. In this fictional example, the **description** and **metadata** properties are never used in SQL queries. The easiest way to design this indexing policy is to include all paths and then selectively exclude the **description** and **metadata** property paths.

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
      "path": "/description/?"
    },
    {
      "path": "/metadata/*"
    }
  ]
}
```

Alternatively, you can exclude all paths and only selectively include the name and price property paths.

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {
      "path": "/name/?"
    },
    {
      "path": "/price/?"
    }
  ],
  "excludedPaths": [
    {
      "path": "/*"
    }
  ]
}
```

> :bulb: Tip
>
> The drawback to this approach is that you will need to update the index anytime you change your schema.

---

# Review write-heavy index patterns

If you want your SQL queries to be endlessly flexible, why exclude any paths at all? Well, each insert or update operation requires the indexer to run to update the inverted index with data from your newly created or updated item. More oversized items, or bulk workloads, can cause the indexing to use many RU/s or take a significant amount of time.

Let’s consider an example JSON object that is much larger than previous examples.

```json
{
  "id": "3324734",
  "name": "Road-200 Green",
  "internal": {
    "tracking": {
      "id": "eac06d51-2462-4bfb-8eb6-46281da16f8e"
    }
  },
  "inStock": true,
  "price": 1303.33,
  "description": "Consequat dolore commodo tempor pariatur consectetur fugiat labore velit aliqua ut anim. Et anim eu ea reprehenderit sit ullamco elit irure laborum sunt ea adipisicing eu qui. Officia commodo ad amet ea consectetur ea est fugiat.",
  "warehouse": {
    "shelfLocations": [
      20,
      37,
      35,
      27,
      38
    ]
  },
  "metadata": {
    "color": "brown",
    "manufacturer": "Fabrikam",
    "supportEmail": "support@fabrik.am",
    "created_by": "sdfuouu",
    "created_on": "2020-05-05T19:21:27.0000000Z",
    "department": "cycling",
    "sku": "RD200-B"
  },
  "tags": [
    "pariatur",
    "et",
    "commodo",
    "ex",
    "tempor",
    "esse",
    "nisi",
    "ullamco",
    "Lorem",
    "ullamco",
    "ex",
    "ea",
    "laborum",
    "tempor",
    "consequat"
  ]
}
```

Using the default indexing policy, this entire item and all paths are indexed and added to the inverted index each time you create a new item or update an existing item. Indexing runs over the entire item, even if you update a single property in the item.

To combat this, you can use an indexing policy that excludes all paths except for the ones you require for your SQL queries. Let’s consider a scenario where our application only issues the following two SQL queries:

```sql
SELECT 
    * 
FROM 
    products p
WHERE
    p.price >= <numeric-value> AND
    p.price <= <numeric-value>
```

```sql
SELECT 
    * 
FROM 
    products p
WHERE
    p.price = <numeric-value>
```

An indexing policy that excludes all paths, except for the **price** property path, would be appropriate here. This policy will still index items, but it will do so quickly because only one property is added to the inverted index.

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {
      "path": "/price/?"
    }
  ],
  "excludedPaths": [
    {
      "path": "/*"
    }
  ]
}
```

> :bulb: Tip
>
> Again, the drawback to this approach is that you will need to update the index anytime you change your schema.

Here is a diagram of the inverted index showing that it only has a single property to traverse and then multiple potential values.

![Inverted tree with a single property node of price and multiple child nodes](https://docs.microsoft.com/en-us/learn/wwl-data-ai/choose-indexes-azure-cosmos-db-sql-api/media/4-inverted-tree.png)

Suppose your application is write-heavy and only ever does point reads using the `id` and `partition key` values. In that case, you can choose to disable indexing entirely using a customized indexing policy.

```json
{
  "indexingMode": "none",
}
```

---

# Exercise - Optimize an Azure Cosmos DB SQL API container's index policy for common operations

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/choose-indexes-azure-cosmos-db-sql-api/5-exercise-optimize-containers-index-policy-common-operations)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge Check

1. Your team has written a SQL query for Azure Cosmos DB SQL API with the following text: SELECT * FROM c WHERE c.sku = 'RD3387G'. Which lookup method will the query engine use for the sku filter?

- [ ] Full scan
- [ ] Index scan.
- [x] Index seek.
> Correct. The equality filter allows the query engine to perform a seek of a specific value.

2. Which property of an indexing policy should you set to disable all indexing for a container?

- [x] indexingMode
> Correct. Setting the indexingMode property to none disables all indexing.
- [ ] excludedPaths
- [ ] automatic

--- 

# Summary

In this module, you explored the common read-heavy and write-heavy application patterns. You also explored common indexing policy patterns to support these patterns.

Now that you have completed this module, you can:

* Customize an indexing policy based on whether your workload is read or write centric

