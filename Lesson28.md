# Create resource template for Azure Cosmos DB SQL API

Learn about automated Azure Cosmos DB SQL API resource deployments using the Azure Resource Manager with JSON and Bicep templates.

**Learning objectives**

After completing this module, you'll be able to:

* Identify the three most common resource types for Azure Cosmos DB SQL API accounts
* Create and deploy an JSON Azure Resource Manager template for an Azure Cosmos DB SQL API account, database, or container
* Create and deploy a Bicep template for an Azure Cosmos DB SQL API account, database, or container
* Manage throughput and index policies using JSON or Bicep templates

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Once an Azure Cosmos DB SQL API account is ready to go through a release lifecycle, it's not uncommon for an operations team to attempt to automate the creation of Azure Cosmos DB resources in the cloud. Automation makes it easier to deploy new environments, restore past environments, or scale a service out.

In Azure, Azure Resource Manager and Bicep templates are two of the ways you can automate the creation of Azure Cosmos Db resources. Azure Resource Manager templates are JavaScript Object Notation (JSON) files that define the infrastructure and configuration for your project. Bicep is an alternative template language that can be used to develop templates.

> :grey_exclamation: Note
>
> For this module, we will use both the `Bicep` and `JSON` syntaxes for `Azure Resource Manager` templates. The hands-on exercise for this module will use both the `Bicep` and `JSON` syntaxes to illustrate the differences.

After completing this module, you'll be able to:

* Identify the three most common resource types for Azure Cosmos DB SQL API accounts
* Create and deploy an JSON Azure Resource Manager template for an Azure Cosmos DB SQL API account, database, or container
* Create and deploy a Bicep template for an Azure Cosmos DB SQL API account, database, or container
* Manage throughput and index policies using JSON or Bicep templates

---

# Understand Azure Resource Manager resources

When creating Bicep files or Azure Resource Manager templates (ARM templates), you need to understand what resource types are available, and what values to use in your template.

Each of the resources available for Azure Cosmos DB is all listed under the `Microsoft.DocumentDB` resource provider:

| Resource type | Description |
|-----|-----|
| Microsoft.DocumentDB/databaseAccounts | Represents an account |
| Microsoft.DocumentDB/databaseAccounts/sqlDatabases | Represents a SQL API database |
| Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers | Represents a SQL API container |

Visually, you can think of these resources as a hierarchy.

![Hierarchy of resources in an Azure Cosmos DB SQL API account](https://docs.microsoft.com/en-us/learn/wwl-data-ai/create-resource-template-for-azure-cosmos-db-sql-api/media/2-hierarchy.png)

This list is not exhaustive, other resources can be created in a template such as:

* Stored procedures
* User-defined functions
* Pre-triggers
* Post-triggers

> :bulb: Tip
>
> These example container-level resources can be created with their code automatically deployed using a template.

---

# Author Azure Resource Manager templates

Authoring a template for an Azure Cosmos DB SQL API account is much like building one from scratch using the portal or from the CLI. There are three primary resources to define in a specific relationship order.

## Empty template

An Azure Resource Manager template is, at its core, a JSON file with a specific syntax you must follow. The default minimal empty template is a JSON document with a `schema` property set to `https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#`, a contentVersion property set to `1.0.0.0`, and an empty resources array. This example illustrates a minimal empty template.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
  ]
}
```

> :grey_exclamation: Note
> 
> All resources we place in this template will be JSON objects within the resources array.

## Account resource

The first resource type to define is Microsoft.DocumentDB/databaseAccounts. This represents an account that is not specific to any API. If the API is not specified, it is inferred to be a SQL API account.

An object for this resource must contain, at a minimum, the following properties:

* name
* location
* properties.databaseAccountOfferType
* properties.locations[].locationName

Here is an example of an account that has a unique name with a prefix of csmsarm and is deployed to West US.

```json
{
  "type": "Microsoft.DocumentDB/databaseAccounts",
  "apiVersion": "2021-05-15",
  "name": "[concat('csmsarm', uniqueString(resourceGroup().id))]",
  "location": "[resourceGroup().location]",
  "properties": {
    "databaseAccountOfferType": "Standard",
    "locations": [
      {
        "locationName": "westus"
      }
    ]
  }
}
```

> :grey_exclamation: Note
>
> You can define more than one location using the locations array.

## Database resource

The next resource is of type Microsoft.DocumentDB/databaseAccounts/sqlDatabases and is a child resource of the account. This relationship is defined using the dependsOn property.

An object for this resource must contain, at a minimum, the following properties:

* name
* properties.resources.id

A database can also optionally contain the following properties:

* properties.options.throughput
* properties.options.autoscaleSettings.maxThroughput

Here is an example of a database that is named cosmicworks.

```json
{
  "type": "Microsoft.DocumentDB/databaseAccounts/sqlDatabases",
  "apiVersion": "2021-05-15",
  "name": "[concat('csmsarm', uniqueString(resourceGroup().id), '/cosmicworks')]",
  "dependsOn": [
    "[resourceId('Microsoft.DocumentDB/databaseAccounts', concat('csmsarm', uniqueString(resourceGroup().id)))]"
  ],
  "properties": {
    "resource": {
      "id": "cosmicworks"
    }
  }
}
```

## Container resource

Within a database, you can define multiple child `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers` resources. Here, you can allocate throughput, configure indexing policy, and set a partition key path.

A container object must contain, at a minimum, the following properties:

* name
* properties.resource.id
* properties.resource.partitionkey.paths[]

A database can also optionally contain the following properties:

* properties.options.throughput
* properties.options.autoscaleSettings.maxThroughput
* properties.resource.indexingPolicy

Here is an example of a container that is named products, has 400 RU/s throughput, and a partition key path of /categoryId.

```json
{
  "type": "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers",
  "apiVersion": "2021-05-15",
  "name": "[concat('csmsarm', uniqueString(resourceGroup().id), '/cosmicworks/products')]",
  "dependsOn": [
    "[resourceId('Microsoft.DocumentDB/databaseAccounts', concat('csmsarm', uniqueString(resourceGroup().id)))]",
    "[resourceId('Microsoft.DocumentDB/databaseAccounts/sqlDatabases', concat('csmsarm', uniqueString(resourceGroup().id)), 'cosmicworks')]"
  ],
  "properties": {
    "options": {
      "throughput": 400
    },
    "resource": {
      "id": "products",
      "partitionKey": {
        "paths": [
          "/categoryId"
        ]
      }
    }
  }
}
```

## Final template

Now that all resources are in place, the template file should now contain the following code.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.DocumentDB/databaseAccounts",
      "apiVersion": "2021-05-15",
      "name": "[concat('csmsarm', uniqueString(resourceGroup().id))]",
      "location": "[resourceGroup().location]",
      "properties": {
        "databaseAccountOfferType": "Standard",
        "locations": [
          {
            "locationName": "westus"
          }
        ]
      }
    },
    {
      "type": "Microsoft.DocumentDB/databaseAccounts/sqlDatabases",
      "apiVersion": "2021-05-15",
      "name": "[concat('csmsarm', uniqueString(resourceGroup().id), '/cosmicworks')]",
      "dependsOn": [
        "[resourceId('Microsoft.DocumentDB/databaseAccounts', concat('csmsarm', uniqueString(resourceGroup().id)))]"
      ],
      "properties": {
        "resource": {
          "id": "cosmicworks"
        }
      }
    },
    {
      "type": "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers",
      "apiVersion": "2021-05-15",
      "name": "[concat('csmsarm', uniqueString(resourceGroup().id), '/cosmicworks/products')]",
      "dependsOn": [
        "[resourceId('Microsoft.DocumentDB/databaseAccounts', concat('csmsarm', uniqueString(resourceGroup().id)))]",
        "[resourceId('Microsoft.DocumentDB/databaseAccounts/sqlDatabases', concat('csmsarm', uniqueString(resourceGroup().id)), 'cosmicworks')]"
      ],
      "properties": {
        "options": {
          "throughput": 400
        },
        "resource": {
          "id": "products",
          "partitionKey": {
            "paths": [
              "/categoryId"
            ]
          }
        }
      }
    }
  ]
}
```

---

# Configure database or container resources

Each template resource uses the same resource type and version between both Azure Resource Manager and Bicep templates. If you learn how to build it in one language, you can easily learn it in the other.

> :grey_exclamation: Note
>
> A Bicep template does not require any "empty" template syntax. You can begin writing your definitions in a blank file.

## Account resource

The Microsoft.DocumentDB/databaseAccounts resource in Bicep must contain the same minimal properties as in an Azure Resource Manager template.

Here is an example of an account that has a unique name with a prefix of csmsarm and is deployed to West US.

```json
resource Account 'Microsoft.DocumentDB/databaseAccounts@2021-05-15' = {
  name: 'csmsbicep${uniqueString(resourceGroup().id)}'
  location: resourceGroup().location
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: 'westus'
      }
    ]
  }
}
```

> :bulb: Tip
>
> If this resource already exists from a previous deployment, the Azure Resource Manager will just skip the resource and move on to the next. This is very handy when building a template incrementally.

## Database resource

This example of a Microsoft.DocumentDB/databaseAccounts/sqlDatabases resource configures a database resource, a slight difference then the JSON template reviewed in a previous unit.

Bicep also requires resources to define a parent property defining relationships as opposed to the verbose dependsOn property.

```bicep
resource Database 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2021-05-15' = {
  parent: Account
  name: 'cosmicworks'
  properties: {
    options: {
      
    }
    resource: {
      id: 'cosmicworks'
    }
  }
}
```

## Container resource

This Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers resource is similar to the JSON equivalent, except it defines a throughput property at this level.

```bicep
resource Container 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2021-05-15' = {
  parent: Database
  name: 'customers'
  properties: {
    resource: {
      id: 'customers'
      partitionKey: {
        paths: [
          '/regionId'
        ]
      }
    }
  }
}
```

## Final template

Now that all resources are in place, the template file should now contain the following code.

```bicep
resource Account 'Microsoft.DocumentDB/databaseAccounts@2021-05-15' = {
  name: 'csmsbicep${uniqueString(resourceGroup().id)}'
  location: resourceGroup().location
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: 'westus'
      }
    ]
  }
}

resource Database 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2021-05-15' = {
  parent: Account
  name: 'cosmicworks'
  properties: {
    options: {
      
    }
    resource: {
      id: 'cosmicworks'
    }
  }
}

resource Container 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2021-05-15' = {
  parent: Database
  name: 'customers'
  properties: {
    resource: {
      id: 'customers'
      partitionKey: {
        paths: [
          '/regionId'
        ]
      }
    }
  }
}
```

---

# Configure throughput with an Azure Resource Manager template

Now that the templates have been defined, use the Azure CLI to deploy either JSON or Bicep Azure Resource Manager templates. To make things easier, the `az deployment group create` command is identical between the two types of templates that can be deployed.

## Deploy Azure Resource Manager template to a resource group

Use the `az deployment group create` with the following arguments to deploy an Azure Resource Manager template to a resource group:

| Argument | Description |
|-----|-----|
| `--resource-group` | The name of the resource group that is the target of the deployment |
| `--template-file` | The name of the file with the resources defined to deploy |

```bash
az deployment group create \
    --resource-group '<resource-group>' \
    --template-file '.\template.json'
```

There are also other optional arguments that you can use to customize the deployment.

For example, you can name a deployment to make it easier to find in logs or the Azure portal using the `--name` argument.

```bash
az deployment group create \
    --resource-group '<resource-group>' \
    --name '<deployment-name>' \
    --template-file '.\template.json'
```

You can also define parameters inline with the command using the `--parameters` argument.

```bash
az deployment group create \
    --resource-group '<resource-group>' \
    --template-file '.\template.json' \
    --parameters name='<value>'
```

If you prefer to use a parameter JSON file, that is also possible with the `--parameters` argument.

```bash
az deployment group create \
    --resource-group '<resource-group>' \
    --template-file '.\template.json' \
    --parameters '@.\parameters.json'
```

## Deploy Bicep template to a resource group

Fortunately, deploying a Bicep template is identical in practice to deploying a JSON template. In this example, a Bicep template is deployed with the only difference being a change in the name of the file's extension.

```bash
az deployment group create \
    --resource-group '<resource-group>' \
    --template-file '.\template.bicep'
```

---

# Manage index policies through Azure Resource Manager templates

It is common to define an indexing policy as part of deploying an Azure Cosmos DB account and its resources in an automated manner. Both the JSON and Bicep syntax for Azure Resource Manager templates supports defining indexing policies natively. However, the syntax can be tricky if you haven't tried it before.

For the examples in this unit, let's assume that we want to deploy the following indexing policy to our `products` container.

## Defining an indexing policy in JSON templates

Let's assume that we want to deploy the following indexing policy to our `products` container in our account.

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {
      "path": "/price/*"
    }
  ],
  "excludedPaths": [
    {
      "path": "/*"
    }
  ]
}
```

The `indexingPolicy` object can be lifted with no changes and set to the `properties.resource.indexingPolicy` property of the `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers`.

```json
{
  "type": "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers",
  "apiVersion": "2021-05-15",
  "name": "[concat('csmsarm', uniqueString(resourceGroup().id), '/cosmicworks/products')]",
  "dependsOn": [
    "[resourceId('Microsoft.DocumentDB/databaseAccounts', concat('csmsarm', uniqueString(resourceGroup().id)))]",
    "[resourceId('Microsoft.DocumentDB/databaseAccounts/sqlDatabases', concat('csmsarm', uniqueString(resourceGroup().id)), 'cosmicworks')]"
  ],
  "properties": {
    "options": {
      "throughput": 400
    },
    "resource": {
      "id": "products",
      "partitionKey": {
        "paths": [
          "/categoryId"
        ]
      },
      "indexingPolicy": {
        "indexingMode": "consistent",
        "automatic": true,
        "includedPaths": [
          {
            "path": "/price/*"
          }
        ],
        "excludedPaths": [
          {
            "path": "/*"
          }
        ]
      }
    }
  }
}
```

## Defining an indexing policy in Bicep templates

Let's assume that we want to deploy the following indexing policy to our customers `container` in our account.

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {
      "path": "/address/*"
    }
  ],
  "excludedPaths": [
    {
      "path": "/*"
    }
  ]
}
```

A few small changes are required to use this indexing policy in Bicep. These changes include:

* Removing the double quotation from property names
* Changing property values from double quotes to single quotes
* Removing commas typically required in JSON

```bicep
resource Container 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2021-05-15' = {
  parent: Database
  name: 'customers'
  properties: {
    resource: {
      id: 'customers'
      partitionKey: {
        paths: [
          '/regionId'
        ]
      }
      indexingPolicy: {
        indexingMode: 'consistent'
        automatic: true
        includedPaths: [
          {
            path: '/address/*'
          }
        ]
        excludedPaths: [
          {
            path: '/*'
          }
        ]
      }
    }
  }
}
```

## Updating an indexing policy on an existing container

If a resource of type `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers` already exists, and all other properties match, you can update the indexing policy by solely changing the values within the `properties.resource.indexingPolicy` property. Azure Resource Manager will only change the indexing policy while keeping the rest of the container intact.

The command for deployment is the same as the initial deployment.

```bash
az deployment group create \
    --resource-group '<resource-group>' \
    --template-file '.\template.json' \
    --name 'jsontemplatedeploy'
```

```bash
az deployment group create \
    --resource-group '<resource-group>' \
    --template-file '.\template.bicep' \
    --name 'biceptemplatedeploy'
```

---

# Exercise: Create an Azure Cosmos DB SQL API container using Azure Resource Manager templates

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/create-resource-template-for-azure-cosmos-db-sql-api/7-exercise-create-container-using-azure-resource-manager-templates)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge check

1. Which property should you set, in Bicep, to establish a relationship between a container and its corresponding database?

- [ ] name
- [ ] dependsOn
- [x] parent
> Correct. The parent property establishes a relationship between two resources in Bicep.

2. Which command should you use to deploy a Bicep or JSON template to Azure Resource Manager?

- [x] az deployment group creates
> Correct. The az deployment group creates command is the correct command to deploy both Bicep and JSON templates.
- [ ] az group deployment creates
- [ ] az bicep deploys

3. You have an Azure Cosmos DB SQL API account name adventureworksllc, a database named cosmicworks, and a container named products. In a JSON Azure Resource Manager template, how do you specify the name property of this resource?

- [ ] products
- [ ] adventureworksllc/cosmicworks/products
> Correct. The name of a container resource is defined in the <account>/<database>/<container> syntax.
- [ ] databaseAccounts/adventureworksllc/sqlDatabases/cosmicworks/containers/products

---

# Summary

In this module, you used Azure Resource Manager and Bicep templates to automate the deployment and management of Azure Cosmos DB SQL API resources in the cloud.

> :bulb; Tip
>
> Remember, the Microsoft.DocumentDB resource provider has many other resources that can be deployed using templates. This module only covered a subset of possible resources.

Now that you have completed this module, you can:

* Describe the differences between the various resource types within the Microsoft.DocumentDB resource provider.
* Create and deploy SQL API accounts, databases, or containers using an Azure Resource Manager or Bicep template.
* Adjust the properties of containers and databases using an Azure Resource Manager or Bicep template.
