---
author: bwren
ms.author: bwren
ms.service: azure-monitor
ms.topic: include
ms.date: 08/24/2023
---

### Design checklist

> [!div class="checklist"]
> - Determine whether to combine your operational data and your security data in the same Log Analytics workspace.
> - Configure pricing tier for the amount of data that each Log Analytics workspace typically collects.
> - Configure data retention and archiving.
> - Configure tables used for debugging, troubleshooting, and auditing as Basic Logs.
> - Limit data collection from data sources for the workspace.
> - Regularly analyze collected data to identify trends and anomalies.
> - Create an alert when data collection is high.
> - Consider a daily cap as a preventative measure to ensure that you don't exceed a particular budget.
> - Set up alerts on Azure Advisor cost recommendations for Log Analytics workspaces.

### Configuration recommendations

| Recommendation | Benefit |
|:---|:---|
| Determine whether to combine your operational data and your security data in the same Log Analytics workspace. | Since all data in a Log Analytics workspace is subject to Microsoft Sentinel pricing if Sentinel is enabled, there might be cost implications to combining this data. See [Design a Log Analytics workspace strategy](../logs/workspace-design.md) for details on making this decision for your environment balancing it with criteria in other pillars. |
| Configure pricing tier for the amount of data that each Log Analytics workspace typically collects. | By default, Log Analytics workspaces will use pay-as-you-go pricing with no minimum data volume. If you collect enough data, you can significantly decrease your cost by using a [commitment tier](../logs/cost-logs.md#commitment-tiers), which allows you to commit to a daily minimum of data collected in exchange for a lower rate. If you collect enough data across workspaces in a single region, you can link them to a [dedicated cluster](../logs/logs-dedicated-clusters.md) and combine their collected volume using [cluster pricing](../logs/cost-logs.md#dedicated-clusters).<br><br>See [Azure Monitor Logs cost calculations and options](../logs/cost-logs.md) for details on commitment tiers and guidance on determining which is most appropriate for your level of usage. See [Usage and estimated costs](../usage-estimated-costs.md#usage-and-estimated-costs) to view estimated costs for your usage at different pricing tiers.  |
| Configure interactive and long-term data retention. | There's a charge for retaining data in a Log Analytics workspace beyond the default of 31 days (90 days if Sentinel is enabled on the workspace and 90 days for Application insights data). Consider your particular requirements for having data readily available for log queries. You can significantly reduce your cost by configuring [long-term retention](../logs/data-retention-configure.md), which allows you to retain data for up to twelve years and still access it occasionally using [search jobs](../logs/search-jobs.md) or [restoring a set of data](../logs/restore.md) to the workspace. |
| Configure tables used for debugging, troubleshooting, and auditing as Basic Logs. | Tables in a Log Analytics workspace configured for [Basic Logs](../logs/logs-table-plans.md) have a lower ingestion cost in exchange for limited features and a charge for log queries. If you query these tables infrequently and don't use them for alerting, this query cost can be more than offset by the reduced ingestion cost. |
| Limit data collection from data sources for the workspace. | The primary factor for the cost of Azure Monitor is the amount of data that you collect in your Log Analytics workspace, so you should ensure that you collect no more data that you require to assess the health and performance of your services and applications. See [Design a Log Analytics workspace architecture](../logs/workspace-design.md) for details on making this decision for your environment balancing it with criteria in other pillars.<br><br>Tradeoff: There might be a tradeoff between cost and your monitoring requirements. For example, you might be able to detect a performance issue more quickly with a high sample rate, but you might want a lower sample rate to save costs. Most environments have multiple data sources with different types of collection, so you need to balance your particular requirements with your cost targets for each. See [Cost optimization in Azure Monitor](../best-practices-cost.md) for recommendations on configuring collection for different data sources. |
| Regularly analyze collected data to identify trends and anomalies.  | Use [Log Analytics workspace insights](../logs/log-analytics-workspace-insights-overview.md) to periodically review the amount of data collected in your workspace. In addition to helping you understand the amount of data collected by different sources, it will identify anomalies and upward trends in data collection that could result in excess cost. Further analyze data collection using methods in [Analyze usage in Log Analytics workspace](../logs/analyze-usage.md) to determine if there's additional configuration that can decrease your usage further. This is particularly important when you add a new set of data sources, such as a new set of virtual machines or onboard a new service. |
| Create an alert when data collection is high. | To avoid unexpected bills, you should be [proactively notified anytime you experience excessive usage](../logs/analyze-usage.md#send-alert-when-data-collection-is-high). Notification allows you to address any potential anomalies before the end of your billing period. |
| Consider a daily cap as a preventative measure to ensure that you don't exceed a particular budget. | A [daily cap](../logs/daily-cap.md) disables data collection in a Log Analytics workspace for the rest of the day after your configured limit is reached. This shouldn't be used as a method to reduce costs as described in [When to use a daily cap](../logs/daily-cap.md#when-to-use-a-daily-cap).<br><br>If you do set a daily cap, in addition to [creating an alert when the cap is reached](../logs/log-analytics-workspace-health.md#view-log-analytics-workspace-health-and-set-up-health-status-alerts), ensure that you also [create an alert rule to be notified when some percentage has been reached (90% for example)](../logs/analyze-usage.md#send-alert-when-data-collection-is-high). This gives you an opportunity to investigate and address the cause of the increased data before the cap shuts off data collection. |
| Set up alerts on Azure Advisor cost recommendations for Log Analytics workspaces. | Azure Advisor recommendations for Log Analytics workspaces proactively alert you when there's an opportunity to optimize your costs. [Create Azure Advisor alerts](../../advisor/advisor-alerts-portal.md) for these cost recommendations: <ul><li>Consider configuring the cost effective Basic logs plan on selected tables - We've identified ingestion of more than 1 GB per month to tables that are eligible for the low cost Basic log data plan. The Basic log plan gives you query capabilities for debugging and troubleshooting at a lower cost.</li><li>Consider changing pricing tier- Based on your current usage volume, investigate changing your pricing (Commitment) tier to receive a discount and reduce costs.</li><li>Consider removing unused restored tables - You have one or more tables with restored data active in your workspace. If you're no longer using a restored data, delete the table to avoid unnecessary charges.</li><li>Data ingestion anomaly was detected - We've identified a much higher ingestion rate over the past week, based on your ingestion in the three previous weeks. Take note of this change and the expected change in your costs.</ul></li>You can also view automatically generated recommendation by selecting **Overview** > **Recommendations** or **Advisor recommendations** from your Log Analytics workspace resource menu.|
 