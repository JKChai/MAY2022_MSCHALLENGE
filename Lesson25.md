# Implement backup and restore for Azure Cosmos DB SQL API

We will learn to use the two backup models Azure Cosmos DB provides.

**Learning objectives**

* Understand the different backup and restore options Azure Cosmos DB provides
* Evaluate periodic backups
* Configure continuous backups
* Do point in time recovery

**Prerequisites**

Before starting this module, you should have experience of building cloud applications with Microsoft C# or a similar programming language.

---

# Introduction

Azure Cosmos DB automatically backups yours data. These backups are taken at regular intervals. The backup process happens behind the scenes and doesn't affect either the performance or availability of your database operations. The backups are encrypted with Microsoft managed keys and stored in a separate storage service. The backups are globally replicated for resiliency against regional disasters. The backups are encrypted while in transfer since they're transferred over a secure non-public network. Restores of backups are done on the same region they were done in. These backups are done in one of two modes, either a periodic or a continuous backup mode.

After completing this module, you'll be able to:

* Understand the different backup and restore options Azure Cosmos DB provides
* Evaluate periodic backups
* Configure continuous backups
* Do point in time recovery

---

# Evaluate periodic backup

Azure Cosmos DB takes automatic backups of your data at regular periodic intervals. Azure Cosmos DB does these backups in the following way:

* A full backup is taken every 4 hours. Only the last two backups are stored by default. Both the backup interval and the retention period can be configured in the Azure portal. This configuration can be set during or after the Azure Cosmos DB account has been created.
* If Azure Cosmos DB's containers or database are deleted, the existing container and database snapshots will be retained for 30 days.
* Azure Cosmos DB backups are stored in Azure Blob storage.
* Backups are stored in the current write region or if using multi-region writes to one of the write regions to guarantee low latency.
* Snapshots of the backup are replicated to another region through geo-redundant storage (GRS). This replication, provides resiliency against regional disasters.
* Backups can't be accessed directly. To restore the backup, a support ticket needs to be opened with the Azure Cosmos DB team.
* Backups won't affect performance or availability. Furthermore, no RUs are consumed during the backup process.

## Backup Storage Redundancy

Azure Cosmos DB backups use by default geo-redundant blob storage that is replicated to a paired region. This backup storage redundancy can be modified either during or after the creation of the account. The redundancy options available to periodic backup mode are:

* Geo-redundant backup storage: The default value. Copies the backup asynchronously across the paired region.
* Zone-redundant backup storage: Copies the backup synchronously across three Azure availability zones in the primary region.
* Locally redundant backup storage: Copies the backup synchronously three times within a single physical location in the primary region.

## Change the default backup interval and retention period

By default, Azure Cosmos DB backups every 4 hours and keeps the last two backups at no extra cost. These settings are set at the Azure Cosmos DB account level and can be configured during or after the account creation. Currently these settings can only be changed from the Azure portal.

### Change backup options for a new account

1. When creating a new Azure Cosmos DB account, select Periodic under the Backup policy tab.
2. Change the backup options as needed.
    * Backup Interval - This setting defines how often is the backup going to be done. Can be changed in minutes or hours. The interval period can be between 1 and 24 hours. The default is 240 minutes.
    * Backup Retention - This setting defines how long should the backups be kept. Can be changed in hours or days. The retention period will be at least two times the backup interval and 720 hours (or 30 days) at the most. The default is 8 Hours.
    * 3 Backup storage redundancy - One of the three redundancy options discussed in the previous section. The default is Geo-redundant backup storage.

### Change backup options for an existing account

1. On the Azure portal, navigate to the Azure Cosmos DB account.
2. Under the *Settings* section, select Backup & Restore.
3. Change the backup options as needed.
    * Backup Interval - This setting defines how often is the backup going to be done. Can be changed in minutes or hours. The interval period can be between 1 and 24 hours. The default is 240 minutes.
    * Backup Retention - This setting defines how long should the backups be kept. Can be changed in hours or days. The retention period will be at least two times the backup interval and 720 hours (or 30 days) at the most. The default is 8 Hours.
    * Backup storage redundancy - One of the three redundancy options discussed in the previous section. The default is Geo-redundant backup storage.

> :grey_exclamation: Note
>
> You must have Azure Cosmos DB owner, contributor, or have the Cosmos DB Operator (CosmosdbBackupOperator) role assigned to configure the backup retention period or backup storage redundancy settings.

## Request to restore a backup

If you need to restore a database or container, you'll need to open a request ticket or call the Azure support team. Azure support is available for all plans except for the Basic plan. You must have Azure Cosmos DB owner, contributor, or have the Cosmos DB Operator (`CosmosdbBackupOperator`) role assigned to request a restore from the portal.

## Considerations for restoring a backup

Scenarios to consider:

* Delete the entire Azure Cosmos DB account.
* Delete one or more Azure Cosmos DB databases.
* Delete one or more Azure Cosmos DB containers.
* Delete or modify the Azure Cosmos DB items within a container. This specific case is typically referred to as data corruption.

A new Azure Cosmos DB account is created to hold the data every time you restore a backup. Data can't be restored to an existing Azure DB account. The account created will have the `<Azure_Cosmos_account_original_name>-restored#` name format, where # is an incremental digit when you're doing multiple restores against the same account.

If you delete the Azure Cosmos DB account, and wish to recover it with the same name, don't recreate it. Just ask the Azure support team to recreate it with the original name. The restored account will have the same provisioned throughput, indexing policies and it is in same region as the original account.

If you delete an Azure Cosmos DB database, you can ask to restore the whole database, or just those containers inside the databases you wish to restore.

If you delete or modify items in a container, and wish to restore that data before the delete or the data modification, you should specify the time to restore to. If your retention period is set to anything less that seven days, it's a good practice to temporarily increase your retention period to at least seven days to make sure your backups aren't overwritten while you open the support ticket.

Settings that are not restored to the new account:

* VNET access control lists
* Stored procedures, triggers, and user-defined functions
* Multi-region settings

## Costs of Extra backups

Azure Cosmos DB includes the option for the retention of two backups for free. Extra backups will be charged on a region-based backup-storing pricing. So for example, if you have a backup interval of 3 hours and you chose a retention period of two days (48 hours), and since two backups are free, you'll pay for (48/3-2) region-based backup-storing price backup size.

## Manage your own backups

While you can call Azure support to restore your backups, there are a couple of other approaches you can do to manage your own backups. Those options are:

* Use Azure Data Factory to periodically copy your data to a storage location.
* Use the Azure Cosmos DB change feed to get the incremental changes and copy those changes to a storage location.

## I restored a copy of my data, now what?

The first step will be to make sure the data wanted was restored from the backup. Once you confirm you have the data, migrate that data back to the original account.

You could migrate the data to the original account the following ways:

* Use Azure Data Factory.
* Use the Azure Cosmos DB data migration tool.
* Use the Azure Cosmos DB change feed.
* Write your own data migration app in your preferred programming language.

> :grey_exclamation: Note
>
> Don't forget to delete the Azure Cosmos DB account your data was restored to, once you migrate the data back to your production account. If you don't delete the account, you'll incur extra costs.

---

# Configure continuous backup and recovery

When using the *continuous backups* mode, backups are continuously taken in every region where the Azure Cosmos DB account exists.

The retention period for the backups is either 30 day or if the Azure Cosmos DB account was created before 30 days, up to the resource creation time. The restore can be done for any point in time timestamp within the retention period. If the Azure Cosmos DB Account is using the **strong consistency level**, the backups taken in the *write region* could be more up to date then the backups taken in the read regions.

## Backup storage redundancy

By default, locally redundant storage accounts are used store the backups in each region. When Availability zones are enabled for a region, the backups are then stored in a Zone-Redundant storage account. This storage redundancy cannot be updated when using the continuous backup mode.

### Change backup options for a new account

* When creating a new Azure Cosmos DB account, select Continuous under the Backup policy tab.

### Change backup options for an existing account

1. In the Azure Cosmos DB account page, under the *Settings* section, choose **Features**.
2. Select `Continuous Backup`, and select `Enable`.

## What is and is not restored?

You can choose to restore any combination of Azure Cosmos DB containers, databases, or an entire account. This action will restore all selected data and its indexes to a new Azure Cosmos DB account. The data containers, databases, or account restored is guaranteed to be consistent up to the restore time specified.

Settings that are not restored to the new account:

* Firewall, VNET, private endpoint settings.
* Consistency settings. By default, the account is restored with session consistency.
* Regions.
* Stored procedures, triggers, UDFs.

## Permissions

The owner of an Azure Cosmos DB account can either start the restore, or grant restore rights to roles or principals.

## Continuous backup mode charges

Backup storage space and restore cost will be added for Azure Cosmos DB accounts that have continuous backup enabled. A separate charge will be added every time a restore is started.

## Limitations when using the continuous backup mode

* Azure Cosmos DB accounts using customer-managed keys are not supported.
* Multi-region write accounts not supported.
* You can't restore an account into a region where the source account did not exist.
* The retention period is 30 days and can't be changed.
* Can't modify or delete IAM policies when restore is in progress.
* Accounts that create unique indexes after the container is created are not supported.
* Point in time restore always restores to a new Azure Cosmos DB account.
* Collection's consistent indexes may still be rebuilding after completing the restore.
* Since TTL container properties are restored with the restore process, restores must be for timestamps before TTL properties were added to a container. This timestamp will prevent data from being deleted right after the restore.
* Azure Synapse Link and continuous backup mode can't coexist in the same database account.

---

# Perform a point-in-time recovery

Azure Cosmos DB account with a continuous backup mode, gives you the ability to do a point in time recovery. This type of recovery will allow you to choose any timestamp within the up to 30-days backup retention period and restore a combination of Azure DB containers, databases, or the accounts.

## Restore Scenarios

Let's review at several point-in-times restores scenarios:

* Restore deleted account - Deleted accounts that can be restored are visible in the Restore pane under the Azure Cosmos DB account list page. The information needed for the restore is the timestamp right before the delete, the account name of the deleted account, and the target name to be restored as. Restores can be performed from the Azure portal, PowerShell, or CLI.
* Restore data of an account in a particular region - If you need a copy in a region of an Azure Cosmos DB Account you can do a point in time restore of the account. The information needed for the restore is the wished timestamp, and the target name to be restored as. Restores can be performed from the Azure portal, PowerShell, or CLI.
* Recover from an accidental write or delete operation within a container with a known restore timestamp - If you know the timestamp when the accidental operation was done, you can do a point in time restore from Azure portal, PowerShell, or CLI into another account at the wished timestamp to recover to.
* Restore an account to a previous point in time before the accidental delete of the database - Under the point in time page, use the event feed pane to determine when the database was deleted, and find the restore time. You can do a point in time restore from Azure portal, PowerShell, or CLI into another account at the wished timestamp to recover to.
* Restore an account to a previous point in time before the accidental delete or modification of the container properties - Under the point in time page, use the event feed pane to determine when the container was created, modified, or deleted and find the restore time. You can do a point in time restore from Azure portal, PowerShell, or CLI into another account at the wished timestamp to recover to.

---

# Exercise: Recover a database or container from a recovery point

> This unit includes a lab to complete.
>
> Use the free resources provided in the lab to complete the exercises in this unit. You will not be charged.
> 
> Microsoft provides this lab experience and related content for educational purposes. All presented information is owned by Microsoft and intended solely for learning about the covered products and services in this Microsoft Learn module.

[Check this out to launch the lab](https://docs.microsoft.com/en-us/learn/modules/implement-backup-restore-for-azure-cosmos-db-sql-api/5-exercise-recover-database-container-from-recovery-point)

> :grey_exclamation: Note
>
> A virtual machine (VM) containing the client tools you need is provided, along with the exercise instructions. Use the button above to open the VM. A limited number of concurrent sessions are available - if the hosted environment is unavailable, try again later.

> :bulb: Tip
>
> Alternatively, if you would like to use a development environment on your own computer, you can use this [setup](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/00-setup-environment.md) guide and follow these [exercise](https://github.com/microsoftlearning/dp-420-cosmos-db-dev/blob/main/instructions/01-create-account.md) instructions. The setup guide is designed for multiple development exercises, and may include software that is not required for this specific exercise. Additionally, due to the range of possible operating systems and setup configurations, we can't provide support if you choose to complete the exercise on your own computer.

When you finish the exercise, end the lab to close the VM. Don't forget to come back and complete the knowledge check to earn points for completing this module!

---

# Knowledge check

1. To restore a periodic backup you must?

- [ ] In the Azure portal Azure Cosmos DB page, select Restore
- [x] Open a ticket with the Azure support team to restore the backup for you
> That's correct. You can't restore an Azure Cosmos DB account or any of its data, only the Azure support team can do that, so a support ticket must be opened to restore the data.
- [ ] Under Azure Cosmos DB account Data Explorer, run the SQL Query restore from "AZUREBLOBSTORAGEPATH", replace AZUREBLOBSTORAGEPATH for the URL of your backup file.

2. Which of the following options must you configure for continuous backup mode to work properly?

- [ ] Backup storage redundancy.
- [ ] Retention period.
- [x] Continuous backup mode doesn't require any setting to be configured.
> That's correct. Unlike periodic backups, there is no need to configure intervals, retention, or storage redundancy.

3. What is a limitation of using continuous backup mode?

- [ ] You can only restore over the existing Azure Cosmos DB account at the container level.
- [ ] Multi-region write accounts not supported.
> That's correct. Currently continuous backups are only supported for an Azure Cosmos DB account configured for single-write region.
- [ ] Firewall and VNET settings are only recovered for the restored Azure Cosmos DB account region.

---

# Summary

In this module, you learned the backup and recovery options for Azure Cosmos DB.

Now that you have completed this module, you can:

* Understand the different backup and restore options Azure Cosmos DB provides
* Evaluate periodic backups
* Configure continuous backups
* Do point in time recovery