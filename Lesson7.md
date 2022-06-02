# Configure the Azure Cosmos DB SQL API SDK

Learn how to configure the Azure Cosmos DB SQL API SDK in various ways including how to integrate with the emulator, implement parallelism, and create a custom logger.

**Learning objectives**

After completing this module, you'll be able to:

* Configure the SDK for offline development
* Troubleshoot common connection errors
* Implement parallelism in the SDK
* Configure logging using the SDK

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Often, you will want to configure the Azure Cosmos DB SQL API SDK for .NET to enable common scenarios, troubleshoot problems, improve performance, or gather deeper insight. The SDK includes a rich set of options, along with a fluent builder, to configure your applications to perform and be managed in a way that is useful to your team.

After completing this module, you'll be able to:

* Configure the SDK for offline development
* Troubleshoot common connection errors
* Implement parallelism in the SDK
* Configure logging using the SDK

---

# Enable Offline Development

As you begin to use Azure Cosmos DB across multiple projects, there will eventually be a need to use and test Azure Cosmos DB in a local environment where you can validate new code quickly without creating a new instance in the cloud. The Azure Cosmos DB emulator is a great tool for common Dev+Test workflows that developers may need to implement on their local machine.

## Azure Cosmos DB emulator

The Azure Cosmos DB emulator is a local environment that is useful to develop and test applications locally without incurring the costs or complexity of an Azure subscription.

The emulator is available to run in Windows, Linux, or as a Docker container image.

The emulator is available as a download from the Microsoft Docs website and supports various APIs depending on the platform. The SQL API is universally supported across all platforms.

The Docker container image for the emulator is published to the Microsoft Container Registry and is syndicated across various container registries such as Docker Hub. To obtain the Docker container image from **Docker Hub**, use the Docker CLI to **pull** the image from `mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator`.

```Bash
docker pull mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator
```

## Configuring the SDK to connect to the emulator

The Azure Cosmos DB emulator uses the same APIs as the cloud service, so connecting to the emulator is not different from connecting to the cloud service. The emulator uses a single fixed account with a static authentication key that is the same across all instances

First, the emulator's endpoint is `https://localhost:<port>/` using SSL with the default port set to 8081. In C# code, you can configure this endpoint as a string variable using this example line of code.

```C#
string endpoint = "https://localhost:8081/";
```

The emulator's key is a static well-known authentication key. The default value for this key is `C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==`. In C# code, you can save this key as a variable using this example line of code.

```C#
string key = "C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==";
```

> :bulb: Tip
>
> You can start the emulator using the **/Key** option to generate a new key instead of using the default key.

Once those variables are set, create the **CosmosClient** like you typically would for a cloud-based account.

```C#
CosmosClient client = new (endpoint, key);
```

---

# Handle Connection Errors

While most of your requests are fine, there are some scenarios where a request can fail for a temporary reason. In these scenarios, it's both normal and expected for you to retry your request after a reasonable amount of time.

## Built-in retry

A transient error is an error that has an underlying cause that soon resolves itself. Applications that connect to your database should be built to expect these transient errors. The Azure Cosmos DB SQL API SDK for .NET has built-in logic to handle common transient failures for read and query requests. The SDK does NOT automatically retry write requests as they are not idempotent.

> :bulb: Tip
>
> Try to always use the latest version of the SDK. The retry logic that is built-in is constantly being improved in newer releases.

If you are writing an application that experiences a write failure, it is up to your application code to implement retry logic is considered a best practice.

As an application developer, it's important to understand the HTTP status codes where retrying your request makes sense. These codes include, but are not limited to:

* **429**: Too many requests
* **449**: Concurrency error
* **500**: Unexpected service error
* **503**: Service unavailable

> :bulb: Tip
>
> If you experience 50x errors indicating issues with service availability, you can file an Azure support issue to receive technical support or report an issue.

There are HTTP error codes, such as **400 (bad request)**, **401 (not authorized)**, **403 (forbidden)**, and **404 (not found)** that indicate a failure client-side that should be fixed in application code and not retried.

---

# Implement threading and parallelism

While the SDK implements thread-safe types and some degrees of parallelism, there are best practices that you can implement in your application code to ensure that the SDK has the best performance it can possibly have in your workload.

## Avoid resource-related timeouts

Many times request timeouts occur due to high CPU or port utilization on client machines rather than a service-side issue. It is important to monitor resource utilization on client machines and scale-out appropriately to avoid SDK errors are retries due to local resource exhaustion.

## Use async/await in .NET

The C# language in .NET has a series of Task-based features to asynchronously invoke SDK client methods. For example, the **CreateDatabaseIfNotExistsAsync** method is invoked asynchronously using the following syntax.

```C#
Database database = await client.CreateDatabaseIfNotExistsAsync("cosmicworks");
```

This syntax uses the `await` keyword to run the task asynchronously and return the result into the indicated variable. Using the asynchronous keywords allows the SDK to manage requests simultaneously in a efficient manner.

Avoid blocking the asynchronous execution using **Task.Wait** or **Task.Result** such as in the example code below.

```C#
Database database = client.CreateDatabaseIfNotExistsAsync("cosmicworks").Result;
```

## Use built-in iterators instead of LINQ methods

LINQ methods such as `ToList` will eagerly and synchronously drain a query while blocking any other calls from executing. For example, this invocation of ToList() will block all other calls and potentially retrieve a large set of data:

```C#
container.GetItemLinqQueryable<T>()
    .Where(i => i.categoryId == 2)
    .ToList<T>();
```

The SDK includes methods such as `ToFeedIterator<T>` that asynchronously retrieves the results of a query without blocking other calls. This example illustrates the same scenario but using the special iterator instead of ToList.

```C#
container.GetItemLinqQueryable<T>()
    .Where(i => i.categoryId == 2)
    .ToFeedIterator<T>();
```

---

## Configure max concurrency, parallelism, and buffered item count

When issuing a query from the SDK, the **QueryRequestOptions** includes a set of properties to tune a query's performance.

### Max item count

All query results in Azure Cosmos DB SQL API are returned as "pages" of results. This property indicates the number of items you would like to return in each "page". The service default is 100 items per page of results. You can set this value to **-1** to set a dynamic page size.

In this example, the **MaxItemCount** property is set to a value of **500**.

```C#
QueryRequestOptions options = new ()
{
    MaxItemCount = 500
};
```

> :bulb: Tip
> 
> If you use a **MaxItemCount** of -1, you should ensure the total response doesn't exceed the service limit for response size. For instance, the max response size is 4 MB.

### Max concurrency

**MaxConcurrency** specifies the number of concurrent operations ran client side during parallel query execution. If set to **1**, parallelism is effectively disabled. If set to **-1**, the SDK manages this setting. Ideally, you would set this value to the number of physical partitions for your container.

In this example, the **MaxConcurrency** property is set to a value of **5**.

```C#
QueryRequestOptions options = new ()
{
    MaxConcurrency = 5
};
```

### Max buffered item count

The **MaxBufferedItemCount** property sets the maximum number of items that are buffered client-side during a parallel query execution. If set to **-1**, the SDK manages this setting. The ideal value for this setting will largely depend on the characteristics of your client machine.

In this example, the **MaxBufferedItemCount** property is set to a value of **5,000**.

```C#
QueryRequestOptions options = new ()
{
    MaxBufferedItemCount = 5000
};
```

> :grey_exclamation: Note
>
> These settings are explored much deeper in other Azure Cosmos DB SQL API modules on issuing queries using the SDK.

---

# Configure logging

There are many scenarios where you wish to log the HTTP requests that the Azure Cosmos DB SQL API SDK performs "under the hood." The SDK includes a fluent client builder class that simplifies the process of injecting custom handlers into the HTTP requests and responses. You can take advantage of this functionality to build a simple logging mechanism.

## Client builder

The **Microsoft.Azure.Cosmos.Fluent.CosmosClient** class is a builder class that fluently configures a new client instance. It comes with multiple methods that are used as an alternative to the **CosmosClientOptions** class including, but not limited to:

| Method | Description |
|-----|-----|
| **WithApplicationRegion** or **WithApplicationPreferredRegions** | Configures preferred region[s] |
| **WithConnectionModeDirect** and **WithConnectionModeGateway** | Sets connection mode |
| **WithConsistencyLevel** | Overrides consistency level |

To use the builder, first you must add a using directive to the **Microsoft.Azure.Cosmos.Fluent** namespace.

```c#
using Microsoft.Azure.Cosmos.Fluent;
```

Now, you can create a new instance of the **CosmosClientBuilder** class passing in a connection string or endpoint+key pair as constructor parameters.

```c#
CosmosClientBuilder builder = new (connectionString);
CosmosClientBuilder builder = new (endpoint, key);
```

At this point, you can add any fluent methods to configure the client. Once you are done with fluent methods, you can invoke the Build method to create an instance of type **CosmosClient**.

```c#
CosmosClient client = builder.Build();
```

## Creating a custom log handler

To log HTTP requests, you will need to create a new class that inherits from the abstract **RequestHandler** class. In the handler, you can add logic before and after the HTTP request is sent. For this example, we will create a handler that performs the following workflow when an HTTP request is sent:

1. Writes the HTTP method and URI of the originating request to the console
2. Sends the request to the base implementation and stores the response in a variable
3. Writes the HTTP status code number and description to the console
4. Returns the response

### Creating a custom RequestHandler implementation

To implement this, we need to create a new class that inherits from **RequestHandler**.

```c#
public class LogHandler : RequestHandler
{   
}
```

The abstract class includes a **SendAsync** method that should be overridden to inject new logic around requests.

```c#
public override async Task<ResponseMessage> SendAsync(RequestMessage request, CancellationToken cancellationToken)
{
}
```

Within the **SendAsync** method, the **RequestUri** and Method properties of the **RequestMessage** parameter are printed to the console. Then, the base **SendAsync** method is invoked to send the actual request and store the response in a local variable.

```c#
Console.WriteLine($"[{request.Method.Method}]\t{request.RequestUri}");

ResponseMessage response = await base.SendAsync(request, cancellationToken);
```

After the response is stored in a local variable, the **StatusCode** of the response is printed in both numeric and string format. Then the response is returned as the result of the asynchronous method.

```c#
Console.WriteLine($"[{Convert.ToInt32(response.StatusCode)}]\t{response.StatusCode}");

return response;
```

Here is the code for the complete class.

```c#
public class LogHandler : RequestHandler
{    
    public override async Task<ResponseMessage> SendAsync(RequestMessage request, CancellationToken cancellationToken)
    {
        Console.WriteLine($"[{request.Method.Method}]\t{request.RequestUri}");

        ResponseMessage response = await base.SendAsync(request, cancellationToken);
        
        Console.WriteLine($"[{Convert.ToInt32(response.StatusCode)}]\t{response.StatusCode}");
        
        return response;
    }
}
```

### Using a custom RequestHandler implementation in the client builder

Once the request handler implementation is ready, invoke the **AddCustomHandler** method of the **CosmosClientBuilder** instance passing in a new instance of the custom request handler.

```C#
builder.AddCustomHandlers(new LogHandler());
```

Here is the code for the complete creation of the client using the builder.

```C#
CosmosClientBuilder builder = new (endpoint, key);

builder.AddCustomHandlers(new LogHandler());

CosmosClient client = builder.Build();
```

### Testing the custom logger

Let's assume we have a fictional scenario where we use our client instance to invoke the **CreateDatabaseIfNotExistsAsync** method. The client instance should check for the existence of the database first, and if it doesn't find the database, it will create a new one using the specified name.

For this fictional scenario, we will use this example line of code to invoke the **CreateDatabaseIfNotExistsAsync** method.

```C#
Database result = await client.CreateDatabaseIfNotExistsAsync("cosmicworks");
```

When you run the application for the first time, the logger will output that it performed the following actions:

1. Sent a HTTP **GET** *request* to query for your specific database at the `dbs/<database-name>` endpoint.
2. Received a *response* of **404** that the database was not found.
3. Sent a HTTP **POST** *request* with the database details in the body of the request to the `dbs/` endpoint.
4. Received a *response* of **201** indicating that the database has been created with the database's details in the response body.

```bash
[GET]   dbs/cosmicworks
[404]   NotFound
[POST]  dbs/
[201]   Created
```

If you ran the application again, the logger will output a much shorter workflow:

1. Sent a HTTP **GET** *request* to query for your specific database at the `dbs/<database-name>` endpoint.
2. Received a *response* of **200** indicating that the database has been found with the database's details in the response body.

```bash
[GET]   dbs/cosmicworks
[200]   OK
```

--- 

# Exercise: Configure the Azure Cosmos DB SQL API SDK for offline development

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

# Knowledge Check

1. Which of these code samples would create a client that could connect to the Azure Cosmos DB emulator using its default port?

- [ ] new CosmosClient("https://documents.azure.com", "C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==");
- [x] new CosmosClient("https://localhost:8081", "C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==");
> That's correct. This code would correctly connect to the emulator using the default port of 8081.
- [ ] new CosmosClient("https://localhost", "C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==");

2. Assuming that you have an Azure Cosmos DB container with 50 partitions, which of the following code will correctly configure the QueryRequestOptions instance to optimize parallelism of requests from the SDK and client?

- [x] new QueryRequestOptions { MaxConcurrency = 50 };
> That's correct. This code will configure the concurrency to match the number of partitions.
- [ ] new QueryRequestOptions { MaxItemCount = 50 };
- [ ] new QueryRequestOptions { MaxBufferedItemCount = 50 };

3. Which class should you inherit from to create a class that intercepts SDK-side HTTP requests and inject extra logic?

- [ ] RequestMessage
- [ ] HttpRequest
- [x] RequestHandler
> That's correct. The RequestHandler abstract class defines an overridable method to inject logic in the HTTP request+response flow.

--- 

# Summary

In this module, you configured the SDK for common scenarios that may come up with a developer team. You specifically; tuned the parallelism options, handled connection issues, configured the SDK to use with the emulator, and built a custom logger.

Now that you have completed this module, you can:

* Use the Azure Cosmos DB emulator with the SDK
* Handle the most common connection errors with the SDK
* Configure the parallelism options in the SDK client
* Create a custom request handler to log HTTP request from the SDK
