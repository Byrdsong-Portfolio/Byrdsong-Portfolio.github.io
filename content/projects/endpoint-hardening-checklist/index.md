---
title: "Tier 1 Endpoint Hardening Checklist"
date: 2026-05-13
draft: false
summary: "A practical hardening checklist covering Windows, macOS, and mobile endpoints — password policies, encryption, patch status, MDM enrollment, and screen lock baselines for Tier 1 IT support."
tags:
  - Endpoint Security
  - Hardening
  - Windows
  - macOS
  - Mobile
  - MDM
  - Checklist
  - Compliance
---

## What Tier 1 Hardening Means

Tier 1 endpoint hardening is not advanced threat modeling — it is the baseline security configuration that every managed device should have before it touches a corporate network. These controls are the ones that security auditors check first, that compliance frameworks like CIS Benchmarks require as Level 1 controls, and that attackers probe for because they are commonly misconfigured or skipped entirely.

For Tier 1 IT support, this checklist serves two purposes: a new-device setup verification before issuing a machine to a user, and a periodic review checklist for existing devices flagged by MDM or escalated through a helpdesk ticket. Every item here maps to a documented security control and can be confirmed without specialized tools — just OS-level settings access and, where applicable, an MDM console.

---

## Windows Endpoints

### Encryption

- [ ] **BitLocker enabled on the OS drive** — Verify in Control Panel > BitLocker Drive Encryption or run `manage-bde -status` in an elevated command prompt. Status should read "Protection On."
- [ ] **Recovery key backed up to Azure AD or Active Directory** — Confirm the key is escrowed; do not rely on the user having the key saved locally.
- [ ] **Removable drives: BitLocker To Go policy applied** — If the org handles sensitive data, USB drives should require encryption before data can be written.

### Authentication and Access

- [ ] **Password complexity policy active** — Minimum 12 characters, requires uppercase, lowercase, number, and special character. Verify via Local Group Policy or confirm MDM compliance policy is enforced.
- [ ] **Screen lock timeout ≤ 5 minutes** — Settings > Personalization > Lock Screen > Screen timeout settings. Idle lock should trigger within 5 minutes.
- [ ] **Screen lock requires password on resume** — Power & Sleep > related settings > require sign-in should be set to "When PC wakes from sleep."
- [ ] **Local administrator accounts audited** — Run `net localgroup administrators` in an elevated prompt. Only expected accounts should appear. Flag any unknown entries.
- [ ] **Guest account disabled** — Verify in Local Users and Groups (lusrmgr.msc) or via PowerShell: `Get-LocalUser Guest | Select-Object Name, Enabled`.
- [ ] **Default Administrator account renamed or disabled** — The built-in `Administrator` account is a known target. It should be disabled or renamed.

### Updates and Patching

- [ ] **Windows Update current** — Settings > Windows Update. No critical or security updates should be pending. Acceptable window: within 30 days of latest Patch Tuesday.
- [ ] **Automatic updates enabled** — Devices should not require manual intervention to receive security patches.

### Endpoint Protection

- [ ] **Microsoft Defender Antivirus active and updated** — Windows Security > Virus & threat protection. Real-time protection should be On; definitions should be current (within 24 hours).
- [ ] **Windows Defender Firewall enabled for all profiles** — Domain, Private, and Public profiles all active. Verify in Windows Security > Firewall & network protection.
- [ ] **No third-party security software conflicts** — If a third-party AV is deployed, Defender should be in passive mode, not disabled. Confirm the deployed solution is active and licensed.

### MDM and Management

- [ ] **Device enrolled in MDM (Intune or equivalent)** — Settings > Accounts > Access work or school. Device should show an MDM enrollment, not just an Azure AD join.
- [ ] **MDM compliance status: Compliant** — Verify in the Intune portal under Devices > All Devices. Non-compliant status indicates a policy gap.
- [ ] **No unenrollment or device retirement pending** — Check the Intune console for any pending wipe or retire actions on the device.

---

## macOS Endpoints

### Encryption

- [ ] **FileVault enabled** — System Settings > Privacy & Security > FileVault. Should show "FileVault is turned on." Recovery key should be escrowed with MDM, not stored locally only.
- [ ] **FileVault recovery key type verified** — Institutional key (managed via MDM) is preferred over personal recovery key for corporate devices.

### Authentication and Access

- [ ] **Screen lock timeout ≤ 5 minutes** — System Settings > Lock Screen. "Require password after screen saver begins or display is turned off" should be set to "Immediately" or a short interval.
- [ ] **Password policy active** — MDM profile or local policy should enforce minimum password length and complexity. Confirm via MDM compliance report or `pwpolicy -getaccountpolicies`.
- [ ] **Automatic login disabled** — System Settings > Users & Groups > Login Options. Automatic login should be Off.
- [ ] **Remote Login (SSH) disabled** — System Settings > General > Sharing. Remote Login should be toggled off unless explicitly needed for a specific role.
- [ ] **Screen sharing disabled** — Same Sharing panel. Screen Sharing and Remote Management should be off on standard user machines.

### Updates and Patching

- [ ] **Software Update current** — System Settings > General > Software Update. macOS version should be within one minor release of the latest, or on the org's approved baseline.
- [ ] **Automatic security response updates enabled** — System Settings > Software Update > Automatic Updates. "Install Security Responses and system files" should be on.

### MDM and Management

- [ ] **MDM profile installed and active** — System Settings > Privacy & Security > Profiles. The corporate MDM enrollment profile should be present.
- [ ] **Profile removal blocked** — For corporate devices, the MDM profile should be supervised and non-removable by the user.
- [ ] **Gatekeeper set to App Store and identified developers** — System Settings > Privacy & Security > Allow applications downloaded from. Should not be set to "Anywhere."

---

## Mobile Endpoints (iOS and Android)

### Authentication

- [ ] **Screen lock with biometric or 6-digit PIN minimum** — No device should be accessible without authentication. Swipe-only unlock is not acceptable for corporate use.
- [ ] **Auto-lock timeout ≤ 5 minutes** — Verify in device settings; MDM policy should enforce this and report compliance status.
- [ ] **Failed attempt wipe policy active** — After a configurable number of failed unlock attempts (typically 10), the device should wipe. Confirm this is set in the MDM compliance policy.

### OS and Patch Status

- [ ] **OS version current or within approved baseline** — Intune and most MDM platforms can enforce a minimum OS version as a compliance requirement. Devices below baseline should be flagged.
- [ ] **Automatic updates enabled** — Both iOS (Settings > General > Software Update > Automatic Updates) and Android (Settings > Software update > Auto download over Wi-Fi) should be on.

### Management and Encryption

- [ ] **Device enrolled in MDM** — Corporate-owned devices should be fully enrolled. BYOD devices should have at minimum an MAM (Mobile Application Management) profile applied.
- [ ] **Device encryption confirmed** — iOS devices are encrypted by default when a passcode is set. Android devices running Android 10+ are encrypted by default; verify in Settings > Security > Encryption.
- [ ] **Remote wipe capability confirmed** — Verify in the MDM console that the device is reachable and a selective or full wipe can be issued. Test this periodically on non-production devices.
- [ ] **App installation restricted to approved sources** — iOS: MDM policy should prevent sideloading. Android: "Install unknown apps" should be disabled for all apps in the MDM policy.

---

## Using This Checklist in an Org Context

**Who runs it:** Tier 1 technicians during device provisioning and during escalated compliance remediation. Tier 2 or IT Security should review flagged items that cannot be resolved at Tier 1.

**How often:** At provisioning for new devices. Quarterly review for devices that MDM has flagged as non-compliant or that have been offline for more than 30 days. Immediately following any user-reported security incident.

**What to do with findings:**
- Items that can be resolved through settings are fixed immediately and documented in the helpdesk ticket.
- Items that require a policy change (e.g., MDM compliance policy is not enforcing screen lock) are escalated to Tier 2 or IT administration with the specific gap documented.
- Devices with multiple failures should be quarantined from the corporate network (if Conditional Access is in place, this may happen automatically) until remediated.
- Findings should be logged against the device asset record. A pattern of recurring failures on the same device is a signal for hardware replacement or user training.

This checklist does not replace MDM automated compliance reporting — it supplements it. MDM tells you what is out of compliance at scale. This checklist tells you what to look for when you have the device in front of you and need to verify or remediate manually.
