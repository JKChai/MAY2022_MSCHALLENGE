# Expand query and transaction functionality in Azure Cosmos DB SQL API

Author user-defined functions and triggers using JavaScript in Azure Cosmos DB SQL API.

**Learning objectives**

After completing this module, you'll be able to:

* Create user-defined functions
* Create triggers

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Azure Cosmos DB provides language-integrated, transactional execution of JavaScript. When using the SQL API in Azure Cosmos DB, you can write triggers and user-defined functions (UDFs) in the JavaScript language. In this module, you will author JavaScript logic that enhances the functionalities of the SQL query languages and point operations.

After completing this module, you'll be able to:

* Create user-defined functions
* Create triggers

---

# Create user-defined functions

User-defined functions (UDFs) are used to extend the Azure Cosmos DB SQL API’s query language grammar and implement custom business logic. UDFs can only be called from inside queries as they enhance and extend the SQL query language.

> :grey_exclamation: Note
>
> UDFs do not have access to the context object and are meant to be used as compute-only code

Here is an example JSON document for a product with a `name` and a `price` property.

```js
{
  "name": "Black Bib Shorts (Small)",
  "price": 80.00
}
```

A simple SQL query to get the data from a container with many items like this one, would be constructed to include both fields.

```sql
SELECT 
    p.name,
    p.price
FROM
    products p
```

UDFs extend the SQL query language by giving you small areas where you can inject simple business logic into a query. Let's take this example, and create a user-defined function to apply business tax. In our example scenario, we want to apply a 15% tax and end up with an ideal result set that includes a third `priceWithTax` property.

```json
[
  {
    "name": "Black Bib Shorts (Small)",
    "price": 80.00,
    "priceWithTax": 92.00
  }
]
```

A user-defined function is defined as a JavaScript function that takes in one or more scalar input[s] and then returns a scalar value as the output.

```js
function name(input) {
    return output;
}
```

In this example function, the scalar input is assumed to be a number that is then multipled by 1.15 to add 15% tax.

```js
function addTax(preTax) {
    return preTax * 1.15;
}
```

The updated query includes a third projected field that references the udf function by using the `udf.addTax()` syntax passing in the `p.price` field as an input parameter and aliasing the output of that field to the name `priceWithTax`.

```sql
SELECT 
    p.name,
    p.price,
    udf.addTax(p.price) AS priceWithTax
FROM
    products p
```

---

# Create user-defined functions with the SDK

The `Scripts` property in the `Microsoft.Azure.Cosmos.Container` class contains a `CreateUserDefinedFunctionAsync` method that is used to create a new user-defined function from code.

> :grey_exclamation: Note
>
> The next set of examples assume that you already have a container variable defined.

To start, define the JavaScript function for the UDF in a string variable.

```cs
string udf = @"function addTax(preTax) {
    return preTax * 1.15;
}";
```

> :bulb: Tip
>
> Alternatively, you can use file APIs such as `System.IO.File` to read a function from a `*.js` file.

Next, create an object of type `Microsoft.Azure.Cosmos.Scripts.UserDefinedFunctionProperties` with the `Id` and `Body` properties set to the unique identifier and content of the UDF, respectively.

```cs
UserDefinedFunctionProperties properties = new()
{
    Id = "addTax",
    Body = udf
};
```

Finally, invoke the `CreateUserDefinedFunctionAsync` method of the container variable to create a new UDF passing in the properties composed earlier.

```cs
await container.Scripts.CreateUserDefinedFunctionAsync(properties);
```

---

# Add triggers to an operation

Triggers are the core way that Azure Cosmos DB SQL API can inject business logic both before and after operations. Triggers are resources stored within a container, and their code is written in JavaScript, much like stored procedures and user-defined functions.

Triggers are defined as JavaScript functions. The function is then executed when the trigger is invoked.

```js
function name() {
}
```

Within the function, the `getContext()` method retrieves a context object, which can be used to perform multiple actions, including:

* Access the HTTP request object (the source of a pre-trigger)
* Access the HTTP response object (the source of a post-trigger)
* Access the corresponding Azure Cosmos DB SQL API container

Using the context object, you can invoke the `getRequest()` or `getResponse()` methods to access the HTTP request and response objects. You can also invoke the `getCollection()` method to access the container using the JavaScript query API.

## Pre-trigger

Pre-triggers are ran before an operation and cannot have any input parameters. They can perform actions such as validate the properties of an item, or inject missing properties.

Let's walk through a simple example where a JSON item is ready to be created in a container.

```json
{
  "id": "caab0e5e-c037-48a4-a760-140497d19452",
  "name": "Handlebar",
  "categoryId": "e89a34d2-47ee-4da8-bcf6-10f552604b79",
  "categoryName": "Accessories",
  "price": 50
}
```

In this example, a pre-trigger will be created that runs before an HTTP POST operation. This trigger will check for the existence of a `label` property. If it does not exist, it will add the label property with a value of `new`. The JavaScript code for this function uses the `getContext()` and `getRequest()` methods to get the current HTTP request, and then the request body.

```js
function addLabel(item) {
    var context = getContext();
    var request = context.getRequest();
    
    var pendingItem = request.getBody();
}
```

Finally, the function will check for the existence of the `label` property, add it if it does not exist, and then return the modified item as the updated request body.

```js
if (!('label' in pendingItem))
    pendingItem['label'] = 'new';

request.setBody(pendingItem);
```

The final pre-trigger function contains the following code:

```js
function addLabel(item) {
    var context = getContext();
    var request = context.getRequest();
    
    var pendingItem = request.getBody();

    if (!('label' in pendingItem))
        pendingItem['label'] = 'new';

    request.setBody(pendingItem);
}
```

If you invoke the create operation using this pre-trigger, you should expect your resulting JSON to include the `label` property thanks to the logic in the trigger.

```json
{
  "id": "caab0e5e-c037-48a4-a760-140497d19452",
  "name": "Handlebar",
  "categoryId": "e89a34d2-47ee-4da8-bcf6-10f552604b79",
  "categoryName": "Accessories",
  "price": 50,
  "label": "new"
}
```

## Post-trigger

Post-triggers run after an operation has completed and can have input parameters even though they are not required. They have action to the HTTP response message right before it is sent to the client. They can perform actions such as updating or creating secondary items based on changes to your original item.

Let's walk through a slightly different example with the same JSON file. Now, a post-trigger will be used to create a second item with a different materialized view of our data. Our goal, is to create a second item with three JSON properties; `sourceId`, `categoryId`, and `displayName`.

```json
{
  "sourceId": "caab0e5e-c037-48a4-a760-140497d19452",
  "categoryId": "e89a34d2-47ee-4da8-bcf6-10f552604b79",
  "displayName": "Handlebar [Accessories]",
}
```

> :grey_exclamation: Note
>
> We are including the categoryId property because all items created within a post-trigger must have the same logical partition key as the original item that was the source of the trigger.

We can start our function by getting both the container and HTTP response using the `getCollection()` and `getResponse()` methods. We will also get the newly created item using the `getBody()` method of the HTTP response object.

```js
function createView() {
    var context = getContext();
    var container = context.getCollection();
    var response = context.getResponse();
    
    var createdItem = response.getBody();
}
```

Using the various properties of our newly created item, we can construct a new JavaScript object.

```js
var viewItem = {
    sourceId: createdItem.id,
    categoryId: createdItem.categoryId,
    displayName: `${createdItem.name} [${createdItem.categoryName}]`
};
```

And then we can use the `createDocument` method to create a new item from our view, and then throw or return if there are any errors or timeouts.

```js
var accepted = container.createDocument(
    container.getSelfLink(),
    viewItem,
    (error, newItem) => {
        if (error) throw error;
    }
);
if (!accepted) return;
```

The final post-trigger function contains the following code:

```js
function createView() {
    var context = getContext();
    var container = context.getCollection();
    var response = context.getResponse();
    
    var createdItem = response.getBody();
    
    var viewItem = {
        sourceId: createdItem.id,
        categoryId: createdItem.categoryId,
        displayName: `${createdItem.name} [${createdItem.categoryName}]`
    };
 
    var accepted = container.createDocument(
        container.getSelfLink(),
        viewItem,
        (error, newItem) => {
            if (error) throw error;
        }
    );
    if (!accepted) return;
}
```

---

# Create and use triggers with the SDK

The `Scripts` property in the `Microsoft.Azure.Cosmos.Container` class contains a `CreateTriggerAsync` method that is used to create a new pre/post trigger from code.

> :grey_exclamation: Note
> 
> The next set of examples assume that you already have a container variable defined.

## Create a pre-trigger

Start by creating a string variable with the definition of your pre-trigger function in JavaScript.

```cs
string preTrigger = @"function addLabel() {
    var context = getContext();
    var request = context.getRequest();
    
    var pendingItem = request.getBody();

    if (!('label' in pendingItem))
        pendingItem['label'] = 'new';

    request.setBody(pendingItem);
}";
```

> :bulb: Tip
> 
> Alternatively, you can use file APIs such as System.IO.File to read a function from a *.js file.

Now, create an object of type `Microsoft.Azure.Cosmos.Scripts.TriggerProperties` with the `Id` and `Body` properties set to the unique identifier and content of the trigger, respectively. Set the `TriggerOperation` property to `TriggerOperation.Create` for this example, and then set the `TriggerType` property to `TriggerType.Pre`.

```cs
TriggerProperties properties = new()
{
    Id = "addLabel",
    Body = preTrigger,
    TriggerOperation = TriggerOperation.Create,
    TriggerType = TriggerType.Pre
};
```

Invoke the `CreateTriggerAsync` method of the container variable to create a new pre-trigger passing in the properties composed earlier.

```cs
await container.Scripts.CreateTriggerAsync(properties);
```

## Create a post-trigger

Creating a post-trigger is almost identical to creating a pre-trigger. Start by creating a string variable with the JavaScript definition of your post-trigger function.

```cs
string postTrigger = @"function createView() {
    var context = getContext();
    var container = context.getCollection();
    var response = context.getResponse();
    
    var createdItem = response.getBody();
    
    var viewItem = {
        sourceId: createdItem.id,
        categoryId: createdItem.categoryId,
        displayName: `${createdItem.name} [${createdItem.categoryName}]`
    };
 
    var accepted = container.createDocument(
        container.getSelfLink(),
        viewItem,
        (error, newItem) => {
            if (error) throw error;
        }
    );
    if (!accepted) return;
}";
```

Now, create an object of type `Microsoft.Azure.Cosmos.Scripts.TriggerProperties` configured almost identically to the same configuration used with the pre-trigger with one key difference. Set the `TriggerType` property to `TriggerType.Post`. Then, invoke the `CreateTriggerAsync` method of the container variable to create a new post-trigger passing in the properties composed earlier.

```cs
TriggerProperties properties = new()
{
    Id = "createView",
    Body = postTrigger,
    TriggerOperation = TriggerOperation.Create,
    TriggerType = TriggerType.Post
};

await container.Scripts.CreateTriggerAsync(properties);
```

> :bulb: Tip
> 
> In these examples, we have exclusively created triggers on the `create` operation. You can also create triggers for other operations in your container.

## Use a trigger in an operation

Now that the triggers have been defined and created within the container, you can use them in an operation on the same container.

Let’s use an example where you create a new item in C# using an anonymous type.

```cs
var newItem = new
{
    id = "caab0e5e-c037-48a4-a760-140497d19452",
    name = "Handlebar",
    categoryId = "e89a34d2-47ee-4da8-bcf6-10f552604b79",
    categoryName = "Accessories",
    price = 50
};
```

Prior to invoking the operation, create an object of type `Microsoft.Azure.Cosmos.ItemRequestOptions`. Within that options object, configure the PreTriggers and PostTriggers property lists to include the triggers you would like enabled for this operation.

```cs
ItemRequestOptions options = new()
{
    PreTriggers = new List<string> { "addLabel" },
    PostTriggers = new List<string> { "createView" }
};
```

> :grey_exclamation: Note
> 
> Remember, triggers are not automatically executed; they must be specified for each database operation where you want them to execute.

Now, invoke the `CreateItemAsync` method of the container object passing in the item to be created and the options object.

```cs
await container.CreateItemAsync(newItem, requestOptions: options);
```

Finally, if you query your container, you will see that two things have happened:

1. The pre-trigger added a `label` property to your first item with a value of new.
2. The post-trigger created a second item with a materialized view of your data.

```json
[
  {
    "id": "caab0e5e-c037-48a4-a760-140497d19452",
    "name": "Handlebar",
    "categoryId": "e89a34d2-47ee-4da8-bcf6-10f552604b79",
    "categoryName": "Accessories",
    "price": 50,
    "label": "new"
  },
  {
    "id": "77875d1e-dac4-3b66-9f9c-ab6747d65952",
    "sourceId": "caab0e5e-c037-48a4-a760-140497d19452",
    "categoryId": "e89a34d2-47ee-4da8-bcf6-10f552604b79",
    "displayName": "Handlebar [Accessories]"
  }
]
```

---

# Exercise: Implement and use user defined functions with the Azure Cosmos DB SDK

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/expand-query-transaction-functionality-azure-cosmos-db-sql-api/6-exercise-implement-use-user-defined-functions-with-the-sdk)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge check

1. You have authored a user-defined function named addTax. You are writing a SQL query to return a flat array of scalar price values with the calculated tax value. Which valid SQL query should you use for this task?

- [ ] SELECT VALUE addTax(p.price) FROM products p
- [x] SELECT VALUE udf.addTax(p.price) FROM products p
> Correct. The udf.addTax() syntax is the correct syntax to user your UDF in a SQL query.
- [ ] SELECT VALUE p.price.addTax() FROM products p

2. You are tasked with taking the date values that are stored in a container, and converting them to a company-specific date format in SQL query results. Which server-side programming construct should you use for this task?

- [x] User-defined function
> Correct. A user-defined function can be used natively in a SQL query and influence the results of that query.
- [ ] Pre-trigger
- [ ] Post-trigger

3. Your team has written validation logic in JavaScript to make sure items are in your required format before committing them to a container. Which server-side programming construct should you use for this task?

- [ ] User-defined function
- [x] Pre-trigger
> Correct. A pre-trigger will run its logic prior to the item being committed to the container. At this point, any validation logic can be executed.
- [ ] Post-trigger

4. Your team has created a set of aggregate metadata items that are required to be modified anytime you successfully create or update an item within your container. Which server-side programming construct should you use for this task?

- [ ] User-defined function
- [ ] Pre-trigger
- [x] Post-trigger
> Correct. A post-trigger will run its logic after the item has successfully been created or updated. At this point, you can update the aggregate metadata item.

---

# Summary

In this module, you authored a user-defined function that enhanced the functionality of a SQL query.

Now that you have completed this module, you can:

* Create a user-defined function and use it in a SQL query
* Create a pre or post trigger that is executed with a point operation
