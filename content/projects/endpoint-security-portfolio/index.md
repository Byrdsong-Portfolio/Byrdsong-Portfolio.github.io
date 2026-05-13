---
title: "Endpoint Security Portfolio"
date: 2026-05-13
draft: false
summary: "A collection of endpoint security work: a PowerShell compliance auditor, an IAM-to-MDM integration analysis, a Tier 1 hardening checklist, and an M365/Intune home lab."
tags:
  - PowerShell
  - Windows
  - macOS
  - Intune
  - MDM
  - IAM
  - Azure AD
  - Endpoint Security
  - Compliance
  - Automation
  - Home Lab
links:
  - type: site
    url: https://github.com/Byrdsong-Portfolio/windows-endpoint-auditor
---

## Windows Endpoint Compliance Auditor

`Invoke-LocalUserAudit.ps1` audits all local Windows user accounts using `Get-LocalUser`, calculates account staleness from last logon date and password age, and exports a timestamped CSV report. It flags accounts that are disabled, have no password set date, or have been inactive for 90 or more days, producing reproducible output that satisfies third-party audit requirements.

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

if (-not (Test-Path $OutputPath)) {
    New-Item -ItemType Directory -Path $OutputPath -Force | Out-Null
}

$timestamp   = Get-Date -Format "yyyyMMdd-HHmmss"
$reportFile  = Join-Path $OutputPath "LocalUserAudit_$timestamp.csv"
$staleDays   = 90

Write-Host "Starting local user audit on $env:COMPUTERNAME..."

$results = Get-LocalUser | ForEach-Object {
    $user = $_

    $passwordAge = $null
    if ($user.PasswordLastSet) {
        $passwordAge = [math]::Floor((New-TimeSpan -Start $user.PasswordLastSet -End (Get-Date)).TotalDays)
    }

    $daysSinceLogon = $null
    if ($user.LastLogon) {
        $daysSinceLogon = [math]::Floor((New-TimeSpan -Start $user.LastLogon -End (Get-Date)).TotalDays)
    }

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
        LastLogon         = if ($user.LastLogon)        { $user.LastLogon.ToString("yyyy-MM-dd")        } else { "Never" }
        DaysSinceLogon    = if ($null -ne $daysSinceLogon) { $daysSinceLogon }                           else { "N/A"   }
        PasswordLastSet   = if ($user.PasswordLastSet)  { $user.PasswordLastSet.ToString("yyyy-MM-dd")  } else { "Never" }
        PasswordAgeDays   = if ($null -ne $passwordAge)  { $passwordAge }                                else { "N/A"   }
        PasswordExpires   = $user.PasswordExpires
        AccountExpires    = if ($user.AccountExpires)   { $user.AccountExpires.ToString("yyyy-MM-dd")   } else { "Never" }
        StaleFlag         = $staleFlag
        AuditTimestamp    = (Get-Date -Format "yyyy-MM-dd HH:mm:ss")
    }
}

$results | Export-Csv -Path $reportFile -NoTypeInformation -Encoding UTF8

$flaggedCount = ($results | Where-Object { $_.StaleFlag }).Count
Write-Host ""
Write-Host "Audit complete."
Write-Host "  Total accounts audited : $($results.Count)"
Write-Host "  Flagged (stale/disabled): $flaggedCount"
Write-Host "  Report saved to        : $reportFile"
Write-Host ""

$results | Format-Table ComputerName, UserName, Enabled, DaysSinceLogon, PasswordAgeDays, StaleFlag -AutoSize
```

Run from an elevated PowerShell session with `.\Invoke-LocalUserAudit.ps1` (outputs to Desktop by default) or pass `-OutputPath "C:\ComplianceReports"` for a custom location. The script requires no external modules. The 90-day staleness threshold aligns with CIS Benchmark account hygiene controls: stale and orphaned local accounts are a common lateral movement vector, and the timestamped CSV creates defensible, reproducible documentation.

---

## IAM to MDM: The Bridge Most IT Teams Miss

MDM without IAM is an incomplete security posture. A fully enrolled, policy-compliant device can still be compromised through credential theft, because device compliance does not validate the identity behind the session. Azure AD Conditional Access bridges this gap by requiring a device to be both Azure AD joined and marked compliant by Intune before granting access to protected applications. Intune compliance policies evaluate BitLocker status, OS version, password requirements, Defender state, and Secure Boot, then propagate that status to Azure AD within minutes. A non-compliant device loses access to Exchange Online, SharePoint, and Teams automatically. The correct architecture treats device trust and identity trust as a single enforcement layer: a correct credential from a non-compliant device gets blocked, and a compliant device with a compromised identity triggers Conditional Access risk controls before access is granted.

---

## Tier 1 Endpoint Hardening Checklist

Baseline controls for Windows, macOS, and mobile devices before any endpoint touches a corporate network.

### Windows

- [ ] BitLocker enabled on OS drive; recovery key escrowed to Azure AD
- [ ] Password complexity enforced via MDM or Group Policy (12+ characters minimum)
- [ ] Screen lock timeout 5 minutes or less; password required on resume
- [ ] Local administrator accounts audited; unknown accounts removed
- [ ] Guest account and default Administrator account disabled or renamed
- [ ] Windows Update current within 30 days; automatic updates enabled
- [ ] Microsoft Defender real-time protection active; definitions current
- [ ] Device enrolled in Intune and reporting Compliant status

### macOS

- [ ] FileVault enabled; recovery key escrowed to MDM (not stored locally)
- [ ] Screen lock requires password immediately on sleep or screensaver
- [ ] Automatic login disabled
- [ ] Remote Login (SSH) and Screen Sharing disabled unless explicitly required
- [ ] Software Update current; automatic security response updates enabled
- [ ] MDM enrollment profile present and set non-removable (supervised)
- [ ] Gatekeeper set to App Store and identified developers

### Mobile (iOS and Android)

- [ ] Screen lock with biometric or 6-digit PIN minimum; swipe-only unlock not acceptable
- [ ] Auto-lock timeout 5 minutes or less, enforced by MDM compliance policy
- [ ] Failed attempt wipe policy active (10 failed attempts maximum)
- [ ] OS version current or within approved baseline; automatic updates enabled
- [ ] Device enrolled in MDM (corporate) or MAM profile applied (BYOD)
- [ ] Remote wipe capability confirmed in MDM console

---

## M365 Endpoint Management Lab

Using a free Microsoft 365 Developer Program tenant (25 E5 licenses, 90-day renewable), I built a working Intune environment from scratch: enrolled a Windows 11 test device via Azure AD join with MDM auto-enrollment, configured a Windows compliance policy, and deployed a Conditional Access policy requiring device compliance for access to all M365 apps. I verified the full enforcement loop by disabling BitLocker on the enrolled device, confirming non-compliant status propagated from Intune to Azure AD within 15 minutes, and verifying that Conditional Access blocked M365 sign-in until compliance was restored.

**What was configured:**

- Windows compliance policy: BitLocker required, minimum OS Windows 11 22H2, 12-character password, 5-minute screen lock, Defender real-time protection required
- Conditional Access policy: all users, Office 365 target apps, grant access to compliant devices only; validated in report-only mode before enabling enforcement
- MDM auto-enrollment: scope set to All in Entra Devices settings so any Azure AD join triggers automatic Intune enrollment
- Device enrollment restrictions: 5-device limit per user, Windows and iOS platforms only, minimum OS version enforced at enrollment time

This lab demonstrates hands-on Intune administration including compliance policy design, Conditional Access testing, and device enrollment configuration in a real M365 tenant.

*[Screenshot placeholder: Intune device detail showing compliant and non-compliant status after BitLocker change]*

*[Screenshot placeholder: Conditional Access block page after device failed compliance evaluation]*
