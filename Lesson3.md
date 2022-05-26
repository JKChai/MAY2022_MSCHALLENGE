# Plan Resource Requirements

Familiarize yourself with the various configuration options for a new Azure Cosmos DB SQL API account.

**Learning objectives**

After completing this module, you'll be able to:

* Evaluate various requirements of your application

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Creating a new Azure Cosmos DB account often requires making many configuration choices that can, at first, be daunting. While the defaults fit numerous scenarios, it makes the most sense to familiarize yourself with the configuration options to ensure that your account and resources are optimally configured for your solution. In this module, you will learn how to prepare and configure an Azure Cosmos DB account and resources for a new solution.

After completing this module, you'll be able to:

* Evaluate various requirements of your application

---

# Understand throughput

Referring to our basic hierarchy of resources, an Azure Cosmos DB SQL API database is a unit of management for a set of schema-agnostic containers. Each container is a unit of scalability for both throughput and storage.

Containers are partitioned horizontally across compute within an Azure region and distributed across all Azure Regions you configure in your Azure Cosmos DB SQL API account.

When configuring Azure Cosmos DB, you can provision throughput at either or both the database and container levels.

## Container-level throughput provisioning

![Throughput provisioned at container level](https://docs.microsoft.com/en-us/learn/wwl-data-ai/plan-resource-requirements/media/2-container.png)

Any throughput provisioned exclusively at the container level is reserved only for this container. This throughput is available only for this container all the time. This throughput is also financially backed by SLAs.

> :grey_exclamation: Note
>
> This is the most commonly used method of manual throughput provisioning.

## Database-level throughput provisioning

![Throughput provisioned at database level](https://docs.microsoft.com/en-us/learn/wwl-data-ai/plan-resource-requirements/media/2-database.png)

Throughput provisioned on a database is shared across all containers in the database. Because all containers share the throughput resources, you may not get predictable performance in a specific container within the database.

## Mixed-throughput provisioning

![Throughput provisioned at both container and database level](https://docs.microsoft.com/en-us/learn/wwl-data-ai/plan-resource-requirements/media/2-mixed.png)

There may be situations where you may want to combine provisioning throughput at the database and container level. A container with provisioned throughput cannot be converted to a shared database container. Conversely, a shared database container cannot be converted to have dedicated throughput.

---

# Evaluate throughput requirements

Request units are a rate-based currency. They are used to make it simple to talk about physical resources like memory, CPU, and IO when performing requests in Azure Cosmos DB. For example, it’s easier to think of 10 request units as roughly twice as much as five request units in a relative sense without worrying about the physical resources that are abstracted away. Request units are used to measure both foreground and background activities.

Every request consumes a fixed number of request units, including but not limited to:

* Reads
* Writes
* Queries
* Stored procedures

## Configuring throughput

When you create a database or container in Azure Cosmos DB, you can provision request units in an increment of request units per second (or RU/s for short). You cannot provision less than 400 RU/s, and they are provisioned in increments of 100.

## Estimating ad-hoc RU/s consumption

Some RU/s are normalized across various access methods, making many common operations predictable. Using this knowledge, you can perform some basic estimations for simple workloads. For example, you can estimate the RU/s required for common database operations such as one RU for a read and six RU/s for a write operation of a 1-KB document in optimal conditions.

![Request units diagram with estimates](https://docs.microsoft.com/en-us/learn/wwl-data-ai/plan-resource-requirements/media/3-request-units.png)

Using this strategy, you should identify your solution's query and access patterns to make an educated guess as to how many request units will be needed in Azure Cosmos DB. To accomplish this, you will want information such as:

* Top five queries
* Number of read operations per second
* Number of write operations per second

> :bulb: Tip
>
> Measuring RU/s for queries should be done at scale. Measuring queries running on a single physical partition will not yield significant data on the actual throughput used in your real world scenario once it is deployed and scaled out.

You can use a spreadsheet application to build a quick table to figure out a rough estimate of your needed request unit capacity. Here's a quick example:

| Operation type | Number of requests per second |Number of RU per request | RU/s needed |
|---------|--------|--------|----------|
| Write Single Document | 10,000 | 10 | 100,000 |
| Top Query #1 | 700 | 100 | 70,000 |
| Top Query #2 | 200 | 100 | 20,000 | 
| Total RU/s | | | 200,000 RU/s |

> :bulb: Tip
>
> You can also run a proof of concept application, and use the request charge property of the SDK to measure the real-world RU charge of running the operations that you intend to make against Azure Cosmos DB.

---

# Evaluate data storage requirements

Planning for a new Azure Cosmos DB account is composed of two components; throughput and storage. While we have already discussed throughput, data in Azure Cosmos DB will also consume SSD storage billed per GB per month.

## Migrating existing transactional workloads

The Azure Cosmos DB Capacity Calculator is a calculator surfaced as an online form to plug in details about your existing data workload to help estimate your application's storage and throughput requirements and translate it to a cost estimate in terms of Azure Cosmos DB.

![Screenshot of the Azure Cosmos DB Capacity Calculator](https://docs.microsoft.com/en-us/learn/wwl-data-ai/plan-resource-requirements/media/4-calculator.png#lightbox)

> :grey_exclamation: Note
>
> The costs detailed in this example may not accurately reflect current storage costs or your current region.

The calculator will inquire about details such as:

* Total data stored
* Whether you expect to perform near real-time analytics
* The anticipated size of documents
* Point reads per second
* Queries per second

After your rough estimate and proof of concept, use the capacity calculator to refine your estimate further to a much more accurate cost to run your solution in Azure Cosmos DB.

---

# Time-to-live (TTL)

Azure Cosmos DB allows you to set the length of time documents live in the database before being automatically purged. A document's "time-to-live" (TTL) is measured in seconds from the last modification and can be set at the container level with the ability to override on a per-item basis.

Once set at the container level, Azure Cosmos DB will automatically purge documents at the specified time since they were last modified. The TTL value is defined as an integer in seconds.

> :bulb: Tip
>
> The maximum TTL value is 2147483647.

TTL expiration is a background task performed in the background using request units and is scheduled when quiescent.

## Configuring TTL on a container

The TTL value for a container is configured using the `DefaultTimeToLive` property of the container's JSON object.

| DefaultTimeToLive | Expiration |
|-----|-----|
| Does not exist | Items are not automatically expired |
| `-1` | Items will not expire by default |
| n | n seconds after last modified time |

The TTL value for an item is configured by setting the `ttl` path of the item. The TTL value for an item will only work if the `DefaultTimeToLive` property is configured for the parent container. If the `ttl` path is configured for the item, it will override the `DefaultTimeToLive` property of the parent container.

## Examples

| Container.DefaultTimeToLive | Item.ttl | Expiration in seconds |
|-----|-----|-----|
| `1000` | null | `1000` |
| `1000` | `-1` | This item will never expire |
| `1000` | `2000` | `2000` |

| Container.DefaultTimeToLive | Item.ttl | Expiration in seconds |
|-----|-----|-----|
| null | null | This item will never expire |
| null | `-1` | This item will never expire |
| null | `2000` | This item will never expire |

---

# Plan for data retention with time-to-live (TTL)

Azure Cosmos DB only bills for storage you directly consume in real time, and you don't have to pre-reserve storage in advance. In high-write scenarios, TTL values can be used to save on data storage costs in Azure Cosmos DB. Data that has already been shipped out to data warehouses or aggregated and stored in other forms or elsewhere can be immediately purged to ensure that you only keep fresh and relevant data in local SSD storage.

Consider solutions such to aggregate and migrate data such as:

* Change feed
* Azure Data Warehouse
* Azure Blob Storage

When designing your solution, game plan how long your data will need to be retained in Azure Cosmos DB before being migrated across your entire Azure solution space to minimize storage costs.

---

# Knowledge check

1. What rate-based currency acronym is used as a simplification of CPU, memory, and IOPS?

    - [x] RU/s
    > That's correct. RU/s is an acronym for request units per second.
    - [ ] TTL
    - [ ] vCPUs

2. Which property of a container should be specified to automatically purge items after a specified number of seconds?

    - [ ] Expiration
    - [x] DefaultTimeToLive
    > That's correct. Specify the DefaultTimeToLive of a container to enable the time-to-live feature for items in the container.
    - [ ] _ttl

---

# Summary

In this module, you reviewed many of the steps you can take to review your existing applications and maps how you see throughput and storage to concepts in terminology in Azure Cosmos DB. You can now use these skills to configure an Azure Cosmos DB SQL API account.

Now that you have completed this module, you can:

* Evaluate the throughput and storage requirements of your applications in the context of Azure Cosmos DB SQL API
