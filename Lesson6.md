# Use the Azure Cosmos DB SQL API SDK

Learn about the Microsoft.Azure.Cosmos library, and then download the library to use in a .NET application.

**Learning objectives**

After completing this module, you'll be able to:

* Integrate the Microsoft.Azure.Cosmos SDK library from NuGet
* Connect to an Azure Cosmos DB SQL API account using the SDK and .NET

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

There are various SDKs available to connect to the Azure Cosmos DB SQL API from many popular programming languages including, but not limited to:

* .NET (C#)
* Java
* Python
* JavaScript (Node.js)

For this section, you will explore and get hands-on with the .NET SDK for the Azure Cosmos DB SQL API.

After completing this module, you'll be able to:

* Integrate the Microsoft.Azure.Cosmos SDK library from NuGet
* Connect to an Azure Cosmos DB SQL API account using the SDK and .NET

---

# Understand the SDK

The **Microsoft.Azure.Cosmos** library is the latest version of the .NET SDK for Azure Cosmos DB SQL API.

The library is open-source and hosted online on GitHub at azure/azure-cosmos-dotnet-v3. The open-source project conforms to the Microsoft Open Source Code of Conduct and accepts contributions and suggestions from the community.

The Microsoft.Azure.Cosmos library includes a namespace of the same name with common classes that you will explore later in this module including, but not limited to:

| Class | Description |
|-----|-----|
|Microsoft.Azure.Cosmos.**CosmosClient** | Client-side logical representation of an Azure Cosmos DB account and the primary class used for the SDK |
| Microsoft.Azure.Cosmos.**Database** | Logically represents a database client-side and includes common operations for database management |
| Microsoft.Azure.Cosmos.**Container** | Logically represents a container client-side and includes common operations for container management |

---

# Import from package manager

The **Microsoft.Azure.Cosmos** library, including all of its previous versions, are hosted on **nuget** to make it easier to import the library into a .NET application.

## Import a Nuget package

To import a NuGet package into a .NET application, you must use the .NET CLI. The CLI includes a `dotnet add` command that is used to add a resource to a .NET project. To specifically add a NuGet package, you must do one of the following things:

### Import the latest version of the package

Invoke the `dotnet add package` command with only the name of the package. For example, this command will import the latest stable version of the Microsoft.Azure.Cosmos library.

```bash
dotnet add package Microsoft.Azure.Cosmos
```

> :bulb: Tip
> 
> This command will only import stable versions of the package. If a newer preview of the package is available, it will import the older stable version. If no stable version is available, it will not import the package at all.

### Import a specific version of the package

Invoke the `dotnet add package` command with the name of the package and the `--version` argument specifying a specific package version. For example, this command will import the **3.22.1** version of the **Microsoft.Azure.Cosmos** library.

```bash
dotnet add package Microsoft.Azure.Cosmos \
    --version 3.22.1
```

> :bulb: Tip
> 
> Specifying the package version is the only way to import preview versions of packages that have not been flagged as stable yet.

## .NET project file

Once imported, the package specification will be added to the **csproj** file for the .NET project. The project file uses the XML format and a new element named **PackageReference** is created within the **ItemGroup** element with the name of the package and the version. In this example, the **3.22.1** version of the **Microsoft.Azure.Cosmos** library was imported into the project from NuGet.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net6.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.Azure.Cosmos" Version="3.22.1" />
  </ItemGroup>
</Project>
```

> :grey_exclamation: Note
>
> The version of the package will be added whether you specfied it in the import command or not. If you did not specify a package version, the version of the latest stable package that was imported is specified in the project file.

---

# Connect to an Online Account

Once the **Microsoft.Azure.Cosmos** library is imported, you can begin using the namespaces and classes within your .NET project.

## Import the namespace

Before using the library, you should import the **Microsoft.Azure.Cosmos** namespace using a using directive. The **using directive** allows you to use types within the namespace without being forced to fully qualify each type.

```cs
using Microsoft.Azure.Cosmos;
```

## Use the CosmosClient class

The two most common ways to create an instance for the **CosmosClient** class is to instantiate it with one of the following two constructors:

* A constructor that takes a single string value representing the connection string for the account.
* A constructor that takes two string values representing the endpoint and a key for the account.

> :grey_exclamation: Note
>
> You can always retrieve the connection string, endpoint, or any of the keys from the Azure portal. For the examples in this section, we will use a fictional endpoint of **https­://dp420.documents.azure.com:443/** and a sample key of **fDR2ci9QgkdkvERTQ==**.

> :bulb: Tip
>
> You can also use the CosmosClient class with the Microsoft Identity Platform directly for Azure AD authentication, but that is beyond the scope of this module.

### Use with a connection string

The **CosmosClient** class has a constructor that only takes a single string value. Pass in the connection string of the account to use this constructor. This example uses a connection string in the `AccountEndpoint=<account-endpoint>;AccountKey=<account-key>` format with the fictional endpoint and key.

```cs
string connectionString = "AccountEndpoint=https­://dp420.documents.azure.com:443/;AccountKey=fDR2ci9QgkdkvERTQ==";

CosmosClient client = new (connectionString);
```

### Use with an endpoint and key

Alternatively, you can use a constructor of the **CosmosClient** class that takes in two string parameters representing the account's **endpoint** and **key** in that order. This example uses the fictional endpoint and key.

```cs
string endpoint = "https­://dp420.documents.azure.com:443/";
string key = "fDR2ci9QgkdkvERTQ==";

CosmosClient client = new (endpoint, key);
```

## Read properties of the account

> :bulb: Tip
>
> At this point, you only have a logical client-side representation of the Azure Cosmos DB SQL API account. The SDK won't initially connect to the account until you perform an operation.

Once the client instance is instantiated, you can use various methods directly. For example, you can asynchronously invoke the **ReadAccountAsync** method to get an object of type **AccountProperties** with various properties.

```cs
AccountProperties account = await client.ReadAccountAsync();
```

The **AccountProperties** class includes useful properties such as, but not limited to:

| Property | Description |
|-----|-----|
| Id | Gets the unique name of the account | 
| ReadableRegions | Gets a list of readable locations for the account |
| WritableRegions | Gets a list of writable locations for the account |
| Consistency | Gets the default consistency level for the account |

## Interact with a database

Once you have a client instance, you can retrieve or create a database using one of three methods:

* Retrieve an existing database using the name
* Create a new database passing in a unique database name
* Have the SDK check for the existence of the database and either create or retrieve it automatically

Any of these three methods will return an instance of type **Database** that you can use to interact with the database.

### Retrieving an existing Database

```cs
Database database = client.GetDatabase("cosmicworks");
```

### Create a new Database

```cs
Database database = await client.CreateDatabaseAsync("cosmicworks");
```

### Create Database if it doesn't already exist

```cs
Database database = await client.CreateDatabaseIfNotExistsAsync("cosmicworks");
```

## Interact with a Container

Now that you have a database instance, you can retrieve or create a container using one of three methods:

* Retrieve an existing container using just the name
* Create a new container passing in a unique container name, partition key path, and the amount of throughput to manually provision
* Have the SDK check for the existence of the container and either create or retrieve it automatically

Any of these three methods will return an instance of type **Container** that you can use to interact with the container.

### Retrieving an existing Container

```cs
Container container = database.GetContainer("products");
```

### Create a new Container

```cs
Container container = await database.CreateContainerAsync(
    "cosmicworks", 
    "/categoryId", 
    400
);
```

### Create Container if it doesn't already exist

```cs
Container container = await database.CreateContainerIfNotExistsAsync(
    "cosmicworks", 
    "/categoryId", 
    400
);
```

---

# Implement Client Singleton

Each instance of the `CosmosClient` class has a few features that are already implemented on your behalf:

* Instances are already thread-safe
* Instances efficiently manage connections
* Instances cache addresses when operating in **direct** mode

Because of this behavior, each time you destroy and recreate and instance within a single .NET **AppDomain**, the new instances lose the benefits of the caching and connection management.

The Azure Cosmos DB SQL API SDK team recommends that you use a single instance per **AppDomain** for the lifetime of the application. This small change to your setup allows for better SDK client-side performance and more efficient connection management.

> :bulb: Tip
> 
> It's simple to use a singleton in a typical .NET console application. For ASP.NET web applications, you should review how to create a singleton instance using the dependency injection framework of your choice.

---

# Configure Connectivity mode

The **CosmosClientOptions** class provides a range of options that you can configure for the client when it connects to an account. These options include, but are not limited to:

* The mode used to connect to the account
* Custom consistency level used specifically for the client instance
* The preferred account region[s]

## Overriding default client options

When connecting to an Azure Cosmos DB account using the **CosmosClient** class, there are a few assumptions that the client makes:

* That you will want to connect to the first writable (primary) region of your account
* That you will use the default consistency level for the account with your read requests
* That you will connect directly to data nodes for requests

> :grey_exclamation: Note
>
> There are other assumptions that are not listed here. These assumptions can also be configured with the **CosmosClient** class.

To configure the client, you will need to create an instance of the **CosmosClientOptions** class and pass in that instance as the last parameter to the **CosmosClient** constructor. Here are two examples using the constructors discussed earlier in this module.

You can either use the constructor that takes in an endpoint and key:

```cs
CosmosClientOptions options = new ();

CosmosClient client = new (endpoint, key, options);
```

Or, use the constructor that takes in a connection string:

```cs
CosmosClientOptions options = new ();

CosmosClient client = new (connectionString, options);
```

## Changing the connection mode

Within the **CosmosClientOptions** class, you can set the **ConnectionMode** property to one of two possible enumerable values:

| Value | Description |
|-----|-----|
| Gateway | All requests are routed through the Azure Cosmos DB gateway as a proxy |
| Direct | The gateway is only used in initialization and to cache addresses for direct connectivity to data nodes |

The default setting is to use the **Direct** connection mode. This example configures the client to use the default settings.

```C#
CosmosClientOptions options = new ()
{
    ConnectionMode = ConnectionMode.Direct
};
```

You can optionally configure the client to always use the gateway as a proxy for requests. This example configures the client to use the **Gateway** connection mode.

```C#
CosmosClientOptions options = new ()
{
    ConnectionMode = ConnectionMode.Gateway
};
```

## Changing the current consistency level

Every Azure Cosmos DB SQL API account has a default consistency level configured. Individual clients can configure a different consistency level for all read requests made with the client. This example illustrates a client configured to use **eventual** consistency.

```C#
CosmosClientOptions options = new ()
{
    ConsistencyLevel = ConsistencyLevel.Eventual
};
```

The **ConsistencyLevel** enumeration has multiple potential values including:

* Bounded Staleness
* ConsistentPrefix
* Eventual
* Session
* Strong

> :bulb: Tip
>
> The **ConsistencyLevel** setting is only used to only *weaken* the consistency level for reads. It cannot be strengthened or applied to writes.

## Setting the preferred application region[s]

By default, the client will use the first writable region for requests. This is typically referred to as the primary region. You can use either the **CosmosClientOptions.ApplicationRegion** or **CosmosClientOptions.ApplicationPreferredRegions** to override this behavior.

The **ApplicationRegion** property sets the single preferred region that the client will connect to for operations. It the configured region is unavailable, the client will default to the fallback priority list set on the account to determine the next region to use. In this example, the preferred region is configured to **West US**.

```c#
CosmosClientOptions options = new ()
{
    ApplicationRegion = "westus"
};
```

> :bulb: Tip
> 
> If your account is not configured for multi-region write, the client will always use the single writable region for write operations and this setting will only impact read operations.

If you would like to create a custom failover/priority list for the client to use for operations, you can use the **ApplicationPreferredRegions** property with a list of regions. This example uses a custom list configured to try **West US** first and then **East US**.

```C#
CosmosClientOptions options = new CosmosClientOptions()
{
    ApplicationPreferredRegions = new List<string> { "westus", "eastus" }
};
```

---

# Exercise: Connect to Azure Cosmos DB SQL API with the SDK

This unit includes a lab to complete.

Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.

Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/use-azure-cosmos-db-sql-api-sdk/7-exercise-connect-to)

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

1. Which class should you use to interact with containers in the Azure Cosmos DB SDK for .NET?

- [ ] DocumentCollection
- [x] Container
> That's correct. The Container type is used to interact with containers in the latest SDK.
- [ ] CosmosContainer

2. Which CosmosClient constructor should you use when you have two string variables for the account endpoint and key along with a CosmosClientOptions object?

- [ ] new CosmosClient(string, CosmosClientOptions)
- [ ] new CosmosClient(CosmosClientOptions)
- [x] new CosmosClient(string, string, CosmosClientOptions)
> That's correct. The constructor that takes two string parameters and a configuration object is designed for use with an endpoint and key.

3. What is the name of the Azure Cosmos DB SDK for .NET on NuGet?

- [x] Microsoft.Azure.Cosmos
> That's correct. Microsoft.Azure.Cosmos refers to the latest version of the .NET SDK for the Azure Cosmos DB SQL API.
- [ ] Microsoft.Azure.Documents
- [ ] Microsoft.Azure.DocumentDB

---

# Summary

In this module, you got hands-on with the latest version of the .NET SDK for Azure Cosmos DB SQL API. You used the SDK to connect to an account and perform a query of common account properties. The skills you learned in this module will be useful as you explore more complex developer concepts for the Azure Cosmos DB SQL API.

Now that you have completed this module, you can:

* Locate and pull the Microsoft.Azure.Cosmos SDK library from NuGet into a .NET project
* Use the SDK library to connect to an existing Azure Cosmos DB SQL API account from a .NET application