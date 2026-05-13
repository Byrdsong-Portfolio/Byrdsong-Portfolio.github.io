---
title: "M365 Endpoint Management Lab"
date: 2026-05-13
draft: false
summary: "A hands-on home lab using a Microsoft 365 Developer tenant to practice Intune device enrollment, compliance policies, and Conditional Access — built to demonstrate real MDM skills before the interview."
tags:
  - Microsoft 365
  - Intune
  - MDM
  - Azure AD
  - Conditional Access
  - Home Lab
  - Endpoint Security
---

## What This Lab Is

Most MDM job postings list Intune experience as a requirement. Most people without a current employer-provided Intune environment have no obvious way to get that experience. The M365 Developer Program solves that problem: Microsoft provides a free 90-day renewable tenant with 25 E5 licenses, which includes Microsoft Intune, Azure Active Directory Premium P2, and the full M365 suite. This lab documents how I used that tenant to build a working Intune environment, enroll a test device, configure compliance policies, and wire up Conditional Access — from scratch, with no prior access to a production Intune environment.

The goal was not a lab for its own sake. It was to be able to speak credibly about Intune configuration in an interview — to have done the thing, not just read about it.

---

## Lab Setup

### Step 1: Create the M365 Developer Tenant

The Microsoft 365 Developer Program is at [developer.microsoft.com/en-us/microsoft-365/dev-program](https://developer.microsoft.com/en-us/microsoft-365/dev-program). Sign up with a personal Microsoft account. Select the "Instant Sandbox" option, which provisions a pre-populated tenant with sample users, teams, and data — faster than building from scratch and more realistic to work with.

The tenant comes with a `.onmicrosoft.com` domain (e.g., `yourname.onmicrosoft.com`), 25 E5 licenses, and a global administrator account. Save the admin credentials in a password manager immediately.

### Step 2: Enable and Access Intune

Intune is included in the E5 license but may not be active by default.

1. Sign into the [Microsoft Entra admin center](https://entra.microsoft.com) with the global admin account
2. Navigate to Microsoft Intune > Overview — the console should be accessible
3. If prompted, confirm the MDM and MAM authority is set to Intune (not SCCM or None)

> [Screenshot: Intune Overview dashboard showing tenant name, enrolled device count, and license status]

### Step 3: Enroll a Test Device

For the Windows enrollment, I used my personal Windows 11 machine in a test configuration. The enrollment method was **User-driven Azure AD Join with Intune auto-enrollment**.

**Enrollment path:**
1. Settings > Accounts > Access work or school > Connect
2. Enter the developer tenant admin credentials
3. Azure AD join option — join this device to Azure Active Directory
4. After join, Intune enrollment triggers automatically based on MDM auto-enrollment policy

**To enable auto-enrollment:**
- Entra admin center > Devices > Device Settings > MDM User Scope: Set to "All" (or a specific group)
- This ensures any Azure AD-joined device auto-enrolls in Intune without additional steps

> [Screenshot: Settings > Accounts showing work/school connection to the developer tenant]

> [Screenshot: Intune Devices > All Devices showing the enrolled test machine with compliance status]

**Alternative for those without a spare device:** The [Intune Simulator](https://endpoint.microsoft.com/#view/Microsoft_Intune_DeviceSettings/DevicesMenu/~/overview) allows policy simulation and compliance status review without a physical device. It does not replace real enrollment experience but is useful for policy design practice.

---

## What Was Configured

### Compliance Policy: Windows 10/11 Baseline

Created under Intune > Devices > Compliance Policies > Create Policy (Windows 10 and later).

**Settings configured:**

| Setting | Value |
|---|---|
| BitLocker | Required |
| Minimum OS version | 10.0.22621 (Windows 11 22H2) |
| Password required | Yes |
| Minimum password length | 12 characters |
| Password complexity | Digits, lowercase, uppercase, special characters |
| Maximum inactivity before screen lock | 5 minutes |
| Firewall | Required |
| Microsoft Defender Antivirus | Required |
| Microsoft Defender real-time protection | Required |

**Actions for non-compliance:**
- Mark device non-compliant: immediately
- Send email to user: after 1 day
- Retire device: after 30 days

> [Screenshot: Intune compliance policy settings page showing the BitLocker and password requirements]

The test device enrolled and reported as compliant within about 15 minutes of enrollment. I then manually disabled BitLocker on the test machine to verify the non-compliant state propagated to Intune and then to Azure AD.

> [Screenshot: Intune device detail showing "Not compliant" status with the specific failing policy item]

### Conditional Access Policy: Require Compliant Device for M365

Created in the Entra admin center under Protection > Conditional Access > New Policy.

**Policy configuration:**

- **Name:** Require compliant device — M365 apps
- **Users:** All users (scoped to the test tenant)
- **Target resources:** Office 365 (includes Exchange Online, SharePoint, Teams)
- **Conditions:** None added (applies to all platforms and locations)
- **Grant:** Grant access — Require device to be marked as compliant
- **Session:** None
- **Enable policy:** On (Report-only first, then On after verifying behavior)

> [Screenshot: Conditional Access policy grant settings showing "Require device to be marked as compliant"]

**Testing the policy:**

From the compliant enrolled device, accessing Outlook Web App worked without additional prompts. After disabling BitLocker (making the device non-compliant), signing out and back in to Outlook triggered a Conditional Access block page indicating the device did not meet compliance requirements.

> [Screenshot: Conditional Access block page shown after device became non-compliant]

This confirmed the end-to-end loop: Intune detects non-compliance → propagates to Azure AD device record → Conditional Access enforces the block at sign-in.

### Device Enrollment Restrictions

Configured under Intune > Devices > Enrollment > Enrollment Restrictions.

- Set a device limit per user (5 devices)
- Restricted enrollment to Windows and iOS platforms only (blocked Android in this test environment to practice restriction configuration)
- Set minimum OS version requirement at enrollment to prevent outdated devices from joining

---

## Key Findings from Building It

**The compliance evaluation loop is faster than I expected.** After a device configuration change, Intune re-evaluates compliance within 8-15 minutes in my testing. Azure AD picked up the new compliance status within another 5-10 minutes. The total lag from "device becomes non-compliant" to "Conditional Access blocks access" was under 30 minutes without any manual refresh.

**Report-only mode for Conditional Access is genuinely useful.** Running the policy in report-only mode first let me see exactly which sign-ins would have been blocked before enabling enforcement. This is the correct production rollout path — I would not recommend enabling a Conditional Access policy in enforcement mode as the first step in a real environment.

**Intune's built-in reporting gaps.** The compliance overview dashboard is useful, but getting the full picture required building a custom filter in Devices > All Devices with the compliance status column added. The default view buries non-compliant devices. In a production environment with hundreds of devices, this would matter.

**Device clean-up policy matters.** Stale enrolled devices persist in the Intune inventory even after the physical device is wiped or retired. Configuring a device clean-up rule (retire devices inactive for 30 days) is necessary to keep the device count accurate.

---

## Why This Lab Matters for the Role

The honest answer is that showing up to an Intune interview having only read documentation is not enough. Interviewers who work in Intune daily will ask follow-up questions that require having touched the actual interface — what does non-compliance look like in the console, how do you test a Conditional Access policy safely, where do you go to see which policy is blocking a specific device.

This lab produced answers to those questions grounded in actual experience, not documentation. It also produced screenshots that can be walked through in a conversation, which is more credible than narrating what the documentation says would happen.

The M365 Developer Program makes this accessible to anyone willing to spend a few hours setting it up. The 90-day window renews if you show active development activity — in practice, continuing to configure and explore the tenant counts. For anyone building toward an endpoint management role, there is no good reason to wait for employer access to get started.
