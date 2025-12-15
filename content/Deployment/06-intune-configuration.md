---
title: Step 06 - Intune Configuration
description: Configure Win32 applications, FSLogix, and default device settings before session host deployment
published: true
date: 2025-12-14T00:00:00.000Z
tags: [Quick-Deploy, Intune, applications, Win32, FSLogix, configuration]
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 06: Intune Configuration

Prepare three layers of Intune configuration before session hosts deploy in Step 09:

1. **Win32 Applications** - Deploy evergreen apps (Chrome, Adobe, etc.) to all AVD devices
2. **FSLogix Configuration** - Set up profile container policies (see Step 07)
3. **Default Device Settings** - Configure Windows behavior, updates, and RDP settings for all AVD devices

## Prerequisites

- [ ] Intune subscription active (included with Microsoft 365 E3/E5)
- [ ] Device groups created: `avd-devices-all`, `avd-devices-pooled`, `avd-devices-personal` (from [[02-identity-setup]])
- [ ] Intune Policy Administrator and Application Manager roles

---

## Section 1: Deploy Win32 Applications

Deploy evergreen applications (Chrome, Adobe Reader, etc.) to all AVD devices using Win32 packages from the evergreen template.

**Repository:** [aidrak/intunewin32-evergreen-template](https://github.com/aidrak/intunewin32-evergreen-template)

This template provides step-by-step instructions and automation for packaging and deploying common enterprise applications as Intune Win32 apps. Refer to the repository for:

- Application packaging instructions (using IntuneWinAppUtil.exe)
- Detection rule examples for each application
- Deployment to device groups (`avd-devices-all`)
- Monitoring and troubleshooting

**Key points:**
- Applications deploy to `avd-devices-all` group (both pooled and personal)
- Applications install after device enrollment in Intune
- Pooled host applications can alternatively be pre-installed in golden image (Step 03)
- Win32 app deployment supports user-context and system-context installation

**Portal:** Intune Admin Center → Apps → Windows → + Add

See [[../Intune/application-deployment-with-intune|Application Deployment with Intune]] for detailed theory and advanced scenarios.

---

## Section 2: Configure FSLogix

FSLogix profile container configuration is handled separately in **[[07-fslogix-configuration|Step 07: FSLogix Configuration via Intune]]**.

Session hosts will receive FSLogix policies during enrollment in Intune, before VMs boot in Step 09.

---

## Section 3: Default Device Configuration Profiles

Configure Intune device configuration profiles for all AVD devices to standardize Windows behavior, security, and optimization settings.

**Portal:** Intune Admin Center → Devices → Configuration profiles → + Create profile

### Profile 1: Disable Shutdown Options

**Name:** `AVD - Disable Shutdown Options`
**Platform:** Windows
**Profile type:** Settings catalog
**Assignment:** `avd-devices-all`

| Setting | Value |
|---------|-------|
| Administrative Templates > Windows Components > File Explorer > Show hibernate in power options menu | Disabled |
| Administrative Templates > Windows Components > File Explorer > Show sleep in power options menu | Disabled |
| Administrative Templates > Start > Hide Hibernate | Enabled |
| Administrative Templates > Start > Hide Restart | Enabled |
| Administrative Templates > Start > Hide Shut Down | Enabled |
| Administrative Templates > Start > Hide Sleep | Enabled |
| Administrative Templates > Start > Hide Switch Account | Enabled |

**Purpose:** Prevents users from shutting down, restarting, or putting session hosts to sleep, ensuring sessions remain available.

---

### Profile 2: Disable RDP Shortpath (Optional)

**Name:** `AVD - Disable RDP Shortpath`
**Platform:** Windows
**Profile type:** Settings catalog
**Assignment:** `avd-devices-all`

| Setting | Value |
|---------|-------|
| Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Azure Virtual Desktop > Enable RDP Shortpath for managed networks | Disabled |
| Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Azure Virtual Desktop > RDP Shortpath > Enable RDP Shortpath for managed network using NAT traversal | Disabled |
| Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Azure Virtual Desktop > RDP Shortpath > Enable RDP Shortpath for public network using NAT traversal | Disabled |
| Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Azure Virtual Desktop > RDP Shortpath > Enable RDP Shortpath for public network using Relay (TURN) | Disabled |

**Purpose:** Disables RDP Shortpath if your environment doesn't support managed networks or requires traditional RDP connectivity. Optional—enable Shortpath for better performance on supported networks.

---

### Profile 3: Default Taskbar Layout

**Name:** `AVD - Default Taskbar Layout`
**Platform:** Windows
**Profile type:** Settings catalog
**Assignment:** `avd-devices-all`

| Setting | Value |
|---------|-------|
| Start > Allow Pinned Folder Music | The shortcut is hidden and disables the setting in the Settings app |
| Start > Start Layout | (XML below) |

**Start Layout XML:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<LayoutModificationTemplate xmlns="http://schemas.microsoft.com/Start/2014/LayoutModification" xmlns:defaultlayout="http://schemas.microsoft.com/Start/2014/FullDefaultLayout" xmlns:start="http://schemas.microsoft.com/Start/2014/StartLayout" xmlns:taskbar="http://schemas.microsoft.com/Start/2014/TaskbarLayout" Version="1">
  <CustomTaskbarLayoutCollection>
    <defaultlayout:TaskbarLayout>
      <taskbar:TaskbarPinList>
        <taskbar:DesktopApp DesktopApplicationID="Microsoft.Windows.Explorer" />
        <taskbar:UWA AppUserModelID="Microsoft.Office.WINWORD.EXE.15" />
        <taskbar:UWA AppUserModelID="Microsoft.Office.EXCEL.EXE.15" />
        <taskbar:UWA AppUserModelID="Microsoft.Office.OUTLOOK.EXE.15" />
      </taskbar:TaskbarPinList>
    </defaultlayout:TaskbarLayout>
  </CustomTaskbarLayoutCollection>
</LayoutModificationTemplate>
```

**Purpose:** Standardizes taskbar layout across all AVD devices. Pin File Explorer and Microsoft Office applications (Word, Excel, Outlook) by default.

---

### Profile 4: Microsoft 365 Auto Updates

**Name:** `AVD - M365 Auto Updates - Monthly Enterprise Channel`
**Platform:** Windows
**Profile type:** Settings catalog
**Assignment:** `avd-devices-all`

| Setting | Value |
|---------|-------|
| Microsoft Office 2016 (Machine) > Updates > Channel Name (Device) | Monthly Enterprise Channel |
| Microsoft Office 2016 (Machine) > Updates > Don't install Microsoft Teams with new installations or updates of Office | Enabled |
| Microsoft Office 2016 (Machine) > Updates > Enable Automatic Updates | Enabled |
| Microsoft Office 2016 (Machine) > Updates > Hide option to enable or disable updates | Enabled |
| Microsoft Office 2016 (Machine) > Updates > Hide Update Notifications | Enabled |
| Microsoft Office 2016 (Machine) > Updates > Update Channel | Enabled |

**Purpose:** Configures Office 365 to auto-update on pooled hosts during maintenance windows. Monthly Enterprise Channel provides stability for multi-user environments.

---

### Profile 5: Windows Update Settings

**Name:** `AVD - Windows Update - Settings`
**Platform:** Windows
**Profile type:** Settings catalog
**Assignment:** `avd-devices-all`

| Setting | Value |
|---------|-------|
| Windows Update For Business > Active Hours End | 4 (4 AM) |
| Windows Update For Business > Active Hours Max Range | 17 hours |
| Windows Update For Business > Active Hours Start | 9 (9 AM) |
| Windows Update For Business > Block "Pause Updates" ability | Block |
| Windows Update For Business > Configure Deadline Grace Period | 2 days |
| Windows Update For Business > Defer Quality Updates Period (Days) | 3 |

**Purpose:** Ensures critical security updates install outside business hours (9 AM - 4 AM), with a 3-day grace period for quality updates to be tested before deployment.

---

## Monitoring and Verification

**Portal:** Intune Admin Center → Devices → Configuration profiles → [Profile Name] → Device and user check-in status

Expected status after 15-30 minutes:
- **Succeeded:** Policy successfully deployed to device
- **Error:** Device failed to receive policy (check device connectivity and Intune enrollment)
- **Conflict:** Device has conflicting policies (resolve by removing duplicates)

### Verify on Session Host

```powershell
# Check Intune policies applied (RDP as admin)
Get-Item "Registry::HKEY_LOCAL_MACHINE\Software\Microsoft\IntuneManagementExtension\Win32Apps"

# Check Windows Update settings
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update"

# Verify Office update channel
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Office\ClickToRun\Configuration"
```

---

## Next Steps

All Intune configuration is now in place:
1. ✅ Win32 applications configured (via evergreen template)
2. ✅ Default device settings configured (Intune profiles)
3. ✅ FSLogix policies ready (Step 07)

Proceed to **[[07-fslogix-configuration|Step 07: FSLogix Configuration via Intune]]** to finalize profile container settings.

---

## Related References

- [[../Intune/application-deployment-with-intune|Application Deployment with Intune]] - Win32 app theory and advanced scenarios
- [[../Images/golden-image-process|Golden Image Process]] - Pre-installing apps on pooled hosts
- [[../Intune/device-configuration|Device Configuration Profiles]] - Managing Intune settings at scale
