# PowerPagesSPA-CosmosDB

## In portal.azure.com
### Provision Azure Cosmos DB
1) Find Azure Cosmos DB -> create new Azure Cosmos DB for NoSQL -> Account name `test-cosmos-db` -> follow other steps -> in Security disable Key-based Authentication -> create
2) Go to the resource -> find CORS -> add in Allowed Origins `https://your-portal.powerappsportals.com, https://127.0.0.1:5501`
3) Go to Data Explorer -> make new Database with id `TestDB` -> in this db make a new Container with id `TestContainer` and Partition key `key01`

Note: when Azure Cosmos DB is provisioned, you have the role `Owner`. With this role, you can only CRUD Databases/Containers, but not Items. Trying to add item you get error:
`Request is blocked because principal [7a723eae-c4d2-48cd-92e5-5545d18bbc61] does not have required RBAC permissions to perform action [Microsoft.DocumentDB/databaseAccounts/readMetadata] on resource [/]`
  
### Add API permissions
Go to App registrations -> find your Power Pages app and open -> API Permissions -> Add a permission -> select Azure Cosmos DB -> select user_impersonation -> add and grant.

### Install Azure PowerShell on your machine
Follow steps from [learn.microsoft.com/en-us/powershell/azure/install-azure-powershell](https://learn.microsoft.com/en-us/powershell/azure/install-azure-powershell?view=azps-14.5.0).

#### Windows

Note: If you have Azure extension with Azure PowerShell in VS Code remove AzureRm as they come in conflict. 

Open PowerShell as Administrator and run:
```powershell
Uninstall-AzureRm
```
Install Azure PowerShell:
```powershell
Install-Module -Name Az -Repository PSGallery -Force -Scope CurrentUser # -AllowClobber
```
Note: add param `-AllowClobber` if error like this is shown:
```powershell
<#
The following commands are already available on this system:
'Login-AzAccount,Logout-AzAccount,Send-Feedback'. This module
'Az.Accounts' may override the existing commands. If you still
want to install this module 'Az.Accounts', use -AllowClobber
parameter.
#>
```
Then run:
```powershell
Run Connect-AzAccount # select your azure account
```
#### MacOS
TBA

### Add roles for your Cosmos DB account



