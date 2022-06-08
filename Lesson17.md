# Search Azure Cosmos DB SQL API data with Azure Cognitive Search

Index Azure Cosmos DB SQL API data with Azure Cognitive Search.

**Learning objectives**

After completing this module, you'll be able to:

* Create an indexer to migrate data from Azure Cosmos DB SQL API to an Azure Cognitive Search index

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Azure Cognitive Search is a cloud-native search service that provides an expanded set of query functionality over a large variety and volume of data. A search service is ideal for scenarios where your developer team would like to implement search-specific features such as filters, faceting, autocomplete, and synonym matching without the complexity of trying to implement these features directly in the database source. In this module, you will explore how Azure Cognitive Search can crawl data from the Azure Cosmos DB SQL API.

After completing this module, you'll be able to:

* Create an indexer to migrate data from Azure Cosmos DB SQL API to an Azure Cognitive Search index

---

# Create an indexer for data in Azure Cosmos DB SQL API

An **Azure Cognitive Search** instance is comprised of a few core components:

* **Indexes** that contain JSON documents that are searchable
* **Indexers** to crawl data from various data sources and insert them into indexes
* **Data Sources** that connect Azure Cognitive Search to various data platforms

![Diagram of an Azure Cognitive Services account where an indexer indexes data from a data source and stores the result in an index](https://docs.microsoft.com/en-us/learn/wwl-data-ai/search-azure-cosmos-db-sql-api-data-azure-cognitive-search/media/2-indexers.png)

In the case of Azure Cosmos DB SQL API, you can configure a container as a **data source**, create a query and a frequency that the **indexer** will use to crawl data, and create a target **index** where the resulting searchable JSON documents are stored.

## Connecting a data source

The first step is to create a data source. The data source points to somewhere where data is stored. With Azure Cosmos DB SQL API, the data source is a reference to an existing account with the following parameters configured:

| Parameter | Value |
|-----|-----|
| Connection string | Connection string for Azure Cosmos DB account |
| Database | Name of target database |
| Collection | Name of target container |
| Query | A SQL query to select items to be indexed |

## Customizing the index

Once the data source is configured, an index should be created that would be the target of the indexing operation. The index contains, at a minimum, a **name** and a **key**. The key refers to a unique identifier field for each JSON document in the index.

Each field in the index should be configured to enable or disable features when searching. These optional features allow extra search functionality on specific fields when it makes sense. For each field, you must configure whether the field is:

| Feature | Description |
|-----|-----|
| Retrievable | Configures the field to be projected in search result sets |
| Filterable | Accepts OData-style filtering on the field |
| Sortable | Enables sorting using the field |
| Facetable | Allows field to be dynamically aggregated and grouped |
| Searchable | Allows search queries to match terms in the field |

## Configuring the indexer

The final step is to configure the indexer’s **name** and **schedule**. The schedule determines how often the indexer will run to pull data from the data source and populate the index with JSON documents.

---

# Implement a change detection policy

The default query for the Azure Cosmos DB SQL API data source is the following SQL query.

```sql
SELECT
    *
FROM
    c
WHERE
    c._ts >= @HighWaterMark
ORDER BY
    c._ts
```

This query only finds items whose **timestamp** (`_ts` property) is greater than or equal to a built-in high watermark field. The high watermark field comes from a built-in change detection policy that attempts to identify whether an item has been changed or not.

To accomplish change detection, the indexer will index all items returned by the query. It will then store a timestamp as the high watermark. The next indexer run will then index items with a timestamp greater than or equal to the stored high watermark. This strategy effectively indexes all items that have been created or changed since the last run.

If your SQL query sorts the items to index using the timestamp, then Azure Cognitive Search can implement incremental progress during indexing. If the indexer fails for a transient reason, sorting the timestamps will allow the indexer to resume indexing from the failure point instead of reindexing the entire container again. This setting must be enabled when configuring the data source.

In the default query example, the result set is sorted using the `_ts` field. If you write a custom query, you must sort using the same field to enable incremental progress when indexing.

---

# Manage a data deletion detection policy

If an item is deleted from a container in Azure Cosmos DB SQL API, that item may not be deleted from the index in Azure Cognitive Search. To enable tracking of deleted items, you must configure a policy to track when an item is deleted.

Azure Cognitive Search supports, exclusively, the ability to track if an item is deleted using a combination of a field and value in a soft-delete scenario. For example, consider this JSON document that has a property named `_isDeleted` with a value of `true`.

```json
{
  "id": "E08E4507-9666-411B-AAC4-519C00596B0A",
  "categoryId": "86F3CBAB-97A7-4D01-BABB-ADEFFFAED6B4",
  "sku": "TI-R092",
  "name": "LL Road Tire",
  "_isDeleted": true
}
```

Using the soft-delete policy, the `softDeleteColumnName` for the data source (Azure Cosmos DB SQL API) would be configured as `_isDeleted`. The `softDeleteMarkerValue` would then be set to `true`. Using this strategy, Azure Cognitive Search will remove items that have been `soft-deleted` from the container.

---

# Exercise - Search data using Azure Cognitive Search and Azure Cosmos DB SQL API

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/search-azure-cosmos-db-sql-api-data-azure-cognitive-search/5-exercise-search-data-use-azure-cognitive-search)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge Check

1. To enable the ability to index changes in Azure Cosmos DB SQL API items, which field should you include in the data source's SQL query?

- [ ] partitionKey
- [ ] id
- [x] _ts
> Correct. The _ts field is required to enable the high watermark policy to track changes to existing items.

2. Your Azure Cosmos DB SQL API solution regularly uses a time-to-live value to automatically delete items after a set amount of time. Which strategy should you use to ensure that the deleted items are also deleted in the search index?

- [x] Configure a soft-delete policy with a tracked column and value
> Correct. A soft-delete column and value is required to enable the soft-delete column deletion detection policy.
- [ ] No configuration is required, Azure Cognitive Search will automatically remove data from the index
- [ ] Configure a high watermark policy that is mapped to the timestamp (_ts) field

---

# Summary

In this module, you explored the Azure Cognitive Search indexer and how it crawls data from Azure Cosmos DB SQL API to put in an index. This data was then used for rich search-specific queries that would be prohibitively difficult to implement in a database store.

Now that you have completed this module, you can:

* Index data in Azure Cosmos DB SQL API for search queries in Azure Cognitive Search
