+++
title = 'Introduction to Azure Resource Graph (ARG)'
date = 2024-02-18
draft = false
+++

Azure Resource Graph is one of those things I stumbled upon while learning (or attempting) kusto query. At first, it was just curiousity, and then I realized the possibilities it opens up. In hindsight, it has always been there, but to truly appreciate ARG you need that _aha_ moment that you'll only get by trying it out! So let's dive into some things I learned along the way...

## What is Azure Resource Graph?

Azure Resource Graph (ARG) helps you explore your resources for a specific tenant, of which you can filter down:

- management group
- subscription
- resource group
- type
- loction
- resourceId

Before ARG, if I wanted to get a list of say all the resources I have, I'd do the following in pwsh

```pwsh
$resources = Get-AzResource | ForEach-Object {
    [PSCustomObject]@{
        ResourceGroupName = $_.ResourceGroupName
        ResourceName = $_.ResourceName
        ResourceType = $_.ResourceType
        Location = $_.Location
    }
}

$resources | Group-Object -Property ResourceType | ForEach-Object {
    [PSCustomObject]@{
        ResourceType = $_.Name
        Count = $_.Count
    }
}
```

Now here's the thing, this will do if you have one subscription. So at first, I wasn't impressed until... I had a customer with 150+ subscriptions that needed managing. Can you imagine all that context switching? The horror. That's where ARG shines!

Here's what the kql would look like:

```pwsh
resources
| summarize count=count() by type
| order by type asc
```

This will give you the same result only faster... AND... on a larger scale. So here's what that query would look like on a larger scale:

```pwsh
resources
| join kind=leftouter(
    resourcecontainers
    | where type=='microsoft.resources/subscriptions'
    | project subscriptionName=name, subscriptionId
) on subscriptionId
| summarize count=count() by subscriptionName,type
| order by subscriptionName,type asc
```

I know, this query may seem daunting, I'm still learning as well. Have a look at the official [documentation](https://learn.microsoft.com/en-us/azure/governance/resource-graph/) to get you started

## ARG and Governance go together

Did someone say [resource tagging](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/govern/guides/standard/prescriptive-guidance#resource-tagging)? Honestly, if there ever was a reason for tagging, this would be it. Imagine having to report on resources you manage without properly tagging them. Now just think about that on a larger scale...

Naming convention is great. ( I know, I know...) if you want to start a heated discussion. People have strong opnions on this subject. Tag your resources please!

ARG and tagging are a match made in heaven!

## ARG in the portal

Ever wonder how the portal gets all the alerts for a specific resource? ARG! Before, I'd do a F12 to see what was going on under the hood. I found out it was just an ARG query, a complicated one, but ARG nonetheless. Turns out that isn't necessary anymore. You can _open query_ in the portal these days. I was blown away and terrified at the same time! Up to that point I'd never seen _**union, set_has_element**_ used in ARG queries. This kql is going to take some getting used to. And that's only ARG for now, imagine going full Azure Data Explorer kql! One can only dream.

## Things to consider

Here are some things to think about.

### ARG kusto query is a subset

I've seen some complex query using the _let_ operator. _Let_ doesn't work in ARG, or maybe I'm not doing it correctly. Which isn't so say it isn't a skill you can't aquire. The _let_ operator works just fine in loganalytics queries (something I'm getting more into when setting up monitoring alerts).

### Correlate ARG with Azure Monitor Logs

You can do quite some nifty stuff by integrating ARG in your LogAnalytic workspace query.

Here's one I borrowed from the online documentation:

_Retrieve performance data related to CPU utilization and filter to resources with the “prod” tag._

```pwsh
InsightsMetrics
| where Name == "UtilizationPercentage"
| lookup (
    arg("").Resources
    | where type == 'microsoft.compute/virtualmachines'
    | project _ResourceId=tolower(id), tags
    )
    on _ResourceId
| where tostring(tags.Env) == "Prod"
```

Be sure to have a look at the [documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/azure-monitor-data-explorer-proxy#query-data-in-azure-resource-graph-by-using-arg-preview). This opens up so many possibilities!

## Conclusion

Learning kusto query is a must! Granted, it can be overwhelming at times, but, like everything, you need to practice. When faced with a complex query, break it down. It is an aquired taste, but one worth cultivating. I'll end on this note but I'll be back with some more ARG soon...

Ttyl/Urv
