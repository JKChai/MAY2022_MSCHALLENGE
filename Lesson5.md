# Move data into and out of Azure Cosmos DB SQL API

Migrate data into and out of Azure Cosmos DB SQL API using Azure services and open-source solutions.

**Learning objectives**

After completing this module, you'll be able to:

* Migrate data using Azure services
* Migrate data using Spark or Kafka

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Shortly after creating a new Azure Cosmos DB account, it’s not uncommon to migrate new data into the account. There are services both in and out of Azure that you may want to consider for this task. In this module, you will learn about various Azure-native and open-source services that could be used to move data into and out of Azure Cosmos DB SQL API.

After completing this module, you'll be able to:

* Migrate data using Azure services
* Migrate data using Spark or Kafka

---

# Move data by using Azure Data Factory

Azure Data Factory is a native service to extract data, transform it, and load it across sinks and stores in an entirely serverless fashion. From a data integration perspective, this means you can marshal data from one datastore to another, regardless of the nuances of each, as long as you can reasonably transform the data between each data paradigm.

## Setup

Azure Cosmos DB SQL API is available as a **linked service** within Azure Data Factory. This type of linked service is supported both as a **source** of data ingest and as a target (**sink**) of data output. For both, the configuration is identical. You can configure the service using the Azure portal, or alternatively using a JSON object.

```JSON
{
    "name": "<example-name-of-linked-service>",
    "properties": {
        "type": "CosmosDb",
        "typeProperties": {
            "connectionString": "AccountEndpoint=<cosmos-endpoint>;AccountKey=<cosmos-key>;Database=<cosmos-database>"
        }
    }
}
```

> :bulb: Tip
> 
> Alternatively, you can use service principals or managed identities to connect to an Azure Cosmos DB SQL API account with Azure Data Factory.

## Read from Azure Cosmos DB

In Azure Data Factory, when reading data from Azure Cosmos DB SQL API, we must configure our linked service as a **source**. This will read data in. To configure this, we must create a SQL query of the data we want to read in. For example, we may write a query such as `SELECT id, categoryId, price, quantity, name FROM products WHERE price > 500` to filter the items from the container that we want to read in from Azure Cosmos DB SQL API to Azure Data Factory to be transformed and then eventually loaded into our destination data store.

In Azure Data Factory, our **source** activity has a configuration JSON object that we can use to set properties such as the query:

```JSON
{
    "source": {
        "type": "CosmosDbSqlApiSource",
        "query": "SELECT id, categoryId, price, quantity, name FROM products WHERE price > 500",
        "preferredRegions": [
            "East US",
            "West US"
        ]        
    }
}
```

## Write to Azure Cosmos DB

In Azure Data Factory, when storing data to Azure Cosmos DB SQL API, we must configure our linked service as a **sink**. This will load out data. To configure this, we must set our write behavior. For example, we may always want to insert our data, or we may want to upsert our data and overwrite any items that may have a matching unique identifier (**id** field).

Our **sink** activity also had a configuration JSON object:

```JSON
"sink": {
    "type": "CosmosDbSqlApiSink",
    "writeBehavior": "upsert"
}
```

---

# Move data by using a Kafka connector

**Apache Kafka** is an open-source platform used to stream events in a distributed manner. Many companies use Kafka for large-scale high-performance data integration scenarios. **Kafka Connect** is a tool within their suite to stream data between Kafka and other data systems. Understandably, this can include Azure Cosmos DB as a source of data or a target (sink) of data.

## Setup

The Kafka Connect connectors for Azure Cosmos DB is available as an open-source project on GitHub at [microsoft/kafka-connect-cosmosdb](https://github.com/microsoft/kafka-connect-cosmosdb). Instructions for downloading and installing the JAR file manually are available at the repository.

## Configuration

Four configuration properties should be set to properly configure connectivity to an Azure Cosmos DB SQL API account.

| Property | Value |
|-----|-----|
| connect.cosmos.connection.endpoint | Account endpoint URI |
| connect.cosmos.master.key | Account key |
| connect.cosmos.databasename | Name of the database resource |
| connect.cosmos.containers.topicmap | Using CSV format, a mapping of the Kafka topics to containers |

## Topics to containers map

Each container should be mapped to a topic. For example, suppose you would like the **products** container to be mapped to the **prodlistener** topic and the **customers** container to the **custlistener** topic. In that case, you should use the following CSV mapping string: `prodlistener#products,custlistener#customers`.

## Write to Azure Cosmos DB

Let’s write data to Azure Cosmos DB by creating a topic. In Apache Kafka, all messages are sent via topics.

You can create a new topic using the `kafka-topics` command. This example will make a new topic named `prodlistener`.

```Bash
kafka-topics --create \
    --zookeeper localhost:2181 \
    --topic prodlistener \
    --replication-factor 1 \
    --partitions 1
```

The following command will start a producer so you can write three records to the inventory topic.

```Bash
kafka-console-producer \
    --broker-list localhost:9092 \
    --topic hotels
```

And in the console, you can then enter these three records to the topic. Once this is done, these records will be committed to the Azure Cosmos DB SQL API container mapped to the topic (**products**).

```JSON
{"id": "0ac8b014-c3f4-4db0-8a1f-434bab460938", "name": "handlebar", "categoryId": "78148556-4e84-44be-abae-9755dde9c9e3"}
{"id": "54ba00da-50cf-44d8-b122-1d18bd1db400", "name": "handlebar", "categoryId": "eb642a5e-0c6f-4c83-b96b-bb2903b85e59"}
{"id": "381dde84-e6c2-4583-b66c-e4a4116f7d6e", "name": "handlebar", "categoryId": "cf8ae707-6d74-4563-831a-06e15a70a0dc"}
```

## Read from Azure Cosmos DB

You can create a source connector in Kafka Connect using a JSON configuration object. In this sample configuration below, most of the properties should be left unchanged, but be sure to change the following values:

| Property | Description |
|-----|-----|
| connect.cosmos.connection.endpoint | Your actual account endpoint URI |
| connect.cosmos.master.key | Your actual account key |
| connect.cosmos.databasename | The name of your actual account database resource |
| connect.cosmos.containers.topicmap | Using CSV format, a mapping of your actual Kafka topics to containers |

```JSON
{
  "name": "cosmosdb-source-connector",
  "config": {
    "connector.class": "com.azure.cosmos.kafka.connect.source.CosmosDBSourceConnector",
    "tasks.max": "1",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "connect.cosmos.task.poll.interval": "100",
    "connect.cosmos.connection.endpoint": "<cosmos-endpoint>",
    "connect.cosmos.master.key": "<cosmos-key>",
    "connect.cosmos.databasename": "<cosmos-database>",
    "connect.cosmos.containers.topicmap": "<kafka-topic>#<cosmos-container>",
    "connect.cosmos.offset.useLatest": false,
    "value.converter.schemas.enable": "false",
    "key.converter.schemas.enable": "false"
  }
}
```

As an illustrative example, using this example configuration table:

| Property | Description |
|-----|-----|
| connect.cosmos.connection.endpoint | https://dp420.documents.azure.com:443/ |
| connect.cosmos.master.key | C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw== |
| connect.cosmos.databasename | cosmicworks |
| connect.cosmos.containers.topicmap | prodlistener#products |

Here is an example configuration file:

```JSON
{
  "name": "cosmosdb-source-connector",
  "config": {
    "connector.class": "com.azure.cosmos.kafka.connect.source.CosmosDBSourceConnector",
    "tasks.max": "1",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "connect.cosmos.task.poll.interval": "100",
    "connect.cosmos.connection.endpoint": "https://dp420.documents.azure.com:443/",
    "connect.cosmos.master.key": "C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==",
    "connect.cosmos.databasename": "cosmicworks",
    "connect.cosmos.containers.topicmap": "prodlistener#products",
    "connect.cosmos.offset.useLatest": false,
    "value.converter.schemas.enable": "false",
    "key.converter.schemas.enable": "false"
  }
}
```

Once configured, data from the Azure Cosmos DB change feed will be published to a Kafka topic.

---

# Move data by using Stream Analytics

Azure Stream Analytics is a real-time event-processing engine designed to process fast streaming data from multiple sources simultaneously. It can aggregate, analyze, transform, and even move data around to other data stores for more profound and further analysis.

## Setup

**Azure Stream Analytics** supports multiple output sinks included Azure Cosmos DB SQL API.

> :grey_exclamation: Note
>
> As of this time, only the SQL API is supported.

## Configuration

Configuring the Azure Cosmos DB SQL API output consists of either selecting the account within your subscription or providing your credentials, which commonly include:

| Property | Description |
|-----|-----|
| `Output alias` | An alias to refer to this output in the query |
| `Account ID` | Account endpoint URI |
| `Account Key` | Account key |
| `Database` | Name of the database resource |
| `Container name` | Name of the container |

The database and container must already exist in the Azure Cosmos DB SQL API account before using the output sink.

## Write to Azure Cosmos DB

Query results from Azure Stream Analytics will be processed as JSON output when written to Azure Cosmos DB SQL API.

Additionally, items are **upserted** to Azure Cosmos DB SQL API based on the value of the id field. Items are typically inserted into Azure Cosmos DB SQL API. If an item already exists with the same unique id, then the operation is assumed to be an **update** operation instead of an **insert** operation.

---

# Move data by using the Azure Cosmos DB Spark connector

With Azure Synapse Analytics and Azure Synapse Link for Azure Cosmos DB, you can create a cloud-native hybrid transactional and analytical processing (HTAP) to run analytics over your data in Azure Cosmos DB SQL API. This connection makes integration over your data pipeline on both ends of your data world, Azure Cosmos DB and Azure Synapse Analytics.

## Setup

First, you should make sure Synapse Link is enabled at the account level. This can be accomplished using the Azure portal or by using the Azure CLI:

```Azure CLI
az cosmosdb create --name <name> --resource-group <resource-group> --enable-analytical-storage true
```

You can also use Azure PowerShell:

```Azure PowerShell
New-AzCosmosDBAccount -ResourceGroupName <resource-group> -Name <name>  -Location <location> -EnableAnalyticalStorage true
```

When creating a container, you should enable analytical storage at the container level on a per container basis. Again this can be accomplished with the portal.

This can also be accomplished with the CLI:

```Azure CLI
az cosmosdb sql container create --resource-group <resource-group> --account <account> --database <database> --name <name> --partition-key-path <partition-key-path> --throughput <throughput> --analytical-storage-ttl -1
```
Or with Azure PowerShell:
```Azure PowerShell
New-AzCosmosDBSqlContainer -ResourceGroupName <resource-group> -AccountName <account> -DatabaseName <database> -Name <name> -PartitionKeyPath <partition-key-path> -Throughput <throughput> -AnalyticalStorageTtl -1
```

> :bulb: Tip
> 
> You can also use the various developers SDKs to enable or disable either analytical storage on a per-container level or Synapse Link at the account level.

## Read from Azure Cosmos DB

> :grey_exclamation: Note
>
> The next couple of Python examples should be performed within your Azure Synapse Analytics workspace.

There are two options to query data from Azure Cosmos DB SQL API. First, you can choose to load to a Spark DataFrame where the metadata is cached. This example uses Python to load a Spark DataFrame that points to an Azure Cosmos DB SQL API account.

```Python
productsDataFrame = spark.read.format("cosmos.olap")\
    .option("spark.synapse.linkedService", "cosmicworks_serv")\
    .option("spark.cosmos.container", "products")\
    .load()
```

Alternatively, you can create a Spark table that points to the Azure Cosmos DB SQL API directly. You can then run SparkSQL queries against the Spark table without impacting the underlying store. This example uses Python to create a Spark table.

```Python
create table products_qry using cosmos.olap options (
    spark.synapse.linkedService 'cosmicworks_serv',
    spark.cosmos.container 'products'
)
```

## Write to Azure Cosmos DB

> :grey_exclamation: Note
>
> The next couple of Python examples should be performed within your Azure Synapse Analytics workspace.

If we want to write new data to Azure Cosmos DB from our Spark DataFrame, we can use the following Python script to append the data in a DataFrame to an existing container.

```Python
productsDataFrame.write.format("cosmos.oltp")\
    .option("spark.synapse.linkedService", "cosmicworks_serv")\
    .option("spark.cosmos.container", "products")\
    .mode('append')\
    .save()
```

> :grey_exclamation: Note
>
> This operation will impact our existing transaction workloads and will consume request units on the Azure Cosmos DB SQL API container[s].

We can even take it further and stream data from a DataFrame, starting from a checkpoint. We can also append this streaming data to an existing container using this example Python script.

```Python
query = productsDataFrame\
    .writeStream\
    .format("cosmos.oltp")\
    .option("spark.synapse.linkedService", "cosmicworks_serv")\
    .option("spark.cosmos.container", "products")\
    .option("checkpointLocation", "/tmp/runIdentifier/")\
    .outputMode("append")\
    .start()

query.awaitTermination()
```

---

# Exercise: Migrate existing data using Azure Data Factory

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/move-data-azure-cosmos-db-sql-api/6-exercise-migrate-existing-data-using-azure-data-factory)

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

# Knowledge check

1. Which type of component in Azure Data Factory will load data out to Azure Cosmos DB SQL API after it has been transformed?

- [x] Sink
> That's correct. A sink is the destination for data after it has been transformed.
- [ ] Source
- [ ] Input

2. After enabling Synapse Link at the Azure Cosmos DB SQL API account level, what should you do before you can use the Spark connector with a specific container?

- [ ] Install Apache Spark in Azure Cosmos DB SQL API
- [ ] No further action is necessary after enabling Synapse Link
- [x] Enable analytical storage at the container level
> That's correct. Each container should have analytical storage enabled.

---

# Summary

In this module, you learned how to migrate data in and out of Azure Cosmos DB using various services in Azure and open-source platforms.

Now that you have completed this module, you can:

* Migrate data using Apache Spark and Apache Kafka
* Migrate data using Azure Data Factory and Azure Stream Analytics.

