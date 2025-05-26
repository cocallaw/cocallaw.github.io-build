---
title: "Monitor Azure Files Usage with Workbooks and Alerts"
date: 2025-06-25 12:00:00 -500
categories: [Azure, Bicep, Monitoring]
tags: [Azure, Bicep, Workbooks, Storage, Monitoring]
author: CLC
description: Learn how to monitor your Azure Files Share usage to avoid quota issues and receive alerts as needed.
toc: true
comments: true
media_subpath: /post-content/06-az-filesusage
---

## Why Monitor File Share Usage?

Azure File Shares have a defined quota. If usage approaches this limit, services may experience interruptions. Azure Monitor provides metrics for:

- **Used Capacity** (in bytes)  
- **Provisioned Quota** (in GiB)

By visualizing these in a centralized dashboard, you get a quick glance at usage across all shares.

## Step 1: Create the Workbook Manually (Optional)

If you'd like to build the workbook manually first:

1. Go to **Azure Monitor > Workbooks > + New**
2. Add a **Metrics Grid**
3. Scope it to your storage accounts and select:
   - Metric: `Used capacity`
   - Metric: `Provisioned capacity`
4. Set visualization to **Table**
5. Add a custom column for usage percentage:
   ```text
   (Used capacity / Provisioned capacity) * 100
   ```
6. Add conditional formatting for values above 80%

Once complete, click **Advanced Editor**, copy the JSON, and save it locally.

## Step 2: Deploy via Bicep

Here’s a Bicep template that deploys the workbook automatically using a pre-defined JSON

```bicep
param location string = resourceGroup().location
param workbookName string = 'FileShareUsageWorkbook'

resource workbook 'Microsoft.Insights/workbooks@2020-10-20' = {
  name: workbookName
  location: location
  kind: 'user'
  properties: {
    displayName: 'Azure File Share Usage Monitor'
    category: 'Storage'
    shared: true
    version: '1.0'
    serializedData: loadTextContent('fileshare-usage-workbook.json')
  }
}
```

## Customizing Scope to Specific Resource Groups or Tags

By default, the workbook queries metrics across all storage accounts and file shares in the selected subscription. However, you may want to limit visibility to certain **resource groups** or filter by **tags** (e.g., show only production workloads).

### Filter by Specific Resource Groups

To scope the workbook to specific resource groups:

#### Option 1: Hardcode Resource IDs

Modify the workbook JSON’s `resources` array with specific resource IDs for your storage accounts:

```json
"resources": [
  "/subscriptions/&lt;sub-id&gt;/resourceGroups/rg-fileshares-01/providers/Microsoft.Storage/storageAccounts/mystorage1",
  "/subscriptions/&lt;sub-id&gt;/resourceGroups/rg-fileshares-02/providers/Microsoft.Storage/storageAccounts/mystorage2"
]
```

This restricts metrics queries to only those accounts.

#### Option 2: Use a Resource Group Parameter

Let the user choose a resource group dynamically:

```json
{
  "name": "resourceGroupParam",
  "type": 3,
  "value": "All",
  "isRequired": false,
  "input": {
    "type": "ResourceGroupPicker"
  }
}
```

Then, update your Kusto queries in the workbook like so:

```kusto
| where ResourceGroup == {resourceGroupParam}
```

### Filter by Tags

If your storage accounts are tagged (e.g., `Environment = Production`), you can filter based on tags directly in Kusto:

```kusto
InsightsMetrics
| where ResourceType == "microsoft.storage/storageaccounts/fileServices/shares"
| where Tags["Environment"] == "Production"
```

**Note:** Not all metric tables include tag data. If filtering by tags doesn’t return expected results, consider enriching the data using Azure Resource Graph or `ResourceContainers`.

## Optional: Add Alerts

You can also set alerts when a share exceeds 80% of its quota:

1. Go to **Azure Monitor > Alerts**
2. Create a rule for the `Used capacity` metric
3. Set a threshold (e.g., 80% of quota)
4. Send notifications or trigger automation

## Wrap Up

With a single Bicep deployment, you now have visibility into your Azure File Share capacity. This is especially useful in enterprise environments with many storage accounts and shares.

Let me know if you'd like a version scoped to specific resource groups, filtered by tags, or integrated with Logic Apps for automation!

## Summary

Here’s a quick summary of how to set up monitoring for Azure File Shares using Azure Monitor Workbooks and Bicep:

- **Metrics like Used Capacity and Provisioned Quota are available by default**, so no extra configuration is needed unless you're exporting to Log Analytics.
- You can **manually create the workbook** via the Azure Portal or **automate it using Bicep** with an exported JSON definition.
- The workbook can be **customized to filter by resource groups or tags** using hardcoded scopes or dynamic parameters.
- You can enhance visibility by **pinning visuals to dashboards** and **setting alerts** when shares approach quota limits.
- This approach provides centralized visibility and proactive management of file share capacity at scale.

Let me know if you’d like the template integrated into a pipeline or extended to include drill-downs, logic apps, or cost visibility!
