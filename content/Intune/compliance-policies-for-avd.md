---
title: Compliance Policies for AVD
description: 
published: true
date: 2025-12-14T04:53:11.479Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:39.346Z
---

# Compliance Policies for AVD

Compliance policies in Microsoft Intune define security and health requirements that devices must meet to be considered "compliant". For AVD session hosts, compliance policies ensure that session hosts meet organizational security standards before users can access corporate resources. When integrated with Conditional Access, compliance policies enforce Zero Trust security by blocking access from non-compliant devices. This page covers compliance policy fundamentals, AVD-specific requirements, integration with Conditional Access, and best practices for MSP environments.

## What are Compliance Policies

A compliance policy is a set of rules that devices must meet to be marked as "compliant" in Intune. Examples include:

- **Security Requirements**: BitLocker encryption enabled, firewall active, antivirus up-to-date
- **Configuration Requirements**: Minimum OS version, password complexity, device PIN required
- **Health Attestation**: Secure Boot enabled, Code Integrity verified

Unlike configuration profiles (which *configure* settings), compliance policies *evaluate* whether settings meet requirements. If a device fails compliance evaluation, it is marked as "non-compliant", and actions can be triggered (send email, block access, retire device).

**Compliance Check Frequency:**
- **Initial Check**: When device enrolls into Intune
- **Periodic Check**: Every 8 hours by default (configurable down to 1 hour)
- **On-Demand Check**: Via Intune Admin Center or Company Portal app

**Compliance States:**
- **Compliant**: Device meets all requirements
- **Non-Compliant**: Device fails one or more requirements
- **In Grace Period**: Device is non-compliant but within grace period (default 1 day)
- **Not Evaluated**: Compliance policy not assigned or device has not checked in

For AVD environments, compliance policies ensure that session hosts are secure before users connect. This is especially important when using Conditional Access to require compliant devices for accessing corporate resources.

## Compliance vs Configuration

It's common to confuse compliance policies with configuration profiles. Here's the difference:

| Aspect | Configuration Profiles | Compliance Policies |
|--------|------------------------|---------------------|
| **Purpose** | **Configure** settings (deploy, enforce) | **Evaluate** whether settings meet requirements |
| **Action** | Changes device settings (e.g., enables BitLocker) | Checks if settings are configured correctly |
| **Enforcement** | Configures the device (pushes settings) | Reports compliance status (evaluates settings) |
| **Example** | Deploy BitLocker encryption settings | Require BitLocker to be enabled |
| **Conditional Access** | Not integrated | Integrated (can require compliant device) |
| **User Impact** | Silent (settings applied automatically) | Visible (users notified if non-compliant) |

### Why Both Matter

You need **both** configuration profiles and compliance policies for complete security:

1. **Configuration Profile**: Deploys BitLocker encryption settings to session hosts
2. **Compliance Policy**: Verifies that BitLocker is actually enabled
3. **Conditional Access**: Blocks access if compliance policy reports device as non-compliant

**Example Scenario:**
- Configuration profile deploys BitLocker settings to enable encryption
- Compliance policy checks if BitLocker is enabled; if not, marks device as non-compliant
- Conditional Access policy requires compliant device for accessing Microsoft 365; non-compliant device is blocked

This layered approach ensures that security settings are not only deployed but also enforced and verified.

## Common Compliance Requirements for AVD

The following compliance requirements are recommended for AVD session hosts. These requirements align with Microsoft security baselines and Zero Trust principles.

### BitLocker Encryption

**Requirement:** BitLocker Drive Encryption enabled for OS drive

**Why:** Protects data at rest if Azure VM disk is compromised or downloaded

**Configuration:**
- **Setting**: Require BitLocker
- **Value**: Enabled
- **Scope**: OS drive (C:)

**Portal:** Intune Admin Center → Devices → Compliance Policies → Create Policy → Windows 10 and later → Device Health → BitLocker → Require BitLocker

**Note:** Azure Managed Disks support encryption-at-rest (SSE) by default, but BitLocker provides an additional layer (encryption-in-use). For AVD, BitLocker is recommended but not mandatory if Azure Disk Encryption is enabled.

**Deployment Workflow:**
1. Create configuration profile to deploy BitLocker settings (enable BitLocker, set encryption method to XTS-AES 256)
2. Assign configuration profile to AVD-Devices-All
3. Create compliance policy requiring BitLocker enabled
4. Assign compliance policy to AVD-Devices-All
5. Wait 24-48 hours for BitLocker encryption to complete and compliance status to update

### Firewall Enabled

**Requirement:** Windows Defender Firewall enabled for all network profiles (Domain, Private, Public)

**Why:** Prevents unauthorized network access to session hosts

**Configuration:**
- **Setting**: Require Firewall
- **Value**: Enabled
- **Network Profiles**: Domain, Private, Public

**Portal:** Intune Admin Center → Devices → Compliance Policies → Create Policy → Windows 10 and later → System Security → Firewall → Require

**Note:** Windows Defender Firewall is enabled by default on Windows 10/11. This compliance check ensures it remains enabled and is not disabled by users or scripts.

### Antivirus Active and Up-to-Date

**Requirement:** Microsoft Defender Antivirus enabled, definitions up-to-date, and real-time protection active

**Why:** Protects session hosts from malware, ransomware, and other threats

**Configuration:**
- **Setting**: Require Microsoft Defender Antimalware
- **Value**: Enabled
- **Additional Checks**:
  - Real-time protection enabled
  - Signature version no older than 7 days

**Portal:** Intune Admin Center → Devices → Compliance Policies → Create Policy → Windows 10 and later → Microsoft Defender Antimalware

**Best Practice:** Combine with Microsoft Defender for Endpoint (MDE) for advanced threat protection, EDR capabilities, and centralized reporting. MDE integrates with Intune compliance policies and can automatically mark devices as non-compliant if threats are detected.

### Minimum OS Version

**Requirement:** Windows 10 21H2 (build 19044) or Windows 11 22H2 (build 22621) or later

**Why:** Ensures session hosts receive security updates and feature support

**Configuration:**
- **Setting**: Minimum OS Version
- **Value**: 10.0.19044 (Windows 10 21H2) or 10.0.22621 (Windows 11 22H2)

**Portal:** Intune Admin Center → Devices → Compliance Policies → Create Policy → Windows 10 and later → Device Properties → Minimum OS Version

**Recommendation:** Set minimum OS version based on your organization's update cadence:
- **Conservative**: Windows 10 21H2 or Windows 11 22H2 (long-term support)
- **Current**: Windows 10 22H2 or Windows 11 23H2 (latest feature updates)

**Impact:** Devices running older OS versions (e.g., Windows 10 20H2) will be marked as non-compliant. Plan golden image updates before enabling this requirement.

### Password Requirements

**Requirement:** Device password required, minimum length, complexity, and expiration

**Why:** Protects against unauthorized local access (though less relevant for cloud-managed AVD with Entra authentication)

**Configuration:**
- **Setting**: Require a password to unlock mobile devices
- **Value**: Enabled
- **Additional Settings**:
  - Minimum password length: 8 characters
  - Password complexity: Require alphanumeric
  - Maximum minutes of inactivity before password is required: 15 minutes
  - Password expiration (days): 90 (or "Never" for modern passwordless environments)

**Portal:** Intune Admin Center → Devices → Compliance Policies → Create Policy → Windows 10 and later → System Security → Password

**Note:** For AVD session hosts managed by Entra ID, local passwords are less relevant since users authenticate via Entra. This requirement is more applicable to personal devices (BYOD) than AVD session hosts. Consider skipping this requirement for AVD environments.

### Secure Boot and Code Integrity

**Requirement:** Secure Boot enabled, Code Integrity policy active

**Why:** Prevents boot-level malware (rootkits, bootkits) from executing

**Configuration:**
- **Setting**: Require Secure Boot
- **Value**: Enabled
- **Additional Checks**:
  - Code Integrity policy active

**Portal:** Intune Admin Center → Devices → Compliance Policies → Create Policy → Windows 10 and later → Device Health → Secure Boot

**Note:** Secure Boot requires UEFI firmware (not legacy BIOS). All modern Azure VMs support UEFI and Secure Boot. Verify that your golden image has Secure Boot enabled during deployment.

## Grace Periods

A grace period allows devices to remain compliant for a specified time after failing a compliance check. This prevents immediate access loss and gives administrators time to remediate issues.

**Default Grace Period:** 1 day (24 hours)

**Configurable Range:** 0 days (immediate non-compliance) to 120 days

**How It Works:**
1. Device fails compliance check (e.g., antivirus definitions outdated)
2. Intune marks device as "In Grace Period" for 24 hours
3. During grace period, device is still considered compliant for Conditional Access
4. After grace period expires, device is marked as "Non-Compliant" and access is blocked (if Conditional Access is configured)

**When to Use Grace Periods:**
- **Pilot Deployments**: Set longer grace periods (7-14 days) to avoid blocking access during testing
- **Production Deployments**: Set shorter grace periods (1-3 days) to enforce compliance quickly
- **Critical Requirements**: Set 0-day grace period for high-risk scenarios (e.g., known malware detected)

**Portal:** Intune Admin Center → Devices → Compliance Policies → Create Policy → Actions for Noncompliance → Mark device noncompliant → Schedule (days after noncompliance)

**Best Practice:** Use 1-day grace period for most compliance requirements. For critical security issues (e.g., known vulnerability detected), use 0-day grace period to block access immediately.

## Actions for Non-Compliance

When a device is marked as non-compliant, Intune can trigger automated actions. These actions can be immediate or scheduled after a delay.

### Mark Device as Non-Compliant

**Action:** Mark device as non-compliant in Intune

**When:** Immediately or after grace period (configurable delay)

**Impact:** Device shows "Non-Compliant" status in Intune. If Conditional Access is configured with "Require compliant device", access is blocked.

**Portal:** Intune Admin Center → Devices → Compliance Policies → [Policy] → Actions for Noncompliance → Add → Mark device noncompliant

**Best Practice:** Set to 1 day (allows grace period for transient issues like temporary antivirus update delays).

### Send Email to End User

**Action:** Send email notification to user's Entra ID email address

**When:** Configurable delay (e.g., immediately, 1 day, 3 days)

**Email Content:** Includes:
- Device name
- Compliance policy that failed
- Remediation steps (e.g., "Enable BitLocker on your device")
- Link to Company Portal for manual sync

**Portal:** Intune Admin Center → Devices → Compliance Policies → [Policy] → Actions for Noncompliance → Add → Send email to end user

**Best Practice:** Send email notification immediately when device is marked non-compliant. Include remediation instructions (e.g., "Contact IT support if issue persists").

**Example Email Template:**
```
Subject: Action Required - Your AVD session host is non-compliant

Your AVD session host "{DeviceName}" does not meet security requirements.

Issue: BitLocker encryption is not enabled

Action Required: Contact IT support to enable BitLocker encryption on your session host.

If you believe this is an error, sync your device in Company Portal and try again.
```

### Send Email to IT Admin

**Action:** Send email notification to IT admin/support team

**When:** Configurable delay (e.g., 3 days, 7 days)

**Email Content:** Includes:
- Device name and user
- Compliance policy that failed
- Last check-in time

**Portal:** Intune Admin Center → Devices → Compliance Policies → [Policy] → Actions for Noncompliance → Add → Send email to IT admin → Enter email addresses

**Best Practice:** Send email to IT admin after 3 days of non-compliance. This gives users time to remediate before escalating to IT.

### Remotely Lock Device

**Action:** Trigger remote lock on device (requires user PIN to unlock)

**When:** Configurable delay (e.g., 7 days, 14 days)

**Impact:** Device is locked and requires user to enter PIN/password to unlock

**Portal:** Intune Admin Center → Devices → Compliance Policies → [Policy] → Actions for Noncompliance → Add → Remotely lock device

**Best Practice:** Rarely used for AVD session hosts (more applicable to mobile devices). If used, set long delay (14-30 days) to avoid locking production session hosts.

### Retire Device

**Action:** Remove device from Intune management and delete company data

**When:** Configurable delay (e.g., 30 days, 60 days)

**Impact:** Device is unenrolled from Intune, company data is deleted, and user can no longer access corporate resources

**Portal:** Intune Admin Center → Devices → Compliance Policies → [Policy] → Actions for Noncompliance → Add → Retire the noncompliant device

**Best Practice:** Use only for extreme cases (e.g., lost device, security breach). For AVD session hosts, consider manual decommissioning instead of automated retire action.

**Recommendation for AVD:** Focus on "Mark device as non-compliant" and "Send email" actions. Avoid "Retire device" for AVD session hosts, as accidental retirement can cause production outages.

## Integration with Conditional Access

Conditional Access policies enforce access controls based on conditions (user, location, device compliance, app, etc.). By integrating compliance policies with Conditional Access, you can block access from non-compliant AVD session hosts.

### Require Compliant Device for AVD Access

**Use Case:** Allow AVD connections only from compliant session hosts

**Configuration:**
1. Create Intune compliance policy (e.g., require BitLocker, firewall, antivirus)
2. Assign compliance policy to AVD-Devices-All group
3. Create Conditional Access policy:
   - Users: All users (or specific AVD user group)
   - Cloud apps: Windows Virtual Desktop (AVD)
   - Conditions: Device platforms = Windows
   - Grant: Require device to be marked as compliant
4. Enable Conditional Access policy

**Portal:** Entra Admin Center → Protection → Conditional Access → New Policy

**Impact:** Users can only connect to AVD from compliant session hosts. Non-compliant session hosts are blocked until compliance is restored.

**Example Policy:**
- **Name**: Require Compliant Device for AVD
- **Assignments**:
  - Users: All AVD users
  - Cloud apps: Windows Virtual Desktop
- **Conditions**:
  - Device platforms: Windows
- **Access Controls**:
  - Grant: Require device to be marked as compliant
- **Enable Policy**: On

### Testing Conditional Access

Before enabling Conditional Access in production, test in "Report-only" mode:

1. Create Conditional Access policy
2. Set "Enable policy" to **Report-only**
3. Monitor sign-in logs for 7 days (Entra Admin Center → Monitoring → Sign-ins)
4. Review "Conditional Access" column to see which sign-ins would be blocked
5. If results are acceptable, change "Enable policy" to **On**

**Best Practice:** Always test Conditional Access policies in Report-only mode before enabling. This prevents accidental lockouts and allows you to verify policy behavior.

### Conditional Access for Hybrid Scenarios

If your organization uses both Entra-joined and Hybrid-joined AVD session hosts, create separate Conditional Access policies:

**Policy 1: Entra-Joined Session Hosts**
- Condition: Device state = Entra joined
- Grant: Require compliant device

**Policy 2: Hybrid-Joined Session Hosts**
- Condition: Device state = Hybrid Entra joined
- Grant: Require compliant device OR Require domain-joined device

This ensures both device types are covered.

## Our Environment Compliance Configuration

### Compliance Policy: AVD Session Hosts - Security Baseline

**Assigned To:** AVD-Devices-All (all AVD session hosts in RG-Azure-VDI-01)

**Requirements:**
- **BitLocker**: Required (OS drive)
- **Firewall**: Enabled (Domain, Private, Public profiles)
- **Antivirus**: Microsoft Defender enabled, real-time protection active, signatures no older than 7 days
- **OS Version**: Windows 10 21H2 (10.0.19044) or later
- **Secure Boot**: Required

**Grace Period:** 1 day

**Actions for Non-Compliance:**
- **Immediately**: Send email to end user
- **1 day**: Mark device as non-compliant
- **3 days**: Send email to IT admin (support@example.com)

**Conditional Access Integration:**
- **Policy Name**: Require Compliant Device for AVD
- **Cloud App**: Windows Virtual Desktop
- **Grant**: Require device to be marked as compliant

### Deployment Process

1. **Create Compliance Policy**
   - Intune Admin Center → Devices → Compliance Policies → Create Policy → Windows 10 and later
   - Configure requirements (BitLocker, firewall, antivirus, OS version)
   - Set grace period to 1 day
   - Add actions for non-compliance (email immediately, mark non-compliant after 1 day)

2. **Assign to Device Group**
   - Assign to AVD-Devices-All group
   - Exclude any test/pilot devices if needed

3. **Monitor Compliance Status**
   - Intune Admin Center → Devices → Compliance Policies → [Policy] → Monitor → Device Status
   - Wait 24-48 hours for all devices to check in and report compliance

4. **Create Conditional Access Policy**
   - Entra Admin Center → Protection → Conditional Access → New Policy
   - Assign to AVD users, Windows Virtual Desktop app
   - Grant: Require device to be marked as compliant
   - Enable policy in Report-only mode

5. **Test Conditional Access (Report-Only)**
   - Monitor sign-in logs for 7 days
   - Verify no unexpected blocks

6. **Enable Conditional Access**
   - Change policy mode from Report-only to On
   - Monitor for 48 hours, verify no user reports of access issues

### Verification

**Check Compliance Status:**
- Intune Admin Center → Devices → All Devices → [Device] → Device Compliance

**Check Conditional Access Status:**
- Entra Admin Center → Monitoring → Sign-in Logs → [User] → Conditional Access tab

**Expected Result:**
- All session hosts in AVD-Devices-All show "Compliant" status
- Sign-in logs show "Success" with "Require compliant device" policy applied

## Best Practices

### Align Compliance Requirements with Security Baselines

**Why:** Microsoft publishes security baselines for Windows 10/11 that represent best-practice security configurations. Aligning compliance policies with these baselines ensures consistency and reduces configuration drift.

**How:** Review Microsoft Security Baselines and configure compliance policies to match (BitLocker, firewall, antivirus, OS version).

**Link:** Microsoft Security Baselines - https://aka.ms/baselines

### Use 1-Day Grace Period for Production

**Why:** Balances security with operational flexibility. Gives time to resolve transient issues (e.g., temporary antivirus update failure) without blocking access.

**Exception:** For critical security requirements (e.g., known malware detected), use 0-day grace period to block access immediately.

### Test Conditional Access in Report-Only Mode First

**Why:** Prevents accidental lockouts and allows you to verify policy behavior before enforcement.

**Workflow:**
1. Create Conditional Access policy in Report-only mode
2. Monitor sign-in logs for 7 days
3. Review which sign-ins would be blocked
4. Adjust policy if needed
5. Enable policy (Report-only → On)

### Send Email Notifications Immediately

**Why:** Informs users of compliance issues and provides remediation instructions before access is blocked.

**Example:** If BitLocker fails compliance, email user immediately with instructions to contact IT support.

### Monitor Compliance Status Weekly

**Why:** Ensures compliance policies are working as expected and identifies devices that are consistently non-compliant.

**Weekly Checklist:**
- Review compliance policy dashboard (Devices → Compliance Policies → [Policy] → Device Status)
- Investigate devices in "Non-Compliant" state for >7 days
- Remediate issues (apply updates, fix configuration errors, retire devices)

### Use Conditional Access for AVD Access

**Why:** Enforces Zero Trust security by blocking access from non-compliant session hosts. Even if user credentials are compromised, attacker cannot connect from non-compliant device.

**Recommended Policy:**
- Users: All AVD users
- Cloud App: Windows Virtual Desktop
- Grant: Require device to be marked as compliant

### Exclude Break-Glass Accounts from Conditional Access

**Why:** Prevents lockout in emergency scenarios (e.g., compliance policy misconfiguration breaks all devices).

**How:**
1. Create Entra ID group: "Conditional Access - Break Glass Exclusion"
2. Add 2-3 emergency admin accounts to group
3. In Conditional Access policy, exclude this group from "Users" assignment

**Important:** Secure break-glass accounts with strong passwords (20+ characters) and store credentials in secure location (password vault, printed copy in safe).

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Device shows "Not Evaluated" compliance status** | Compliance policy not assigned or device has not checked in | 1. Verify compliance policy is assigned to device group containing the session host<br>2. Force device sync (Intune Admin Center → Devices → [Device] → Sync)<br>3. Wait up to 8 hours for initial check-in |
| **Device non-compliant for BitLocker but encryption is enabled** | BitLocker recovery key not escrowed to Entra ID | 1. On session host, run: `BackupToAAD-BitLockerKeyProtector C: -KeyProtectorId {GUID}`<br>2. Get Key Protector ID: `(Get-BitLockerVolume -MountPoint C:).KeyProtector`<br>3. Verify key backed up: Entra Admin Center → Devices → All Devices → [Device] → BitLocker Keys |
| **Device non-compliant for antivirus but Defender is active** | Antivirus signatures outdated (>7 days old) | 1. Force signature update: `Update-MpSignature`<br>2. Verify update: `Get-MpComputerStatus` (check `AntivirusSignatureLastUpdated`)<br>3. Force compliance sync: Intune Admin Center → Devices → [Device] → Sync |
| **Conditional Access blocks compliant device** | Compliance status not synced to Entra ID or CA evaluation delay | 1. Force device compliance sync (Intune Admin Center → Devices → [Device] → Sync)<br>2. Wait 15 minutes for compliance status to sync to Entra ID<br>3. User signs out and signs back in to refresh token<br>4. Check sign-in logs for CA evaluation result |
| **Compliance policy shows "Error" state** | Setting not supported on OS version or device type | 1. Review error details (Devices → All Devices → [Device] → Device Compliance → [Policy])<br>2. If error is "Setting not applicable", verify OS version supports setting<br>3. Remove unsupported setting from policy or upgrade OS version |
| **All devices non-compliant after policy creation** | Policy requirements too strict or golden image missing required settings | 1. Review compliance requirements (e.g., minimum OS version set too high)<br>2. Update golden image to meet requirements (enable BitLocker, update OS version)<br>3. Redeploy session hosts from updated golden image |

## Summary

Compliance policies are essential for enforcing security standards and integrating Zero Trust access controls for AVD session hosts. By defining requirements (BitLocker, firewall, antivirus), setting grace periods, and integrating with Conditional Access, you ensure that only secure, compliant session hosts can access corporate resources.

**Key Takeaways:**
- Use **compliance policies** to evaluate security requirements (BitLocker, firewall, antivirus)
- Use **configuration profiles** to deploy security settings (enable BitLocker, configure firewall)
- Set **1-day grace period** for production deployments (balances security with flexibility)
- Send **email notifications** immediately to inform users of compliance issues
- Integrate with **Conditional Access** to block access from non-compliant devices (Zero Trust enforcement)
- Test Conditional Access in **Report-only mode** before enabling (prevents accidental lockouts)

## Next Steps

After configuring compliance policies:

1. **Application Deployment** - Deploy Win32 apps and Microsoft Store apps via Intune (Page 4)
2. **Windows Update Policies** - Control update deployment with update rings and feature update policies (Page 5)

Proceed to the next page to configure Application Deployment for AVD session hosts.