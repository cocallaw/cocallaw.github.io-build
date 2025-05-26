---
title: "Export BitLocker Keys from Entra with Graph PowerShell"
date: 2025-05-12 05:00:00 -500
categories: [PowerShell, Azure, Microsoft Graph]
tags: [PowerShell, Azure, CrowdStrike, BitLocker, Entra, Microsoft Graph]
author: CLC
description: How to retrieve BitLocker recovery keys from Entra using PowerShell and Microsoft Graph.
toc: true
comments: true
media_subpath: /post-content/03-blkeysentra
---

On the day of the [CrowdStrike outage](https://en.wikipedia.org/wiki/2024_CrowdStrike-related_IT_outages), countless Windows devices across the world became unstable—many of them failing to boot or respond. For some machines administrators needed one critical thing to be able to access the disk and attempt repair: the BitLocker recovery key.

If the Windows machine was registered with Intune/Entra, these recovery keys were likely stored securely in your tenant. Working with different customers and the help of PowerShell and Microsoft Graph, a script was created over that weekend that was able to retrieve BitLocker Keys quickly and at scale so that they could be provided to whatever the next step was in the recovery process.

The Script: [cocallaw/Entra-Get-BLKeys](https://github.com/cocallaw/Entra-Get-BLKeys)

---

## The Problem

- Windows machines were unable to boot due to a bad kernel driver.
- To recovery, admins needed to access the disk and remove the problematic driver.
- If BitLocker encryption was enabled on the OS disk, the machine would require a recovery key to unlock the drive.
- Admins needed the recovery keys for numerous machines, but they were not easily accessible at scale.

---

## The Solution: PowerShell + Microsoft Graph

The following PowerShell script automates the retrieval of BitLocker recovery keys for a list of machine names. It queries Microsoft Graph to find devices by name, then pulls their associated keys.

### Pre-req:

- The machine must be enrolled in Intune/Entra
- The account connecting must have `Device.Read.All` and `BitLockerKey.Read.All` permissions in Microsoft Graph.
- A CSV file with the one column named `MachineName` and the names of machines in the column.

---

## Running the Script

1. Save the script as `Get-BitLockerKeys.ps1`.
1. Open PowerShell and run the script with the path to your CSV file as the parameter `-MachinesCSV`.
1. The script will output a CSV file named `DeviceRecoveryKeys.csv`.

{% raw %}

```powershell
.\Get-BitLockerKeys.ps1 -MachinesCSV "C:\path\to\your\machines.csv"
```

{% endraw %}

---

## Section 1: Input Validation

The first part ensures that the file provided by the `MachinesCSV` parameter exists, is a `.csv`, and contains the expected header.

{% raw %}

```powershell
Param (
    [string]$MachinesCSV
)

if (-not (Test-Path -Path $MachinesCSV)) {
    Write-Error "The file '$MachinesCSV' does not exist. Please provide a valid file path."
    exit
}

if (-not ($MachinesCSV -like "*.csv")) {
    Write-Error "The file '$MachinesCSV' is not a CSV file."
    exit
}

$csvContent = Import-Csv -Path $MachinesCSV
if (-not $csvContent[0].PSObject.Properties.Name -contains "MachineName") {
    Write-Error "The CSV must contain a 'MachineName' column."
    exit
}
```

{% endraw %}

---

## Section 2: Module Check & Authentication

This section checks that the Microsoft Graph module is available and current, then connects with the appropriate scopes.

{% raw %}

```powershell
$module = Get-Module -ListAvailable -Name Microsoft.Graph
if (-not $module) {
    Write-Error "Microsoft Graph PowerShell module is not installed."
    exit
} elseif ($module.Version -lt [Version]"2.5.0") {
    Write-Error "Please update the Microsoft Graph module."
    exit
}

try {
    Connect-MgGraph -Scopes "Device.Read.All", "BitLockerKey.Read.All" -NoWelcome
} catch {
    Write-Error "Failed to connect to Microsoft Graph."
    exit
}

Import-Module Microsoft.Graph.Identity.DirectoryManagement -Force
```

{% endraw %}

---

## Section 3: Device Lookup

Once connected this part queries Microsoft Graph to find matching device records by names as listed in the CSV file provided.

{% raw %}

```powershell
$vmNames = (Import-Csv -Path $MachinesCSV).MachineName
$devices = @()

foreach ($vmName in $vmNames) {
    $devices += Get-MgDevice -Filter "displayName eq '$vmName'"
}
```

{% endraw %}

---

## Section 4: Retrieve BitLocker Keys

With the list of machines stored in `$devices`, we pull recovery keys per device and if no recovery key info is found set `NoKeyFound` as the value for the Recovery Key and ID.

{% raw %}

```powershell
$deviceRecoveryKeys = @()

foreach ($device in $devices) {
    $recoveryKeys = Get-MgInformationProtectionBitlockerRecoveryKey -Filter "deviceId eq '$($device.DeviceId)'"
    if ($recoveryKeys) {
        foreach ($key in $recoveryKeys) {
            $deviceRecoveryKeys += [PSCustomObject]@{
                DeviceName    = $device.DisplayName
                DeviceId      = $device.DeviceId
                RecoveryKeyId = $key.Id
                RecoveryKey   = (Get-MgInformationProtectionBitlockerRecoveryKey -BitlockerRecoveryKeyId $key.Id -Property key).key
            }
        }
    } else {
        $deviceRecoveryKeys += [PSCustomObject]@{
            DeviceName    = $device.DisplayName
            DeviceId      = $device.DeviceId
            RecoveryKeyId = "NoKeyFound"
            RecoveryKey   = "NoKeyFound"
        }
    }
}
```

{% endraw %}

---

## Section 5: Export to CSV

Finally, the key information is output to a CSV file named `DeviceRecoveryKeys.csv` and displays the path.

{% raw %}

```powershell
$outputFilePath = "DeviceRecoveryKeys.csv"
$deviceRecoveryKeys | Export-Csv -Path $outputFilePath -NoTypeInformation

$absolutePath = Resolve-Path -Path $outputFilePath
Write-Output "Device recovery keys have been exported to '$absolutePath'."
```

{% endraw %}

## Section 6: Sample Output

The output CSV `DeviceRecoveryKeys.csv` will follow this format:

{% raw %}

```csv
"DeviceName","DeviceId","RecoveryKeyId","RecoveryKey"
"test-vm-0","90e61314-d6dd-4ff8-a878-754b5b60a8be","11135913-8395-458b-a7bf-dd7ef10214a2","375177-019569-186362-280841-610973-240801-321277-046255"
"test-vm-1","c7165ba4-0fac-4ec5-82bf-5e4ff05dcdc8","NoKeyFound","NoKeyFound"
```

{% endraw %}

---

## Full Script

For easy copy/pasting, the entire script is below in one block. You can also find it in the GitHub Repo [cocallaw/Entra-Get-BLKeys](https://github.com/cocallaw/Entra-Get-BLKeys).

{% raw %}

```powershell
Param (
    [string]$MachinesCSV
)

if (-not (Test-Path -Path $MachinesCSV)) {
    Write-Error "The file '$MachinesCSV' does not exist. Please provide a valid file path."
    exit
}

if (-not ($MachinesCSV -like "*.csv")) {
    Write-Error "The file '$MachinesCSV' is not a CSV file."
    exit
}

$csvContent = Import-Csv -Path $MachinesCSV
if (-not $csvContent[0].PSObject.Properties.Name -contains "MachineName") {
    Write-Error "The CSV must contain a 'MachineName' column."
    exit
}

$module = Get-Module -ListAvailable -Name Microsoft.Graph
if (-not $module) {
    Write-Error "Microsoft Graph PowerShell module is not installed."
    exit
} elseif ($module.Version -lt [Version]"2.5.0") {
    Write-Error "Please update the Microsoft Graph module."
    exit
}

try {
    Connect-MgGraph -Scopes "Device.Read.All", "BitLockerKey.Read.All" -NoWelcome
} catch {
    Write-Error "Failed to connect to Microsoft Graph."
    exit
}

Import-Module Microsoft.Graph.Identity.DirectoryManagement -Force

$vmNames = (Import-Csv -Path $MachinesCSV).MachineName
$devices = @()

foreach ($vmName in $vmNames) {
    $devices += Get-MgDevice -Filter "displayName eq '$vmName'"
}

$deviceRecoveryKeys = @()

foreach ($device in $devices) {
    $recoveryKeys = Get-MgInformationProtectionBitlockerRecoveryKey -Filter "deviceId eq '$($device.DeviceId)'"
    if ($recoveryKeys) {
        foreach ($key in $recoveryKeys) {
            $deviceRecoveryKeys += [PSCustomObject]@{
                DeviceName    = $device.DisplayName
                DeviceId      = $device.DeviceId
                RecoveryKeyId = $key.Id
                RecoveryKey   = (Get-MgInformationProtectionBitlockerRecoveryKey -BitlockerRecoveryKeyId $key.Id -Property key).key
            }
        }
    } else {
        $deviceRecoveryKeys += [PSCustomObject]@{
            DeviceName    = $device.DisplayName
            DeviceId      = $device.DeviceId
            RecoveryKeyId = "NoKeyFound"
            RecoveryKey   = "NoKeyFound"
        }
    }
}

$outputFilePath = "DeviceRecoveryKeys.csv"
$deviceRecoveryKeys | Export-Csv -Path $outputFilePath -NoTypeInformation
$absolutePath = Resolve-Path -Path $outputFilePath
Write-Output "Device recovery keys have been exported to '$absolutePath'."
```

{% endraw %}

---

## Permissions Reminder

This script requires delegated or app permissions for:

- `Device.Read.All`
- `BitLockerKey.Read.All`

---

## Notes

- Recovery keys are only available if the device was properly enrolled and reporting compliance.
- Permissions must be delegated or granted to the appropriate Graph scopes
- The script exports the keys in plaintext, be cautious with the output file or update the script to output in a more secure manner
- This script is designed for educational purposes and should be tested in a safe environment before use in production.

## Final Thoughts

This was a good reminder that recovery isn’t just about backups—it’s also about access. PowerShell and Microsoft Graph made it possible to recover quickly during the CrowdStrike outage by tapping into the organization’s existing processes and tooling.

The script is a great example of how to leverage existing tools to solve real-world problems. If you have any questions or need help with the script, feel free to reach out.
