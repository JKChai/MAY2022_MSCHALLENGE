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

Resource hierarchy with database highlighted and multiple example child containers

Container
Containers are the fundamental unit of scalability in Azure Cosmos DB SQL API. Typically, you provision throughput at the container level. Azure Cosmos DB SQL API will automatically and transparently partition the data in a container. You can also optionally configure an indexing policy or a default time-to-live value at the container level.

Resource hierarchy with a set of containers highlighted

Item[s]
An Azure Cosmos DB SQL API resource container is a schema-agnostic container of arbitrary user-generated JSON items. The SQL API for Azure Cosmos DB stores individual documents in JSON format as items within the container. Azure Cosmos DB SQL API natively supports JSON files and can provide fast and predictable performance because write operations on JSON documents are atomic.

 Tip

Containers can also store JavaScript based stored procedures, triggers and user-defined-functions (UDFs)

Resource hierarchy with items highlighted and other example children resources of containers
