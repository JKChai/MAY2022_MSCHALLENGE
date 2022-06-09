# Implement a non-relational data model

Identify an application’s key access patterns. Define the entities’ data model, along with containers to store the data with a partition key that will result in an efficient and scalable data store for the application.

**Learning objectives**

In this module, you will:

* Determine access patterns for data.
* Apply data model and partitioning strategies to support an efficient and scalable NoSQL database.

**Prerequisites**

* Familiarity with Azure Cosmos DB concepts, like databases, containers, documents, and throughput in Request Units per second (RU/s)
* Familiarity with using Azure Cosmos DB resources and data in the Azure portal, like using Data Explorer, running queries, and viewing query stats
---

# Introduction

Azure Cosmos DB is Microsoft's fully managed NoSQL database on Azure. As a NoSQL database, Azure Cosmos DB is both horizontally scalable and nonrelational.

Horizontal scalability allows Azure Cosmos DB support data sizes well beyond that of a typical relational database. Horizontal scalability also means that the database provides predictable performance.

To achieve this level of scalability, users need to understand the concepts, techniques unique to NoSQL databases for modeling and partitioning data.

## Scenario

Imagine that you work for a retail startup that's designing a database to manage online orders. You're working on a proposal for an efficient database design using Cosmos DB core (SQL) API. You've been provided an entity-relationship model to start from. You want to provide the maximum scalability, performance, and efficiency possible and to achieve this, the data will need to be modeled correctly.

The following entity-relationship diagram (ER model) provides you with the details of the nine entities you will be working with. The relational model has nine entities in their own tables.

![Diagram that shows the relational model for our example application.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/1-full-relational-model.png)

## What will we be doing?

In this module, we'll take our existing relational data model and redesign it as a NoSQL database for our e-commerce application. During this process, you'll learn the following concepts:

* **Differences between relational versus NoSQL databases**: You'll explore some of the differences between NoSQL databases and relational databases and why they are that way.
* **Using application data access patterns to model data**: You'll learn how understanding the way an application reads and writes data influences how to model it for a NoSQL database.
* **Embedding versus referencing**: You'll learn when you should embed data within the same document versus when you should store data as a separate document.
* **Choosing a partition key**: You'll learn key concepts needed for choosing the best partition key to avoid hot partitions and optimize workloads that are either read or write heavy, or both.
* **Modeling lookup or reference data**: Finally, you'll learn how to model data that's used as a lookup or reference for other data.

## What is the main goal?

When you finish this module and the companion module, "Optimize your database by using advanced modeling patterns for Azure Cosmos DB," you'll have the knowledge and skills to properly model and partition data for a NoSQL database deployed on Azure Cosmos DB.

After completing this module, you’ll be able to:

* Determine access patterns for data.
* Apply data model and partitioning strategies to support an efficient and scalable NoSQL database.

---

# What's the difference between NoSQL and relational databases?

In general, NoSQL databases like Azure Cosmos DB are characterized as being both horizontally scalable and nonrelational.

## Horizontal scale versus vertical scale

Relational databases typically grow by increasing the size of the VM or compute that they're hosted on. NoSQL databases scale up by adding more servers or nodes. These nodes are also known as physical partitions in Cosmos DB. Data stored on these physical partitions needs to be organized so that it can be efficiently accessed later.

Data is predictably routed to different physical partitions by using the value of a required property on each document. This property is called a container's partition key, this partition key needs to be specified when creating the container. Passing the partition key when data is written or read from a container ensures that operations are efficient.

Although the need for a partition key might appear to be constraint, it has some enormous benefits. Typically, relational database possibly will grow to less than 100 TB at most. A NoSQL database can grow to unlimited size, and can do so without any impact on response times when it's accessing data from any single partition.

Additionally, as partitions are added, so too is more compute added and the amount of processing that is supported by the database simultaneously grows.

## Nonrelational versus relational databases

The second defining characteristic of a NoSQL database is that there are no foreign keys, constraints, or enforced relationships of any kind between pieces of data. Because data in a NoSQL database is stored on different physical servers, enforcing constraints or relationships, or placing locks on data would result in negative or unpredictable performance.

However, not having enforced relationships doesn't mean that you can't manage entities that have relationships in a NoSQL database, it just means that you need to do it differently.

## Why are these database types so different?

Understanding how the economics of computing has changed since relational databases were first introduced can help explain why these two types of databases are so different.

When relational databases were invented in 1970, the cost of storage and memory were relative to compute was high. The goal of normalizing a database model was to reduce duplicate data and thus cost within a database. The database engine would apply locks and latches to enforce strict ACID (atomicity, consistency, isolation, durability) semantics as it performed operations on all the needed pieces of data together. The locks on data ensured consistent data, but with trade-offs in concurrency, latency, and availability.

Today, the cost of storage and memory is relatively cheap compared to compute, thus, to be cost effective we no longer need optimize for storage efficiency. With workloads requiring increasing levels of concurrency and availability and lower latencies, there was a need for a new type of database that's optimized for these requirements, and so NoSQL databases were born.

It is also for these reasons, that one of the goals in modeling data for a NoSQL database is to do so in a manner that ensures reading or writing data is compute-efficient. In-part because relational operators like cross-document joins don't exist in NoSQL databases, data must be stored as the application uses it for it to be the most efficient. Often data needs to be denormalized, duplicated, or otherwise stored in a way that breaks many of the relational normalization rules that are used for relational data modeling.

## Can you use NoSQL for relational workloads?

Can you use NoSQL for relational workloads?
At this point, you might be wondering whether NoSQL databases are appropriate to use for relational workloads. And the answer is yes! NoSQL databases can absolutely be used for workloads where relationships between different entities exist.

NoSQL databases are often used when a relational database can't meet the desired performance, scale, or availability needs of the application.

The techniques for designing a NoSQL database are different from the techniques for modeling data for a relational database. These techniques are also not intuitive for someone with a background in relational database design. Some of the best practices that you learn for building relational databases don't translate well to non-relational database design. Those relational database best practices are often antipatterns when you're designing for a NoSQL database.

For the rest of this module and in the advanced modeling module, we'll step you through the techniques that are used to model data in a manner that will result in a high-performance NoSQL database.

---

# Identify access patterns for your app

When you're designing a data model for a NoSQL database, the objective is to ensure that operations on data are done in the fewest requests. To do this, you need to understand the relationships between the data and how data will be accessed by the application. These access patterns are important because they, along with the relationships, will determine how the properties of the various entities are grouped together and stored in documents within containers in Azure Cosmos DB SQL API databases.

In Cosmos DB SQL API, documents are called Items and containers are often synonymously referred to as collections.

## Identify access patterns for customer entities

Let's start with the customer entities in our e-commerce database. The following diagram shows three entities and the relationships between them. The three entities are **Customer**, **CustomerAddress**, and **CustomerPassword**. The **Customer** entity has a 1:Many relationship to **CustomerAddress**. **Customer** has a 1:1 relationship to **CustomerPassword**.

![Diagram that shows the relational model for customer entities.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/3-customer-relational-model.png)

In our application, we'll perform three operations on the customer entities:

* Create a customer: When a new user first visits the e-commerce site, a new customer will be created.
* Update a customer: When an existing user updates their profile information, their customer record will be updated.
* Retrieve a customer: When an existing user visits the site, they'll sign in with their password. During that same session, they'll need to access other customer data (such as address) to purchase new items.

For each of these operations, we need all this data at the same time. These entities can be modeled as separate documents, but it would require multiple round trips to the server to create, update, and retrieve the customer data.

## Model customer entities

Azure Cosmos DB stores data as JSON, so we can model the 1:Many relationship between Customer and CustomerAddress and embed the customer address data as an array. For the 1:1 relationship between Customer and CustomerPassword, we can embed that as an object in our new single customer document. Then the e-commerce application can create, edit, or retrieve customer data in a single request.

The following diagram shows what our customer entity looks like.

![Diagram that shows a modeled customer document.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/3-modeled-customer-document.png)

---

# When to embed or reference data

In the previous unit, we embedded the customer address and password data into a new customer document. That action reduces the number of requests, which improves performance and reduces cost. However, you can't always embed data. There are rules for when you should embed data in a document instead of referencing it in a different row.

## When should you embed data?

Embed data in a document when the following criteria apply to your data:

* **Read or updated together**: Data that's read or updated together is nearly always modeled as a single document. This is especially true because our objective for our NoSQL model is to reduce the number of requests to our database. In our scenario, all of the customer entities are read or written together.
* **1:1 relationship**: For example, Customer and CustomerPassword have a 1:1 relationship.
* **1:Few relationship**: In a NoSQL database, it's necessary to distinguish 1:Many relationships as bounded or unbounded. Customer and CustomerAddress have a bounded 1:Many relationship because customers in an e-commerce application normally have only a handful of addresses to ship to. When the relationship is bounded, this is a 1:Few relationship.

## When should you reference data?

Reference data as separate documents when the following criteria apply to your data:

* **Read or updated independently**: This is especially true where combining entities that would result in large documents. Updates in Azure Cosmos DB require the entire item to be replaced. If a document has a few properties that are frequently updated alongside a large number of mostly static properties, it's much more efficient to split the document into two. One document then contains the smaller set of properties that are updated frequently. The other document contains the static, unchanging values.
* **1:Many relationship**: This is especially true if the relationship is unbounded. Azure Cosmos DB has a maximum document size of 2 MB. So in situations where the 1:Many relationship is unbounded or can grow extremely large, data should be referenced, not embedded.
* **Many:Many relationship**: We'll explore an example of this relationship in a later unit with product tags.

Separating these properties reduces throughput consumption for more efficiency. It also reduces latency for better performance.

---

# Exercise: Measure performance for customer entities

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/implement-non-relational-data-model/5-exercise-measure-performance-for-customer-entities)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Choose a partition key

Remember that data in JSON documents is stored in Azure Cosmos DB databases within containers that are in turn distributed across physical partitions and where the data is routed to the appropriate physical partition based on the value of a partition key.

![Diagram that illustrates the physical partitions in Azure Cosmos DB.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/6-physical-partitions.png)

The partition key is a required document property that ensures documents with the same partition key value are routed to and stored within a specific physical partition. A physical partition supports a fixed maximum amount of storage and throughput (RU/s). When the capacity of a logical partition, which we will introduce in the next section, gets close to the maximum storage, Azure Cosmos DB adds another physical partition to the container. Azure Cosmos DB automatically distributes the logical partitions across the available physical partitions, again using the partition key value to so in a predictable way.

In this unit, you'll learn more about logical partitions and how to avoid hot partitions. This information will help us choose the appropriate partition key for the customer data in our scenario.

In Azure Cosmos DB, you increase storage and throughput by adding more physical partitions to access and store data. The maximum storage size of a physical partition is 50 GB, and the maximum throughput is 10,000 RU/s.

## Logical partitions in Azure Cosmos DB

“The partition key ensures documents with the same partition key value are considered to belong to the same logical partition and are routed to and stored within a specific physical partition. The concept of a logical partition allows for the grouping together of documents with same partition key value. Multiple logical partitions can be stored within a single physical partition and the container can have an unlimited number of logical partitions. Individual logical partitions are moved to new physical partitions as a unit as the container grows. Moving logical partitions as a unit ensures that all documents within it reside on the same physical partition. The maximum size for a logical partition is 20 GB. Using a partition key with high cardinality to allows you to avoid this 20-GB limit by spreading your data across a larger number of logical partitions. 

![Diagram that shows the relationship between the physical and logical partitions.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/6-relationship-between-physical-and-logical-partitions.png)

A partition key provides a way to route data for a logical partition. It's a property that exists within every document in your container that routes your data. A container is another abstraction and is for all data stored with the same partition key. The partition key is defined when you create a container.

In the following example, the container has a partition key of `/username`.

![Diagram that shows an example where the partition key is username.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/6-container-partition-key.png)

## Avoid hot partitions

When you're modeling data for Azure Cosmos DB, it's critically important that the partition key that you choose results in an even distribution of data and requests across physical partitions in your container. This is especially true when containers grow larger and have an increasing number of physical partitions.

If you don't test the design of your database under load during development, a poor choice for partition key might not be revealed until the application is in production and significant data has already been written.

When data is not partitioned correctly, it can result in *hot partitions*. Hot partitions prevent your application workload from scaling, and they can occur on both storage and throughput.

### Storage hot partitions

A hot partition on storage occurs when you have a partition key that results in highly asymmetric storage patterns. As an example, consider a multitenant application that uses TenantId as its partition key with five tenants: A to F. Tenants B,C,D and E are very small, Tenant D has a little more data. However Tenant A is massive and quickly hits the 20-GB limit for its partition. In this scenario, we need to select a different partition key that will spread the storage across more logical partitions.

![Diagram that shows a storage distribution skew.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/6-storage-distribution-skew.png)

### Throughput hot partitions

Throughput can suffer from hot partitions when most or all of the requests go to the same logical partition.

It's important to understand the access patterns for your application to ensure that requests are spread as evenly as possible across partition key values. When throughput is provisioned for a container in Azure Cosmos DB, it's allocated evenly across all the physical partitions within a container.

As an example, if you have a container with 30,000 RU/s, this workload is spread across the three physical partitions for the same six tenants mentioned earlier. So each physical partition gets 10,000 RU/s. If tenant D consumes all of its 10,000 RU/s, it will be rate limited because it can't consume the throughput allocated to the other partitions. This results in poor performance for tenant C and D, and leaving unused compute capacity in the other physical partitions and remaining tenants. Ultimately, this partition key results in a database design where the application workload can't scale.

![Diagram that shows a throughput hot partition.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/6-hot-partition-throughput.png)

When data and requests are spread evenly, the database can grow in a way that fully utilizes both the storage and throughput. The result will be the best possible performance and highest efficiency. In short, the database design will scale.

![Diagram that shows the data and requests spread evenly across partitions.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/6-partitions-even.png)

## Consider reads versus writes

When you're choosing a partition key, you also need to consider whether the data is read heavy or write heavy. You should seek to distribute write-heavy requests with a partition key that has high cardinality.

For read-heavy workloads, you should ensure that queries are processed by one or a limited number of physical partitions by including an `WHERE` clause with an equality filter on the partition key, or an IN operator on a subset of partition key values in your queries.

In scenarios where the application workload is both write heavy and read heavy, there is a solution. We'll explore that in the next module.

The following illustration shows a container that's partitioned by username. This query will hit only a single logical partition, so its performance will always be good.

![Diagram that shows a partition query for username.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/6-in-partition-query.png)

A query that filters on a different property, such as `favoriteColor`, would "fan out" to all partitions in the container. This is also known as a cross-partition query. Such a query will perform as expected when the container is small and occupies only a single partition. However, as the container grows and there are increasing number of physical partitions, this query will become slower and more expensive because it will need to check every partition to get the results whether the physical partition containers data related to the query or not.

## Choose a partition key for customers

Now that you understand partitioning in Azure Cosmos DB, we can decide on a partition key for our customer data. As we covered earlier, we perform three operations on customers: create a customer, update a customer, and retrieve a customer. In this case, we'll retrieve the customer by their id. Because that operation will be called the most, it makes sense to make the customer's id the partition key for the container.

![Diagram that shows the customer partition key as id.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/6-customer-partition-key.png)

You might worry here that making the id the partition key means that we'll have as many logical partitions as there are customers, with each logical partition containing only a single document. Millions of customers would result in millions of logical partitions.

But this is perfectly fine! Logical partitions are a virtual concept, and there's no limit to how many logical partitions you can have. Azure Cosmos DB will collocate multiple logical partitions on the same physical partition. As logical partitions in number or in size, Cosmos DB will move them to new physical partitions when needed.

---

# Model small lookup entities

Our data model includes two small reference data entities, `ProductCategory` and `ProductTag`. These entities are used for reference values and are related to other entities though a `1:Many relationship`.

![Diagram that shows the relationship of the product category, product, product tags, and product tag tables.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/7-product-relational-model.png)

In this unit, we'll model the `ProductCategory` and `ProductTag` entities in our document model.

## Model product categories

Firstly for categories, we'll model the data with its id and name columns as the only properties and put it in a new container called `ProductCategory`.

Next we need to choose a partition key. Let's explore the operations that we need to perform on this data.

We'll create a new product category, edit a product category, and then list all product categories. Creating and editing product categories are not frequently run operations. Our e-commerce application will often list all product categories when customers visit the website. So the last operation is the one we'll run the most.

The query for this last operation will look like this: `SELECT * FROM c`. With id as the selected partition key this query will now be cross-partition, even though we want to try to optimize these read-heavy operations use only a single partition if possible. We also know that the data for product category will never grow near 20 GB in size, so how would this information help us in modeling the data in a way that will result in a single partition query when we list all product categories.

![Diagram that shows the cross-partition query for listing all product categories.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/7-product-category-model.png)

In order to coerce this small amount of data back into a single partition, we can add entity discriminator property to our schema and use this as the partition key for this container. By assigning this property a constant value for all documents of this type in the container, we ensure that we now have a single partition query. In this case, we will call the property `type` and give a constant value of `category`. Our query would now look like: `SELECT * FROM c WHERE type = ”category”`.

![Diagram that shows the product category modeled with the partition key as type and the value as category.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/7-product-category-model-type.png)

## Model product tags

Next up is the `ProductTag` entity. This entity is nearly identical in function to `ProductCategory` entity we discussed in the previous section. Let's take the same approach here and model the document to contain id and name properties and create an entity discriminator property called `type`, in this case with a constant value of tag. Let's create a new container called `ProductTag` and make `type` the new partition key.

![Diagram that shows the modeled product tag container with the partition key as type and the value as tag.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-non-relational-data-model/media/7-product-tag-model.png)

Some people find this technique for modeling small lookup tables strange. However, modeling our data this way gives us an opportunity to make a further optimization in the next module.

---

# Knowledge Check

1. Which of the following criteria would define a good candidate to embed two entities in a single document schema?

- [ ] Many: Many relationships
- [x] Read or updated together
> That's correct. This is correct because Data that's read or updated together is nearly always modeled as a single document. This is especially true because our objective for our NoSQL model is to reduce the number of requests to our database.
- [ ] Read or updated independently

2. Consider the following scenario: Your company has customers across four countries, with hundreds of thousands of customers for countries 1, and 2 and a few thousand customers for countries 3 and 4. Requests for each country total approximately 50,000 RU/s every hour of the day. Your application team proposes to use countryId as the partition key for this container. Which of the following statements is true?

- [ ] This partition key will prevent throughput hot partitions.
- [ ] This partition key will cause fan outs when filtering by countryId.
- [x] This partition key could cause storage hot partitions
> That's correct. Storage hot partitions are likely for regions 1 or 2 since they would have to bulk of the data stored.

3. Why is understanding the access pattern of your application and how to use this information in your data model design important?

- [ ] Understanding the access pattern of your application helps you identify where to save space when storing our data.
- [x] Understanding the access pattern of your application helps you identify how to access your data with fewer requests.
> That's correct. The main goal of your application is usually to process the data as soon as possible, so we usually design our NoSQL databases to reduce the number of requests the app does to read and/or write the data.
- [ ] Understanding the access pattern of your applications wouldn't helps you identify the correct schema of your data.

---

# Summary

In this module, you learned key concepts and techniques for modeling and partitioning data for NoSQL databases like Azure Cosmos DB. We applied these to our e-commerce application that we needed to migrate from a relational database to a NoSQL database. The things that you learned in this module include:

* Differences between relational versus NoSQL databases: You learned how NoSQL databases like Azure Cosmos DB are horizontally scalable, whereas relational databases are typically vertically scalable.
* Using access patterns to model data: You learned how understanding an application's access patterns to data plays an important role in how to model and partition data.
* Embedding versus referencing: You learned when you should embed different entities within the same document versus when you should reference the data and store it as separate rows.
* Choosing a partition key: You learned key concepts for choosing a partition key. These concepts include how to avoid hot partitions and how to handle workloads that are both read and write heavy.
* Modeling lookup or reference data: Finally, you learned how to model data that's used as a lookup or reference for other data.

We applied all of these concepts and techniques to a relational database to model it for a NoSQL database. We modeled the three customer entities and embedded them in a single document. This resulted in an increase in performance by reducing the number of requests for the data.

We also modeled the product category and product tag entities. And we used a special technique to reduce the overall storage and throughput required for small lookup tables.

Now that you have completed this module, you can:

* Determine access patterns for data.
* Apply data model and partitioning strategies to support an efficient and scalable NoSQL database.