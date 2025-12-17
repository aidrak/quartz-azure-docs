---
title: Intune Prerequisites for AVD
description: 
published: true
date: 2025-12-14T04:53:15.078Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:42.927Z
---

# Intune Prerequisites for AVD

Microsoft Intune provides modern device management capabilities for Azure Virtual Desktop session hosts. When AVD session hosts are Entra joined, they automatically enroll into Intune, enabling centralized configuration management, compliance enforcement, and application deployment. This page covers the licensing requirements, Entra ID configuration, and enrollment strategies for AVD environments.

## What is Microsoft Intune

Microsoft Intune is a cloud-based Mobile Device Management (MDM) and Mobile Application Management (MAM) service that enables organizations to control how devices and applications are used. For AVD deployments, Intune provides:

- **Configuration Management**: Deploy settings, security baselines, and policies to session hosts
- **Compliance Enforcement**: Ensure devices meet organizational security requirements
- **Application Deployment**: Distribute and manage applications (though less common for pooled AVD)
- **Update Management**: Control Windows Update deployment through update rings and feature update policies
- **Inventory and Reporting**: Track device health, compliance status, and application inventory

Intune integrates seamlessly with Entra ID (formerly Azure AD) and Conditional Access, creating a comprehensive security and management platform. For AVD specifically, Intune excels at managing personal (persistent) desktops where users have dedicated session hosts, and provides baseline configuration management for pooled (non-persistent) environments.

## Licensing Requirements

### Required Licenses

Microsoft Intune requires one of the following licenses per user:

**Option 1: Microsoft 365 Bundles (Recommended)**
- **Microsoft 365 E3** - Includes Intune Plan 1, Office 365 E3, Windows 10/11 Enterprise E3
- **Microsoft 365 E5** - Includes Intune Plan 1, Office 365 E5, Windows 10/11 Enterprise E5, advanced security features
- **Microsoft 365 F3** - Frontline workers, includes Intune Plan 1 with limited features

**Option 2: Standalone Intune**
- **Microsoft Intune Plan 1** - Standalone license for device management (approximately $8/user/month)
- **Microsoft Intune Plan 2** - Adds advanced endpoint analytics and remote help capabilities

**Option 3: Enterprise Mobility + Security (EMS)**
- **EMS E3** - Includes Intune Plan 1, Entra ID P1, Azure Information Protection P1
- **EMS E5** - Includes Intune Plan 2, Entra ID P2, Azure Information Protection P2, Microsoft Defender for Identity

### License Assignment

Licenses must be assigned to **users**, not devices. When a user signs into an Entra-joined AVD session host:

1. The device checks if the user has an Intune license
2. If licensed, the device enrolls into Intune under that user's account
3. Device-based policies (configuration profiles, compliance) then apply to the session host

For shared pooled AVD environments, you typically assign licenses to a dedicated "AVD Service Account" or use the primary administrator account that provisions the session hosts.

### AVD Licensing Considerations

- **AVD Access Rights**: Separate from Intune; requires Microsoft 365 E3/E5, Windows E3/E5, or AVD per-user access license
- **Multi-Session Licensing**: Windows 10/11 Enterprise multi-session (pooled AVD) requires Microsoft 365 or Windows E3/E5 per user
- **Personal AVD**: Personal desktops can use Windows 10/11 Pro with Microsoft 365 licensing

> **Best Practice:** For MSP environments, use Microsoft 365 E3 or E5 bundles to get AVD rights, Intune, and Office 365 in a single license. This simplifies licensing compliance and provides the best value.

## Entra ID Requirements

Microsoft Intune relies on Entra ID for identity, authentication, and device management. The following Entra ID prerequisites are required:

### Entra ID Tenant

- Active Entra ID (Azure AD) tenant
- Users must have Entra ID accounts (cloud-only or synchronized from on-premises AD via Entra Connect)
- Entra ID P1 or P2 (included in Microsoft 365 E3/E5) recommended for Conditional Access integration

### Device Join Type

AVD session hosts can join Entra ID in two ways:

**Entra Join (Recommended for AVD)**
- Devices join directly to Entra ID (cloud-native)
- Automatically enrolls into Intune when MDM auto-enrollment is configured
- Best for cloud-first organizations or new AVD deployments
- No on-premises AD dependency for device management
- Our environment: **RG-Azure-VDI-01** uses Entra Join for all session hosts

**Hybrid Entra Join**
- Devices join both on-premises AD and Entra ID
- Requires additional configuration (Group Policy or Intune enrollment script) to enroll into Intune
- Best for organizations with existing on-premises AD infrastructure and GPO investments
- Enables gradual migration from GPO to Intune

> **Note:** For new AVD deployments, use **Entra Join**. It eliminates the need for domain controllers in Azure, reduces complexity, and provides automatic Intune enrollment. Hybrid join is only necessary if you have strict requirements for on-premises AD group policy or legacy applications that require domain authentication.

## Automatic Enrollment Configuration

Automatic enrollment links Entra ID device registration with Intune enrollment, so Entra-joined devices automatically enroll into Intune management.

### Configure MDM Auto-Enrollment

**Portal:** Entra Admin Center → Mobility (MDM and MAM) → Microsoft Intune

1. **Navigate to Auto-Enrollment Settings**
   - Entra Admin Center (https://entra.microsoft.com)
   - Identity → Devices → Mobility (MDM and MAM)
   - Select **Microsoft Intune**

2. **Configure MDM User Scope**
   - **None**: Disables auto-enrollment (not recommended)
   - **Some**: Auto-enroll devices for selected security groups
   - **All**: Auto-enroll all Entra-joined devices (recommended for AVD)

   For AVD environments, set to **All** or create a security group containing AVD users.

3. **Configure MAM User Scope (Optional)**
   - MAM (Mobile Application Management) applies to mobile devices (iOS/Android)
   - For AVD (Windows only), MAM is typically set to **None**

4. **Set MDM URLs (Automatic)**
   - Terms of Use URL: Auto-populated by Microsoft
   - Discovery URL: `https://enrollment.manage.microsoft.com/enrollmentserver/discovery.svc`
   - Compliance URL: `https://portal.manage.microsoft.com/?portalAction=Compliance`

   These are automatically configured when you select Microsoft Intune as the MDM provider.

5. **Save Configuration**

### Verification

After configuring auto-enrollment, new Entra-joined AVD session hosts will automatically enroll into Intune:

1. **Join session host to Entra ID** during deployment
2. **User signs in** with licensed Entra ID account
3. **Intune enrollment triggers automatically** (check Intune Admin Center → Devices → All Devices)
4. **Policies begin applying** within 8 hours (force sync with `C:\Windows\System32\deviceenroller.exe /c /AutoEnrollMDM`)

Our environment (RG-Azure-VDI-01) has auto-enrollment configured for all users, so session hosts in **hp-pooled-prod1** and **hp-personal-prod1** automatically appear in Intune after deployment.

## Enrollment Strategies for AVD

### Entra Join with Auto-Enrollment (Recommended)

**Best for:** New AVD deployments, cloud-first organizations

**Configuration:**
- Deploy AVD session hosts with Entra join enabled
- Configure Intune auto-enrollment (MDM scope = All)
- Session hosts automatically enroll on first user sign-in

**Advantages:**
- Zero manual enrollment steps
- No domain controllers required
- Simplified architecture
- Seamless Conditional Access integration

**Example (ARM Template):**
```json
{
  "name": "sessionHost01",
  "properties": {
    "additionalUnattendContent": [
      {
        "componentName": "Microsoft-Windows-UnattendedJoin",
        "content": "<AzureADJoin>true</AzureADJoin>"
      }
    ]
  }
}
```

Our environment uses this method exclusively for all session hosts in RG-Azure-VDI-01.

### Hybrid Join with GPO Enrollment

**Best for:** Organizations with existing on-premises AD infrastructure

**Configuration:**
- Deploy AVD session hosts with Hybrid Entra join
- Create Group Policy to trigger Intune enrollment:
  - Computer Configuration → Policies → Administrative Templates → Windows Components → MDM
  - Enable "Enable automatic MDM enrollment using default Azure AD credentials"
- Session hosts enroll on next group policy refresh

**Advantages:**
- Maintains on-premises AD integration
- Gradual migration from GPO to Intune
- Supports legacy applications requiring domain authentication

**Disadvantages:**
- Requires domain controllers in Azure (additional cost)
- More complex architecture
- Manual GPO configuration required

### Hybrid Join with Intune Enrollment Script

**Best for:** Hybrid environments where GPO cannot be used

**Configuration:**
- Deploy AVD session hosts with Hybrid Entra join
- Run enrollment script via Azure VM Run Command or startup script:
  ```powershell
  $enrollmentUrl = "https://enrollment.manage.microsoft.com/enrollmentserver/discovery.svc"
  deviceenroller.exe /c /AutoEnrollMDM
  ```

**Advantages:**
- Works without Group Policy
- Can be integrated into deployment automation

**Disadvantages:**
- Requires scripting and automation
- Less elegant than auto-enrollment

## Enrollment Restrictions

Intune allows you to control which devices can enroll and which device types are allowed.

### Configure Enrollment Restrictions

**Portal:** Intune Admin Center → Devices → Enrollment → Enrollment restrictions

1. **Device Type Restrictions**
   - Allow/block Windows, iOS, Android, macOS
   - For AVD, typically allow **Windows only**

2. **Device Limit Restrictions**
   - Maximum devices per user (default: 5)
   - For personal AVD, increase to 10-15 if users have multiple persistent desktops
   - For pooled AVD, this setting is less relevant (devices enroll under service accounts)

3. **Platform Restrictions**
   - Minimum OS version (e.g., Windows 10 20H2 or later)
   - Block personally owned devices (recommended for AVD to prevent personal devices from enrolling)

### Best Practices for AVD

- **Device Type:** Allow Windows only
- **Ownership:** Block personal devices (set device ownership to "Corporate" during AVD deployment)
- **OS Version:** Require Windows 10 21H2 or Windows 11 22H2 minimum
- **Device Limit:** 5-10 devices per user (accommodates multiple AVD sessions)

> **Security Tip:** Use enrollment restrictions to prevent non-AVD devices from enrolling. Combine with Conditional Access policies that require "Intune compliant device" for AVD access.

## Our Environment Setup

### Configuration Summary

**Environment:** RG-Azure-VDI-01 (Azure Resource Group)

**Device Groups:**
- **avd-devices-all**: Dynamic group containing all AVD session hosts
- **avd-devices-pooled**: Dynamic group for pooled session hosts (hp-pooled-prod)
- **avd-devices-personal**: Dynamic group for personal session hosts (hp-personal-prod)

**Enrollment Method:**
- **Entra Join** with automatic MDM enrollment
- Session hosts auto-enroll on first user sign-in
- No domain controllers required

**Licensing:**
- Users assigned Microsoft 365 E3 licenses (includes Intune, AVD rights, Office 365)

**Host Pools:**
- **hp-pooled-prod1**: Non-persistent pooled desktops (Entra joined, auto-enrolled)
- **hp-personal-prod1**: Persistent personal desktops (Entra joined, auto-enrolled)

**Storage Accounts:**
- **intunescripts121125**: Azure Storage account for PowerShell scripts and Win32 app packages (used for Intune app deployment)

### Dynamic Group Queries

These queries automatically populate device groups based on session host properties:

**avd-devices-all:**
```
(device.displayName -startsWith "vm-pooled-") -or (device.displayName -startsWith "vm-personal-")
```

**avd-devices-pooled:**
```
(device.displayName -startsWith "vm-pooled-")
```

**avd-devices-personal:**
```
(device.displayName -startsWith "vm-personal-")
```

> **Note:** These queries match our naming conventions where session hosts use patterns like `vm-pooled-001`, `vm-pooled-002`, `vm-personal-011`, etc.

## Key Concepts

### MDM vs MAM

- **MDM (Mobile Device Management)**: Manages the entire device, including OS settings, policies, and applications. Used for corporate-owned devices like AVD session hosts.
- **MAM (Mobile Application Management)**: Manages specific applications and their data without controlling the entire device. Used for personal mobile devices (BYOD).

For AVD, you use **MDM only** since session hosts are corporate-owned infrastructure.

### User-Based vs Device-Based Licensing

- **User-Based**: Licenses assigned to users; devices inherit enrollment when users sign in (Intune default)
- **Device-Based**: Licenses assigned to devices directly (supported for Intune Device Licenses, rarely used)

For AVD, use **user-based licensing**. Assign Intune licenses to users, and their session hosts automatically enroll.

### Enrollment vs Registration

- **Registration**: Device identity created in Entra ID (happens during Entra join)
- **Enrollment**: Device enrolls into Intune management (happens after registration when auto-enrollment is configured)

Both happen automatically for Entra-joined AVD session hosts when auto-enrollment is enabled.

## Best Practices

### Use Entra Join for New Deployments

**Why:** Simplifies architecture, eliminates domain controllers, enables automatic Intune enrollment, and integrates seamlessly with Conditional Access.

**When to use Hybrid Join:** Only if you have strict requirements for on-premises AD group policy or legacy applications that cannot authenticate via Entra ID.

### Assign Microsoft 365 Bundles Instead of Standalone Licenses

**Why:** Microsoft 365 E3/E5 bundles include Intune, AVD access rights, Windows Enterprise, and Office 365 in a single license. This simplifies license management and reduces costs compared to purchasing standalone licenses separately.

**Cost Comparison:**
- Microsoft 365 E3: ~$36/user/month (includes everything)
- Standalone: Intune ($8) + AVD ($9) + Windows E3 ($7) + Office 365 E3 ($23) = $47/user/month

### Enable Auto-Enrollment for All Users

**Why:** Eliminates manual enrollment steps, ensures consistent policy application, and reduces administrative overhead.

**Configuration:** Entra Admin Center → Mobility (MDM and MAM) → Microsoft Intune → MDM User Scope = **All**

### Use Dynamic Device Groups

**Why:** Automatically organize session hosts based on properties (e.g., pooled vs personal, production vs test). Policies and apps can then target these groups without manual device assignment.

**Example:** Create "avd-devices-pooled" dynamic group with query `(device.displayName -startsWith "vm-pooled-")`, then assign configuration profiles to this group.

### Set Enrollment Restrictions

**Why:** Prevents unauthorized devices from enrolling, enforces minimum OS versions, and limits device sprawl.

**Recommended Restrictions:**
- Device type: Windows only
- Ownership: Corporate only (block personal devices)
- Minimum OS: Windows 10 21H2 or Windows 11 22H2
- Device limit: 10 per user

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Session host does not appear in Intune after deployment** | Auto-enrollment not configured, or user lacks Intune license | 1. Verify MDM auto-enrollment is enabled (Entra Admin Center → Mobility → Microsoft Intune → MDM User Scope = All)<br>2. Confirm user has Intune license (Entra Admin Center → Users → Licenses)<br>3. Force enrollment: `deviceenroller.exe /c /AutoEnrollMDM` |
| **Enrollment fails with "0x80180002" error** | User lacks Intune license or MAM-only license assigned | Assign full Intune license (Microsoft 365 E3/E5 or Intune Plan 1), not just MAM license |
| **Hybrid-joined session host does not enroll** | GPO not configured or not applied | 1. Verify GPO: Computer Configuration → Windows Components → MDM → "Enable automatic MDM enrollment" is enabled<br>2. Force GPO update: `gpupdate /force`<br>3. Check event log: Applications and Services → Microsoft → Windows → DeviceManagement-Enterprise-Diagnostics-Provider |
| **Policies not applying after enrollment** | Device not in targeted group, or policy conflict | 1. Verify device membership in targeted Entra ID group (Intune Admin Center → Devices → All Devices → [Device] → Groups)<br>2. Check policy status: Devices → All Devices → [Device] → Device Configuration → Monitor<br>3. Force policy sync: Devices → All Devices → [Device] → Sync |
| **Enrollment succeeds but compliance shows "Not Evaluated"** | Compliance policy not assigned or not targeting device | 1. Create compliance policy (Devices → Compliance Policies → Create Policy)<br>2. Assign to device group (avd-devices-all)<br>3. Wait up to 8 hours or force sync |
| **"Too many devices enrolled" error** | User exceeded device limit (default 5) | Increase device limit: Devices → Enrollment → Enrollment Restrictions → Device Limit Restrictions → Set limit to 10-15 |

> **Note:** Intune policy application can take up to 8 hours for initial check-in. To force immediate sync, go to Intune Admin Center → Devices → All Devices → [Device] → Sync, or run `C:\Program Files\Microsoft Intune Management Extension\Microsoft.Management.Services.IntuneWindowsAgent.exe` on the session host.

## Verification Checklist

Before deploying AVD session hosts, verify the following prerequisites:

- [ ] **Licensing**: Users have Microsoft 365 E3/E5 or Intune Plan 1 licenses assigned
- [ ] **Auto-Enrollment**: MDM auto-enrollment configured in Entra Admin Center (MDM User Scope = All)
- [ ] **Enrollment Restrictions**: Device type, ownership, and OS version restrictions configured
- [ ] **Device Groups**: Dynamic groups created for avd-devices-all, avd-devices-pooled, avd-devices-personal
- [ ] **Test Enrollment**: Deploy one test session host and verify it appears in Intune within 8 hours
- [ ] **Policy Assignment**: At least one configuration profile or compliance policy assigned to AVD device groups

Once these prerequisites are met, AVD session hosts will automatically enroll into Intune on first user sign-in, and policies will begin applying.

## Next Steps

After configuring Intune prerequisites:

1. **Device Configuration Profiles** - Deploy settings and policies to session hosts (Settings Catalog, ADMX templates)
2. **Compliance Policies** - Enforce security baselines (BitLocker, firewall, antivirus)
3. **Application Deployment** - Deploy applications via Intune (Win32 apps, Microsoft Store)
4. **Windows Update Policies** - Control update deployment (update rings, feature updates)

Proceed to the next page to configure Device Configuration Profiles for AVD session hosts.