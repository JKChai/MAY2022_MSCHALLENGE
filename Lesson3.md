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