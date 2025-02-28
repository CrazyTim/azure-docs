---
title: "include file"
description: "include file"
services: storage
author: alexwolfmsft
ms.service: storage
ms.topic: include
ms.date: 09/09/2022
ms.author: alexwolf
ms.custom: include file
---

When developing locally with Passwordless authentication, make sure the user account that connects to Cosmos DB is assigned a role with the correct permissions to perform data operations. Currently, Azure Cosmos DB for NoSQL does not include built-in roles for data operations, but you can create your own using the Azure CLI or PowerShell.

Roles consist of a collection of permissions or actions that a user is allowed to perform, such as read, write, and delete. You can read more about [configuring role based access control (RBAC)](../../../articles/cosmos-db/how-to-setup-rbac.md) in the cosmos security configuration documentation. 

## Create the custom role

Create roles using the `az role definition create` command. Pass in the Cosmos DB account name and resource group, followed by a body of JSON that defines the custom role. The following example creates a role named `PasswordlessReadWrite` with permissions to read and write items in Cosmos DB containers. The role is also scoped to the account level using `/`.

```azurecli
az cosmosdb sql role definition create 
    --account-name passwordlessnosql
    --resource-group  passwordlesstesting 
    --body '{
    "RoleName": "PasswordlessReadWrite",
    "Type": "CustomRole",
    "AssignableScopes": ["/"],
    "Permissions": [{
        "DataActions": [
            "Microsoft.DocumentDB/databaseAccounts/readMetadata",
            "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*",
            "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/*"
        ]
    }]
}'
```

When the command completes, copy the ID value from the `name` field and paste it somewhere for later use.

Next, assign the role you created to the user account or service principal that will connect to Cosmos DB. During local development, this will generally be your own account that is logged into Visual Studio or the Azure CLI.

Retrieve the details of your account using the `az ad user` command.

```azurecli
az ad user show --id "<your-email-address>"
```

Copy the value of the `id` property out of the results and paste it somewhere for later use.

Finally, assign the custom role you created to your user account using the `az cosmosdb sql role assignment create` command and the IDs you copied previously.

```azurecli
az cosmosdb sql role assignment create 
    --account-name passwordlessnosql
    --resource-group passwordlesstesting
    --scope "/" 
    --principal-id <your-user-id>
    --role-definition-id <your-custom-role-id> 
```