---
title: "Windows Endpoint Compliance Auditor"
date: 2026-05-13
draft: false
summary: "A PowerShell script that audits local Windows user accounts, detects disabled or stale accounts, and exports a timestamped CSV compliance report — built for sysadmins who need defensible documentation."
links:
  - type: site
    url: https://github.com/Byrdsong-Portfolio/windows-endpoint-auditor
tags:
  - PowerShell
  - Windows
  - Endpoint Security
  - Automation
  - Compliance
  - Scripting
---

## The Problem with Manual Auditing

Every Windows environment accumulates accounts over time — service accounts that outlived the project they were created for, former employees whose profiles were never disabled, test accounts someone spun up and forgot. Manual auditing means opening Local Users and Groups, clicking through each account, noting what you see, and hoping nothing changed by the time you write the report. It is error-prone, time-consuming, and produces documentation that a third-party auditor cannot reproduce or verify.

Compliance frameworks like CIS Benchmarks, NIST 800-53, and most enterprise security baselines require regular review of local user accounts as part of account hygiene and least-privilege enforcement. The question is not whether to do it — it is whether you can prove you did it, and prove it consistently.

---

## What the Script Does

`Invoke-LocalUserAudit.ps1` runs against the local machine using `Get-LocalUser` and WMI to pull every local account's status, last logon, and password metadata. It calculates how stale each account is, applies a flag for accounts that meet stale or disabled criteria, and exports the full result to a timestamped CSV.

**What it collects per account:**

- Username and display name
- Enabled/disabled status
- Last logon date and days since last logon
- Password last set date and password age in days
- Whether the password is set to expire
- Account expiration date (if set)
- A `StaleFlag` boolean (true if disabled, no password set date, or inactive 90+ days)
- Audit timestamp and computer name for multi-machine traceability

---

## The Script

```powershell
<#
.SYNOPSIS
    Audits local Windows user accounts and exports a compliance report to CSV.

.DESCRIPTION
    Enumerates all local user accounts, checks enabled status, last logon date,
    and password age. Flags accounts that are disabled, have no password set date,
    or have not logged on in 90 or more days. Exports results to a timestamped CSV.

.PARAMETER OutputPath
    Directory where the CSV report is saved. Defaults to the current user's Desktop.

.EXAMPLE
    .\Invoke-LocalUserAudit.ps1
    .\Invoke-LocalUserAudit.ps1 -OutputPath "C:\AuditReports"

.NOTES
    Requires: Windows 10/11 or Windows Server 2016+, PowerShell 5.1+, local admin rights.
#>

param (
    [string]$OutputPath = "$env:USERPROFILE\Desktop"
)

# Verify output directory exists
if (-not (Test-Path $OutputPath)) {
    New-Item -ItemType Directory -Path $OutputPath -Force | Out-Null
}

$timestamp   = Get-Date -Format "yyyyMMdd-HHmmss"
$reportFile  = Join-Path $OutputPath "LocalUserAudit_$timestamp.csv"
$staleDays   = 90

Write-Host "Starting local user audit on $env:COMPUTERNAME..."

$results = Get-LocalUser | ForEach-Object {
    $user = $_

    # Calculate password age
    $passwordAge = $null
    if ($user.PasswordLastSet) {
        $passwordAge = [math]::Floor((New-TimeSpan -Start $user.PasswordLastSet -End (Get-Date)).TotalDays)
    }

    # Calculate days since last logon
    $daysSinceLogon = $null
    if ($user.LastLogon) {
        $daysSinceLogon = [math]::Floor((New-TimeSpan -Start $user.LastLogon -End (Get-Date)).TotalDays)
    }

    # Flag: disabled, no password date, or stale (inactive >= 90 days)
    $staleFlag = (
        (-not $user.Enabled) -or
        ($null -eq $user.PasswordLastSet) -or
        ($null -ne $daysSinceLogon -and $daysSinceLogon -ge $staleDays)
    )

    [PSCustomObject]@{
        ComputerName      = $env:COMPUTERNAME
        UserName          = $user.Name
        FullName          = $user.FullName
        Enabled           = $user.Enabled
        LastLogon         = if ($user.LastLogon)       { $user.LastLogon.ToString("yyyy-MM-dd")       } else { "Never" }
        DaysSinceLogon    = if ($null -ne $daysSinceLogon) { $daysSinceLogon }                          else { "N/A"   }
        PasswordLastSet   = if ($user.PasswordLastSet) { $user.PasswordLastSet.ToString("yyyy-MM-dd") } else { "Never" }
        PasswordAgeDays   = if ($null -ne $passwordAge) { $passwordAge }                                else { "N/A"   }
        PasswordExpires   = $user.PasswordExpires
        AccountExpires    = if ($user.AccountExpires)  { $user.AccountExpires.ToString("yyyy-MM-dd")  } else { "Never" }
        StaleFlag         = $staleFlag
        AuditTimestamp    = (Get-Date -Format "yyyy-MM-dd HH:mm:ss")
    }
}

# Export to CSV
$results | Export-Csv -Path $reportFile -NoTypeInformation -Encoding UTF8

# Console summary
$flaggedCount = ($results | Where-Object { $_.StaleFlag }).Count
Write-Host ""
Write-Host "Audit complete."
Write-Host "  Total accounts audited : $($results.Count)"
Write-Host "  Flagged (stale/disabled): $flaggedCount"
Write-Host "  Report saved to        : $reportFile"
Write-Host ""

# Print table to console for quick review
$results | Format-Table ComputerName, UserName, Enabled, DaysSinceLogon, PasswordAgeDays, StaleFlag -AutoSize
```

---

## Security Rationale

This script maps directly to two foundational security controls:

**Least Privilege** — accounts that no one uses are attack surface. A disabled account can still be re-enabled through misconfiguration or credential reuse if it was never audited. Knowing which accounts exist and which are active is the first step in enforcing least-privilege at the endpoint level.

**Account Hygiene** — stale accounts with old passwords are a vector for lateral movement. In incident response scenarios, auditors routinely trace compromised accounts back to service accounts or former-employee accounts that were never deprovisioned. Timestamped CSV exports create a paper trail that demonstrates proactive control.

The 90-day threshold for the `StaleFlag` aligns with CIS Benchmark recommendations and most enterprise password rotation policies. It is configurable via the `$staleDays` variable at the top of the script.

---

## Requirements

| Requirement | Detail |
|---|---|
| OS | Windows 10, Windows 11, or Windows Server 2016+ |
| PowerShell | 5.1 or higher |
| Permissions | Local administrator rights required |
| Modules | None — uses built-in `Microsoft.PowerShell.LocalAccounts` |

Run from an elevated PowerShell session:

```powershell
# Run with default output to Desktop
.\Invoke-LocalUserAudit.ps1

# Specify a custom output directory
.\Invoke-LocalUserAudit.ps1 -OutputPath "C:\ComplianceReports"
```

---

## Sample Output

```
ComputerName  UserName         Enabled  DaysSinceLogon  PasswordAgeDays  StaleFlag
------------  --------         -------  --------------  ---------------  ---------
DESKTOP-01    Administrator    False    N/A             N/A              True
DESKTOP-01    Isaiah           True     2               47               False
DESKTOP-01    svc_backup       True     112             438              True
DESKTOP-01    Guest            False    N/A             N/A              True
```

In this example, `svc_backup` is still enabled but has not logged on in 112 days and has a password nearly 15 months old — exactly the kind of account that gets missed in manual reviews and targeted in credential-based attacks.
