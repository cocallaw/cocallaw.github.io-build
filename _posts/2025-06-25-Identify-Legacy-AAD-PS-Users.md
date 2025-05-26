---
title: "Identify AzureAD PowerShell Users in your Tenant before Retirement"
date: 2025-06-25 12:00:00 -500
categories: [Azure, PowerShell, Entra]
tags: [AzureAD, Microsoft Graph, PowerShell, Entra ID, Reporting, Security]
author: CLC
description: 
toc: true
comments: true
media_subpath: /post-content/04-gh-notifyslackdiscord
---

Microsoft is retiring the legacy **AzureAD** and **MSOnline** PowerShell modules. These modules have been widely used for years to manage users, groups, roles, and other directory objects in Azure Active Directory (now Entra ID). With their deprecation, it’s critical to identify any users or automation that still rely on them — and plan a migration to Microsoft Graph PowerShell.

In this post, I’ll walk you through how to:

- Identify sign-ins using the AzureAD PowerShell app ID
- Correlate those sign-ins with Entra alerts
- Export and report this data via PowerShell in CSV and HTML formats
- Generate test sign-ins if you need data for validation

## What's Being Retired?

The following modules are being deprecated:

- `AzureAD` (`AzureAD.Standard.Preview` as well)
- `MSOnline`

These modules authenticate via a legacy app registration (`1b730954-1685-4b74-9bfd-dac224a7b894`) that shows up in sign-in logs. Microsoft Graph PowerShell is the official replacement.

## Option 1: Find AzureAD PowerShell Sign-In Activity with KQL

You can use Microsoft Graph API and Kusto Query Language (KQL) to identify sign-ins tied to the legacy AzureAD app.

### KQL Query

```sql
SigninLogs
| where AppId == "1b730954-1685-4b74-9bfd-dac224a7b894" // AzureAD PowerShell app ID
| project UserPrincipalName, IPAddress, ResourceDisplayName, AuthenticationRequirement, ConditionalAccessStatus, Status, CreatedDateTime
| order by CreatedDateTime desc
```

## Option 2: Use a PowerShell Script to Query Logs and Generate Reports

### Prerequisites

#### Microsoft Graph PowerShell

In order to run the script, you need to have the **Microsoft Graph PowerShell SDK** installed. If you haven't done so already, install it using the following command:

```powershell
Install-Module Microsoft.Graph -Scope CurrentUser
```

#### Permissions

Ensure you have the following permissions granted to your account to run the script:

- `AuditLog.Read.All`
- `Directory.Read.All`
- `IdentityRiskEvent.Read.All`

### What the Script Does

This script connects to Microsoft Graph, retrieves sign-in logs for the AzureAD PowerShell app, and correlates them with identity risk events. It then generates a report in both CSV and HTML formats for easy review.

## The Full PowerShell Script

Here’s the complete PowerShell script, you can also run this script directly from your PowerShell terminal by running the following command:

```powershell
Invoke-Expression $(Invoke-WebRequest -uri aka.ms/wvdmsixps -UseBasicParsing).Content
```
or the shorthand version:
```powershell
iwr -useb aka.ms/wvdmsixps | iex
```
You can also copy and save the full script below or find the current version on GitHub at [cocallaw/Entra-AAD-PS-Retirement](https://github.com/cocallaw/Entra-AAD-PS-Retirement/).

```powershell
function Get-ValidatedDate($prompt, $default) {
    do {
        $inDate = Read-Host $prompt
        if ([string]::IsNullOrWhiteSpace($inDate)) { return $default }
        $parsed = $null
        $valid = [datetime]::TryParseExact($inDate, 'yyyy-MM-dd', $null, 'None', [ref]$parsed)
        if ($valid) { return $parsed }
        Write-Warning "Invalid date format. Please use yyyy-MM-dd."
    } while ($true)
}

$InformationPreference = 'SilentlyContinue'
# Check if Microsoft.Graph module is installed
if (-not (Get-Module -ListAvailable -Name Microsoft.Graph)) {
    Write-Warning "Microsoft.Graph module is not installed. Please install it using: Install-Module Microsoft.Graph -Scope CurrentUser"
    exit
}
Write-Information "Loading Microsoft.Graph module"
Import-Module Microsoft.Graph -ErrorAction SilentlyContinue

# Connect interactively
$requiredScopes = @("AuditLog.Read.All", "Directory.Read.All", "IdentityRiskEvent.Read.All")
$needConnect = $false

if (-not (Get-MgContext)) {
    Write-Information "Connecting to Microsoft Graph..."
    $needConnect = $true
}
else {
    $currentScopes = (Get-MgContext).Scopes
    if (-not $currentScopes -or ($requiredScopes | Where-Object { $_ -notin $currentScopes })) {
        Write-Information "Existing session missing required scopes. Reconnecting..."
        $needConnect = $true
    }
    else {
        Write-Information "Already connected to Microsoft Graph. Reusing existing session."
    }
}

if ($needConnect) {
    Connect-MgGraph -Scopes $requiredScopes
}

# Prompt date range
$startDate = Get-ValidatedDate "Enter start date (yyyy-MM-dd), or press Enter for 30 days ago" (Get-Date).AddDays(-30)
$endDate = Get-ValidatedDate "Enter end date (yyyy-MM-dd), or press Enter for today" (Get-Date)

# Prompt username filter (optional)
$usernameFilter = Read-Host "Enter username filter (optional), or press Enter to skip"

$aadAppId = '1b730954-1685-4b74-9bfd-dac224a7b894'

$startDateUTC = [datetime]::Parse($startDate).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$endDateUTC = ([datetime]::Parse($endDate).AddDays(1).AddSeconds(-1)).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

$filter = "appId eq '$aadAppId' and createdDateTime ge $startDateUTC and createdDateTime le $endDateUTC"
$uri = "https://graph.microsoft.com/v1.0/auditLogs/signIns?`$filter=$( [uri]::EscapeDataString($filter) )"

$allSignIns = @()
do {
    try {
        $response = Invoke-MgGraphRequest -Uri $uri -Method GET
    }
    catch {
        Write-Error "Failed to retrieve sign-in logs: $_"
        Disconnect-MgGraph
        exit
    }
    $allSignIns += $response.value
    $uri = $response.'@odata.nextLink'
} while ($uri)

if ($allSignIns.Count -eq 0) {
    Write-Warning "No sign-in logs found for the specified filters."
    Disconnect-MgGraph
    exit
}

# Add progress indication for large datasets
$progressCount = 0
$totalCount = $allSignIns.Count
$results = $allSignIns | ForEach-Object {
    $progressCount++
    if ($progressCount % 100 -eq 0) {
        Write-Progress -Activity "Processing sign-in logs" -Status "$progressCount of $totalCount" -PercentComplete (($progressCount / $totalCount) * 100)
    }
    [PSCustomObject]@{
        UserPrincipalName = $_.userPrincipalName
        AppDisplayName    = $_.appDisplayName
        IPAddress         = $_.ipAddress
        Location          = (($_.location.city, $_.location.state, $_.location.countryOrRegion) -join ", ").Trim(', ')
        SignInTime        = $_.createdDateTime
        Status            = if ($_.status.errorCode -eq 0) { 'Success' } else { 'Failure' }
        RiskLevel         = $_.riskLevelAggregated
        CorrelationId     = $_.correlationId
    }
}
Write-Progress -Activity "Processing sign-in logs" -Completed

# Apply username filter if specified
if (-not [string]::IsNullOrWhiteSpace($usernameFilter)) {
    $results = $results | Where-Object { $_.UserPrincipalName -like "*$usernameFilter*" }
}

if ($results.Count -eq 0) {
    Write-Warning "No sign-in logs match the username filter."
    Disconnect-MgGraph
    exit
}

# Export CSV
$csvPath = Join-Path $PWD "LegacyAzureAD_SignIns_Report.csv"
try {
    $results | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
    Write-Information "CSV report saved to: $csvPath"
}
catch {
    Write-Error "Failed to export CSV file: $_"
}

# Export HTML with sorting functionality
$htmlPath = Join-Path $PWD "LegacyAzureAD_SignIns_Report.html"

$htmlHeader = @"
<style>
    table {border-width: 1px; border-style: solid; border-color: black; border-collapse: collapse;}
    th {border-width: 1px; padding: 5px; border-style: solid; border-color: black; background-color: #0078D4; color: white; cursor: pointer;}
    td {border-width: 1px; padding: 5px; border-style: solid; border-color: black;}
    .sorted-asc::after { content: " ▲"; }
    .sorted-desc::after { content: " ▼"; }
    tr:nth-child(even) {background-color: #f2f2f2;}
    tr:hover {background-color: #ddd;}
</style>
<script>
function sortTable(n) {
  var table, rows, switching, i, x, y, shouldSwitch, dir, switchcount = 0;
  table = document.getElementsByTagName("table")[0];
  switching = true;
  // Set the sorting direction to ascending:
  dir = "asc";
  
  // Remove sort indicators from all headers
  var headers = table.getElementsByTagName("th");
  for (i = 0; i < headers.length; i++) {
    headers[i].classList.remove("sorted-asc", "sorted-desc");
  }
  
  /* Make a loop that will continue until no switching has been done: */
  while (switching) {
    // Start by saying: no switching is done:
    switching = false;
    rows = table.rows;
    /* Loop through all table rows (except the first, which contains table headers): */
    for (i = 1; i < (rows.length - 1); i++) {
      // Start by saying there should be no switching:
      shouldSwitch = false;
      /* Get the two elements you want to compare, one from current row and one from the next: */
      x = rows[i].getElementsByTagName("TD")[n];
      y = rows[i + 1].getElementsByTagName("TD")[n];
      
      // Check if the two rows should switch place:
      if (dir == "asc") {
        if (isDate(x.innerHTML) && isDate(y.innerHTML)) {
          // Date comparison
          if (new Date(x.innerHTML) > new Date(y.innerHTML)) {
            shouldSwitch = true;
            break;
          }
        } else {
          if (x.innerHTML.toLowerCase() > y.innerHTML.toLowerCase()) {
            shouldSwitch = true;
            break;
          }
        }
      } else if (dir == "desc") {
        if (isDate(x.innerHTML) && isDate(y.innerHTML)) {
          // Date comparison
          if (new Date(x.innerHTML) < new Date(y.innerHTML)) {
            shouldSwitch = true;
            break;
          }
        } else {
          if (x.innerHTML.toLowerCase() < y.innerHTML.toLowerCase()) {
            shouldSwitch = true;
            break;
          }
        }
      }
    }
    if (shouldSwitch) {
      /* If a switch has been marked, make the switch and mark that a switch has been done: */
      rows[i].parentNode.insertBefore(rows[i + 1], rows[i]);
      switching = true;
      // Each time a switch is done, increase this count by 1:
      switchcount ++;
    } else {
      /* If no switching has been done AND the direction is "asc", set the direction to "desc" and run the while loop again. */
      if (switchcount == 0 && dir == "asc") {
        dir = "desc";
        switching = true;
      }
    }
  }
  
  // Add sort indicator to the clicked header
  if (dir === "asc") {
    headers[n].classList.add("sorted-asc");
  } else {
    headers[n].classList.add("sorted-desc");
  }
}

// Helper function to detect dates
function isDate(value) {
  // Simple date detection for common formats
  const datePattern = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}|^\d{1,2}\/\d{1,2}\/\d{4}/;
  return datePattern.test(value);
}
</script>
<h1>Legacy Azure AD PowerShell Sign-In Report</h1>
<p>Report generated: $(Get-Date -Format "yyyy-MM-dd HH:mm")</p>
<p>Date range: $($startDate.ToString('yyyy-MM-dd')) to $($endDate.ToString('yyyy-MM-dd'))</p>
"@

# Generate HTML with clickable headers
$preContent = $htmlHeader
$postContent = "<p><em>Click on column headers to sort</em></p>"

$html = $results | ConvertTo-Html -Property UserPrincipalName, AppDisplayName, IPAddress, Location, SignInTime, Status, RiskLevel, CorrelationId -PreContent $preContent -PostContent $postContent

# Add onclick handlers to table headers
$html = $html -replace '<th>([A-Za-z]+)</th>', '<th onclick="sortTable($i++)">$1</th>'
# Reset the index counter for onclick handlers
$html = $html -replace '\$i\+\+', '0'
$html = $html -replace 'onclick="sortTable\(0\)">UserPrincipalName', 'onclick="sortTable(0)">User Principal Name'
$html = $html -replace 'onclick="sortTable\(1\)">AppDisplayName', 'onclick="sortTable(1)">App Display Name'
$html = $html -replace 'onclick="sortTable\(3\)">Location', 'onclick="sortTable(3)">Location'
$html = $html -replace 'onclick="sortTable\(4\)">SignInTime', 'onclick="sortTable(4)">Sign-In Time'
$html = $html -replace 'onclick="sortTable\(5\)">Status', 'onclick="sortTable(5)">Status'
$html = $html -replace 'onclick="sortTable\(6\)">RiskLevel', 'onclick="sortTable(6)">Risk Level'
$html = $html -replace 'onclick="sortTable\(7\)">CorrelationId', 'onclick="sortTable(7)">Correlation ID'
$html = $html -replace 'onclick="sortTable\(2\)">IPAddress', 'onclick="sortTable(2)">IP Address'

# Save the HTML file
$html | Set-Content -Path $htmlPath

Write-Information "HTML report saved to: $htmlPath"

# Open HTML report automatically
Start-Process $htmlPath

```

## Final Recommendations

- Identify remaining users/scripts using the legacy modules now.
- Begin migrating those to **Microsoft Graph PowerShell**.
- Use sign-in log reports to monitor future regressions or unauthorized usage.

## Additional Resources

- [Microsoft: Retirement of AzureAD and MSOnline modules](https://learn.microsoft.com/powershell/azuread/)
- [Microsoft Graph PowerShell SDK](https://learn.microsoft.com/powershell/microsoftgraph/)
