# README

## TL;DR

Taking this Microsoft challenge as one of the ways to develop myself as part of my journey to become a true software engineer. As a BI developer & data engineer focusing on NoSQL, this challenge fits the most to my career and study.

## Azure Cosmos DB Developer Challenge

> Design and implement data models; design and implement data distribution; integrate an Azure Cosmos DB solution; optimize an Azure Cosmos DB solution; and maintain an Azure Cosmos DB solution.

> "Azure Cosmos DB is Microsoft's proprietary globally distributed, multi-model database service "for managing data at planet-scale" launched in May 2017.[https://azure.microsoft.com/en-us/services/cosmos-db/#overview] It is schema-agnostic, horizontally scalable, and generally classified as a NoSQL database." from wikipedia.

## Introduction to Azure Cosmos DB SQL API

Learn about the Azure Cosmos DB SQL API and determine if it is a good fit for your application.

<style>
smalltitle { font-weight:700;font-size:20px }
</style>

<smalltitle>Learning objectives</smalltitle>

After completing this module, you’ll be able to:

* Evaluate whether Azure Cosmos DB SQL API is the right database for your application.
* Describe how the features of the Azure Cosmos DB SQL API are appropriate for modern applications.

<smalltitle>Prerequisites</smalltitle>

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

### Introduction

Today's apps deliver innovation in all facets of life. For a business to remain competitive, companies must build apps and products that work with real-time data, are resilient, and flexible.

Modern apps thrive on real-time data from different sources and shaped in different forms. An apps' usefulness is often in their ability to move and use data.

Developers require flexibility in their platforms so they can be responsive to business changes. Developers also require their entire application ecosystem to flexibly handle changes in the velocity, volume, or shape of their data. This flexibility enables developers to develop new features more rapidly than they ever have before.

#### Scenario

Suppose you work as the lead developer at a retail company. Your team is building your online storefront. The new storefront will be designed to be accessible across various devices including mobile. The team expects a spike in demand when the storefront is published and various "grand opening" sales begin.

As the lead developer, you have been tasked with identifying a database platform. The database platforms you consider should be able to service the data your team will generate and collect over time. The selected database should also be able to handle a large variety of data, at high volumes and velocity. Your database solution needs to scale quickly and with little friction in order to handle this demand that is both growing and variable.

#### Azure Cosmos DB

Azure Cosmos DB is a fast NoSQL database service for modern app development at any scale.

Here, you'll see how Azure Cosmos DB and its SQL API can be used for this type of business problem. You'll also learn a bit about how the database works. At the end, this module will help you decide if Azure Cosmos DB's SQL API is a good choice for your solutions.

## What is Azure Cosmos DB SQL API

Let's start with a few definitions and a quick tour through Azure Cosmos DB SQL API. This overview should help you see whether Azure Cosmos DB might be a good fit for your work.

### What is a NoSQL database?

Developers require new kinds of databases that can address the needs of modern apps. NoSQL databases were designed to address these needs and unique challenges such as:

High volumes of data
Data with many different sources and forms
Dynamic data schemas to store different types of data
Using high-velocity and/or real-time data
NoSQL databases are defined by common characteristics they share rather than a specific formal definition. These characteristics include:

Data store is non-relational
Designed for scale-out
Does not enforce a specific schema
NoSQL databases generally do not enforce relational constraints or put locks on data, making writes fast. They are also often designed for horizontally scale via sharding or partitioning, which allows them to maintain high-performance regardless of size.

While there are many NoSQL data models, four broad data model families are commonly used when modeling data in a NoSQL database:

Diagram showing various NoSQL models including; a key-value, document, graph, and column-family store

Moving forward, we will focus on the document data model, the data model supported by Azure Cosmos DB SQL API

Why use a NoSQL database with the document data model?
The document data model breaks data down into individual document entities. A document can be any structured data type, but JSON is commonly used as the data format. The Azure Cosmos DB SQL API supports JSON natively.

Illustration of a hierarchical document data model that includes parent entities, child entities, and lines connecting them

A document is an atomic entity and can have its own data form, regardless of what is stored in other documents in the same database. Because of this flexibility, there is no need for a pre-defined schema making it easier to build new applications rapidly. Additionally, this flexibility enables scenarios where different types of data can be stored together and where models can evolve over the lifetime of an application.

What is a JSON document?
JavaScript Object Notation, or JSON, is a lightweight data format. JSON was built to be highly compatible with the literal notation of an object in the JavaScript language. Many frameworks, browsers, and even databases support JavaScript natively making JSON a popular format for transmitting and storing data.

Here is an example of a JSON document:

JSON

Copy
{
  "device": {
    "type": "mobile"
  },
  "sentTime": "2019-11-12T13:08:42",
  "spoolRefs": [
    "6a86682c-be5a-4a4a-bacd-96c4d1c7ece6",
    "79e78fe2-93aa-4688-89db-a7278b034aa6"
  ]
}
As you can see, JSON is a relatively readable data format that clearly exposes its content. JSON is also relatively easy to parse and use in JavaScript applications.

What is Azure Cosmos DB SQL API?
Azure Cosmos DB SQL API is a fast NoSQL database service that offers rich querying over diverse data, helps deliver configurable and reliable performance, is globally distributed, and enables rapid development.

Illustration of a world map with four globally distributed nodes that are connected via lines

The SQL API is the core or native API for working with documents. The SQL API supports fast, flexible development utilizing JSON documents, a query language with a familiar syntax, and client libraries for popular programming languages. Azure Cosmos DB provides other APIs, such as Mongo, Gremlin, and Cassandra, that offer compatibility with each database ecosystem while still mapping to the same underlying infrastructure of the native SQL API.

Azure Cosmos DB SQL API has a few advantages such as:

Guaranteed speed at any scale—even through bursts—with instant, limitless elasticity, fast reads, and multi-master writes, anywhere in the world
Fast, flexible app development with SDKs for popular languages, a native SQL API along with APIs for MongoDB, Cassandra, and Gremlin, and no-ETL (extract, transform, load) analytics
Ready for mission-critical applications with guaranteed business continuity, 99.999-percent availability, and enterprise-grade security
Fully managed and cost-effective serverless database with instant, automatic scaling that responds to application needs
These capabilities make Azure Cosmos DB ideally suited for modern application development. Azure Cosmos DB SQL API is especially suited for applications that:

Experience unpredictable spikes and dips in traffic
Generate lots of data
Need to deliver real-time user experiences
Are depended upon for business continuity
The Azure Cosmos DB SQL API can arbitrarily store native JSON documents with flexible schema. Data is indexed automatically and is available for query using a flavor of the SQL query language designed for JSON data. The SQL API can be accessed using SDKs for popular frameworks such as .NET, Python, Java, and Node.js.