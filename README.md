# Power Pages SPA with Azure Cosmos DB

Query Azure Cosmos DB with MSAL token from browser.

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

Az PowerShell commands for Azure Cosmos DB [learn.microsoft.com/en-us/powershell/module/az.cosmosdb](https://learn.microsoft.com/en-us/powershell/module/az.cosmosdb/?view=azps-14.5.0#cosmos-db)

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

#### Other relevant commands

##### List all active role assignments on your Cosmos DB account
Run:
```powershell
Get-AzCosmosDBSqlRoleAssignment `
  -ResourceGroupName rg-dev-001 `
  -AccountName test-cosmos-db
```
Response: 
```powershell
Id               : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleAssignmen 
                   ts/ea55e97d-808f-414d-a5dc-41fcf9fc08f0
Scope            : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/dbs/TestDB/coll 
                   s/TestContainer
RoleDefinitionId : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleDefinitio
                   ns/667aef15-cfd8-4c28-ab31-e9bf39cee554
PrincipalId      : 0a726c36-7f85-4281-9b46-279b1e9eb331

Id               : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleAssignmen
                   ts/ba8ea356-3a19-4ad8-b45f-3540f0d501ee
Scope            : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/dbs/TestDB/coll 
                   s/TestContainer
RoleDefinitionId : /subscriptions/c15ff08c-5669-485a-ae22-300c2f5920ec/resourceGroups/rg-dev-001/providers/Microsoft.DocumentDB/databaseAccounts/test-cosmos-db/sqlRoleDefinitio 
                   ns/6aa1c516-89e6-42db-a92a-9efbd692fbfc
PrincipalId      : 0a726c36-7f85-4281-9b46-279b1e9eb331
```

##### Remove Role Assignment
Run:
```powershell
Remove-AzCosmosDBSqlRoleAssignment `
  -AccountName test-cosmos-db `
  -ResourceGroupName rg-dev-001 `
  -Id ea55e97d-808f-414d-a5dc-41fcf9fc08f0 <# id of the role assignment. This id will remove role assignment "Write TestDb/TestContainer" from your Cosmos DB account #> `
  -PassThru # to see result of the operation
```
Response: 
```powershell
true # if -PassThru is set or nothing
```

##### Remove Role Definition
Run:
```powershell
Remove-AzCosmosDBSqlRoleDefinition `
  -ResourceGroupName rg-dev-001 `
  -AccountName test-cosmos-db `
  -Id 667aef15-cfd8-4c28-ab31-e9bf39cee554 <# id of the role definition. This id will remove both assignment and role "Write TestDb/TestContainer" from your Cosmos DB #> `
  -PassThru # to see result of the operation
```
Response: 
```powershell
true # if -PassThru is set or nothing
```

### Query Cosmos DB with `fetch` from your portal

[Microsoft Authentication Library for JavaScript (MSAL.js) for Browser-Based Single-Page Applications](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-browser) is used for authenticating with the Azure tenant and getting access tokens.

A chain of functions (piping) is used to get a token for a resource. 

Source code will be provided later.

#### Get token

##### Get URI for your Cosmos DB 
Go to portal.azure.com -> test-cosmos-db resource -> Overview -> URI -> <https>://test-cosmos-db.documents.azure.com:443/
##### MSAL token scope for your Cosmos DB
Remove port from the URI and add user_impersonation => `https://test-cosmos-db.documents.azure.com/user_impersonation`

Requesting token with MSAL methods like `acquireTokenSilent`, `acquireTokenPopup` or `acquireTokenRedirect` will give the following response: 
```json
{
    "token_type": "Bearer",
    "scope": "<https>://test-cosmos-db.documents.azure.com/user_impersonation",
    "expires_in": 4684,
    "ext_expires_in": 4684,
    "access_token": "eyJ0eXAiOiJKV1...DYL4VxOhiTQ",
    "refresh_token": "1.AUEBa_qS...rkh56Ln5FC2C9I",
    "refresh_token_expires_in": 79933,
    "id_token": "eyJ0eXAiOiJKV1Qi...oKE_UPehkAkuw",
    "client_info": "eyJ1aWQiOiI...3RkYnIiOiJFVSJ9"
}
```

#### Query Cosmos DB API

Azure Cosmos DB REST API Reference [learn.microsoft.com/en-us/rest/api/cosmos-db](https://learn.microsoft.com/en-us/rest/api/cosmos-db/)
Request Headers [learn.microsoft.com/en-us/rest/api/cosmos-db/common-cosmosdb-rest-request-headers](https://learn.microsoft.com/en-us/rest/api/cosmos-db/common-cosmosdb-rest-request-headers)

##### Get a Collection

Reference [learn.microsoft.com/en-us/rest/api/cosmos-db/get-a-collection](https://learn.microsoft.com/en-us/rest/api/cosmos-db/get-a-collection)

Request:
```javascript
fetch(
    'https://test-cosmos-db.documents.azure.com/dbs/TestDB/colls/TestContainer/docs',
    {
        headers: new Headers({
            Authorization: encodeURIComponent(
                `type=aad&ver=1.0&sig=${access_token}`
            ),
            'x-ms-date': new Date().toUTCString(),
            'x-ms-version': '2018-12-31',
        }),
    }
)
```

Response: 
```json
{
    "_rid": "MutdAITczYs=",
    "Documents": [
        {
            "id": "02",
            "partitionKey": "key02",
            "_rid": "MutdAITczYsCAAAAAAAAAA==",
            "_self": "dbs/MutdAA==/colls/MutdAITczYs=/docs/MutdAITczYsCAAAAAAAAAA==/",
            "_etag": "\"b500d7d3-0000-0d00-0000-690205270000\"",
            "_attachments": "attachments/",
            "_ts": 1761740071
        }
    ],
    "_count": 1
}
```

##### Send SQL Query

Reference [learn.microsoft.com/en-us/rest/api/cosmos-db/querying-cosmosdb-resources-using-the-rest-api](https://learn.microsoft.com/en-us/rest/api/cosmos-db/querying-cosmosdb-resources-using-the-rest-api)

Request:
```javascript
fetch(
    'https://test-cosmos-db.documents.azure.com/dbs/TestDB/colls/TestContainer/docs',
    {
        method: 'POST',
        headers: new Headers({
          Authorization: encodeURIComponent(
            `type=aad&ver=1.0&sig=${access_token}`
          ),
          'x-ms-date': new Date().toUTCString(),
          'x-ms-version': '2018-12-31',
          'Content-Type': 'application/query+json',
          'x-ms-documentdb-isquery': true,
          'x-ms-documentdb-query-enablecrosspartition': true
        }),
        body: JSON.stringify({query: 'SELECT * FROM TestContainer'})
    }
)
```

Response: 
```json
{
    {
        "_rid": "MutdAITczYs=",
        "Documents": [
            {
                "id": "02",
                "partitionKey": "key02",
                "_rid": "MutdAITczYsCAAAAAAAAAA==",
                "_self": "dbs/MutdAA==/colls/MutdAITczYs=/docs/MutdAITczYsCAAAAAAAAAA==/",
                "_etag": "\"b500d7d3-0000-0d00-0000-690205270000\"",
                "_attachments": "attachments/",
                "_ts": 1761740071
            }
        ],
        "_count": 1
    }
}
```

##### Create a Document

Reference [learn.microsoft.com/en-us/rest/api/cosmos-db/create-a-document](https://learn.microsoft.com/en-us/rest/api/cosmos-db/create-a-document)

Request:
```javascript
fetch(
    'https://test-cosmos-db.documents.azure.com/dbs/TestDB/colls/TestContainer/docs',
    {
        method: 'POST',
        headers: new Headers({
          Authorization: encodeURIComponent(
            `type=aad&ver=1.0&sig=${access_token}`
          ),
          'x-ms-date': new Date().toUTCString(),
          'x-ms-version': '2018-12-31',
          'x-ms-documentdb-partitionkey': '["keyFromPortals"]'
        }),
        body: JSON.stringify({
            id: `${Date.now()}`,
            partitionKey: 'keyFromPortals',
            someOtherData: 'here'
        })
    }
)
```

Response: 
```json
{
    "id": "1761841215641",
    "partitionKey": "keyFromPortals",
    "someOtherData": "here",
    "_rid": "MutdAITczYsIAAAAAAAAAA==",
    "_self": "dbs/MutdAA==/colls/MutdAITczYs=/docs/MutdAITczYsIAAAAAAAAAA==/",
    "_etag": "\"91015851-0000-0d00-0000-690390400000\"",
    "_attachments": "attachments/",
    "_ts": 1761841216
}
```











