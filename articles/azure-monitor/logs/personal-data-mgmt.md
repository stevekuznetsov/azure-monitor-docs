---
title: Managing personal data in Azure Monitor Logs and Application Insights
description: This article describes how to manage personal data stored in Azure Monitor Log Analytics and the methods to identify and remove it.
ms.topic: conceptual
author: guywild
ms.author: guywild
ms.reviewer: meirm
ms.date: 06/28/2022
# Customer intent: As an Azure Monitor admin user, I want to understand how to manage personal data in logs that Azure Monitor collects.

---

# Managing personal data in Azure Monitor Logs and Application Insights

Log Analytics is a data store where personal data is likely to be found. Application Insights stores its data in a Log Analytics partition. This article explains where Log Analytics and Application Insights store personal data and how to manage this data.

In this article, _log data_ refers to data sent to a Log Analytics workspace, while _application data_ refers to data collected by Application Insights. If you're using a workspace-based Application Insights resource, the information on log data applies. If you're using a classic Application Insights resource, the application data applies.

[!INCLUDE [gdpr-dsr-and-stp-note](~/reusable-content/ce-skilling/azure/includes/gdpr-dsr-and-stp-note.md)]

## Permissions required

| Action | Permissions required |
|:-------|:---------------------|
| Purge data from a Log Analytics workspace | `Microsoft.OperationalInsights/workspaces/purge/action` permissions to the Log Analytics workspace, as provided by the [Log Analytics Contributor built-in role](./manage-access.md#log-analytics-contributor), for example |


## Strategy for personal data handling

While it's up to you and your company to define a strategy for handling personal data, here are a few approaches, listed from most to least preferable from a technical point of view:

* Stop collecting personal data, or obfuscate, anonymize, or adjust collected data to exclude it from being considered "personal". This is _by far_ the preferred approach, which saves you the need to create a costly and impactful data handling strategy.
* Normalize the data to reduce negative affects on the data platform and performance. For example, instead of logging an explicit User ID, create a lookup to correlate the username and their details to an internal ID that can then be logged elsewhere. That way, if a user asks you to delete their personal information, you can delete only the row in the lookup table that corresponds to the user. 
* If you need to collect personal data, build a process using the purge API path and the existing query API to meet any obligations to export and delete any personal data associated with a user.

## Where to look for personal data in Log Analytics

Log Analytics prescribes a schema to your data, but allows you to override every field with custom values. You can also ingest custom schemas. As such, it's impossible to say exactly where personal data will be found in your specific workspace. The following locations, however, are good starting points in your inventory.

> [!NOTE]
> Some of the queries below use `search *` to query all tables in a workspace. We highly recommend you avoid using `search *`, which creates a highly inefficient query, whenever possible. Instead, query a specific table.

### Log data

* **IP addresses**: Log Analytics collects various IP information in multiple tables. For example, the following query shows all tables that collected IPv4 addresses in the last 24 hours:
    ```
    search * 
    | where * matches regex @'\b((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)(\.|$)){4}\b' //RegEx originally provided on https://stackoverflow.com/questions/5284147/validating-ipv4-addresses-with-regexp
    | summarize count() by $table
    ```
    
* **User IDs**: You'll find user usernames and user IDs in various solutions and tables. You can look for a particular username or user ID across your entire dataset using the search command:
    ```
    search "<username or user ID>"
    ```
    
  Remember to look not only for human-readable usernames but also for GUIDs that can be traced back to a particular user.
* **Device IDs**: Like user IDs, device IDs are sometimes considered personal data. Use the approach listed above for user IDs to identify tables that hold personal data. 
* **Custom data**: Log Analytics lets you collect custom data through custom logs, custom fields, the [HTTP Data Collector API](../logs/data-collector-api.md), and as part of system event logs. Check all custom data for personal data.
* **Solution-captured data**: Because the solution mechanism is open-ended, we recommend reviewing all tables generated by solutions to ensure compliance.

### Application data

* **IP addresses**: While Application Insights obfuscates all IP address fields to `0.0.0.0` by default, it's fairly common to override this value with the actual user IP to maintain session information. Use the query below to find any table that contains values in the *IP address* column other than `0.0.0.0` in the last 24 hours:
    ```
    search client_IP != "0.0.0.0"
    | where timestamp > ago(1d)
    | summarize numNonObfuscatedIPs_24h = count() by $table
    ```
    
* **User IDs**: By default, Application Insights uses randomly generated IDs for user and session tracking in fields such as *session_Id*, *user_Id*, *user_AuthenticatedId*, *user_AccountId*, and *customDimensions*. However, it's common to override these fields with an ID that's more relevant to the application, such as usernames or Microsoft Entra GUIDs. These IDs are often considered to be personal data. We recommend obfuscating or anonymizing these IDs. 
* **Custom data**: Application Insights allows you to append a set of custom dimensions to any data type. Use the following query to identify custom dimensions collected in the last 24 hours:
    ```
    search * 
    | where isnotempty(customDimensions)
    | where timestamp > ago(1d)
    | project $table, timestamp, name, customDimensions 
    ```
    
* **In-memory and in-transit data**: Application Insights tracks exceptions, requests, dependency calls, and traces. You'll often find personal data at the code and HTTP call level. Review exceptions, requests, dependencies, and traces tables to identify any such data. Use [telemetry initializers](../app/api-filtering-sampling.md) where possible to obfuscate this data.
* **Snapshot Debugger captures**: The [Snapshot Debugger](../app/snapshot-debugger.md) feature in Application Insights lets you collect debug snapshots when Application Insights detects an exception on the production instance of your application. Snapshots expose the full stack trace leading to the exceptions and the values for local variables at every step in the stack. Unfortunately, this feature doesn't allow selective deletion of snap points or programmatic access to data within the snapshot. Therefore, if the default snapshot retention rate doesn't satisfy your compliance requirements, we recommend you turn off the feature.

## Exporting and deleting personal data

We __strongly__ recommend you restructure your data collection policy to stop collecting personal data, obfuscate or anonymize personal data, or otherwise modify such data until it's no longer considered personal. In handling personal, data you'll incur costs in defining and automating a strategy, building an interface through which your customers interact with their data, and ongoing maintenance. It's also computationally costly for Log Analytics and Application Insights, and a large volume of concurrent Query or Purge API calls can negatively affect all other interactions with Log Analytics functionality. However, if you have to collect personal data, follow the guidelines in this section.

> [!IMPORTANT]
>  While most purge operations complete much quicker, **the formal SLA for the completion of purge operations is set at 30 days** due to their heavy impact on the data platform. This SLA meets GDPR requirements. It's an automated process, so there's no way to expedite the operation. 

### View and export

Use the [Log Analytics query API](/rest/api/loganalytics/dataaccess/query) or the [Application Insights query API](/rest/api/application-insights/query) for view and export data requests. 

> [!NOTE]
> You can't use the Log Analytics query API on that have the [Basic and Auxiliary table plans](data-platform-logs.md#table-plans). Instead, use the [Log Analytics /search API](basic-logs-query.md#run-a-query-on-a-basic-or-auxiliary-table).

You need to implement the logic for converting the data to an appropriate format for delivery to your users. [Azure Functions](https://azure.microsoft.com/services/functions/) is a great place to host such logic.


### Delete

> [!WARNING]
> Deletes in Log Analytics are destructive and non-reversible! Please use extreme caution in their execution.

Azure Monitor's [Purge API](/rest/api/loganalytics/workspacepurge/purge) lets you delete personal data. Use the purge operation sparingly to avoid potential risks, performance impact, and the potential to skew all-up aggregations, measurements, and other aspects of your Log Analytics data. See the [Strategy for personal data handling](#strategy-for-personal-data-handling) section for alternative approaches to handling personal data.

Purge is a highly privileged operation. Grant the _Data Purger_ role in Azure Resource Manager cautiously due to the potential for data loss.

To manage system resources, we limit purge requests to 50 requests an hour. Batch the execution of purge requests by sending a single command whose predicate includes all user identities that require purging. Use the [in operator](/azure/kusto/query/inoperator) to specify multiple identities. Run the query before executing the purge request to verify the expected results.

> [!IMPORTANT]
> Use of the Log Analytics or Application Insights Purge API does not affect your retention costs. To lower retention costs, you must decrease your data retention period.

#### Log data

* The [Workspace Purge POST API](/rest/api/loganalytics/workspacepurge/purge) takes an object specifying parameters of data to delete and returns a reference GUID. 
* The [Get Purge Status POST API](/rest/api/loganalytics/workspace-purge/get-purge-status) returns an 'x-ms-status-location' header that includes a URL you can call to determine the status of your purge operation. For example:

    ```
    x-ms-status-location: https://management.azure.com/subscriptions/[SubscriptionId]/resourceGroups/[ResourceGroupName]/providers/Microsoft.OperationalInsights/workspaces/[WorkspaceName]/operations/purge-[PurgeOperationId]?api-version=2015-03-20
    ```

> [!NOTE]
> You can't purge data from tables that have the [Basic and Auxiliary table plans](data-platform-logs.md#table-plans).

#### Application data

* The [Components - Purge POST API](/rest/api/application-insights/components/purge) takes an object specifying parameters of data to delete and returns a reference GUID.
* The [Components - Get Purge Status GET API](/rest/api/application-insights/components/get-purge-status) returns an 'x-ms-status-location' header that includes a URL you can call to determine the status of your purge operation. For example:

   ```
   x-ms-status-location: https://management.azure.com/subscriptions/[SubscriptionId]/resourceGroups/[ResourceGroupName]/providers/microsoft.insights/components/[ComponentName]/operations/purge-[PurgeOperationId]?api-version=2015-05-01
   ```

## Next steps
- Learn more about [how Log Analytics collects, processes, and secures data](../logs/data-security.md).
- Learn more about [how Application Insights collects, processes, and secures data](/previous-versions/azure/azure-monitor/app/data-retention-privacy).