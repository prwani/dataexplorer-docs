---
title: 'Query data in Azure Monitor with Azure Data Explorer (Preview)'
description: 'In this topic, query data in Azure Monitor by creating an Azure Data Explorer proxy for cross product queries with Application Insights and Log Analytics'
services: data-explorer
author: orspod
ms.author: orspodek
ms.reviewer: rkarlin
ms.service: data-explorer
ms.topic: conceptual
ms.date: 01/28/2020

#Customer intent: I want to query data in Azure Monitor using Azure Data Explorer by creating an Azure Data Explorer (ADX) proxy for cross product queries with Log Analytics and Application Insights 
---

# Query data in Azure Monitor using Azure Data Explorer (Preview)

The Azure Data Explorer proxy cluster (ADX Proxy) is an entity that enables you to perform cross product queries between Azure Data Explorer, [Application Insights (AI)](/azure/azure-monitor/app/app-insights-overview), and [Log Analytics (LA)](/azure/azure-monitor/platform/data-platform-logs) in the [Azure Monitor](/azure/azure-monitor/) service. You can map Azure Monitor Log Analytics workspaces or Application Insights apps as proxy clusters. You can then query the proxy cluster using Azure Data Explorer tools and refer to it in a cross cluster query. The article shows how to connect to a proxy cluster, add a proxy cluster to Azure Data Explorer Web UI, and run queries against your AI apps or LA workspaces from Azure Data Explorer.

The Azure Data Explorer proxy flow: 

![ADX proxy flow](media/adx-proxy/adx-proxy-flow.png)

## Prerequisites

> [!NOTE]
> The ADX Proxy is in preview mode. [Connect to the proxy](#connect-to-the-proxy) to enable the ADX proxy feature for your clusters. Contact the [ADXProxy](mailto:adxproxy@microsoft.com) team with any questions.

## Connect to the proxy

1. Verify your Azure Data Explorer native cluster (such as *help* cluster) appears on the left menu before you connect to your Log Analytics or Application Insights cluster.

    ![ADX native cluster](media/adx-proxy/web-ui-help-cluster.png)

1. In the Azure Data Explorer UI (https://dataexplorer.azure.com/clusters), select **Add Cluster**.

1. In the **Add Cluster** window, add the URL of the LA or AI cluster. 
    
    * For LA: `https://ade.loganalytics.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.operationalinsights/workspaces/<workspace-name>`
    * For AI: `https://ade.applicationinsights.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.insights/components/<ai-app-name>`

    * Select **Add**.

    ![Add cluster](media/adx-proxy/add-cluster.png)

    >[!NOTE]
    >If you add a connection to more than one proxy cluster, give each a different name. Otherwise they'll all have the same name in the left pane.

1. After the connection is established, your LA or AI cluster will appear in the left pane with your native ADX cluster. 

    ![Log Analytics and Azure Data Explorer clusters](media/adx-proxy/la-adx-clusters.png)

> [!NOTE]
> The number of Azure Monitor workspaces that can be mapped is limited to 100.

## Run queries

You can run the queries using client tools that support Kusto queries, such as: Kusto Explorer, ADX Web UI, Jupyter Kqlmagic, Flow, PowerQuery, PowerShell, Jarvis, Lens, REST API.

> [!TIP]
> * Database name should have the same name as the resource specified in the proxy cluster. Names are case sensitive.
> * In cross cluster queries, make sure that the naming of Application Insights apps and Log Analytics workspaces is correct.
> * If names contain special characters, they're replaced by URL encoding in the proxy cluster name. 
> * If names include characters that don't meet [KQL identifier name rules](kusto/query/schema-entities/entity-names.md), they are replaced by the dash **-** character.

### Direct query from your LA or AI ADX Proxy cluster

Run queries on your LA or AI cluster. Verify that your cluster is selected in the left pane. 
 
```kusto
Perf | take 10 // Demonstrate query through the proxy on the LA workspace
```

![Query LA workspace](media/adx-proxy/query-la.png)

### Cross query of your LA or AI ADX Proxy cluster and the ADX native cluster 

When you run cross cluster queries from the proxy, verify your ADX native cluster is selected in the left pane. The following examples demonstrate combining ADX cluster tables (using `union`) with LA workspace.

```kusto
union StormEvents, cluster('https://ade.loganalytics.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.operationalinsights/workspaces/<workspace-name>').database('<workspace-name>').Perf
| take 10 
```

```kusto
let CL1 = 'https://ade.loganalytics.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.operationalinsights/workspaces/<workspace-name>';
union <ADX table>, cluster(CL1).database(<workspace-name>).<table name>
```

   [ ![Cross query from the Azure Data Explorer proxy](media/adx-proxy/cross-query-adx-proxy.png)](media/adx-proxy/cross-query-adx-proxy.png#lightbox)

Using the [`join` operator](kusto/query/joinoperator.md), instead of union, may require a [`hint`](kusto/query/joinoperator.md#join-hints) to run it on an Azure Data Explorer native cluster (and not on the proxy). 

## Function supportability

The Azure Data Explorer proxy cluster supports functions for both Application Insights and Log Analytics.
This capability enables cross-cluster queries to reference an Azure Monitor tabular function directly.
The following commands are supported by the proxy:

* `.show functions`
* `.show function {FunctionName}`
* `.show database {DatabaseName} schema as json`

The following image depicts an example of querying a tabular function from the Azure Data Explorer Web UI. 
To use the function, run the name in the Query window.

  [ ![Query a tabular function from Azure Data Explorer Web UI](media/adx-proxy/function-query-adx-proxy.png)](media/adx-proxy/function-query-adx-proxy.png#lightbox)

> [!NOTE]
> Azure Monitor only supports tabular functions. Tabular functions don't support parameters.

## Additional syntax examples

The following syntax options are available when calling the Application Insights (AI) or Log Analytics (LA) clusters:

|Syntax Description  |Application Insights  |Log Analytics  |
|----------------|---------|---------|
| Database within a cluster that contains only the defined resource in this subscription (**recommended for cross cluster queries**) |   cluster(`https://ade.applicationinsights.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.insights/components/<ai-app-name>').database('<ai-app-name>`) | cluster(`https://ade.loganalytics.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.operationalinsights/workspaces/<workspace-name>').database('<workspace-name>`)     |
| Cluster that contains all apps/workspaces in this subscription    |     cluster(`https://ade.applicationinsights.io/subscriptions/<subscription-id>`)    |    cluster(`https://ade.loganalytics.io/subscriptions/<subscription-id>`)     |
|Cluster that contains all apps/workspaces in the subscription and are members of this resource group    |   cluster(`https://ade.applicationinsights.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>`)      |    cluster(`https://ade.loganalytics.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>`)      |
|Cluster that contains only the defined resource in this subscription      |    cluster(`https://ade.applicationinsights.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.insights/components/<ai-app-name>`)    |  cluster(`https://ade.loganalytics.io/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.operationalinsights/workspaces/<workspace-name>`)     |

## Next steps

[Write queries](write-queries.md)
