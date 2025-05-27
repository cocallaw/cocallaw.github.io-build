---
title: "Identify AzureAD PowerShell Users in your Tenant before Retirement"
date: 2025-05-27 12:00:00 -500
categories: [Azure, PowerShell, Entra]
tags: [AzureAD, Microsoft Graph, PowerShell, Entra ID, Reporting, Security]
author: CLC
description: 
toc: true
comments: true
media_subpath: /post-content/05-aadpsrpttool
---

Microsoft is retiring the legacy **AzureAD** and **MSOnline** PowerShell modules. These modules have been widely used for years to manage users, groups, roles, and other directory objects in Azure Active Directory (now Entra ID). With their deprecation, it’s critical to identify any users or automation that still rely on them — and plan a migration.

In this post, I’ll walk you through how to:

- Identify sign-ins using the AzureAD PowerShell App ID with Kusto Query Language (KQL)
- Use a PowerShell script to query sign-in logs and generate reports
- Export and report this data via PowerShell in CSV and HTML formats

## What's Being Retired?

The following modules are being deprecated:

- `AzureAD` (`AzureAD.Standard.Preview` as well)
- `MSOnline`

These modules authenticate using the app registration (`1b730954-1685-4b74-9bfd-dac224a7b894`) that shows up in sign-in logs. Depending on the use case, customers can use either [Microsoft Graph PowerShell](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview?view=graph-powershell-1.0) or [Microsoft Entra PowerShell](https://learn.microsoft.com/en-us/powershell/entra-powershell/overview?view=entra-powershell) replacement.

## Option 1: Find AzureAD PowerShell Sign-In Activity with KQL

You can use Kusto Query Language (KQL) to identify sign-ins tied to the legacy AzureAD app in your Entra ID tenant. This is useful for quickly finding users or scripts that still rely on the AzureAD PowerShell module.

If you have your sign-in logs being sent to a **Log Analytics Workspace** or **Microsoft Sentinel** you can use this Kusto query as a starting point to filter sign-in logs for the AzureAD PowerShell app ID and return relevant details.

### KQL Query

```sql
SigninLogs
| where AppId == "1b730954-1685-4b74-9bfd-dac224a7b894" // AzureAD PowerShell app ID
| project UserPrincipalName, IPAddress, ResourceDisplayName, AuthenticationRequirement, ConditionalAccessStatus, Status, CreatedDateTime
| order by CreatedDateTime desc
```

## Option 2: Use a PowerShell Script to Query Logs and Generate Reports

If you prefer a more automated approach, you can use a PowerShell script to query the sign-in logs for the AzureAD PowerShell app and generate reports in both CSV and HTML formats. This script will help you identify users that still rely on the legacy AzureAD module and easily filter. 

The full script is available on GitHub at [cocallaw/Entra-AAD-PS-Retirement](https://github.com/cocallaw/Entra-AAD-PS-Retirement/)

### Prerequisites

#### PowerShell Core

Ensure you have PowerShell Core installed on your machine. This script is designed to run on [PowerShell Core (7.x)](https://learn.microsoft.com/powershell/scripting/install/installing-powershell) for cross-platform compatibility. This script has been tested on Windows and MacOS only.

#### Microsoft Graph PowerShell

This script utilizes the [Microsoft Graph PowerShell SDK](https://learn.microsoft.com/powershell/microsoftgraph/installation) in order to retrieve logs from the tenant. If you haven't done so already, install it using the following command:

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

#### Sample Output

#### CSV Report

The script generates a CSV file with the following format:
```csv
"UserPrincipalName","AppDisplayName","IPAddress","Location","SignInTime","Status","RiskLevel","CorrelationId"
"jane.doe@contoso.com","Azure Active Directory PowerShell","2605:<IPV6-Address>:40bd","Charlotte, North Carolina, US","5/26/2025 8:46:59 PM","Success","none","0d3b4be7-ce67-4adf-a8c7-506078f9a013"
"john.doe@contoso.com","Azure Active Directory PowerShell","XXX.YYY.ZZZ.VVV","Charlotte, North Carolina, US","5/26/2025 8:15:56 PM","Success","none","932246df-d54b-4b4d-94ec-d29427766b11"
"john.doe@contoso.com","Azure Active Directory PowerShell","XXX.YYY.ZZZ.VVV","Charlotte, North Carolina, US","5/26/2025 8:15:24 PM","Success","none","a9f18460-db0a-4ab6-8d6b-cc0f8c468a5a"
"john.doe@contoso.com","Azure Active Directory PowerShell","XXX.YYY.ZZZ.VVV","Charlotte, North Carolina, US","5/20/2025 5:34:50 PM","Success","none","279e3fe5-34ca-4bbf-a208-906d8f25c90c"
```

#### HTML Report

The script generates an interactive HTML report with sortable columns, which can be opened in any web browser. The HTML report includes the same data as the CSV but is formatted for easier viewing and analysis.

![Sample-HTML-Report](/sample-html-report.png)

#### Key Components

##### **Prerequisites & Authentication**

- Validates Microsoft Graph PowerShell module is installed and loaded
- Connects to Microsoft Graph with required permissions
- Handles existing connections intelligently

   ```powershell
   # Check if Microsoft Graph commands are available
   if (-not (Get-Command Get-MgContext -ErrorAction SilentlyContinue)) {
       # Only then check if module is installed
       if (-not (Get-Module -ListAvailable -Name Microsoft.Graph)) {
           $WarningPreference = 'Continue' 
           Write-Warning "Microsoft.Graph module is not installed. Please install it using: Install-Module Microsoft.Graph -Scope CurrentUser"
           exit
       }
       # Try to load the module silently
       Import-Module Microsoft.Graph.Authentication -DisableNameChecking -Force -ErrorAction SilentlyContinue
   }
   
   # Connect with required permissions
   $requiredScopes = @("AuditLog.Read.All", "Directory.Read.All", "IdentityRiskEvent.Read.All")
   if (-not (Get-MgContext)) {
       Connect-MgGraph -Scopes $requiredScopes
   }
   ```

##### **Date Range Input**

- Prompts user for start and end dates, defaulting to the last 30 days
- Converts dates to proper format for Graph API queries

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
    
    # Prompt date range, defaulting to 30 days
    $startDate = Get-ValidatedDate "Enter start date (yyyy-MM-dd), or press Enter for 30 days ago" (Get-Date).AddDays(-30)
    $endDate = Get-ValidatedDate "Enter end date (yyyy-MM-dd), or press Enter for today" (Get-Date)
    
    # Format dates for Graph API
    $startDateUTC = [datetime]::Parse($startDate).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
    $endDateUTC = ([datetime]::Parse($endDate).AddDays(1).AddSeconds(-1)).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
  ```

##### **Date Collection**

- Queries Microsoft Graph API for sign-in logs with the AzureAD PowerShell AppID
- Handles pagination for large result sets

    ```powershell
    $aadAppId = '1b730954-1685-4b74-9bfd-dac224a7b894'
    $filter = "appId eq '$aadAppId' and createdDateTime ge $startDateUTC and createdDateTime le $endDateUTC"
    $uri = "https://graph.microsoft.com/v1.0/auditLogs/signIns?`$filter=$( [uri]::EscapeDataString($filter) )"
    
    Write-Host "Retrieving sign-in logs for appId '$aadAppId' from $startDate to $endDate"
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
    ```

##### **Data Processing**

- Extracts relevant details from each sign-in (user, IP, location, status)
- Supports optional username filtering
- Formats data for reporting

    ```powershell
    $progressCount = 0
    $totalCount = $allSignIns.Count
    $results = $allSignIns | ForEach-Object {
        $progressCount++
        # Display progress every 100 items
        if ($progressCount % 100 -eq 0) {
            $percentComplete = [math]::Round(($progressCount / $totalCount) * 100, 1)
            Write-Host "Processing: $progressCount of $totalCount ($percentComplete%)" -ForegroundColor Cyan
        }
        # Create PSCustomObject for each sign-in with the relevant properties
        [PSCustomObject]@{
            # Extracted properties such as UserPrincipalName, CorrelationId
        }
    }
    
    # Apply username filter if specified
    if (-not [string]::IsNullOrWhiteSpace($usernameFilter)) {
        $results = $results | Where-Object { $_.UserPrincipalName -like "*$usernameFilter*" }
    }
    ```

##### **Report Generation**

- Creates a CSV file for data analysis
- Generates an interactive HTML report with sortable columns
- Automatically opens the report based on your operating system

    ```powershell
    # Export CSV
    $csvPath = Join-Path $PWD "LegacyAzureAD_SignIns_Report.csv"
    $results | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
    
    # Export HTML with interactive features
    $htmlPath = Join-Path $PWD "LegacyAzureAD_SignIns_Report.html"
    
    # HTML header with sorting JavaScript
    $htmlHeader = @"<style>
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
      // Sorting logic for table columns
      // ...
    }
    </script>
    "@
    
    # Open HTML report based on platform
    try {
        if ($IsWindows -or $env:OS -match 'Windows') {
            Start-Process $htmlPath
        }
        elseif ($IsMacOS -or (uname) -eq 'Darwin') {
            Invoke-Expression "open '$htmlPath'"
        }
        elseif ($IsLinux -or (uname) -eq 'Linux') {
            Invoke-Expression "xdg-open '$htmlPath'"
        }
    }
    catch {
        Write-Warning "Could not automatically open the report: $_"
    }
    ```

## The Full PowerShell Script

Here’s the complete PowerShell script, you can copy and save it or find the current version on GitHub at [cocallaw/Entra-AAD-PS-Retirement](https://github.com/cocallaw/Entra-AAD-PS-Retirement/).

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
$WarningPreference = 'SilentlyContinue' # Suppress warnings globally

Write-Host "Starting Legacy Azure AD PowerShell Sign-In Report script"
Write-Host "Checking prerequisites"

# Check if Microsoft Graph commands are available
if (-not (Get-Command Get-MgContext -ErrorAction SilentlyContinue)) {
    # Only then check if module is installed
    if (-not (Get-Module -ListAvailable -Name Microsoft.Graph)) {
        $WarningPreference = 'Continue' # Temporarily restore warnings for important messages
        Write-Warning "Microsoft.Graph module is not installed. Please install it using: Install-Module Microsoft.Graph -Scope CurrentUser"
        exit
    }
    # Module is installed but commands aren't available, so try to load it silently
    $oldWarningPreference = $WarningPreference
    $WarningPreference = 'SilentlyContinue' # Suppress warnings during module import
    try {
        # Try minimal import
        Import-Module Microsoft.Graph.Authentication -DisableNameChecking -Force -ErrorAction SilentlyContinue
        if (-not (Get-Command Get-MgContext -ErrorAction SilentlyContinue)) {
            # Try full module import with all error suppression flags
            Import-Module Microsoft.Graph -SkipEditionCheck -DisableNameChecking -Force -ErrorAction SilentlyContinue
        }
    }
    catch {
        # Silently ignore errors - we'll check command availability instead
    }
    $WarningPreference = $oldWarningPreference
    # Verify everything loaded correctly
    if (-not (Get-Command Get-MgContext -ErrorAction SilentlyContinue)) {
        Write-Error "Failed to load Microsoft Graph commands. Please start with a fresh PowerShell session."
        exit 1
    }
}
Write-Host "Microsoft Graph commands are available"

$requiredScopes = @("AuditLog.Read.All", "Directory.Read.All", "IdentityRiskEvent.Read.All")
$needConnect = $false
if (-not (Get-MgContext)) {
    Write-Host "Connecting to Microsoft Graph..."
    $needConnect = $true
}
else {
    $currentScopes = (Get-MgContext).Scopes
    if (-not $currentScopes -or ($requiredScopes | Where-Object { $_ -notin $currentScopes })) {
        Write-Host "Existing session missing required scopes. Reconnecting..."
        $needConnect = $true
    }
    else {
        Write-Host "Already connected to Microsoft Graph. Reusing existing session."
    }
}
if ($needConnect) {
    Connect-MgGraph -Scopes $requiredScopes
}

# Prompt date range, defaulting to 30 days
$startDate = Get-ValidatedDate "Enter start date (yyyy-MM-dd), or press Enter for 30 days ago" (Get-Date).AddDays(-30)
$endDate = Get-ValidatedDate "Enter end date (yyyy-MM-dd), or press Enter for today" (Get-Date)
# Prompt username filter (optional)
$usernameFilter = Read-Host "Enter username filter (optional), or press Enter to skip"
$aadAppId = '1b730954-1685-4b74-9bfd-dac224a7b894'

$startDateUTC = [datetime]::Parse($startDate).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$endDateUTC = ([datetime]::Parse($endDate).AddDays(1).AddSeconds(-1)).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

$filter = "appId eq '$aadAppId' and createdDateTime ge $startDateUTC and createdDateTime le $endDateUTC"
$uri = "https://graph.microsoft.com/v1.0/auditLogs/signIns?`$filter=$( [uri]::EscapeDataString($filter) )"

Write-Host "Retrieving sign-in logs for appId '$aadAppId' from $startDate to $endDate"
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
Write-Host "Retrieved $($allSignIns.Count) sign-in logs for appId '$aadAppId' from $startDate to $endDate"
Write-Host "Processing sign-in logs"
$progressCount = 0
$totalCount = $allSignIns.Count
$results = $allSignIns | ForEach-Object {
    $progressCount++
    # Display progress every 100 items
    if ($progressCount % 100 -eq 0) {
        $percentComplete = [math]::Round(($progressCount / $totalCount) * 100, 1)
        Write-Host "Processing: $progressCount of $totalCount ($percentComplete%)" -ForegroundColor Cyan
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
Write-Host "Processing complete: $totalCount items processed" -ForegroundColor Green

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
    Write-Host "CSV report saved to: $csvPath"
}
catch {
    Write-Error "Failed to export CSV file: $_"
}

# Export HTML
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
<title>Legacy Azure AD PowerShell Sign-In Report - $($startDate.ToString('yyyy-MM-dd')) to $($endDate.ToString('yyyy-MM-dd'))</title>
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
    switching = false;
    rows = table.rows;
    /* Loop through all table rows (except the first, which contains table headers): */
    for (i = 1; i < (rows.length - 1); i++) {
      shouldSwitch = false;
      /* Get the two elements you want to compare, one from current row and one from the next: */
      x = rows[i].getElementsByTagName("TD")[n];
      y = rows[i + 1].getElementsByTagName("TD")[n];
      
      if (!x || !y) continue;
      
      // Check if the two rows should switch place:
      if (dir == "asc") {
        if (isDate(x.innerHTML) && isDate(y.innerHTML)) {
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
      rows[i].parentNode.insertBefore(rows[i + 1], rows[i]);
      switching = true;
      switchcount++;
    } else {
      if (switchcount == 0 && dir == "asc") {
        dir = "desc";
        switching = true;
      }
    }
  }
  
  // Add sort indicator to the clicked header
  headers[n].classList.add(dir === "asc" ? "sorted-asc" : "sorted-desc");
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

# Generate HTML without the replacement logic first
$preContent = $htmlHeader
$postContent = "<p><em>Click on column headers to sort</em></p>"

# Generate basic HTML
$html = $results | ConvertTo-Html -Property UserPrincipalName, AppDisplayName, IPAddress, Location, SignInTime, Status, RiskLevel, CorrelationId -PreContent $preContent -PostContent $postContent

# Create a more reliable method to replace the header names and add onclick handlers
$columnOrder = @('UserPrincipalName', 'AppDisplayName', 'IPAddress', 'Location', 'SignInTime', 'Status', 'RiskLevel', 'CorrelationId')
$headerDisplayNames = @{
    'UserPrincipalName' = 'User Principal Name'
    'AppDisplayName'    = 'App Display Name'
    'IPAddress'         = 'IP Address'
    'Location'          = 'Location'
    'SignInTime'        = 'Sign-In Time'
    'Status'            = 'Status'
    'RiskLevel'         = 'Risk Level'
    'CorrelationId'     = 'Correlation ID'
}

# Process the header row directly using a more reliable HTML parsing approach
for ($i = 0; $i -lt $columnOrder.Count; $i++) {
    $colName = $columnOrder[$i]
    $displayName = $headerDisplayNames[$colName]
    $oldHeader = "<th>$colName</th>"
    $newHeader = "<th onclick=""sortTable($i)"">$displayName</th>"
    $html = $html -replace [regex]::Escape($oldHeader), $newHeader
}

# Save the HTML file
$html | Set-Content -Path $htmlPath

Write-Host "HTML report saved to: $htmlPath"

# Open HTML report based on platform
try {
    if ($IsWindows -or $env:OS -match 'Windows') {
        Start-Process $htmlPath
    }
    elseif ($IsMacOS -or (uname) -eq 'Darwin') {
        Invoke-Expression "open '$htmlPath'"
    }
    elseif ($IsLinux -or (uname) -eq 'Linux') {
        Invoke-Expression "xdg-open '$htmlPath'"
    }
    else {
        Write-Information "HTML report saved to: $htmlPath (please open manually)"
    }
}
catch {
    Write-Warning "Could not automatically open the report: $_"
    Write-Host "HTML report saved to: $htmlPath (please open manually)"
}


```

## Final Recommendations

- Identify remaining users/scripts using the legacy modules now.
- Continue to use sign-in log reports to monitor usage burn down or unauthorized usage leading up to the future retirement date.
- Begin migrating those to [Microsoft Graph PowerShell](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview?view=graph-powershell-1.0) or [Microsoft Entra PowerShell](https://learn.microsoft.com/en-us/powershell/entra-powershell/overview?view=entra-powershell).


## Additional Resources

- [PowerShell Gallery: AzureAD Module](https://www.powershellgallery.com/packages/AzureAD/2.0.2.182)
- [Microsoft Graph PowerShell SDK](https://learn.microsoft.com/powershell/microsoftgraph/)
- [Microsoft Entra PowerShell](https://learn.microsoft.com/en-us/powershell/entra-powershell/overview?view=entra-powershell).
