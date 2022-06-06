# Define indexes in Azure Cosmos DB SQL API

Discover indexes and indexing policies in Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Create and execute a SQL query
* Project query results
* Use built-in functions in a query

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

By default, Azure Cosmos DB automatically indexes all paths of items stored using the SQL API. The automatic indexing feature is excellent for developing new applications as you can create complex queries almost immediately. In this module, you will learn how Azure Cosmos DB SQL API indexes JSON items and how the default indexing policy makes it easier to get started building applications quickly.

After completing this module, you'll be able to:

* Describe an index and indexing policy in Azure Cosmos DB SQL API
* Identify indexing policy data types in Azure Cosmos DB SQL API

---

# Understand Indexes

Every Azure Cosmos DB SQL API container has a built-in policy that determines how each item should be indexed. By default, this policy dictates that create, update, or delete operations for any item should update the index and that the index should include all properties of every item. This intelligent default is excellent at the start of many solutions as you get good and predictable query performance without having to dive too deeply into tuning an index.

Let’s review an example of the default policy in action.

Here, we have a JSON object representing a product named **Touring-1000 Blue** with three tags (**bike**, **touring**, and **blue**). You should pay attention to the number of things in the **tags** array.

```json
{
  "name": "Touring-1000 Blue",
  "tags": [
    {
      "name": "bike"
    },
    {
      "name": "touring"
    },
    {
      "name": "blue"
    }
  ]
}
```

If we were to represent this JSON object as a tree, this representation would include traversal paths for both the **name** property and its value (**Touring-1000 Blue**). The tree also contains three traversal paths for the three objects in the **tags** array, each with a leaf node for their **name** properties and respective values.

!['Visual tree representation of the Touring-1000 Blue JSON object'](https://docs.microsoft.com/en-us/learn/wwl-data-ai/define-indexes-azure-cosmos-db-sql-api/media/2-tour-tree.png)

As a counterpoint, here is another JSON object representing a product named **Mountain-400-W Silver** that only contains two tags (**bike** and **silver**). This object is also unique in that it includes a **sku** property with a value of **BK-M38S-38**.

```json
{
  "name": "Mountain-400-W Silver",  
  "sku": "BK-M38S-38",
  "tags": [
    {
      "name": "bike"
    },
    {
      "name": "silver"
    }
  ]
}
```

The tree representation for this JSON object includes simple traversal paths for the **name** and **sku** properties. It also contains two traversal paths for the two objects in the **tags** array and corresponding leaf nodes for their **name** properties and respective values.

![Visual tree representation of the Mountain-400-W Silver JSON object](https://docs.microsoft.com/en-us/learn/wwl-data-ai/define-indexes-azure-cosmos-db-sql-api/media/2-mountain-tree.png)

To conceptualize our container’s index, we tend to think of the index as a union of all trees for each item in the container. Altogether, this creates an **inverted index** which gives our database engine something fast and efficient to traverse when performing query operations. Each node in the tree has metadata indicating which items in our index matches that specific node.

![Inverted index for both the Touring-1000 Blue and Mountain-400-W Silver JSON objects](https://docs.microsoft.com/en-us/learn/wwl-data-ai/define-indexes-azure-cosmos-db-sql-api/media/2-inverted-tree.png)

To illustrate this example, consider the following SQL query:

```sql
SELECT * FROM products p WHERE p.sku = 'BK-M38S-38'
```

Instead of traversing through each item individually, the search engine will traverse the inverted index. For this query, let’s walk through a sample traversal:

1. First, the search engine will start at the root. As of now, all items match.
2. Next, the search will traverse the **sku** property. Now, only the **#2** item matches.
3. Finally, the search will end at the **BK-M38S-28** node. Still, only the **#2** item matches.

The search results are that the #2 item (**Mountain-400-W Silver**) matches, and the SQL query will return all fields from this item.

![Example of a search traversal of the inverted index highlighting only the sku property and BK-M38S-38 value path](https://docs.microsoft.com/en-us/learn/wwl-data-ai/define-indexes-azure-cosmos-db-sql-api/media/2-search-tree-01.png)

Using another example SQL query:

```sql
SELECT p.id FROM products p WHERE p.name = 'Touring-1000 Blue'
```

You can walk through a similar traversal:

1. Starting at the root, all items match.
2. Moving to the **name** property, Still, all items match.
3. Finally, ending at the **Touring-1000 Blue** node, Only the **#1** item matches.

The search results are that the #1 item (**Touring-1000 Blue**) matches, and the SQL query will return only the id field from this item.

!["Example of a search traversal of the inverted index highlighting only the name property and Touring-1000 Blue value path"](https://docs.microsoft.com/en-us/learn/wwl-data-ai/define-indexes-azure-cosmos-db-sql-api/media/2-search-tree-02.png)

---

# Understand Indexing Policies

All data in Azure Cosmos DB SQL API containers is indexed by default. This occurs because the container includes a default **indexing policy** that’s applied to all newly created containers. The default indexing policy consists of the following settings:

* The inverted index is updated for all create, update, or delete operations on an item
* All properties for every item is automatically indexed
* Range indexes are used for all strings or numbers

Indexing policies are defined and managed in JSON. The default indexing policy for a new container in Azure Cosmos DB SQL API includes the following components:

| Component | Description | Default value |
|-----|-----|-----|
| Indexing mode | Configures whether indexing is enabled (Consistent) or not (None) for the container | Consistent |
| Automatic | Configures whether automatically indexes items as they are written | Enabled |
| Included paths | Set of paths to include in the index | All (*) |
| Excluded paths | Set of paths to exclude from the index | _etag property path |

The default indexing policy, in JSON, contains the following content:

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

## Configure indexing mode

There are two primary indexing mode options for most Azure Cosmos DB SQL API containers.

The **consistent** indexing mode updates the index synchronously as your perform individual operations that modify an item (create, update, or delete). This indexing mode will be the standard choice for most containers to ensure the index is updated as items change.

The **none** indexing mode completely disables indexing on a container. This indexing mode is a scenario-specific mode where the indexing operation is either unnecessary or could impact the solution's overall performance. Two examples include:

* A bulk operation to create, update, or delete multiple documents may benefit from disabling indexing during the bulk execution period. Once the bulk operations are complete, the indexing mode can be switched back to **consistent**.
* Solutions that use containers as a pure key-value store only perform point-read operations. These containers do not benefit from the secondary indexes created by running the indexer.

## Including and excluding paths

Indexing policies specify paths that are either explicitly included or excluded from the index. Using a path syntax, you can author expressions to describe various paths within a JSON item.

Consider this JSON object that represents a product item in our Azure Cosmos DB SQL API container:

```json
{
  "id": "8B363B8B-378E-402A-9E68-A935302000B8",
  "name": "HL Touring Frame - Yellow, 46",
  "category": {
    "id": "F3FBB167-11D8-41E4-84B4-5AAA92B1E737",
    "name": "Components, Touring Frames"
  },
  "metadata": {
    "sku": "FR-T98Y-46"
  },
  "price": 1003.91,
  "tags": [
    {
      "name": "accessory"
    },
    {
      "name": "yellow"
    },
    {
      "name": "frame"
    }
  ]
}
```

Three primary operators are used when defining a property path:

* The `?` operator indicates that a path terminates with a string or number (scalar) value
* The `[]` operator indicates that this path includes an array and avoids having to specify an array index value
* The `*` operator is a wildcard and matches any element beyond the current path

Using these operators, you can create a few example property path expressions for the example JSON item:

| Path expression | Description |
|-----|-----|
| /* | All properties |
| /name/? | The scalar value of the name property |
| /category/* | All properties under the category property |
| /metadata/sku/? | The scalar value of the metadata.sku property |
| /tags/[]/name/? | Within the tags array, the scalar values of all possible name properties |

--- 

# Reviewing Indexing Policy Strategies


In the simplest terms, an indexing policy is really two sets of include/exclude expressions that are evaluated to determine which actual properties are indexed. By this very nature, conflicts between include and exclude paths will occur, as the existence of conflicts is by design. When a conflict occurs, the expression with the most precision takes precedence. For example, let’s evaluate these two include and exclude path sets:

* Included path: `/category/name/?`
* Excluded path: `/category/*`

The exclude path excludes all possible properties within the **category** path, however the include path is more precise and specifically includes the **category.name** property. The result is that all properties within **category** are not indexed, with the sole exception being the **category.name** property.

Indexing policies must include the root path and all possible values (`/*`) as either an included or excluded path. More customizations exist as a level of precision beyond that base. This leads to two fundamental indexing strategies you will see in many examples.

First, some indexing policies include all possible properties in the root path, and use the excluded paths list to exclude specific properties or paths as appropriate. This strategy is the case for the default indexing policy. What follows is an example indexing policy that includes all properties except **category.id**:

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
      "path": "/category/id/?"
    }
  ]
}
```

> :bulb: Tip
> 
> In general, it's better to include all paths by default and only exclude certain paths from the index. With this strategy, you can modify your schema and the index will still work with the new set of properties.

Second, other indexing policies may choose to exclude all possible properties in the root path and then selectively include specific properties of paths. This example indexing policy excludes all properties and selectively indexes only the **name** and **tags[].name** properties:

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {
      "path": "/name/?"
    },
    {
      "path": "/tags/[]/name/?"
    }
  ],
  "excludedPaths": [
    {
      "path": "/*"
    }
  ]
}
```

---

# Exercise: Review the default index policy for an Azure Cosmos DB SQL API container with the portal

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/define-indexes-azure-cosmos-db-sql-api/5-exercise-review-default-index-policy-for-container-portal)

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

1. You have created a bulk operation in Azure Cosmos DB SQL API that will insert hundreds of thousands of items into a container. You need to temporarily disable indexing and re-enable it after the bulk operation is complete. Prior to starting the bulk operation, which indexing mode should you configure in the indexing policy?

- [x] None
> That's correct. Setting the indexing mode to none will disable indexing. This setting can be changed back at a later point in time.
- [ ] Consistent
- [ ] Automatic

2. You are authoring a new indexing policy for a container in Azure Cosmos DB SQL API. You would like to define an included path that includes all possible properties from the root of any JSON document. Which path expression should you use?

- [x] /*
> That's correct. The wildcard operator is used to match any elements below the referenced node.
- [ ] /[]
- [ ] /?

---

# Summary

In this module, you explored the underlying mechanisms of indexes and indexing policies in Azure Cosmos DB SQL API. You reviewed the default indexing policy and then parsed the policy to understand the default behavior of queries against a new container.

Now that you have completed this module, you can:

* Describe the relationship between the index and a corresponding indexing policy
* Identify the various indexing policy data types