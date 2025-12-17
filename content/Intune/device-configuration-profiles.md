---
title: Device Configuration Profiles
description: 
published: true
date: 2025-12-14T04:53:13.302Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:41.100Z
---

# Device Configuration Profiles

Device configuration profiles are the primary method for deploying settings, policies, and configurations to AVD session hosts managed by Intune. Unlike traditional Group Policy (GPO), Intune configuration profiles are cloud-native, declarative, and enforce settings continuously rather than only at startup or user logon. This page covers the types of configuration profiles, AVD-specific settings, targeting strategies, and troubleshooting approaches for managing session hosts at scale.

## What are Configuration Profiles

A configuration profile is a collection of device or user settings that Intune applies to managed endpoints. Profiles can control:

- **Operating System Settings**: Windows features, security options, power management
- **Application Configurations**: FSLogix, OneDrive, Microsoft Edge, Office 365
- **Network Settings**: VPN, Wi-Fi, proxy configurations
- **Security Baselines**: Microsoft-recommended security settings for Windows 10/11
- **Custom Settings**: OMA-URI configurations for advanced or unsupported settings

Configuration profiles differ from Group Policy in several key ways:

**Group Policy (GPO):**
- Requires domain controllers
- Applied only at startup, user logon, or manual refresh (gpupdate)
- Settings can be overridden by local administrators
- Requires ADMX templates to be distributed

**Intune Configuration Profiles:**
- Cloud-native, no domain controllers required
- Continuously enforced (re-applied every 8 hours, or on-demand via sync)
- Settings cannot be overridden locally (enforced by Intune agent)
- Settings Catalog provides modern UI for all Windows settings

For AVD environments, configuration profiles are critical for managing pooled (non-persistent) desktops where users do not have dedicated session hosts. Profiles ensure consistent configuration even when users connect to different session hosts in the pool.

## Profile Types

Intune offers several types of configuration profiles. The modern approach is to use **Settings Catalog** for all new policies, but legacy profile types remain for backward compatibility.

### Settings Catalog (Recommended)

The Settings Catalog is the modern, unified interface for configuring Windows settings in Intune. It replaces legacy ADMX templates and device restriction templates with a searchable catalog of over 4,000 settings.

**Advantages:**
- **Searchable**: Find settings by name or description (e.g., search "FSLogix" to find all FSLogix-related settings)
- **Organized**: Settings grouped by category (System, User Rights, Windows Components, etc.)
- **Modern**: Continuously updated with new Windows settings from each feature update
- **Conflict Resolution**: Built-in conflict detection and reporting
- **Cloud-Native**: No ADMX files to distribute

**When to Use:**
- All new configuration profiles for AVD
- FSLogix settings
- RDP properties
- Windows Update settings
- OneDrive Known Folder Move

**Example Use Cases:**
- Configure FSLogix profile container settings
- Set timezone redirection for RDP sessions
- Enable OneDrive Files On-Demand and Known Folder Move
- Configure Windows Defender settings

**Portal:** Intune Admin Center → Devices → Configuration Profiles → Create → New Policy → Windows 10 and later → Settings Catalog

### Templates (Legacy ADMX)

Templates are legacy configuration profiles based on ADMX (Administrative Template) files. Microsoft is migrating all template settings to the Settings Catalog, but templates remain for backward compatibility.

**Profile Types:**
- **Administrative Templates**: ADMX-based policies (same as GPO ADMX templates)
- **Device Restrictions**: Simplified UI for common device settings
- **Endpoint Protection**: Antivirus, firewall, BitLocker settings
- **Identity Protection**: Windows Hello for Business, Credential Guard
- **Kiosk**: Single-app or multi-app kiosk mode (rarely used for AVD)

**When to Use:**
- Legacy deployments with existing ADMX template profiles
- Transitioning from GPO to Intune (import ADMX settings)
- Specific features not yet available in Settings Catalog (rare)

**Migration Recommendation:** Migrate all template-based profiles to Settings Catalog. Microsoft is deprecating templates in favor of Settings Catalog.

### Custom Profiles (OMA-URI)

Custom profiles allow you to configure settings using OMA-URI (Open Mobile Alliance Uniform Resource Identifier) paths. This is useful for advanced configurations not available in Settings Catalog or Templates.

**When to Use:**
- Configuring settings not available in Settings Catalog (very rare as of 2024)
- Testing new Windows features before Settings Catalog support is added
- Applying registry-based settings via CSP (Configuration Service Provider)

**Example OMA-URI:**
```
OMA-URI: ./Device/Vendor/MSFT/Policy/Config/FSLogix~Policy~FSLogix~Profiles/ProfilesProfileType
Data Type: Integer
Value: 0 (Redirected Profile)
```

**Recommendation:** Avoid custom profiles unless absolutely necessary. Settings Catalog covers 99% of common use cases and is easier to manage.

## AVD-Specific Configuration Settings

The following settings are critical for AVD environments. All of these can be configured via Settings Catalog.

### FSLogix Profile Container Settings

FSLogix is the Microsoft-recommended solution for user profile management in AVD. These settings should be deployed to all pooled AVD session hosts.

**Portal:** Intune Admin Center → Devices → Configuration Profiles → Create → Settings Catalog

**Settings to Configure:**

| Setting | Category | Value | Purpose |
|---------|----------|-------|---------|
| **Enabled** | FSLogix → Profiles | Enabled | Activates FSLogix profile containers |
| **VHDLocations** | FSLogix → Profiles | `\\intunescripts121125.file.core.windows.net\profiles` | Path to Azure Files share for profiles |
| **SizeInMBs** | FSLogix → Profiles | 30000 (30 GB) | Maximum profile container size |
| **IsDynamic** | FSLogix → Profiles | Enabled | Enable dynamic VHD expansion (saves storage) |
| **VolumeType** | FSLogix → Profiles | VHDX | Use VHDX format (more resilient than VHD) |
| **LockedRetryCount** | FSLogix → Profiles | 3 | Retry attempts if profile is locked |
| **LockedRetryInterval** | FSLogix → Profiles | 15 (seconds) | Wait time between retry attempts |
| **DeleteLocalProfileWhenVHDShouldApply** | FSLogix → Profiles | Enabled | Clean up local profiles on FSLogix-enabled hosts |

**Example Configuration:**
1. Create new Settings Catalog profile: "AVD - FSLogix Profile Containers"
2. Search for "FSLogix" in Settings Catalog
3. Add all settings above
4. Assign to **avd-devices-pooled** group

> **Note:** FSLogix settings apply only to pooled AVD environments where users do not have persistent desktops. Personal AVD hosts can use FSLogix for profile portability, but it's less critical since users have dedicated session hosts.

### RDP Properties

RDP properties control the user experience for AVD sessions. These settings override host pool RDP properties configured in Azure Portal.

**Settings to Configure:**

| Setting | Category | Value | Purpose |
|---------|----------|-------|---------|
| **Timezone Redirection** | Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host → Device and Resource Redirection | Enabled | Use client's timezone in session |
| **Audio Redirection** | Remote Desktop Services → Device and Resource Redirection | Redirect to client | Play audio on client device |
| **Clipboard Redirection** | Remote Desktop Services → Device and Resource Redirection | Enabled | Allow copy/paste between client and session |
| **Printer Redirection** | Remote Desktop Services → Device and Resource Redirection | Enabled | Map client printers in session |
| **Drive Redirection** | Remote Desktop Services → Device and Resource Redirection | Disabled | Block drive mapping (security best practice) |
| **Session Time Limits - Disconnected Sessions** | Remote Desktop Services → Session Time Limits | 8 hours | Automatically log off disconnected sessions |
| **Session Time Limits - Active Sessions** | Remote Desktop Services → Session Time Limits | 24 hours | Maximum active session time |

**Example Configuration:**
1. Create new Settings Catalog profile: "AVD - RDP Properties"
2. Search for "Remote Desktop Services" in Settings Catalog
3. Add settings above
4. Assign to **avd-devices-all** group

### Windows Update Settings

Control Windows Update behavior for AVD session hosts. Note that for pooled AVD, updates are typically applied to the golden image, not individual session hosts.

**Settings to Configure:**

| Setting | Category | Value | Purpose |
|---------|----------|-------|---------|
| **Configure Automatic Updates** | Administrative Templates → Windows Components → Windows Update | 4 - Auto download and schedule install | Automatic update installation |
| **Scheduled Install Day** | Windows Update | 0 (Every day) | Install updates daily |
| **Scheduled Install Time** | Windows Update | 03:00 | Install at 3 AM (outside business hours) |
| **No Auto-Restart with Logged On Users** | Windows Update | Enabled | Prevent automatic restarts during user sessions |
| **Auto-Restart Notification Schedule** | Windows Update | 2 hours | Warn users 2 hours before restart |

**Recommendation for Pooled AVD:** Instead of applying updates to individual session hosts, update the golden image and redeploy session hosts. This ensures consistency and avoids update-related issues during user sessions.

**Recommendation for Personal AVD:** Use Windows Update for Business (covered in Page 5: Windows Update Policies) to control update deployment with pilot and production rings.

### OneDrive Known Folder Move

Known Folder Move (KFM) redirects user's Desktop, Documents, and Pictures folders to OneDrive, providing automatic backup and cross-device sync.

**Settings to Configure:**

| Setting | Category | Value | Purpose |
|---------|----------|-------|---------|
| **Silently move Windows known folders to OneDrive** | Administrative Templates → OneDrive | Enabled | Automatically redirect Desktop, Documents, Pictures |
| **Tenant ID** | OneDrive | Your Entra Tenant ID | Link to organization's OneDrive |
| **Prevent users from redirecting their Windows known folders to their PC** | OneDrive | Enabled | Block users from disabling KFM |
| **Prompt users to move Windows known folders to OneDrive** | OneDrive | Disabled | Silent redirect (no user prompts) |

**Example Configuration:**
1. Create new Settings Catalog profile: "AVD - OneDrive Known Folder Move"
2. Search for "OneDrive" in Settings Catalog
3. Add settings above, set Tenant ID to your Entra tenant ID
4. Assign to **avd-devices-all** group

> **Best Practice:** Known Folder Move works seamlessly with FSLogix. OneDrive syncs files to the cloud, while FSLogix stores user profile data (registry, app settings). This combination provides fast logon (FSLogix) and data protection (OneDrive).

### Timezone Redirection

Timezone redirection ensures that the AVD session uses the client's local timezone, not the Azure datacenter's timezone.

**Settings to Configure:**

| Setting | Category | Value | Purpose |
|---------|----------|-------|---------|
| **Allow time zone redirection** | Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host → Device and Resource Redirection | Enabled | Use client's timezone in session |

**Example Configuration:**
1. Create new Settings Catalog profile: "AVD - Timezone Redirection"
2. Search for "time zone redirection" in Settings Catalog
3. Enable setting
4. Assign to **avd-devices-all** group

**Impact:** Users see correct local time in their AVD session, which is critical for applications like Outlook calendar, time-stamped documents, and scheduling tools.

## Targeting with Device Groups

Configuration profiles can be assigned to users or devices. For AVD, **device-based assignment** is recommended because settings apply consistently regardless of which user logs in.

### Dynamic Device Groups

Use Entra ID dynamic groups to automatically organize session hosts based on naming conventions or device properties.

**Our Environment Groups:**

**avd-devices-all** (All AVD session hosts)
```
(device.displayName -startsWith "vm-pooled-") -or (device.displayName -startsWith "vm-personal-")
```

**avd-devices-pooled** (Pooled session hosts only)
```
(device.displayName -startsWith "vm-pooled-")
```

**avd-devices-personal** (Personal session hosts only)
```
(device.displayName -startsWith "vm-personal-")
```

### Profile Assignment Strategy

**Assign to avd-devices-all:**
- RDP properties (applies to all AVD types)
- Timezone redirection (applies to all AVD types)
- OneDrive Known Folder Move (applies to all AVD types)
- Security baselines (applies to all AVD types)

**Assign to avd-devices-pooled:**
- FSLogix profile container settings (pooled only, not needed for personal with local profiles)
- Windows Update settings (if updating individual hosts; otherwise update golden image)

**Assign to avd-devices-personal:**
- Windows Update rings (personal desktops can be updated via Intune)
- User-specific application configurations

### Assignment Filters (Advanced)

Assignment filters allow fine-grained targeting based on device properties beyond group membership.

**Example Use Cases:**
- Assign configuration only to session hosts in specific Azure regions
- Differentiate between production and test environments
- Apply different settings based on OS version (Windows 10 vs Windows 11)

**Portal:** Intune Admin Center → Tenant Administration → Filters → Create → Windows

**Example Filter:**
```
(device.osVersion -startsWith "10.0.22") -and (device.deviceName -contains "prod")
```

This filter targets Windows 11 (OS version 10.0.22xxx) production session hosts.

## Conflict Resolution

When multiple configuration profiles configure the same setting with different values, Intune uses the following conflict resolution rules:

### Conflict Resolution Rules

1. **Most Restrictive Wins**: If one profile sets a value to "Enabled" and another sets it to "Disabled", the most restrictive (Enabled) applies
2. **Last Write Wins (Same Priority)**: If two profiles have the same priority and configure the same setting, the last-applied profile wins (non-deterministic)
3. **Explicit Priority**: Assign priority values to profiles (0 = highest priority)

### Avoid Conflicts

**Best Practices:**
- **One Setting, One Profile**: Configure each setting in only one profile
- **Hierarchical Profiles**: Use separate profiles for different scopes (e.g., "AVD - Global Settings", "AVD - Pooled Settings")
- **Descriptive Naming**: Name profiles clearly (e.g., "AVD - FSLogix Profile Containers" instead of "Profile 1")
- **Document Assignments**: Maintain a spreadsheet or wiki page documenting which profiles target which groups

### Detecting Conflicts

**Portal:** Intune Admin Center → Devices → All Devices → [Select Device] → Device Configuration

The Device Configuration page shows:
- **Profile Assignment Status**: Success, Error, Conflict
- **Settings Configured**: Which profiles applied which settings
- **Conflicts**: Settings with conflicting values from multiple profiles

**Resolution:** If a conflict is detected, edit one of the profiles to remove the conflicting setting, or assign the profiles to mutually exclusive groups.

## Reporting and Troubleshooting

### Profile Deployment Status

**Portal:** Intune Admin Center → Devices → Configuration Profiles → [Select Profile] → Monitor → Device Status

This report shows:
- **Succeeded**: Devices that successfully applied the profile
- **Error**: Devices with errors (setting not supported, permission issue, etc.)
- **Conflict**: Devices with conflicting settings from multiple profiles
- **Not Applicable**: Devices where profile does not apply (e.g., iOS profile assigned to Windows device)

**Export Report:** Click "Export" to download CSV of device status.

### Device Configuration View

**Portal:** Intune Admin Center → Devices → All Devices → [Select Device] → Device Configuration

This view shows all configuration profiles assigned to the device and their status:
- **Profile Name**: Name of assigned profile
- **Profile Type**: Settings Catalog, Administrative Template, etc.
- **State**: Success, Error, Conflict, Pending
- **Last Check-In**: Timestamp of last policy sync

Click on a profile to see detailed error messages.

### Common Errors and Solutions

| Error Code | Message | Cause | Solution |
|------------|---------|-------|----------|
| **-2016281112 (0x87D1FDE8)** | Remediation failed | Setting not supported on this OS version | Verify OS version supports the setting (e.g., Windows 11-only settings fail on Windows 10) |
| **-2016281113 (0x87D1FDE7)** | Setting not applicable | Setting applies to different device type (e.g., mobile vs desktop) | Remove setting from profile or assign profile to correct device type |
| **-2016345060 (0x87D13B9C)** | Conflict detected | Multiple profiles configure the same setting with different values | Edit one profile to remove the conflicting setting or adjust priority |
| **-2016345112 (0x87D13B68)** | Policy processing failed | Syntax error in OMA-URI or registry value | Verify OMA-URI path and data type are correct |
| **0x80004005** | Unspecified error | Permissions issue, corrupted profile, or device not communicating with Intune | Force device sync (Devices → [Device] → Sync), check device can reach `*.manage.microsoft.com` |

### Force Policy Sync

If a profile shows "Pending" for more than 8 hours, force a sync:

**Portal Method:**
1. Intune Admin Center → Devices → All Devices → [Select Device]
2. Click **Sync** at the top toolbar
3. Wait 5-10 minutes, refresh Device Configuration page

**Session Host Method (via RDP):**
1. Open Company Portal app (search for "Company Portal" in Start menu)
2. Click **Settings** (gear icon)
3. Click **Sync**

**PowerShell Method (via RDP or VM Run Command):**
```powershell
# Force policy sync
$session = New-CimSession
$syncML = @"
<SyncML xmlns='SYNCML:SYNCML1.2'>
  <SyncBody>
    <Alert>
      <CmdID>1</CmdID>
      <Data>1201</Data>
    </Alert>
    <Final/>
  </SyncBody>
</SyncML>
"@
Invoke-CimMethod -CimSession $session -Namespace root\cimv2\mdm\dmmap -ClassName MDM_DeviceManagement_Provider -MethodName SyncMLRequest -Arguments @{SyncML = $syncML}
```

### Intune Diagnostic Logs

Collect detailed logs for Microsoft support:

**Session Host Method (via RDP):**
1. Open Company Portal app
2. Click **Settings** (gear icon)
3. Click **Export Logs**
4. Save ZIP file to local desktop, then upload to support ticket

**Event Viewer Method (via RDP):**
1. Open Event Viewer (eventvwr.msc)
2. Navigate to: Applications and Services → Microsoft → Windows → DeviceManagement-Enterprise-Diagnostics-Provider → Admin
3. Look for Event ID 1000 (policy applied) or Event ID 2000 (policy error)

## Best Practices

### Use Settings Catalog for All New Profiles

**Why:** Settings Catalog is the modern, searchable, cloud-native approach. Templates and ADMX are legacy and will be deprecated.

**Migration Path:** Identify all template-based profiles, recreate them in Settings Catalog, assign to same groups, then delete template profiles.

### Create Separate Profiles for Different Scopes

**Why:** Easier to troubleshoot, reduces conflicts, and allows granular assignment.

**Example Structure:**
- **AVD - Global Settings** (assigned to avd-devices-all): RDP properties, timezone redirection, security baselines
- **AVD - Pooled FSLogix** (assigned to avd-devices-pooled): FSLogix profile container settings
- **AVD - Personal Updates** (assigned to avd-devices-personal): Windows Update rings

### Use Descriptive Naming Conventions

**Why:** Makes it easy to identify profile purpose and target group.

**Naming Pattern:** `[Scope] - [Category] - [Purpose]`

**Examples:**
- AVD - Global - RDP Properties
- AVD - Pooled - FSLogix Profile Containers
- AVD - Personal - Windows Update Ring - Production

### Assign Profiles to Device Groups, Not User Groups

**Why:** For AVD, device-based assignment ensures settings apply regardless of which user logs in. This is critical for pooled environments where users do not have dedicated session hosts.

**Exception:** User-specific settings (e.g., OneDrive KFM for specific departments) can be assigned to user groups.

### Test Profiles in Pilot Group Before Broad Deployment

**Why:** Prevents production outages from misconfigured settings.

**Workflow:**
1. Create "avd-devices-pilot" dynamic group (e.g., session hosts with "test" in display name)
2. Assign new configuration profile to pilot group
3. Monitor for 48 hours, check for errors
4. If successful, reassign to production group (avd-devices-all or avd-devices-pooled)

### Monitor Profile Deployment Weekly

**Why:** Catch errors early, ensure compliance, and identify devices that are not receiving policies.

**Weekly Checklist:**
- Review profile deployment status for all profiles (Devices → Configuration Profiles → [Profile] → Device Status)
- Investigate any devices in "Error" or "Conflict" state
- Verify new session hosts appear in device groups and receive policies within 24 hours

## Summary

Device configuration profiles are essential for managing AVD session hosts at scale. By using Settings Catalog, targeting device groups, and monitoring deployment status, you can ensure consistent configuration across all session hosts without the complexity of domain controllers and Group Policy.

**Key Takeaways:**
- Use **Settings Catalog** for all new profiles (modern, searchable, cloud-native)
- Configure **FSLogix settings** for pooled AVD environments (profile portability)
- Configure **RDP properties** for user experience (timezone redirection, clipboard, audio)
- Assign profiles to **device groups** (avd-devices-pooled, avd-devices-personal) for consistent application
- Monitor **deployment status** weekly to catch errors early

## Next Steps

After configuring device configuration profiles:

1. **Compliance Policies** - Enforce security baselines (BitLocker, firewall, antivirus) and integrate with Conditional Access
2. **Application Deployment** - Deploy Win32 apps and Microsoft Store apps via Intune (Page 4)
3. **Windows Update Policies** - Control update deployment with update rings and feature update policies (Page 5)

Proceed to the next page to configure Compliance Policies for AVD session hosts.