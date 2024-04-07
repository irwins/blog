+++
title = 'Getting started with shared queries in Azure Resource Graph (ARG)'
date = 2024-04-07T08:17:21+02:00
draft = false
+++

I'm a fan of Azure Resource Graph! When I started with arg in the portal I noticed I coud save my query as private or shared. At that time I didn't think much of until I wanted a better way to manage my queries. That was my _aha_ moment :grin:. Shared queries makes for a better management experience of your queries.

## What are Azure Resource Graph shared queries?

Microsoft's [documentation](https://learn.microsoft.com/en-us/azure/governance/resource-graph/tutorials/create-share-query) explains it best:
>A Shared query is an Azure Resource Manager resource that can be managed with Azure role-based access control (Azure RBAC) and protected with resource locks. When you share queries, you help your team realize goals of consistency and efficiency through repetition.

I quickly realised sharing kql code manually would be a chore. Hardcoding kql in script is a recipe for disaster. Shared queries makes for a better and efficient querying experience. Definitely a step in the right direction. But working in the portal isn't really my thing, so lets automate creating shared queries with a little bicep & Powershell shall we?

## Managing shared query code

I definitely wanted to have the code in json format as this is easy to import in bicep. After a few attempts, I came up with the following:

- Use a psd1 file to contain the necessary data.
- Import the pds1 file.
- Convert To Json for later use.

Why go through all that hassle? I wanted to seperate maintenance & execution. Create a psd1 per shared query makes for easier overview and maintenance by the team. Also, for some reason my code wasn't formatting the way I expected, this works.

Here's what a psd1 shared query looks like for bastion

_**alert-bastion.psd1**_

```powershell
@{
    name        = 'alert-bastion'
    description = 'ManagedServices shared query alert bastion'
    code        = @"
resources
| join kind=leftouter(
    resourcecontainers
    | where type=='microsoft.resources/subscriptions'
    | project subscriptionName=name, subscriptionId
) on subscriptionId
| where type == "microsoft.network/bastionhosts"
| extend details = pack_all()
| project subscriptionName, subscriptionId, location,resourceGroup,name,type,id,details
| order by subscriptionName asc
"@
}
```

In order to process all the psd1 files we'll use the following PowerShell script:

_**Convert-QueriesToJson.ps1**_

```powershell
[CmdletBinding()]
param(
    [string]$QueriesPath = "queries/src"
)

$sharedQueries = Get-ChildItem -Path $QueriesPath -Recurse -Include *.psd1

$jsonQueries = $sharedQueries |
ForEach-Object {
    Import-LocalizedData `
        -BaseDirectory $_.DirectoryName `
        -FileName $_.Name
}

$jsonQueries | ConvertTo-Json | Out-File "queries/src/sharedQueries.json"
```

This imports the psd1 files and converts to a Json array:

_**sharedQueries.json**_

```json
[
    {
        "description": "ManagedServices shared query alert application gateway",
        "name": "alert-application-gateway",
        "code": "resources\n| join kind=leftouter(\n    resourcecontainers\n    | where type=='microsoft.resources/subscriptions'\n    | project subscriptionName=name, subscriptionId\n) on subscriptionId\n| where type =~ 'Microsoft.Network/applicationGateways'\n| extend skuName = properties.sku.name\n| extend details = pack_all()\n| where skuName != 'WAF_Medium'\n| project subscriptionName, subscriptionId, location,resourceGroup,name,skuName,type,id,details\n| order by subscriptionName asc"
    },
    {
        "description": "ManagedServices shared query alert bastion",
        "name": "alert-bastion",
        "code": "resources\n| join kind=leftouter(\n    resourcecontainers\n    | where type=='microsoft.resources/subscriptions'\n    | project subscriptionName=name, subscriptionId\n) on subscriptionId\n| where type == \"microsoft.network/bastionhosts\"\n| extend details = pack_all()\n| project subscriptionName, subscriptionId, location,resourceGroup,name,type,id,details\n| order by subscriptionName asc"
    }
    //...
]
```

Doing it this way, I made sure my kql _code_ was formatted correctly.

## Deploy shared queries (bicep)

Deploying the queries using bicep is pretty straightforward:

- Load sharedQueries.json usingbicep's json function
- Iterate over the array

A little preparation goes a long way...:wink:

_**main.bicep**_

```bicep
param tags object

var jsonQueries = json(loadTextContent('../src/sharedQueries.json'))

resource sharedQueries 'Microsoft.ResourceGraph/queries@2018-09-01-preview' = [for query in jsonQueries: {
  name: query.name
  location: 'global'
  tags: tags
  properties: {
    query: query.code
    description: query.description
  }
}]

output deployedSharedQueries array = [for (query, i) in jsonQueries: {
  name: sharedQueries[i].name
  id: sharedQueries[i].id
}]
```

Here's the bicepparam file:

_**main.dev.bicepparam**_

```bicep
using 'main.bicep'

param tags = {
  environment: 'dev'
  createdBy: '<ContactInfo>'
}
```

Shared queries supports RBAC, so you could get creative on who has access... Just puttin it out there... :wink:

I enjoy using az cli when it comes to prototyping...

_**deploy.azcli**__

```bash
### LOGIN TENANT
TENANT_ID='your_tenant_id'
SUBSCRIPTION_ID='<your_subscription>'
LOCATION=westeurope

az login --tenant $TENANT_ID
az account set --subscription $SUBSCRIPTION_ID

### CREATE RESOURCE GROUP
RESOURCE_GROUP_NAME=rg-az-monitor
az group create -n $RESOURCE_GROUP_NAME -l $LOCATION

### Deploy main bicep template using bicep parameters file

### Lint bicep file
az bicep lint --file queries/infra/main.bicep

### Validate bicep file
az deployment group validate \
    --resource-group $RESOURCE_GROUP_NAME \
    --parameters queries/infra/main.dev.bicepparam

### Preview bicep file
az deployment group what-if \
    --resource-group $RESOURCE_GROUP_NAME \
    --parameters queries/infra/main.dev.bicepparam \
    --template-file queries/infra/main.bicep

utcNow=$(date -u +"%Y%m%d%H%M%S")

az deployment group create \
    --name shared-queries-$utcNow \
    --resource-group $RESOURCE_GROUP_NAME \
    --parameters queries/infra/main.dev.bicepparam
```

## Running a shared query

Running a [shared query](https://learn.microsoft.com/en-us/azure/governance/resource-graph/concepts/query-language#shared-query-syntax-preview) using the shared query URI makes it all worth it. The documentation does state that it's in preview, so fingers crossed.

>To call a shared query inside a Resource Graph query, use the {{shared-query-uri}}

This is how to do this using PowerShell

``` powershell
$sharedQueryResults = Search-AzGraph -Query "{{$SharedResourceIdQuery}}" -UseTenantScope -First 1000
```

The URI takes you directly to the query, no need for context switching! Queries are limited to 1000 result. You can [paginate](https://learn.microsoft.com/en-us/azure/governance/resource-graph/paginate-powershell) query results or...

Use your shared query and refine down the line

``` powershell
$sharedQueryResults = Search-AzGraph -Query "{{$SharedResourceIdQuery}} | where <fill_in>" -UseTenantScope -First 1000
```

## Conclusion

Shared queries definitely makes for a better management experience for you and your team. By keeping the kql code central, you won't need to update any hardcoded kql queries in scripts!
The query code can be revised and results reviewed at any time! What's not to love about that? :grin: I hope this useful in some way. Happy share querying!

Ttyl/Urv

## Additional resources:

- [Azure Resource Graph documentation](https://learn.microsoft.com/en-us/azure/governance/resource-graph/)

## Appendix

- [Sample Resource Graph queries for common scenarios](https://learn.microsoft.com/en-us/azure/governance/resource-graph/samples/samples-by-table?tabs=azure-cli)
