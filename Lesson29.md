# Build multi-item transactions with the Azure Cosmos DB SQL API

Author stored procedures using JavaScript in Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Author stored procedure
* Rollback stored procedure transactions

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

When using the SQL API in Azure Cosmos DB, you can write stored procedures in the JavaScript language that performs multiple operations as a single logical unit. In this module, you will author JavaScript stored procedures that execute directly inside the database engine.

After completing this module, you'll be able to:

* Author stored procedure
* Rollback stored procedure transactions

---

# Understand transactions in the context of JavaScript SDK

In a database, a transaction is typically defined as a sequence of point operations grouped together into a single unit of work. It's expected that a transaction provides `ACID` guarantees:

* `Atomicity` guarantees that all the work done inside a transaction is treated as a single unit where either all of it is committed or none.
* `Consistency` makes sure that the data is always in a healthy internal state across transactions.
* `Isolation` guarantees that no two transactions interfere with each other – generally, most commercial systems provide multiple isolation levels that can be used based on the application's needs.
* `Durability` ensures that any change that's committed in the database will always be present.

In Azure Cosmos DB SQL API, a stored procedure executes one or more operations as a single unit of work within the same scope. Stored procedures are registered in containers, and run within the scope of that specific container. Stored procedures are registered in containers, and run within the scope of that specific container.

> :grey_exclamation: Note
>
> Stored procedures are scoped to a single logical partition. You cannot execute a stored procedure that performs operations across logical partition key values.

![Transaction with multiple steps within a single scope](https://docs.microsoft.com/en-us/learn/wwl-data-ai/build-multi-item-transactions-azure-cosmos-db-sql-api/media/2-transaction.png)

Transactions occurs server-side in Azure Cosmos DB SQL API, so they must adhere to the same limitations as many other HTTP requests. All operations within a stored procedure must completed within a bounded amount of time. Specifically, the operations must be complete with the server `request timeout` duration.

For long-running lists of operations, a helper boolean value is returned by any JavaScript function that performs an operation indicating whether that operation is expected to complete within the request timeout duration. If the Boolean is `true`, you can continue on with the stored procedure. Once that Boolean is `false`, then the stored procedure must finalize as soon as possible. At this point, it is common to return a pointer so that subsequent calls to the stored procedure can start from the pointer instead of rewinding progress all the way to the beginning of the long-running list of operations.

![Transaction that returns a pointer after indication of a pending timeout](https://docs.microsoft.com/en-us/learn/wwl-data-ai/build-multi-item-transactions-azure-cosmos-db-sql-api/media/2-continuation.png)

---

# Author Stored procedures

Transactions are defined as JavaScript functions. The function is then executed when the stored procedure is invoked.

```js
function name() {
}
```

Within the function, the `getContext()` method retrieves a context object, which can be used to perform multiple actions, including:

* Access the HTTP response object
* Access the corresponding Azure Cosmos DB SQL API container

Using the context object, you can invoke the `getResponse()` method to access the HTTP response object to perform actions such as returning a `HTTP OK (200)` and setting the response's body to a static string.

```js
function greet() {
    var context = getContext();
    var response = context.getResponse();
    response.setBody("Hello, Learn!");
}
```

Again, use the context object, you can invoke the `getCollection()` method to access the container using the JavaScript query API.

```js
function createProduct(item) {
    var context = getContext();
    var container = context.getCollection(); 
}
```

At this point, you can perform typical operations such as creating a new document.

```js
function createProduct(item) {
    var context = getContext();
    var container = context.getCollection(); 
    container.createDocument(
        container.getSelfLink(),
        item
    );
}
```

This stored procedure is almost complete. While this code will run fine, it does stand the risk of swallowing errors and potentially not returning if the stored procedure has exceeded the timeout. We should update the code by implementing two more changes:

* Store the boolean return value of container.createDocument, and then use that to determine if we should return from the function due to an impending server timeout.
* Add a third parameter to container.createDocument to handle potential errors and set the response of this stored procedure to the newly created item returned from the operation.

```js
function createProduct(item) {
    var context = getContext();
    var container = context.getCollection(); 
    var accepted = container.createDocument(
        container.getSelfLink(),
        item,
        (error, newItem) => {
            if (error) throw error;
            context.getResponse().setBody(newItem)
        }
    );
    if (!accepted) return;
}
```

> :bulb: Tip
> 
> Alternatively, you can use the `__` (double underline) shortcut as an equivalent to `getContext().getCollection()`.

---

# Rollback transactions

Transactions are deeply and natively integrated into Azure Cosmos DB SQL API’s JavaScript programming model. Inside a JavaScript function, all operations are automatically wrapped under a single transaction. If the function completes without any exception, all data changes are committed. Azure Cosmos DB’s SQL API will roll back the entire transaction if a single exception is thrown from the script.

Effectively, the start of the JavaScript function is similar to a `BEGIN TRANSACTION` statement in a database system, with the end of the function scope being the functional equivalent of `COMMIT TRANSACTION`. If any error is thrown, that’s the functional equivalent of `ROLLBACK TRANSACTION`.

![Illustration of the begin and commit of an implicit transaction using a JavaScript stored procedure](https://docs.microsoft.com/en-us/learn/wwl-data-ai/build-multi-item-transactions-azure-cosmos-db-sql-api/media/4-rollback.png)

In code, this is surfaced simply by throwing any error in JavaScript:

```js
throw new Error('Something');
```

Using the create item example from earlier in this module, you can create a callback function to determine if the operation returned an error from the server. If so, you can rethrow the error immediately to short-circuit your code and cause the entire stored procedure transaction to be rolled back.

```js
(error, newItem) => {
    if (error) throw error;
    // Do something with item
}
```

---

# Create stored procedures with the JavaScript SDK

Creating a stored procedure using the .NET SDK requires the use of a special `Scripts` property in the `Microsoft.Azure.Cosmos.Container` class. Let’s start with an example that assumes a container instance in a variable named `container`.

1. First, define the JavaScript function for the stored procedure in a string variable.

```cs
string sproc = @"function greet() {
    var context = getContext();
    var response = context.getResponse();
    response.setBody('Hello, Learn!');
}";
```

> :bulb: Tip
> 
> Alternatively, you can use file APIs such as System.IO.File to read a function from a *.js file.

2. Next, create an object of type Microsoft.Azure.Cosmos.Scripts.StoredProcedureProperties with the Id and Body properties set to the unique identifier and content of the stored procedure, respectively.

```cs
StoredProcedureProperties properties = new()
{
    Id = "greet",
    Body = sproc
};
```

> :bulb: Tip
> 
> Alternatively, you can provide the identifier and body of the stored procedure as constructor parameters.

```cs
StoredProcedureProperties properties = new("greet", sproc);
```

3. Now, use the `CreateStoredProcedureAsync<>` method of the container variable to create a new stored procedure passing in the properties composed earlier.

```cs
await container.Scripts.CreateStoredProcedureAsync(properties);
```

If you'd like to parse the results, the `CreateStoredProcedureAsync<>` method returns an object of type `Microsoft.Azure.Cosmos.Scripts.StoredProcedureResponse` that contains metadata about the newly created stored procedure within the container.

---

# Exercise: Create a stored procedure with the Azure portal

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/build-multi-item-transactions-azure-cosmos-db-sql-api/6-exercise-create-stored-procedure-azure-portal)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge check

1. At the end of your stored procedure, you would like to set the HTTP response to a static string of Test. Which line of JavaScript code should you use to accomplish this task?

- [x] context.getResponse().setBody('Test');
> That's correct. The HTTP response object includes a setBody function used to set the body of the HTTP response.
- [ ] context.setBody('Test');
- [ ] context.getCollection().setBody('Test');

2. You are authoring a stored procedure in JavaScript and would like to manually roll back a transaction in your code for a certain condition. Which line of code should you use to roll back a transaction?

- [ ] getContext().rollback();
- [ ] throw new Error();
> That's correct. Throwing an exception will roll back the entire transaction.
- [ ] return;

3. Your code contains a variable of type Microsoft.Azure.Cosmos.Container named container and another variable of type Microsoft.Azure.Cosmos.Scripts.StoredProcedureProperties named props. Which code block below would asynchronously create a new stored procedure using the two variables?

- [ ] await container.StoredProcedures.CreateStoredProcedureAsync(props);
- [ ] await container.CreateStoredProcedureAsync(props);
- [ ] await container.Scripts.CreateStoredProcedureAsync(props);
> That's correct. The Scripts property of the Container class includes a method named CreateStoredProcedureAsync.

4. Your stored procedure creates three items with three distinct unique identifiers and logical partition key values. When running your stored procedure, you encounter an error. What is the cause for this error?

- [ ] Stored procedures are scoped to only a single item
- [x] Stored procedures are scoped to only a single logical partition
> This's correct. You cannot execute a stored procedure that performs operations across logical partition key values.
- [ ] Stored procedures are scoped to only a single unique identifier

---

# Summary

In this module, you created stored procedures that executed JavaScript logic directly within the database engine.

Now that you have completed this module, you can:

* Create a stored procedure using the portal or the .NET SDK
* Roll back a transaction within a stored procedure by throwing an error

