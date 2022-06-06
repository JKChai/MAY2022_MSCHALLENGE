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

