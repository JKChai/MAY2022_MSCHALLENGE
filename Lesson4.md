# Configure Azure Cosmos DB SQL API database and containers

Select between the various throughput offerings in Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Compare the various service and throughput offerings for Azure Cosmos DB
* Migrate between standard and autoscale throughput

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Applications come in many different patterns, they may have predictable usage, or they could be spiked with sudden traffic surges. Azure Cosmos DB has multiple ways to host workloads that map directly to how applications run in the real world, whether the applications are predictable or entirely growth-based.

After completing this module, you'll be able to:

* Compare the various service and throughput offerings for Azure Cosmos DB
* Migrate between standard and autoscale throughput

---

# Serverless

Azure Cosmos DB serverless is a consumption-based model where each request consumes request units. The consumption model eliminates the need to pre-provision throughput request units ahead of time.

Remember, when using Azure Cosmos DB, you typically express database options as a cost described in Request Units per second.

## What are use cases for serverless?

![Diagram of individual requests consuming RU/s](https://docs.microsoft.com/en-us/learn/wwl-data-ai/configure-azure-cosmos-db-sql-api/media/2-serverless.png)

Serverless is great for applications with unpredictable or bursty traffic. You can use serverless with an application such as:

* A new application with hard to forecast users loads
* A new prototype application within your organization
* Serverless compute integration with a service like Azure Functions
* Just getting started with Azure Cosmos DB as a new developer
* Low traffic application that doesn’t send or receive numerous data

---

# Compare serverless vs. provisioned throughput

How do you choose between serverless and provisioned throughput?

## Compare workloads

Provisioned throughput is ideal for workloads with predictable traffic patterns that require sustained and predictable performance with minimal variance.

On the other hand, serverless can handle workloads that have wildly varying traffic and low average-to-peak traffic ratios.

## Compare request units

Provisioned throughput makes some number of request units available each second to each container for database operations. The number of request units can be updated either manually or via autoscale.

Serverless doesn’t require any planning or automatic provisioning and can deliver throughput up to a documented service limit.

## Compare global distribution

Provisioned throughput supports distributing your data to an unlimited number of Azure regions.

Serverless accounts can only run in a single Azure region.

## Compare storage limits

Provisioned throughput allows you to store unlimited data in a container.

Serverless only allows up to 50 GB of data in a container.

---

# Autoscale throughput

We can often make an educated guess about where our workload will be as far as throughput, but we won’t know exactly where it lands until it’s in production. We also may know our operational tolerances. We know:

* The maximum amount of money we are willing to spend
* The minimum amount of performance we are ready to tolerate.

With all of this information in mind, we could define a range. This range would represent our application running at a comfortable performance level without overspending without our knowledge.

![Application with usage oscillating between maximum potential spend and minimum performance](https://docs.microsoft.com/en-us/learn/wwl-data-ai/configure-azure-cosmos-db-sql-api/media/4-autoscale-1.png)

With Azure Cosmos DB autoscale, we can define a range of request units per second (RU/s) to scale our database or container automatically and instantly. The throughput RU/s is scaled based on real-time usage instantly.

![Autoscale between max and min RU/s](https://docs.microsoft.com/en-us/learn/wwl-data-ai/configure-azure-cosmos-db-sql-api/media/4-autoscale-2.png)

Autoscale is great for workloads with variable or unpredictable traffic patterns and can minimize unused capacity that would typically be pre-provisioned.

---

# Compare autoscale vs. standard (manual) throughput

How do you choose between autoscale and standard throughput?

## Compare workloads

Standard throughput is again suited for workloads with steady traffic.

Autoscale throughput is better suited for unpredictable traffic. Autoscale can ensure that your actual Azure Cosmos DB provisioned throughput oscillates between your minimal acceptable performance and maximum allowed spend.

## Compare request units

Standard throughput requires a static number of request units to be assigned ahead of time.

With autoscale, you only set the maximum, and the minimum billed will be 10% of the maximum when there are zero requests.

## Compare scenarios

You want to use standard throughput provisioning in scenarios when your team can accurately predict the amount of throughput your application needs, and your team suspects these needs will not change over time. Also throughput provisioning is ideal for scenarios where the full RU/s provisioned is consumed for > 66% of hours per month.

Autoscale throughput is helpful if your team cannot predict your throughput needs accurately or otherwise use the max throughput amount for < 66% of hours per month.

## Compare rate-limiting

The standard throughput will always remain static at the set RU/s that is provisioned. Requests beyond this will be rate-limited, with a response indicating that a wait should be attempted before retrying.

Autoscale will scale up to the max RU/s before similarly rate-limiting responses

---

# Migrate between standard (manual) and autoscale throughput

Existing containers can be migrated to and from autoscale using the Azure portal, Azure CLI, or Azure PowerShell. During the migration process, the system will automatically apply a request unit per second (RU/s) value to the container.

> :grey_exclamation: Note
> 
> You can always change this RU/s value after the migration has occurred.

---

# Exercise: Configure throughput for Azure Cosmos DB SQL API with the Azure portal

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/configure-azure-cosmos-db-sql-api/7-exercise-configure-throughput-for-azure-portal)

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

1. Your Azure Cosmos DB container has manually provisioned throughput. Incoming requests have exceeded the provisioned request units per second (RU/s). What happens next?

- [x] Azure Cosmos DB will rate-limit subsequent requests.
> That's correct. Manually provisioned throughput will rate-limit requests, with a response indicating that a wait should be attempted before retrying.
- [ ] Azure Cosmos DB will scale RU/s up to a service maximum of 50,000 RU/s.
- [ ] Azure Cosmos DB will scale RU/s up to a maximum RU/s that you have previously configured.

2. While building a proof of concept, your web development team was able to successfully estimate the throughput needs of your application within a 5% margin of error and does not expect any significant variances over time. When running in production, the team expects your workload to be extraordinarily stable. What type of throughput option should you consider?

- [ ] Autoscale
- [ ] Serverless
- [x] Standard
> That's correct. Standard throughput is suited for workloads with steady traffic.

---

# Summary

In this module, we talked about the patterns and needs of different applications and how they mapped to the throughput offerings of Azure Cosmos DB. With serverless, autoscale, or standard provisioning, you have many different ways to service the needs of your applications.

Now that you have completed this module, you can:

* Compare serverless and provisioned throughput models
* Compare and migrate between autoscale and standard throughput containers
