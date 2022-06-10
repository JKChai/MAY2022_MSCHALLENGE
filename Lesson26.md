# Implement security in Azure Cosmos DB SQL API

We will learn the different security models that Azure Cosmos DB uses.

**Learning objectives**

* Implement network level access control
* Review data encryption options
* Use role-based access control (RBAC)
* Access Account Resources using Azure Active Directory
* Understand Always Encrypted

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Data security is a shared responsibility between you, and your database provider. On the client side you should keep your servers patches up to date, use HTTP/TLS encryption and make sure your administrative accounts have strong passwords. On the cloud side, Azure Cosmos DB fulfills a large set of security requirements to secure your data. Let's take a quick look on how Azure Cosmos DB secures your data, later in this module we will cover some of these requirements in more detail.

| Security Requirements | Azure Cosmos DB's security approach |
|-----|-----|
| Network security | Azure Cosmos DB supports policy driven IP-based access controls for inbound firewall support. Azure Cosmos DB alloys you to enable a combination of IPs and IP ranges. Only machines with the proper IPs or inside the IP ranges can access the Azure Cosmos DB resources, all other machines are blocked by Azure Cosmos DB. |
| Authorization | Azure Cosmos DB uses hash-based message authentication code (HMAC) for authorization. You can use either a primary key, or a resource token allowing fine-grained access to a resource such as a document. |
| Users and permissions | Using the primary key for the account, you can create user resources and permission resources per database. The resource token is then used during authentication to provide or deny access to the resource. |
| Active directory integration (Azure RBAC) | You can also provide or restrict access to the Cosmos account, database, container, and offers (throughput) using Access control (IAM) in the Azure portal. IAM provides role-based access control and integrates with Active Directory. You can use built in roles or custom roles for individuals and groups. |
| Global replication | In the context of security, global replication ensures data protection against regional failures. |
| Regional failovers | If you have replicated your data in more than one data center, Azure Cosmos DB automatically rolls over your operations should a regional data center go offline. |
| Local replication | Even within a single data center, Azure Cosmos DB automatically replicates data for high availability giving you the choice of consistency levels. |
| Restore deleted data | The automated online backups can be used to recover data you may have accidentally deleted up to ~30 days after the event. |
| Protect and isolate sensitive data | Documents and backups are now encrypted at rest. Personal data and other confidential data can be isolated to specific container and read-write, or read-only access can be limited to specific users. |
| Monitor for attacks | By using audit logging and activity logs, you can monitor your account for normal and abnormal activity. |
| Respond to attacks | Once you have contacted Azure support to report a potential attack, a 5-step incident response process is kicked off to restore normal service security and operations as quickly as possible after an issue is detected and an investigation is started. |
| Geo-fencing | Azure Cosmos DB ensures data governance for sovereign regions (for example, Germany, China, US Gov). |
| Protected facilities | Data in Azure Cosmos DB is stored on SSDs in Azure's protected data centers. |
| HTTPS/SSL/TLS encryption | All connections to Azure Cosmos DB support HTTPS. Azure Cosmos DB also supports TLS 1.2. |
| Encryption at rest | All data stored into Azure Cosmos DB is encrypted at rest. |
| Patched servers | As a managed database, Azure Cosmos DB eliminates the need to manage and patch servers, that's done for you, automatically. |
| Administrative accounts with strong passwords | Security via TLS and HMAC secret based authentication is baked in by default. |

After completing this module, you'll be able to:

* Implement network level access control
* Review data encryption options
* Use role-based access control (RBAC)
* Access Account Resources using Azure Active Directory
* Understand Always Encrypted

---

# Implement network-level access control

Azure Cosmos DB supports IP-based access controls for inbound firewall support. Configuring a firewall for your Azure Cosmos account makes the account accessible only from an approved set of machines and/or cloud services. The firewall will only help secure the network layer. Connections will still require the caller to present a valid authorization token.

## IP access control

By default Azure Cosmos DB accounts allow all networks and IPs from the internet to access the account. The connection request is only required to present a valid authorization token. To configure network security for your Azure Cosmos DB account, you'll need to provide a single IPv4 or CIDR range to limit the access from only the provided IP or IP range. Connections from any IP that is not part of the provided allowed IP list, will receive a 403 (Forbidden) response. Since you might be using the Azure portal to run scripts in the Azure Cosmos DB account Data Explorer, or might be reviewing metrics of the account, you should also enable the Allow access from Azure Portal option in the firewall setup. To use these Azure Cosmos DB features in the Azure portal, you'll also need to select `+ Add my current IP (#.#.#.#)` to add your current IP.

Access to an Azure Cosmos DB account can also be done with subnet and VNET access control. You can combine both access for a specific subnet within a VNET and the IP-based firewall to limit access from public IPs. Regardless of the network security used, you still need to present a valid authorization token.

You can set up your Azure Cosmos DB account firewall rules:

* From the Azure portal.
* Using an Azure Resource Manager template.
* Through Azure CLI or Azure PowerShell by updating the ipRangeFilter property

## Configure an IP firewall by using the Azure portal

Configure an IP firewall by using the Azure portal:

1. On the Azure Cosmos DB account page **Select Firewall and virtual networks**.
2. Select the **Selected networks** option.

![Diagram that shows the firewall setting options.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-security-azure-cosmos-db-sql-api/media/2-firewall-setting.png)

3. (optional) To add a virtual network:

    a. Select `+ Add existing virtual` to pick from a list of already existing virtual networks. In this example only one existing virtual network exists, virtualnetwork1.

    ![Diagram that shows a list of existing virtual networks.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-security-azure-cosmos-db-sql-api/media/2-firewall-add-existing-virtual-network.png)

    b. Select `+ Add new virtual network`.

    ![Diagram that shows the options to create a new virtual network.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-security-azure-cosmos-db-sql-api/media/2-firewall-add-new-virtual-network.png)

4. (optional) Select `+ Add my current IP (#.#.#.#)` to be able to connect from your current client. Review your client IP displayed in this option and validate if it's the correct Public IP address. This option helps simplify development, since apps running on your client IP would be allowed to connect to the Azure Cosmos DB account.
5. Under Firewall, add a single IPv4 or CIDR range. It's this option, that will limit the access from only the provided IP or IP ranges. Add as many IPs or IP ranges as needed. You'll need to get the Public IP addresses from cloud services, virtual machines, or any computer that needs to connect to the Azure Cosmos DB. If those services, physical or virtual machines have an IP that is inside an allowed Virtual Network, there might be no need to add their public IP.
6. You can enable requests to access the Azure portal by selecting the Allow access from Azure portal option.
7. (optional) You can enable access from other sources within Azure by selecting the Accept connections from within Azure datacenters option. This option provides access to the Azure Cosmos DB account from services that don't provide a static IP like Azure Stream Analytics and Azure Functions. This option allows all request from Azure, including request that could originate from other customer subscriptions. The list of IPs allowed by this option is wide, so it limits the effectiveness of a firewall policy. Use this option only if your requests don’t originate from static IPs or subnets in virtual networks.

## Troubleshoot issues with an IP access control policy

Let's review some scenarios of issues with when an IP access control policy is enabled:

### Azure portal blocked

When the IP access control policy is enabled, to enable portal data-plane operations like browsing containers and querying documents, you need to select Allow access from Azure portal option in the Azure Cosmos DB account Firewall and virtual networks settings.

### SDK blocked

If your applications are getting a generic 403 Forbidden response, verify the allowed IP list for your account, and make sure that the correct policy configuration is applied to your Azure Cosmos DB account.

### Source IPs in blocked in requests

To troubleshoot, enable diagnostic logging on your Azure Cosmos DB account. The 403 return codes will be logged for firewall-related messages. You can filter these messages to return the source IPs for the blocked requests.

### Requests from a subnet with a service endpoint for Azure Cosmos DB enabled

Requests from a subnet in a virtual network that has a service endpoint for Azure Cosmos DB enabled sends the virtual network and subnet identity to Azure Cosmos DB accounts. These requests don't have the public IP of the source, so IP filters reject them. To allow access from specific subnets in virtual networks, add an access control list.

### Private IP addresses in list of allowed addresses

Make sure that no private IP address is specified in the allowed addresses list.

---

# Review data encryption options

Azure Cosmos DB stores its primary databases on Solid-State Driver (SSDs). Media attachments and backups are stored in Azure Blob storage, which is usually placed on HHDs. All Azure Cosmos DB regions have encryption turned on for all user data. Azure Cosmos DB now uses encryption at rest for all its databases, backups, and media. Encryption at rest refers to encryption of data on nonvolatile storage devices like HHDs and SSDs. Additionally, when Azure Cosmos DB data is in transit, or over the network, that data is also encrypted. Encryption at rest and in transit means that Azure Cosmos DB is using end-to-end encryption. There's no extra cost for end-to-end encryption.

All this encryption is automatic, so you don't have to enable any setting to use it, which means that encryption is on by default. Azure Cosmos DB uses EAS-256 encryption on all regions where the account is running at. Azure Cosmos DB data is encrypted and decrypted on the fly with service-managed keys without affecting performance or availability. These service-managed keys are managed by Microsoft, but you have to option to use your own customer-managed keys. Microsoft has a set of internal guidelines for encryption key rotation, which Cosmos DB follows.

## Azure Cosmos DB at rest encryption implementation

Azure Cosmos DB encryption at rest uses security technologies like secure key storage systems, encrypted networks, and cryptographic APIs. The storage of encrypted data and the management of the keys is done separately. Systems that decrypt and process data have to communicate with systems that manage keys as indicated in the diagram below.

![Diagram that shows the systems that decrypt and process data have to communicate with systems that manage keys.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-security-azure-cosmos-db-sql-api/media/3-encryption-keys.png)

---

# Use role-based access control (RBAC)

Azure role-based access control (RBAC) is provided in Azure Cosmos DB to do common management operations. Role base access control will grant or deny access to resources and operation on Azure Cosmos DB resources. These roles can be assigned with an Azure Active Directory account, to users, groups, service principals, or managed instances. Role assignments aren't used for data plane operations, they're scoped to control plane access only, such as the Azure Cosmos DB account, databases, containers, and offers (throughput). Let's take a look at these built-in roles supported by Azure Cosmos DB.

## Built-in roles

| Built-in role | Description |
|-----|-----|
| DocumentDB Account Contributor | Can manage Azure Cosmos DB accounts. |
| Cosmos DB Account Reader | Can read Azure Cosmos DB account data. |
| Cosmos Backup Operator | Can submit a restore request for Azure portal for a periodic backup enabled database or a container. Can modify the backup interval and retention on the Azure portal. Can't access any data or use Data Explorer. |
| CosmosRestoreOperator | Can do restore action for Azure Cosmos DB account with continuous backup mode. |
| Cosmos DB Operator | Can provision Azure Cosmos accounts, databases, and containers. Can't access any data or use Data Explorer. |

## Identity and access management (IAM)

Azure RBAC on Azure Cosmos DB is setup using the Access Control (IAM) pane in the Azure portal. Individuals or groups can be assigned built-in roles or custom roles.

![Diagram that shows the Azure Cosmos DB Identity and access management role-based access control options.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-security-azure-cosmos-db-sql-api/media/4-use-role-based-access-control.png)

# Custom Controls

Custom roles provide users a way to create Azure role definitions with a custom set of resource provider operations. Custom roles can be applied to Active Directory service principal and be used to define RBACs just like the built-in roles.

# Preventing changes from the Azure Cosmos DB SDKs

Clients connecting from Azure Cosmos DB SDK can be locked down to prevent from changing any property for the Azure Cosmos accounts, databases, containers, and throughput. The operations involving reading and writing data to Cosmos containers themselves would not affected. This lock down could be desirable to have a higher degree of control and governance for production environments. Only users with the right Azure role and Active Directory credentials, including Managed Service Identities, can change resource properties when this feature is enabled.

If your applications do any of the following actions, they'll no longer be able to do these actions through the Cosmos DB SDK, tools that connect via account keys or from the Azure portal. These actions will need to be executed from Azure Resource Manager Templates, PowerShell, Azure CLI, REST, or Azure Management Library.

* Any change to the Cosmos account including any properties or adding or removing regions.
* Creating, deleting child resources such as databases and containers.
* Updating throughput on database or container level resources.
* Modifying container properties including index policy, TTL and unique keys.
* Modifying stored procedures, triggers or user-defined functions.

---

# Access account resources using Azure Active Directory

Azure Cosmos DB exposes a built-in role-based access control (RBAC) system that lets you:

* Authenticate your data requests with an Azure Active Directory (Azure AD) identity.
* Authorize your data requests with a fine-grained, role-based permission model.

To set up these roles, we'll review the permission model, role definitions ad role assignments RBAC concepts in more detail. Azure portal support for role management is'not available yet.

![Diagram that shows the role-based access control options for data access.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-security-azure-cosmos-db-sql-api/media/5-role-assignment.png)

## Permission model

The permission model covers reading and writing data operations against a database. It won't cover management resource operations like creating, replacing, deleting databases, containers, throughput, Stored Procedures, triggers, or User-defined functions.

The permission model will allow you to grant or deny the following actions:

| Name | Corresponding database operation(s) |
|-----|-----|
| `Microsoft.DocumentDB/databaseAccounts/readMetadata` | Read account metadata. See Metadata requests for details. |
| `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/create` | Create a new item. |
| `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read` | Read an individual item by its ID and partition key (point-read). |
| `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/replace` | Replace an existing item. |
| `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/upsert` | "Upsert" an item, which means create it if it doesn't exist, or replace it if it exists. |
| `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/delete` | Delete an item. |
| `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/executeQuery` | Execute a SQL query. |
| `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/readChangeFeed` | Read from the container's change feed. |
| `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/executeStoredProcedure` | Execute a stored procedure. |
| `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/manageConflicts` | Manage conflicts for multi-write region accounts (that is, list and delete items from the conflict feed). |

Wildcards are also supported for both the container and item levels.

* `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/*`
* `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*`

## Metadata requests

Metadata requests don't return any data stored inside your Azure Cosmos DB account. Metadata requests are issued by the SDKs as read-only request during initialization and to serve specific data requests. They could return information like the partition key of a container, regions the account is in or a list of a container's partitions.

Metadata requests are covered by the action:

* `Microsoft.DocumentDB/databaseAccounts/readMetadata`

They can be assigned at the account, database, or container scope. The actions allowed are:

| Scope | Requests allowed by the action |
|-----|-----|
| Account | 
- Listing the databases under the account 
- For each database under the account, the allowed actions at the database scope |
| Database |	
- Reading database metadata
- Listing the containers under the database
- For each container under the database, the allowed actions at the container scope |
| Container	| 
- Reading container metadata
- Listing physical partitions under the container
- Resolving the address of each physical partition |

## Role definitions

Role definitions, contains a list of allowed actions. Azure Cosmos DB can use either built-in or custom role definitions, let's review those definitions further.

### Built-in role definitions

Azure Cosmos DB exposes the following two built-in roles definitions:

| ID | Name | Included actions |
| 00000000-0000-0000-0000-000000000001 | Cosmos DB Built-in Data Reader |
- Microsoft.DocumentDB/databaseAccounts/readMetadata
- Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read
- Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/executeQuery
- Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/readChangeFeed |
| 00000000-0000-0000-0000-000000000002 | Cosmos DB Built-in Data Contributor | 
- Microsoft.DocumentDB/databaseAccounts/readMetadata
- Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/*
- Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/* |

### Custom Role definition

You can define custom role definition using Azure PowerShell, Azure CLI, or Azure Resource Manager templates. When creating a custom role definition, you need to provide:

* The name of your Azure Cosmos DB account.
* The resource group containing your account.
* The type of the role definition: CustomRole.
* The name of the role definition.
* A list of actions that you want the role to allow.
* One or multiple scope(s) that the role definition can be assigned at; supported scopes are:
    *  `/` (account-level),
    *  `/dbs/<database-name>` (database-level),
    * `/dbs/<database-name>/colls/<container-name>` (container-level).

## Role assignments

The final component to define our data plane role base access control is our Roll Assignment. Role definitions get assigned to specific Azure Active Directory identities through role assignments. This assignment also defines the scope that the role definition applies to, the account, the database, or the container. You can define role assignments using Azure PowerShell, Azure CLI, or Azure Resource Manager templates. When creating a role assignment, you need to provide:

* The name of your Azure Cosmos DB account.
* The resource group containing your account.
* The ID of the role definition to assign.
* The principal ID of the identity that the role definition should be assigned to.
* The scope of the role assignment; supported scopes are:
    * `/` (account-level)
    * `/dbs/<database-name>` (database-level)
    * `/dbs/<database-name>/colls/<container-name>` (container-level)

The scope must match or be a subscope of one of the role definition's assignable scopes.

## Initialize the SDK with Azure AD

To use Azure Cosmos DB RBAC, you'll no longer pass the primary key. You'll pass an instance of a `TokenCredential` class. This instance will provide the Azure Cosmos DB SDK the context to fetch the needed Azure Active Directory token for of the identity you'll use. Your `TokenCredentialinstance` must resolve to the identity (principal ID) that you've assigned your roles to. In this example, a service principal is used with a `ClientSecretCredential` instance.

```cs
TokenCredential servicePrincipal = new ClientSecretCredential(
    "<azure-ad-tenant-id>",
    "<client-application-id>",
    "<client-application-secret>");
CosmosClient client = new CosmosClient("<account-endpoint>", servicePrincipal);
```

## Use data explorer

The data explorer on your Azure Cosmos DB pane doesn't support the Azure Cosmos DB RBAC yet. To use your Azure AD identity when exploring your data, you must use the Azure Cosmos DB Explorer instead. When you access the Azure Cosmos DB Explorer with the specific `?feature.enableAadDataPlane=true` query parameter and sign in.

## Audit data requests

When using the Azure Cosmos DB RBAC, diagnostic logs will now get identity and authorization information for each data operation. This information will allow you to retrieve the Azure Active Directory identity for every request sent to your Azure Cosmos DB account.

The identity and authorization data will also be added to the DataPlaneRequest logs. You should now see the columns `aadPrincipalId_g` and `aadAppliedRoleAssignmentId_g` for the principal ID of Azure Active Directory and the role assignment used respectively.

## Enforcing RBAC as the only authentication method

When using RBAC, you can disable the Azure Cosmos DB account primary and secondary key if you wish to use RBAC exclusively. Disabling the accounts can be done by setting the `disableLocalAuth` to true when creating or updating your Azure Cosmos DB account using Azure Resource Manager templates.

## Limits

* You can create up to 100 role definitions and 2,000 role assignments per Azure Cosmos DB account.
* You can only assign role definitions to Azure AD identities belonging to the same Azure AD tenant as your Azure Cosmos DB account.
* Azure AD group resolution isn't currently supported for identities that belong to more than 200 groups.
* The Azure AD token is currently passed as a header with each individual request sent to the Azure Cosmos DB service, increasing the overall payload size.

---

# Understand Always Encrypted

*Always encrypted* encrypts sensitive data like credit card numbers or payroll information inside client-side applications. The database receives the data already encrypted from the client side, and doesn't contain the decryption keys. This means that the database has no way to decrypt this data. When the client requires the data, the database will send the encrypted data back and since the client has the decryption keys, the client will decrypt the data.

*Always Encrypted* brings client-side encryption capabilities to Azure Cosmos DB. Since the data is encrypted on the client side, the Azure Cosmos DB service is never provided with the decryption keys or the unencrypted data. You own, control, and manage the decryption keys. These keys are stored in Azure Key Vault, you can apply policies to control what keys and secrets each client can access.

## Always encrypted concepts

*Always encrypted* uses encryption keys and decryption policies.

### Encryption keys

#### Data Encryption keys

Always encrypted requires that you create data encryption keys (DEK) ahead of time. The DEKs are created at client-side using the Azure Comsos DB SDK. These DEKs are stored in the Azure Cosmos DB service. The DEKs are defined at the database level so they can be shared across multiple containers. Each DEK you create can be used to encrypt only one property, or can be used to encrypt many properties. You can have multiple DEKs per databases.

#### Customer-managed keys

A DEK must be wrapped by a customer-managed key (CMK) before it stored in Azure Cosmos DB. Since CMKs control the wrapping and unwrapping of the DEKs, they control the access to the data that is encrypted with those DEKs. CMK storage is designed as an extensible/plug-in model, with a default implementation that expects them to be stored in Azure Key Vault. The relationship between these components is displayed in the following diagram.

![Diagram that shows the always encrypted encryption keys and how are connected with Azure Cosmos DB.](https://docs.microsoft.com/en-us/learn/wwl-data-ai/implement-security-azure-cosmos-db-sql-api/media/6-always-encrypted-cmk-dek.png)

### Encryption policy

Encryption policies are container-level specifications describing how the JSON properties should be encrypted. These policies are similar to indexing policies in structure. In the current release, you must create these policies at the container creation time and can't be updated once they're created.

For each property that you want to encrypt, the encryption policy defines:

* The path of the property in the form of `/property`. Only top-level paths are currently supported, nested paths such as `/path/to/property` aren't supported.
* The ID of the DEK to use when encrypting and decrypting the property.
* An encryption type. It can be either randomized or deterministic.
* The encryption algorithm to use when encrypting the property. The specified algorithm can override the algorithm defined when creating the key if they're compatible.

You can't encrypt the ID or the container's partition key.

#### Randomized vs. deterministic encryption

Because the Azure Cosmos DB service might need to support some querying capabilities over the encrypted data, and it can't evaluate the data in plain text, Always Encrypted has more than one encryption type. The encryption types supported by Always Encrypted are:

* Deterministic encryption: It always generates the same encrypted value for any given plain text value and encryption configuration. Using deterministic encryption allows queries to do equality filters on encrypted properties. However, it may allow attackers to guess information about encrypted values by examining patterns in the encrypted property. This is especially true if there's a small set of possible encrypted values, such as True/False, or North/South/East/West region.
* Randomized encryption: It uses a method that encrypts data in a less predictable manner. Randomized encryption is more secure, but prevents queries from filtering on encrypted properties.

## Setting up and using Always Encrypted with Azure Cosmos DB

Setting up Always encrypted will be a multi-step process, from setting your CMK on the Azure Key Vault to creating your DEK and finally creating containers with the encryption policies.

### Setup Azure Key Vault

Before we create our CMK, we need to create a new or use an existing Azure Key Vault to store the CMK in.

1. In the Azure portal, follow the instructions to create a new Azure Key Vault or pick and existing one.
2. Under Keys, create a new Key.
3. Once the key is created, browse to its current version, and copy its full key identifier: `https://<my-key-vault>.vault.azure.net/keys/<key>/<version>`. If you would like to always use the latest version of the key, just omit the key version at the end of the key identifier.

An Azure Active Directory identity will be needed to grant the Azure Cosmos DB SDK access to the Azure Key Vault instance. An Azure AD application or a managed instance is commonly used as a proxy between the client code and the Azure Key Vault instance. To use an AD application as the proxy use these steps:

1. Create a new Azure Active Directory application and add a client secret as described in this quickstart.
2. In the Azure Key Vault instance, under Access policies, select + Add Access Policy and add a new policy:
    a. In Key permissions, select Get, List, Unwrap Key, Wrap Key, **Verify, and Sign.
    b. In Select principal, search for the AAD application you've created before.

### Initialize the SDK

To use Always Encrypted, an instance of an `EncryptionKeyStoreProvider` must be attached to your Azure Cosmos DB SDK instance. This object is used to interact with the key store hosting your CMKs. The default key store provider for Azure Key Vault is named `AzureKeyVaultKeyStoreProvider`. To use the `AzureKeyVaultKeyStoreProvider`, you'll need to add the `Microsoft.Data.Encryption.AzureKeyVaultProvider` package.

The following snippets show how to use the identity of an Azure AD application with a client secret.

```cs
var tokenCredential = new ClientSecretCredential(
    "<aad-app-tenant-id>", "<aad-app-client-id>", "<aad-app-secret>");
var keyStoreProvider = new AzureKeyVaultKeyStoreProvider(tokenCredential);
var client = new CosmosClient("<connection-string>")
    .WithEncryption(keyStoreProvider);
```

### Create a data encryption key

Once we created the CMK in the Azure Key Vault, its time to create our DEK in the parent database. To create this DEK, we'll use the `CreateClientEncryptionKeyAsync` method and pass the following information:

* A string identifier that will uniquely identify the key in the database.
* The encryption algorithm intended to be used with the key. Only one algorithm is currently supported.
* The key identifier of the CMK stored in Azure Key Vault. This parameter is passed in a generic `EncryptionKeyWrapMetadata` object where the name can be any friendly name you want, and the value must be the key identifier.

The following snippets show how we create this DEK in .NET.

```cs
var database = client.GetDatabase("my-database");
await database.CreateClientEncryptionKeyAsync(
    "my-key",
    DataEncryptionKeyAlgorithm.AEAD_AES_256_CBC_HMAC_SHA256,
    new EncryptionKeyWrapMetadata(
        keyStoreProvider.ProviderName,
        "akvKey",
        "https://<my-key-vault>.vault.azure.net/keys/<key>/<version>"));
```

It's a good security practice to rotate your CMKs regularly. You should also rotate your CMK if you suspect that the current CMK has been compromised. Once the CMK is rotated, you provide that new CMK identifier for the DEK rewrapper to start using it. This operation doesn't affect the encryption of your data, but the protection of the DEK. Review the following script that rewraps the new CMK to the DEK:

```cs
await database.RewrapClientEncryptionKeyAsync(
    "my-key",
    new EncryptionKeyWrapMetadata(
        keyStoreProvider.ProviderName,
        "akvKey",
        " https://<my-key-vault>.vault.azure.net/keys/<new-key>/<version>"));
```

### Create a container with encryption policy

Now that we've configured both the CMK and DEK, its time to create a new container using the .NET SDK. The following snippets show how we create the container `my-container` using the DEK `my-key` created in the previous example. This container will have two-encryption policy on properties, `property1` and, `property2` and a partition key `partition-key`. Both properties will use the `my-key` DEK, but one will use encryption type deterministic, while the other will use randomized.

```cs
var path1 = new ClientEncryptionIncludedPath
{
    Path = "/property1",
    ClientEncryptionKeyId = "my-key",
    EncryptionType = EncryptionType.Deterministic.ToString(),
    EncryptionAlgorithm = DataEncryptionKeyAlgorithm.AEAD_AES_256_CBC_HMAC_SHA256.ToString()
};
var path2 = new ClientEncryptionIncludedPath
{
    Path = "/property2",
    ClientEncryptionKeyId = "my-key",
    EncryptionType = EncryptionType.Randomized.ToString(),
    EncryptionAlgorithm = DataEncryptionKeyAlgorithm.AEAD_AES_256_CBC_HMAC_SHA256.ToString()
};
await database.DefineContainer("my-container", "/partition-key")
    .WithClientEncryptionPolicy()
    .WithIncludedPath(path1)
    .WithIncludedPath(path2)
    .Attach()
    .CreateAsync();
```

## Write encrypted data

When an Azure Cosmos DB document is written, the SDK evaluates the encryption policies to determine if any properties need to be encrypted and how to encrypt them. If a property needs to be encrypted, it creates a base 64 string in place of the original text.

### Encryption of complex types

* When the property to encrypt is a JSON array, every entry of the array is encrypted.
* When the property to encrypt is a JSON object, only the leaf values of the object get encrypted. The intermediate subproperty names remain in plain text form.

## Read encrypted items

If you're fetching a single item by `ID` and partition key, running queries, or reading the change feed, no extra steps will be required to decrypt the encrypted properties. This is because the SDK figures out which properties need to be decrypted by looking up the encryption policy.

### Filter queries on encrypted properties

The `AddParameterAsync` method passes the value of the query parameter used in queries that filter on encrypted properties. This method takes the following arguments:

* The name of the query parameter.
* The value to use in the query.
* The path of the encrypted property (as defined in the encryption policy).

We can see an example that uses the `AddParameterAsync` below:

```cs
var queryDefinition = container.CreateQueryDefinition(
    "SELECT * FROM c where c.property1 = @Property1");
await queryDefinition.AddParameterAsync(
    "@Property1",
    1234,
    "/property1");
```

Encrypted properties can only be used in equality filters (`WHERE c.property = @Value`). Any other usage will return unpredictable and wrong query results.

### Reading documents when only a subset of properties can be decrypted

Different document properties in the same container can use different encryption policies. Each policy can use different CMKs to encrypt the properties. If your client has access to some of the CMKs used to decrypt some of the properties, but not access to other CMKs to decrypt other properties, you can still partially query the documents with the properties you can decrypt. You should just remove those properties from your queries, which you don't have access to their CMKs. For example, if `property1` was encrypted with `key1` and `property2` was encrypted with `key2`, and your app only has access to `key1`, your query should ignore `property2`. This query could look like, `SELECT c.property1, c.property3 FROM c`.

---

# Exercise: Store Azure Cosmos DB SQL API account keys in Azure Key Vault

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/implement-security-azure-cosmos-db-sql-api/7-exercise-store-account-keys-azure-key-vault)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge check

1. One reason your request are getting a 403 status code returned is?

- [ ] Because by default Azure Cosmos DB blocks access to all incoming IPs.
- [ ] Because by default Azure Cosmos DB allows access to all incoming IPs.
- [x] Because the firewall is enabled, and your application's IP must be added to the firewall's allowed access list.
> That's correct. If the firewall is enabled, your application's IP must be added to the allowed access list.

2. What do you need to do to use the Data Explorer with RBAC?

- [ ] Access the Azure Cosmos DB Data Explorer with the specific ?feature.enableAadDataPlane=true query parameter and sign-in.
- [ ] Use CLI or PowerShell to enable the feature enableAadDataPlane=true.
- [x] Data Explorer doesn't support RBAC.
> That's correct. Data Explorer doesn't support RBAC, to query your data, use Azure Cosmos DB Explorer instead.

3. What is the proper set of steps to set up Always Encrypted for Cosmos DB?

- [ ] Create a CMKs in Azure Key Vault, Use the SDK to create the DEKs (with the CMKs) on a database, add the DEK and the encryption policies to an existing container.
- [x] Create a CMKs in Azure Key Vault, Use the SDK to create the DEKs (with the CMKs) on a database, create a container with the DEK and encryption policies.
> That's correct. You can only set up Always Encrypted at the container's creation time.
- [ ] Create a CMKs in Azure Key Vault, Use the SDK to create the DEKs (with the CMKs) on a new or existing container with the encryption policies.

---

# Summary

In this module, you learned the security options for Azure Cosmos DB.

Now that you have completed this module, you can:

* Implement network level access control
* Review data encryption options
* Use role-based access control (RBAC)
* Access Account Resources using Azure Active Directory
* Understand Always Encrypted