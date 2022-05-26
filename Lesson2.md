# Try Azure Cosmos DB SQL API

Learn how to use the Azure Cosmos DB SQL to create an account, and then use the account to create Cosmos DB resources.

## Learning objectives

After completing this module, you'll be able to:

* Create a new Azure Cosmos DB SQL API account
* Create database, container, and item resources for an Azure Cosmos DB SQL API account

## Prerequisites

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

The first step to getting started with Azure Cosmos DB is to create a new account. You will learn, here, the basic hierarchy of resources in an Azure Cosmos DB SQL API account and how to create an account along with those resources.

After completing this module, you'll be able to:

* Create a new Azure Cosmos DB SQL API account
* Create database, container, and item resources for an Azure Cosmos DB SQL API account

---

# Explore resources

An Azure Cosmos DB SQL API account is composed of a basic hierarchy of resources that include:

* An account
* One or more databases
* One or more containers
* Many items

![Hierarchy of Azure Cosmos DB resources including an account, then a child set of databases, child set of containers, and then finally a child set of items](https://docs.microsoft.com/en-us/learn/wwl-data-ai/try-azure-cosmos-db-sql-api/media/2-hiearchy.png)

Let's explore each item in this hierarchy.

## Account

Each tenant of the Azure Cosmos DB service is created by provisioning a database account. Accounts are the fundamental units of distribution and high availability. At the account level, you can configure the region[s] for your data in Azure Cosmos DB SQL API. Accounts also contain the globally unique DNS name used for API requests

![Resource hierarchy with account highlighted and associated with a DNS name and key](https://docs.microsoft.com/en-us/learn/wwl-data-ai/try-azure-cosmos-db-sql-api/media/2-account.png)

## Database

A database is a logical unit of management for containers in Azure Cosmos DB SQL API. An Azure Cosmos DB database manages users, permissions, and containers. Within the database, you can find one or more containers. You can also elect to provision throughput for your data here at the database level.

![Resource hierarchy with database highlighted and multiple example child containers](https://docs.microsoft.com/en-us/learn/wwl-data-ai/try-azure-cosmos-db-sql-api/media/2-database-diag.png)

## Container

Containers are the fundamental unit of scalability in Azure Cosmos DB SQL API. Typically, you provision throughput at the container level. Azure Cosmos DB SQL API will automatically and transparently partition the data in a container. You can also optionally configure an indexing policy or a default time-to-live value at the container level.

![Resource hierarchy with a set of containers highlighted](https://docs.microsoft.com/en-us/learn/wwl-data-ai/try-azure-cosmos-db-sql-api/media/2-container-diag.png)

## Item[s]

An Azure Cosmos DB SQL API resource container is a schema-agnostic container of arbitrary user-generated JSON items. The SQL API for Azure Cosmos DB stores individual documents in JSON format as items within the container. Azure Cosmos DB SQL API natively supports JSON files and can provide fast and predictable performance because write operations on JSON documents are atomic.

> :bulb: Tip
>
> Containers can also store JavaScript based stored procedures, triggers and user-defined-functions (UDFs)

![Resource hierarchy with items highlighted and other example children resources of containers](https://docs.microsoft.com/en-us/learn/wwl-data-ai/try-azure-cosmos-db-sql-api/media/2-item.png)

---

# Review basic operations

There are a few basic operations that you will need to perform anytime you create any Azure Cosmos DB SQL API account resource in Azure.

## Creating a new account

The first step to getting started with Azure Cosmos DB is to create a new account.

When creating a new account in the portal, you must first select an API for your workload. The API selection cannot be changed after the account is created. For the remainder of this section, we will assume that the SQL API has been selected.

![Select API option in the portal with a list of all current APIs including SQL, MongoDB, Graph, Table, and Cassandra](https://docs.microsoft.com/en-us/learn/wwl-data-ai/try-azure-cosmos-db-sql-api/media/3-select-api.png#lightbox)

Next, the Azure portal will use a step-by-step wizard with tabs for various configuration options. Here you can configure options such as:

* The globally unique name of your account
* The location (Azure region) for the account
* Capacity mode (provisioned throughput or serverless)

![Wizard with various tabs and options for creating a new Azure Cosmos DB SQL API account](https://docs.microsoft.com/en-us/learn/wwl-data-ai/try-azure-cosmos-db-sql-api/media/3-account-wizard.png#lightbox)

> :grey_exclamation: Note
>
> Only the options in the **Basics** tab are required to create an Azure Cosmos DB account.

## Creating a new database

Databases are logical units of management in Azure Cosmos DB SQL API, and don't require much to create. You only need a unique database name within the account to create a new database.

> :grey_exclamation: Note
>
> However, if you choose to provision throughput at the database level, configuring the database may require additional steps. This is explored deeper in other Azure Cosmos DB SQL API topics.

## Creating a new container

Containers are the primary unit of scalability in Azure Cosmos DB SQL API. When creating a container, you should specify:

* The parent database
* A unique name for the container with the database
* The path for the partition key value
* *Optional*: provisioned throughput if not inferred from database provisioning.

The Azure Cosmos DB service will automatically and transparently partition your data based on the value of the partition key for each individual item.

## Creating simple items

Once the database and container resources exist, you are ready to create your first item. In Azure Cosmos DB SQL API, an item is a JSON document.

> :grey_exclamation: Note
>
> JavaScript Object Notation (JSON) is an open standard file format, and data interchange format, that uses human-readable text to store and transmit data objects consisting of attribute–value pairs and array data types (or any other serializable value)

JSON is a language-independent data format with well-defined data types and near universal support across a diverse range of services and programing languages. Here is an example of a JSON document that could be an item in an Azure Cosmos DB account:

```JSON
{
  "id": "0012D555-C7DE",
  "type": "customer",
  "fullName": "Franklin Ye",
  "title": null,
  "emailAddress": "fye@cosmic.works",
  "creationDate": "2014-02-05",
  "addresses": [
    {
      "addressLine": "1796 Westbury Drive",
      "cityStateZip": "Melton, VIC 3337 AU"
    },
    {
      "addressLine": "9505 Hargate Court",
      "cityStateZip": "Bellflower, CA 90706 US"
    }
  ],
  "password": {
    "hash": "GQF7qjEgMk=",
    "salt": "12C0F5A5"
  },
  "salesOrderCount": 2
}
```

---

# Exercise: Create an Azure Cosmos DB SQL API account

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/try-azure-cosmos-db-sql-api/4-exercise-create-account)

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

# Summary

In this module, you learned and performed the core operations that are necessary anytime you create an Azure Cosmos DB SQL API account. These core operations will be repeated often as you explore deeper concepts for the Azure Cosmos DB SQL API.

Now that you have completed this module, you can:

* Create a new Azure Cosmos DB account resource that uses the SQL API
* Create database and container resources using the Data Explorer
* Create item resources using the Data Explorer
