# PowerPagesSPA-CosmosDB

## In portal.azure.com
### Provision Azure Cosmos DB
1) Find Azure Cosmos DB -> create new Azure Cosmos DB for NoSQL -> Account name `test-cosmos-db` -> follow other steps -> in Security disable Key-based Authentication -> create
2) Go to the resource -> find CORS -> add in Allowed Origins `https://your-portal.powerappsportals.com, https://127.0.0.1:5501`
3) Go to Data Explorer -> make new Database with id `TestDB` -> in this db make a new Container with id `TestContainer` and Partition key `key01`

Note: when Azure Cosmos DB is provisioned, you have the role `Owner`. With this role, you can only CRUD Databases/Containers, but not Items. Trying to add item you get error:

```
Request is blocked because principal [7a723eae-c4d2-48cd-92e5-5545d18bbc61] does not have required RBAC permissions to perform action [Microsoft.DocumentDB/databaseAccounts/readMetadata] on resource [/]
```
2 access categories exist with Cosmos DB RBAC:

1) **Control plane role-based access** allowing to CRUD of databases and containers. Roles exists in portal under the name 'Cosmos DB Operator'. This role lets you manage Azure Cosmos DB accounts, but not access data in them, prevents access to account keys and connection strings. Owner of the Cosmos DB has such permissions by default.
2) **Data plane role-based access** allowing to CRUD items within a container. Not provided to the owner, does not exist in the portal. CLI must be used.
  
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

### Grant data plane role-based access for your Cosmos DB account to user or group

I follow this guide [learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-connect-role-based-access-control](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-connect-role-based-access-control?pivots=azure-powershell#grant-data-plane-role-based-access).

#### 1. Get list of all role definitions associated with your Azure Cosmos DB account

Run:
```powershell
Get-AzCosmosDBSqlRoleDefinition `
  -ResourceGroupName rg-dev-001 `
  -AccountName test-cosmos-db
```
Response: 
```powershell
Id                         : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleDefinitions/00000000-0000-0000-0000-000000000001
RoleName                   : Cosmos DB Built-in Data Reader
Type                       : BuiltInRole
AssignableScopes           : {/subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db}
Permissions.DataActions    : {Microsoft.DocumentDB/databaseAccounts/readMetadata, Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/executeQuery, Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/readChangeFeed,      
                            Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read}
Permissions.NotDataActions :

Id                         : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleDefinitions/00000000-0000-0000-0000-000000000002
RoleName                   : Cosmos DB Built-in Data Contributor
Type                       : BuiltInRole
AssignableScopes           : {/subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db}
Permissions.DataActions    : {Microsoft.DocumentDB/databaseAccounts/readMetadata, Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/*, Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*}
Permissions.NotDataActions :
```

We can assign these 2 BuildInRoles, but with custom roles we can limit the scope to specific databases/containers. Let's do this.

#### 2. Create a new custom role definition for Reader

Run:
```powershell
New-AzCosmosDBSqlRoleDefinition `
    -AccountName test-cosmos-db `
    -ResourceGroupName rg-dev-001 `
    -Type CustomRole `
    -RoleName "Read TestDB/TestContainer" `
    -DataAction (
        "Microsoft.DocumentDB/databaseAccounts/readMetadata", 
        "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/executeQuery", 
        "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/readChangeFeed", 
        "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read"
    ) `
    -AssignableScope "/dbs/TestDB/colls/TestContainer" # we limit the scope to the created database/container
```
Response: 
```powershell
Id                         : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleDefinitions/6aa1c516-89e6-42db-a92a-9efbd692fbfc
RoleName                   : Read TestDB/TestContainer
Type                       : CustomRole
AssignableScopes           : {/subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/dbs/TestDB/colls/TestContainer}
Permissions.DataActions    : {Microsoft.DocumentDB/databaseAccounts/readMetadata, Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/executeQuery, Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/readChangeFeed, Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read}
Permissions.NotDataActions :
```

#### 3. Create a new custom role definition for Writer

Run:
```powershell
New-AzCosmosDBSqlRoleDefinition `
    -AccountName test-cosmos-db `
    -ResourceGroupName rg-dev-001 `
    -Type CustomRole `
    -RoleName "Write TestDB/TestContainer" `
    -DataAction (
        "Microsoft.DocumentDB/databaseAccounts/readMetadata", 
        "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/*", 
        "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*"
    ) `
    -AssignableScope "/dbs/TestDB/colls/TestContainer"
```
Response: 
```powershell
Id                         : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleDefinitions/667aef15-cfd8-4c28-ab31-e9bf39cee554
RoleName                   : Write TestDb/TestContainer
Type                       : CustomRole
AssignableScopes           : {/subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/dbs/TestDB/colls/TestContainer}
Permissions.DataActions    : {Microsoft.DocumentDB/databaseAccounts/readMetadata, Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/*,Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*}
Permissions.NotDataActions :
```

#### 4. Get ID of your current Cosmos DB account

Two options here:

##### Option 1
Run:
```powershell
Get-AzCosmosDBAccount `
  -ResourceGroupName rg-dev-001 `
  -Name test-cosmos-db `
  | Select -Property Id # this line is optional and limits output only to id 
```
Response: 
```powershell
/subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db
```
##### Option 2

The same can be taken from the url bar when you are in the portal.azure.com -> your Azure Cosmos DB resource.

#### 5. Assign roles to a user or group

I will assign a role to Azure Security Group of my team. This group has id 0a726c36-7f85-4281-9b46-279b1e9eb331

Run:
```powershell
New-AzCosmosDBSqlRoleAssignment `
  -AccountName test-cosmos-db `
  -ResourceGroupName rg-dev-001 `
  -RoleDefinitionName "Read TestDB/TestContainer" <# or instead use -RoleDefinitionId "6aa1c516-89e6-42db-a92a-9efbd692fbfc" #> `
  -Scope "/dbs/TestDB/colls/TestContainer" <# scope shall be the same as on the role, otherwise error is out #> `
  -PrincipalId 0a726c36-7f85-4281-9b46-279b1e9eb331 # my security group
```
Response: 
```powershell
Id               : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleAssignments/ba8ea356-3a19-4ad8-b45f-3540f0d501ee
Scope            : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/dbs/TestDB/colls/TestContainer
RoleDefinitionId : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleDefinitions/6aa1c516-89e6-42db-a92a-9efbd692fbfc
PrincipalId      : 0a726c36-7f85-4281-9b46-279b1e9eb331
```

Do the same step for role `Write TestDb/TestContainer` with id `667aef15-cfd8-4c28-ab31-e9bf39cee554`.

Now, after making sure that you are a member of the security group 0a726c36-7f85-4281-9b46-279b1e9eb331, in portal.azure.com try to create items in TestDB/TestContainer and you shall succeed.


