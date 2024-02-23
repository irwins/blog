+++
title = 'Azure OpenAI Assistant'
date = 2024-02-23T08:18:21+01:00
draft = true
+++

## OpenAI is the next shiny thing

I've been following OpenAI developement closely ever since its inception. I was a bit hesitant on upgrading, fearing I wouldn't use its full potential (Also I'm quite frugal...:smirk:). Fortunately there's also a pay-as-you-go plan for using the API directly. This gives you more control on your credits.

When Microsoft announced their version, I was excited about it, as I'm always in Azure. I also have a monthly credit to spend. You do need to [apply](https://aka.ms/oaiapply) for it, but that process is quite easy. Once you gain access the fun can begin!

FYI, the beauty of Azure OpenAI is that it [follows](https://platform.openai.com/docs/assistants/overview) the REST API model of OpenAI closely. That makes transitioning smoothly from one to the other. Have a look at the [documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/overview) for a better understanding of the capabilities.

## What is Azure OpenAI Assistant

I like OpenAI's definition:

> An Assistant has instructions and can leverage models, tools, and knowledge to respond to user queries.

Think of an assitant as a soundboard, where you you can exchange thoughts and ideas. For better results you need to optimize your instructions, but that is a topic for another time.

It took me quite some time to grasp this concept. At first I was using the assistant as a google search. An assistant is more than that. With proper instructions, you can have pretty meaningful conversations. It all comes down to your prompting skills. [Prompt engineering](https://platform.openai.com/docs/guides/prompt-engineering) is a thing! It really is the gateway for better results. How to get better at it? Trial and error... Like everything, practice, practice, practice!

## Setting up Azure OpenAI instance

Once you're eligble for the OpenAI you can deploy your OpenAI resource. Assistant is currently in _Preview_ and only available from specific regions. I've defaulted to _East US 2_.

You can do this all in the [portal](https://learn.microsoft.com/en-us/azure/ai-services/openai/assistants-quickstart?tabs=command-line&pivots=rest-api#set-up), but I was curious if there was support for bicep deployment. There is...:smile:

```bicep
@minLength(2)
@maxLength(64)
@description('Name of the AzOpenAI instance')
param name string = //Instance name. This has to be unique

@description('AzOpenAI instance kind')
param accountKind string = 'OpenAI'

@minLength(1)
@description('Primary location for all resources')
param location string = 'eastus2'

param deployments array = [
  {
    name: 'assistant'
    model: 'gpt-35-turbo'
    format: 'OpenAI'
    version: '0301'
    sku: 'Standard'
    capacity: 120
  }
]

param sku string = 'S0'

param tags object = {
  environment:'DEV'
  createdBy: //Your persona
}

resource open_ai_account 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
  name: name
  location: location
  tags:tags
  kind: accountKind
  sku: {
    name: sku
  }
  properties: {
    customSubDomainName: toLower(name)
  }
}

resource open_ai_deployment 'Microsoft.CognitiveServices/accounts/deployments@2023-05-01' = [for deployment in deployments: {
  parent: open_ai_account
  name: deployment.name
  sku: {
    name: deployment.sku
    capacity: deployment.capacity
  }
  properties: {
      model: {
        name: deployment.model
        format: deployment.format
        version: deployment.version
      }
      raiPolicyName: 'Microsoft.Default'
      versionUpgradeOption: 'OnceNewDefaultVersionAvailable'
  }
}]

output endpoint string = open_ai_account.properties.endpoint
output id string = open_ai_account.id
output name string = open_ai_account.name
```

Here's the bicepparam to accompany the bicep file.

```bicep
using 'main.bicep'

param location = 'eastus2'
param name = //Unique instance name
param sku = 'S0'
param tags = {
  environment: 'DEV'
  createdBy: 'Irwin Strachan'
}
```

At this point a pipeline may be a bit much, so here's how to deploy using the az cli.
What? No PowerShell?!? A poor craftsman always blames his tools...There is merit in having more than one skill...:wink:

```bash
### LOGIN TENANT
TENANT_ID= #Your Tenant
SUBSCRIPTION_ID= #Subscription you applied

az login --tenant $TENANT_ID
az account set --subscription $SUBSCRIPTION_ID

### CREATE RESOURCE GROUP
RESOURCE_GROUP_NAME=rg-openai
LOCATION=eastus2

az group create -n $RESOURCE_GROUP_NAME -l $LOCATION

### Deploy main bicep template using bicep parameters file

### Lint bicep file
az bicep lint --file #Refer to your bicep file

### Validate bicep file
az deployment group validate \
    --resource-group $RESOURCE_GROUP_NAME \
    --parameters #Refer to your bicepparam file

### Preview bicep file
az deployment group what-if \
    --resource-group $RESOURCE_GROUP_NAME \
    --parameters #Refer to your bicepparam file\
    --template-file #Refer to your bicep file

az deployment group create \
    --resource-group $RESOURCE_GROUP_NAME \
    --parameters #Refer to your bicepparam file
```

Ok now that we've deployed our instance we can get down to business!

## Using Aure OpenAI Assistant

As an introduction I'd advice using the _Studio_ to browse around. Once your comfortable you can give Doug Finke his [PowerShellAIAssistant](https://github.com/dfinke/PowerShellAIAssistant) module a try. It supports both OpenAI & AzureOpenAI :smile:. Doug has contributed quite a few popular modules over the years with [ImportExcel](https://www.powershellgallery.com/packages/ImportExcel/7.8.6) being "the chef's kiss"

Ok so up to this point, you've done your deployment and now you can create your assistant...

### Instructions


### Conversing with AzOpenAI assistant

![Chat 01](../../images/azure-openai-assistant/chat-01.png)