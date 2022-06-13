# Write management scripts for Azure Cosmos DB SQL API

Learn how to manage Azure Cosmos DB SQL API accounts, databases, and containers using the Azure CLI.

**Learning objectives**

After completing this module, you'll be able to:

* View arguments, groups, and subgroups for a specific CLI command
* Create Azure Cosmos DB accounts, databases, and containers using the CLI
* Manage an indexing policy using the CLI
* Configure database or container throughput using the CLI
* Initiate failovers and manage failover regions using the CLI

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

The Azure CLI is a suite of commands used to administratively manage Azure resources across various platforms and operating systems. The Azure CLI ships with a set of commands tailored specifically to manage Azure Cosmos DB resources. These commands cover the most common administrative scenarios for Azure Cosmos DB accounts, and any underlying resources.

> :grey_exclamation: Note
>
> Beyond what ships with the Azure CLI, the Azure Cosmos DB team is constantly producing new commands that are available as Azure CLI extensions that can be installed when they are needed. We will focus on core commands that ship with the CLI.

After completing this module, you'll be able to:

* View arguments, groups, and subgroups for a specific CLI command
* Create Azure Cosmos DB accounts, databases, and containers using the CLI
* Manage an indexing policy using the CLI
* Configure database or container throughput using the CLI
* Initiate failovers and manage failover regions using the CLI

---

# Create resources

When managing Azure Cosmos DB SQL API accounts using the Azure CLI, most commands will be broken down into two core CLI command groups:

* **az cosmosdb**: This group contains the commands required to create and manage a new Azure Cosmos DB account.
* **az cosmosdb sql**: This subgroup of the az cosmosdb group contains the commands to manage SQL API-specific resources such as databases and containers.

## Azure Cosmos DB account group commands

To create a new account, you will need to use the **az cosmosdb create** command. This command only requires, at a minimum, the name of the new account to create and the target resource group for the account resource. When creating the new account, you must ensure that the name you use is globally unique to avoid the command failing due to a name conflict with another service instance.

> :bulb: Tip
>
> By default, this command will create a new account using the SQL API.

```bash
az cosmosdb create \
    --name '<account-name>' \
    --resource-group '<resource-group>'
```

When creating a new account, you can also optionally specify extra parameters to control the characteristics of the newly created account. For example, this command enables the free tier and sets the default consistency level to **Strong**.

```bash
az cosmosdb create \
    --name '<account-name>' \
    --resource-group '<resource-group>' \
    --default-consistency-level 'strong' \
    --enable-free-tier 'true'
```

In most cases, you will want to specify the region where you wish to deploy your Azure Cosmos DB account. You can use the `--locations` argument to specify one or more target locations for your account.

```bash
az cosmosdb create \
    --name '<account-name>' \
    --resource-group '<resource-group>' \
    --locations regionName='eastus'
```

Later, you will learn how to specify multiple target locations while setting the appropriate failover priorities for each location.

> :grey_exclamation:
>
> When in doubt, you can always look up the help documentation for commands by using the `--help` argument in these examples: `az cosmosdb --help` & `az cosmosdb create --help`.

## Azure Cosmos DB SQL API subgroup commands

The `az cosmosdb sql` command group contains multiple commands and subgroups to manage individual resources within the SQL API. For example, you can use the `az cosmosdb sql database create` command to create a new database within an existing account. In this example, the `--name` argument refers to the name of the new database, and the `--account-name` argument refers to the name of the account created previously.

```bash
az cosmosdb sql database create \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --name '<database-name>'
```

The `az cosmosdb sql container create` command creates a new container within an existing database. The `--name` argument in this context refers to the name of the new container, while the `--database-name` argument refers to the name of the previously created database.

This creation command also requires, at a minimum, a value for the `--partition-key-path` argument as this argument is required for all containers in Azure Cosmos DB SQL API.

```bash
az cosmosdb sql container create \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --database-name '<database-name>' \
    --name '<container-name>' \
    --throughput '400' \
    --partition-key-path '<partition-key-path-string>'
```

> :grey_exclamation:
>
> When in doubt, you can always look up the help documentation for commands by using the `--help` argument in these examples: `az cosmosdb sql --help`, `az cosmosdb sql database --help`, & `az cosmosdb sql container --help`.

---

# Manage index policies

When you create a new container, you can specify the indexing policy at the point of creation using the CLI.

> :grey_exclamation: Note
> 
> Remember, if you do not specify the indexing policy, the default policy will be used.

To create a new container with a custom index policy, you must use the `--idx` argument in one of two ways.

## JSON file

You can pass in the name of a JSON file with a custom index policy. Let's assume that you have an indexing policy defined in a file named `policy.json`.

```bash
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
      "path": "/headquarters/*"
    },
    {
      "path": "/\"_etag\"/?"
    }
  ]
}
```

You only need to pass in the filename to the CLI command for the `--idx` argument.

```bash
az cosmosdb sql container create \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --database-name '<database-name>' \
    --name '<container-name>' \
    --partition-key-path '<partition-key-path-string>' \
    --idx '@.\policy.json' \
    --throughput '400'
```

## Raw JSON string

Alternatively, you can pass in a raw JSON string with the indexing policy defined. This example uses the same JSON indexing policy defined as a minified string:

```bash
az cosmosdb sql container create \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --database-name '<database-name>' \
    --name '<container-name>' \
    --partition-key-path '<partition-key-path-string>' \
    --idx '{\"indexingMode\":\"consistent\",\"automatic\":true,\"includedPaths\":[{\"path\":\"/*\"}],\"excludedPaths\":[{\"path\":\"/headquarters/*\"},{\"path\":\"/\\\"_etag\\\"/?\"}]}' \
    --throughput '400'
```

> :bulb: Tip
>
> If you are planning to use a raw JSON string with the `--idx` argument, you should read up on your shell's specific behavior around escaping and processing strings. Common shells like Bash and PowerShell can have wildly different behaviors when processing JSON string literals.

---

# Configure database or container-provisioned throughput

You can manage the provisioned throughput for both `containers` and `databases` using the CLI. Each of the container and database CLI groups includes a `throughput` subgroup with an `update` command.

## Update container throughput

The `az cosmosdb sql container throughput update` command can change the throughput of the indicated container using the `--throughput` argument. In this example, the container is updated to `1,000 RU/s` throughput.

```bash
az cosmosdb sql container throughput update \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --database-name '<database-name>' \
    --name '<container-name>' \
    --throughput '1000'
```

## Update database throughput

Similar to updating throughput for a container, a database's throughput can be changed using the `--throughput` argument of the `az cosmosdb sql database throughput update` command. In this example, the database is updated to `4,000 RU/s` throughput.

```bash
az cosmosdb sql database throughput update \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --name '<database-name>' \
    --throughput '4000'
```

---

# Migrate between standard and autoscale throughput

Containers that manually provisioned throughput can be migrated to autoscale throughput. The `az cosmosdb sql container throughput migrate` command contains a special argument named `--throughput-type` that can be set to a value of `autoscale` or `manual`.

Use the `az cosmosdb sql container throughput migrate` command with the `--throughput-type` argument set to autoscale to migrate a container to `autoscale` throughput.

```bash
az cosmosdb sql container throughput migrate \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --database-name '<database-name>' \
    --name '<container-name>' \
    --throughput-type 'autoscale'
```

Once migrated, you can manage the maximum allowed throughput by using the `az cosmosdb sql container throughput update` command and the `--max-throughput` argument. In this example, the maximum throughput is changed to `5,000 RU/s`.

```bash
az cosmosdb sql container throughput update \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --database-name '<database-name>' \
    --name '<container-name>' \
    --max-throughput '5000'
```

You can also invoke the `az cosmosdb sql container throughput show` command to view the minimum throughput of an autoscale container

```bash
az cosmosdb sql container throughput show \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --database-name '<database-name>' \
    --name '<container-name>' \
    --query 'resource.minimumThroughput' \
    --output 'tsv'
```

At any time, you can migrate the container back to manually provisioned throughput by invoking the `az cosmosdb sql container throughput migrate` command again with the `--throughput-type` argument set to `manual`

```bash
az cosmosdb sql container throughput migrate \
    --account-name '<account-name>' \
    --resource-group '<resource-group>' \
    --database-name '<database-name>' \
    --name '<container-name>' \
    --throughput-type 'manual'
```

---

# Change region failover priority

The `az cosmosdb update` command is used to change the characteristics of an Azure Cosmos DB account. These characteristics includes changing the regions, modifying failover priorities, and enabling multi-region writes.

Let's assume that we have an Azure Cosmos DB account that we created in the `East US` region using this command. We will use this account as an assumption throughout the next set of examples.

```bash
az cosmosdb create \
    --name '<account-name>' \
    --resource-group '<resource-group>' \
    --locations regionName='eastus'
```

## Add account regions

Assuming that our Azure Cosmos DB account only contains data in a single region (`East US`), we can add more regions using the `az cosmosdb update` command. When adding new regions, it's important to configure the failover priorities of each region using the unique syntax for this command. In this example, we configured the `East US` region as the first priority, replicated data to the `West US` region and set it as the second priority, and then finally replicated data to the `Central US` region and set it as the third priority for failover.

```bash
az cosmosdb update \
    --name '<account-name>' \
    --resource-group '<resource-group>' \
    --locations regionName='eastus' failoverPriority=0 isZoneRedundant=False \
    --locations regionName='westus2' failoverPriority=1 isZoneRedundant=False \
    --locations regionName='centralus' failoverPriority=2 isZoneRedundant=False
```

> :grey_exclamation: Note
>
> If you are performing any operation involving an account region, you cannot many any other changes to your account and must wait for the operation to complete.

## Enable automatic failover

Even though we have data replicated across multiple regions, we still need to enable the automatic failover mechanisms in Azure Cosmos DB. The `--enable-automatic-failover` argument takes a boolean value to enable or disable this feature. This example enables the feature using the priorities we set in previous examples.

```bash
az cosmosdb update \
    --name '<account-name>' \
    --resource-group '<resource-group>' \
    --enable-automatic-failover 'true'
```

## Change failover priorities

As of now, the Azure Cosmos DB account is configured for automatic failover with the following priorities:

| Region | Failover Priority |
|-----|-----|
| East US | 0 |
| West US 2 | 1 |
| Central US | 2 |

To change the priority values, use the `az cosmosdb failover-priority-change` command with the `failover-policies` argument. The `failover-policies` argument has a unique `<region-name>=<value>` syntax. In this example, the `West US 2` and `Central US` regions will switch priorities.

```bash
az cosmosdb failover-priority-change \
    --name '<account-name>' \
    --resource-group '<resource-group>' \
    --failover-policies 'eastus=0' 'centralus=1' 'westus2=2'
```

> :grey_exclamation: Note
>
> Even if you are not changing the priorities of every region, you must include all regions in the `failover-policies` argument.

The Azure Cosmos DB account would then be configured for automatic failover with these new priority values:

| Region | Failover Priority |
|-----|-----|
| East US | 0 |
| Central US 2 | 1 |
| West US | 2 |

## Enable multi-region write

The `az cosmosdb update` command can be used with the `--enable-multiple-write-locations` argument to enable or disable multi-region writes using a boolean value. In this example, multi-region writes is enabled on our account.

```bash
az cosmosdb update \
    --name '<account-name>' \
    --resource-group '<resource-group>' \
    --enable-multiple-write-locations 'true'
```

## Remove account regions

To remove a region from an Azure Cosmos DB account, use the `az cosmosdb update` command to specify the locations that you want to remain using the `--locations` argument one or more times. Any location that is not included in the list will be removed from the account.

```bash
az cosmosdb update \
    --name '<account-name>' \
    --resource-group '<resource-group>' \
    --locations regionName='eastus' failoverPriority=0 isZoneRedundant=False \
    --locations regionName='westus2' failoverPriority=1 isZoneRedundant=False
```

---

# Initiate failovers

Changing the region with priority `= 0` will trigger a manual failover for an Azure Cosmos account.

The `az cosmosdb update` command is used to update the failover policies for an account. If you use this command and change the failover priority for the region that is already set to `0`, the command will trigger a manual failover.

Let's assume that we have an Azure Cosmos DB account with the following regions:

| Region | Failover Priority |
|-----|-----|
| East US | 0 |
| West US 2 | 1 |

To trigger a manual failover, use the `az cosmosdb update` command with the `--failover-policies` argument switching the priorities of the two regions.

```bash
az cosmosdb failover-priority-change \
    --name '<account-name>' \
    --resource-group '<resource-group>' \
    --failover-policies 'westus2=0' 'eastus=1'
```

> :bulb: tip
>
> Any priority change to a region that is `!= 0` will not trigger a failover.

---

# Exercise: Adjust provisioned throughput using an Azure CLI script

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/write-scripts-for-azure-cosmos-db-sql-api/8-exercise-adjust-provisioned-throughput-using-azure-cli-script)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge check

1. Which Azure CLI command will create a new Azure Cosmos DB account using the SQL API?

- [x] az cosmosdb create
> That's correct. The az cosmosdb create command will create a SQL API account by default.
- [ ] az cosmosdb sql create
- [ ] az cosmosdb create --api sql

2. Which Azure CLI command will change the maximum amount of throughput for a container using autoscale throughput?

- [ ] az cosmosdb sql container throughput migrate --throughput-type
- [ ] az cosmosdb sql container throughput update --throughput
- [x] az cosmosdb sql container throughput update --max-throughput
> That's correct. The update command with the --max-throughput argument changes the maximum amount of throughput for a container.

3. How can you initiate a manual failover to another region in your Azure Cosmos DB SQL API account?

- [ ] Use the az cosmosdb failover-priority-change --failover-policies command to change the priority of any region.
- [x] Use the az cosmosdb failover-priority-change --failover-policies command ensuring that you specifically change the priority of the region with a priority = 0.
> That's correct. Changing the priority of the primary region will cause a manual failover event.
- [ ] Use the az cosmosdb update command to "bounce" the service

---

# Summary

In this module, you learned how to use the `az cosmosdb` command group and many of the SQL API relevant subgroups to create and manage resources within an Azure Cosmos DB account.

> :bulb: Tip
>
> The Azure CLI also includes command subgroups for managing Azure Cosmos DB accounts created using other APIs.

Now that you have completed this module, you can:

* View properties of a specific CLI command using the `--help` argument
* Use `az cosmosdb` to create a new Azure Cosmos DB SQL API account using the CLI
* Use `az cosmosdb create` to create databases and containers using the CLI
* Adjust indexing policy and throughput using the CLI
* Initiate failovers and manage failover regions using the CLI
