---
title: Application Deployment with Intune
description: 
published: true
date: 2025-12-14T04:53:09.838Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:37.667Z
---

# Application Deployment with Intune

Microsoft Intune provides centralized application deployment capabilities for managed devices, including AVD session hosts. While application deployment via Intune works seamlessly for personal (persistent) AVD environments, pooled (non-persistent) environments should primarily rely on applications installed in the golden image rather than Intune deployment. This page covers Intune app types, Win32 app packaging, assignment strategies, and best practices for both pooled and personal AVD deployments.

## App Types in Intune

Intune supports multiple app types, each suited for different deployment scenarios. For AVD session hosts running Windows 10/11, the most common types are Win32 apps and Microsoft Store apps.

### Microsoft Store Apps (Modern)

**What:** Applications distributed via Microsoft Store for Business or Microsoft Store (consumer)

**Format:** MSIX, MSIX Bundle, or Appx packages

**Examples:**
- Microsoft To Do
- Microsoft Whiteboard
- Netflix (consumer apps)
- Custom line-of-business MSIX apps

**Deployment Method:**
1. Intune Admin Center → Apps → Windows → Add → Microsoft Store app (new)
2. Search for app in Microsoft Store
3. Select app, assign to device or user group
4. Intune automatically downloads and installs app from Microsoft Store

**Advantages:**
- Simple deployment (no packaging required)
- Automatic updates via Microsoft Store
- Clean uninstall (no leftover files or registry keys)

**Disadvantages:**
- Limited to apps available in Microsoft Store
- Requires internet connectivity for installation
- Not suitable for traditional desktop apps (.exe, .msi)

**Best For:**
- Modern apps (UWP, MSIX)
- Apps available in Microsoft Store
- Personal AVD deployments (automatic updates)

**AVD Recommendation:** Use for modern apps on personal AVD hosts. Avoid for pooled AVD (install in golden image instead).

### Win32 Apps (Most Common)

**What:** Traditional Windows desktop applications (.exe, .msi)

**Format:** .intunewin package (created with IntuneWinAppUtil.exe)

**Examples:**
- Adobe Acrobat Reader
- Google Chrome
- Zoom
- Custom line-of-business apps

**Deployment Method:**
1. Package app with IntuneWinAppUtil.exe
2. Upload .intunewin package to Intune
3. Configure detection rules, requirements, install/uninstall commands
4. Assign to device or user group
5. Intune deploys app via Intune Management Extension (IME)

**Advantages:**
- Supports all Windows desktop apps (.exe, .msi)
- Flexible installation (silent install, custom scripts)
- Dependencies and supersedence support
- Full control over install/uninstall behavior

**Disadvantages:**
- Requires packaging with IntuneWinAppUtil.exe
- More complex than Microsoft Store apps
- Manual updates (no automatic update mechanism)

**Best For:**
- Traditional desktop apps (.exe, .msi)
- Custom line-of-business apps
- Apps requiring custom install scripts
- Apps not available in Microsoft Store

**AVD Recommendation:** Use for personal AVD hosts only. For pooled AVD, install apps in golden image (faster logon, consistent experience).

### MSIX / MSIX Bundle

**What:** Modern Windows app package format (successor to Appx)

**Format:** .msix or .msixbundle files

**Examples:**
- Custom line-of-business apps packaged as MSIX
- Apps from Microsoft Store for Business
- Enterprise apps distributed internally

**Deployment Method:**
1. Intune Admin Center → Apps → Windows → Add → Line-of-business app
2. Upload .msix or .msixbundle file
3. Assign to device or user group
4. Intune installs app via MSIX installer

**Advantages:**
- Clean install/uninstall (containerized)
- Supports modern Windows features (app attach, MSIX app attach)
- Can be signed with enterprise certificate

**Disadvantages:**
- Requires repackaging traditional apps to MSIX format
- Not all apps can be converted to MSIX (kernel drivers, services)
- Limited ISV support (most vendors still distribute .exe/.msi)

**Best For:**
- Custom line-of-business apps built as MSIX
- Apps requiring clean install/uninstall
- Future AVD deployments with MSIX app attach

**AVD Recommendation:** Use MSIX app attach for pooled AVD (attach/detach apps dynamically). For personal AVD, standard Win32 apps are sufficient.

### Line of Business Apps (MSI)

**What:** Windows Installer packages (.msi)

**Format:** .msi files

**Examples:**
- Custom enterprise apps distributed as MSI
- Third-party apps with MSI installers

**Deployment Method:**
1. Intune Admin Center → Apps → Windows → Add → Line-of-business app
2. Upload .msi file
3. Configure install command (e.g., `msiexec /i app.msi /quiet`)
4. Assign to device or user group
5. Intune installs app via Windows Installer

**Advantages:**
- Simple deployment for MSI-based apps
- Standard Windows installation method

**Disadvantages:**
- Limited to MSI format (no .exe support)
- Less flexible than Win32 apps (no custom scripts)

**Best For:**
- Simple MSI-based apps
- Apps without complex install requirements

**AVD Recommendation:** Use Win32 app type instead (supports both .exe and .msi). Line-of-business app type is legacy.

### Web Links

**What:** Shortcuts to web applications (displayed in Company Portal or Start menu)

**Format:** URLs

**Examples:**
- SharePoint sites
- Internal web apps
- SaaS applications (Salesforce, Workday)

**Deployment Method:**
1. Intune Admin Center → Apps → Windows → Add → Web link
2. Enter URL and display name
3. Assign to user group
4. Link appears in Company Portal or Start menu

**Advantages:**
- No installation required
- Instant access to web apps
- Easy to update (change URL)

**Disadvantages:**
- Not a true application (just a shortcut)
- Requires browser access

**Best For:**
- Web-based SaaS applications
- Internal web portals

**AVD Recommendation:** Use for web apps. Users can also bookmark URLs in browser, so this is optional.

## Win32 App Packaging

Win32 apps are the most common app type for AVD deployments. Packaging Win32 apps requires the **IntuneWinAppUtil.exe** tool to create .intunewin packages.

### Packaging Process

**Step 1: Download IntuneWinAppUtil.exe**

Download from Microsoft: https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool

**Step 2: Prepare App Source Files**

Create a folder containing:
- App installer (.exe or .msi)
- Any dependencies (DLLs, config files)
- Install script (optional, for custom install logic)

**Example Folder Structure:**
```
C:\AppPackaging\AdobeReader\
  ├── AcroRdrDC2300820360_en_US.exe
  └── install.ps1 (optional)
```

**Step 3: Package with IntuneWinAppUtil.exe**

Run IntuneWinAppUtil.exe from command line:

```powershell
.\IntuneWinAppUtil.exe `
  -c "C:\AppPackaging\AdobeReader" `
  -s "AcroRdrDC2300820360_en_US.exe" `
  -o "C:\AppPackaging\Output"
```

**Parameters:**
- `-c`: Source folder (folder containing app installer)
- `-s`: Setup file (installer filename)
- `-o`: Output folder (where .intunewin file will be created)

**Output:** `AcroRdrDC2300820360_en_US.intunewin` (ready to upload to Intune)

**Step 4: Upload to Intune**

1. Intune Admin Center → Apps → Windows → Add → Windows app (Win32)
2. Click "Select app package file", upload .intunewin file
3. Configure app information (name, description, publisher)
4. Configure install/uninstall commands
5. Configure detection rules
6. Configure requirements
7. Assign to device or user group

### Detection Rules

Detection rules determine if an app is already installed. Intune uses detection rules to avoid reinstalling apps unnecessarily.

**Detection Rule Types:**

**1. File or Folder Detection**

Checks if a file or folder exists at a specified path.

**Example (Adobe Reader):**
- Path: `C:\Program Files\Adobe\Acrobat DC\Acrobat`
- File: `Acrobat.exe`
- Detection Method: File exists

**When to Use:** Simple apps with predictable install locations.

**2. Registry Detection**

Checks if a registry key or value exists.

**Example (Adobe Reader):**
- Path: `HKEY_LOCAL_MACHINE\SOFTWARE\Adobe\Acrobat Reader\DC`
- Value: `Version`
- Detection Method: String comparison (equals `23.008.20360`)

**When to Use:** Apps that register version info in registry (most MSI-based apps).

**3. Script Detection (PowerShell)**

Runs a PowerShell script to determine if app is installed. Script must output `$true` if installed, `$false` if not installed.

**Example (Adobe Reader):**
```powershell
$installed = Get-ItemProperty -Path "HKLM:\SOFTWARE\Adobe\Acrobat Reader\DC" -Name "Version" -ErrorAction SilentlyContinue
if ($installed.Version -eq "23.008.20360") {
    Write-Output "Installed"
    exit 0
} else {
    exit 1
}
```

**When to Use:** Complex detection logic (check version, check multiple files, check service status).

**Best Practice:** Use script detection for maximum flexibility. Registry detection is simpler but may not catch all scenarios.

### Requirements

Requirements define prerequisites that devices must meet before app installation.

**Common Requirements:**

| Requirement | Example | Purpose |
|-------------|---------|---------|
| **Operating System Architecture** | 64-bit (x64) | Ensure app is compatible with OS architecture |
| **Minimum OS Version** | Windows 10 21H2 (10.0.19044) | Ensure app is compatible with OS version |
| **Disk Space** | 500 MB | Ensure device has enough disk space |
| **Memory** | 4 GB | Ensure device has enough RAM |
| **Processor** | 2 cores | Ensure device has sufficient CPU |

**Portal:** Intune Admin Center → Apps → [App] → Properties → Requirements

**Best Practice:** Set realistic requirements based on app vendor recommendations. Overly strict requirements (e.g., 32 GB RAM for a simple app) can prevent installation on valid devices.

### Dependencies and Supersedence

**Dependencies:** Apps that must be installed before the current app

**Example:** App A requires .NET Framework 4.8
- App A has dependency: Microsoft .NET Framework 4.8
- Intune installs .NET Framework 4.8 first, then installs App A

**Supersedence:** Apps that replace older versions

**Example:** Adobe Reader DC 2023 supersedes Adobe Reader DC 2022
- Assign Adobe Reader DC 2023 with supersedence relationship to Adobe Reader DC 2022
- Intune uninstalls Adobe Reader DC 2022, then installs Adobe Reader DC 2023

**Portal:** Intune Admin Center → Apps → [App] → Properties → Dependencies / Supersedence

**Best Practice:** Use dependencies for apps with prerequisites (e.g., Visual C++ Runtime, .NET Framework). Use supersedence to simplify app updates (uninstall old version, install new version).

## Assignment Types

Intune apps can be assigned to users or devices with different intent levels.

### Required (Auto-Install)

**Behavior:** App installs automatically on assigned devices/users, no user interaction required

**When to Use:**
- Core business apps (Microsoft 365 Apps, Adobe Reader)
- Security tools (antivirus, VPN client)
- Apps needed by all users

**Example:** Assign Microsoft 365 Apps as "Required" to all AVD session hosts

**Portal:** Intune Admin Center → Apps → [App] → Properties → Assignments → Add Group → Required

**Installation Timeline:**
- Device assignment: Installs at next device check-in (every 8 hours)
- User assignment: Installs at user's first logon

**Best Practice:** Use "Required" for essential apps. Avoid overusing "Required" (slows down logon, increases storage).

### Available (Company Portal)

**Behavior:** App appears in Company Portal; user can install on-demand

**When to Use:**
- Optional apps (productivity tools, utilities)
- Department-specific apps (only some users need)
- Apps users can choose to install

**Example:** Assign Adobe Photoshop as "Available" to Design department users

**Portal:** Intune Admin Center → Apps → [App] → Properties → Assignments → Add Group → Available for enrolled devices

**Installation Timeline:** User installs from Company Portal when needed

**Best Practice:** Use "Available" for optional apps. Reduces deployment overhead and gives users control.

### Uninstall

**Behavior:** App is automatically uninstalled from assigned devices/users

**When to Use:**
- Removing deprecated apps (e.g., old version superseded by new version)
- Removing apps from specific departments (e.g., remove Photoshop from non-design users)

**Example:** Assign Adobe Reader DC 2022 as "Uninstall" after deploying Adobe Reader DC 2023

**Portal:** Intune Admin Center → Apps → [App] → Properties → Assignments → Add Group → Uninstall

**Best Practice:** Use "Uninstall" sparingly. Test uninstall command to ensure clean removal (no leftover files/registry keys).

## App Deployment Strategy for AVD

### Pooled AVD (Non-Persistent)

**Recommendation:** Install apps in golden image, **not via Intune**

**Why:**
- **Faster Logon**: Apps are pre-installed in image; no installation at logon
- **Consistent Experience**: All users get same apps; no variability
- **Reduced Network Traffic**: No repeated downloads of same app to each session host
- **Lower Storage**: Single copy of app in image, not duplicated per session host

**When to Use Intune for Pooled AVD:**
- **Emergency Patches**: Critical security update needed immediately (deploy via Intune, then update golden image)
- **User-Specific Apps**: Apps needed by only a subset of users (assign as "Available" in Company Portal)
- **Testing**: Pilot new apps before adding to golden image

**Best Practice:** Maintain a curated list of apps in golden image. Use Intune only for exceptions.

**Example Golden Image App List:**
- Microsoft 365 Apps (Word, Excel, PowerPoint, Outlook)
- Adobe Acrobat Reader DC
- Google Chrome
- Microsoft Teams
- Zoom
- VPN client

**Update Process:**
1. Update golden image with new app versions monthly
2. Redeploy session hosts from updated golden image
3. Drain old session hosts (block new sessions, wait for existing sessions to end)
4. Delete old session hosts

### Personal AVD (Persistent)

**Recommendation:** Deploy apps via Intune as "Required" or "Available"

**Why:**
- **Automatic Updates**: Intune can deploy new app versions automatically
- **User-Specific Apps**: Users get apps specific to their role/department
- **Flexible Deployment**: Add/remove apps without redeploying session hosts

**Assignment Strategy:**
- **Core Apps**: Assign as "Required" (Microsoft 365 Apps, Adobe Reader, VPN client)
- **Optional Apps**: Assign as "Available" (Photoshop, Visio, Project)
- **Department Apps**: Assign to specific user groups (Finance users get QuickBooks, Design users get Adobe Creative Suite)

**Best Practice:** Use Intune for all app deployment on personal AVD. Update golden image with core apps only (OS, FSLogix, Intune agent).

**Example App Assignment:**
- **All Users (Required)**: Microsoft 365 Apps, Adobe Reader, Google Chrome, VPN client
- **Design Department (Available)**: Adobe Photoshop, Adobe Illustrator, Adobe InDesign
- **Finance Department (Available)**: QuickBooks, SAP GUI

### Microsoft 365 Apps Deployment

Microsoft 365 Apps (Office suite) is the most common app deployed to AVD session hosts. Intune provides a dedicated app type for Microsoft 365 Apps.

**Deployment Method:**

1. **Intune Admin Center → Apps → Windows → Add → Microsoft 365 Apps**
2. **Configure App Suite**:
   - Select apps: Word, Excel, PowerPoint, Outlook, Teams (deselect OneNote, Publisher if not needed)
   - Architecture: 64-bit (recommended for AVD)
   - Update Channel: Monthly Enterprise Channel (recommended for AVD)
   - Version to Install: Latest
   - Languages: English (United States), or multiple languages
3. **Configure Settings**:
   - Accept EULA on behalf of user: Yes
   - Shared computer activation: Yes (required for pooled AVD)
   - Install background service for updates: Yes
4. **Assign**:
   - Pooled AVD: Install in golden image, not via Intune
   - Personal AVD: Assign as "Required" to AVD-Users-All group

**Portal:** Intune Admin Center → Apps → Windows → Add → Microsoft 365 Apps

**Shared Computer Activation:**
- Required for pooled AVD (multiple users on same session host)
- Activates Office for each user's account (not the device)
- Each user gets separate license activation

**Update Channel:**
- **Monthly Enterprise Channel**: Recommended for AVD (monthly feature/security updates, predictable)
- **Current Channel**: Consumer-focused, updates every 2-4 weeks (less predictable)
- **Semi-Annual Enterprise Channel**: Updates every 6 months (stable, but slower security updates)

**Best Practice for Pooled AVD:** Install Microsoft 365 Apps in golden image with Shared Computer Activation enabled. For personal AVD, deploy via Intune as "Required".

## Storage for Intune App Packages

Intune stores uploaded .intunewin packages in Azure Storage. For large app catalogs, consider using a dedicated Azure Storage account to host packages.

**Our Environment:**
- **Storage Account**: intunescripts121125
- **Container**: app-packages (hypothetical; create if needed)
- **Purpose**: Host .intunewin packages, PowerShell scripts, and detection scripts

**Benefits:**
- Centralized storage for app packages
- Version control (store multiple versions of same app)
- Backup and restore (export packages for disaster recovery)

**Setup:**
1. Create Azure Storage account (already exists: intunescripts121125)
2. Create blob container: "app-packages"
3. Upload .intunewin files to container
4. Reference blob URL in Intune app deployment (not supported directly; upload via Intune portal instead)

**Note:** Intune does not support direct referencing of .intunewin files from Azure Storage. Packages must be uploaded via Intune Admin Center. However, Azure Storage can be used for source files (e.g., download .exe installer from Azure Storage during packaging).

## Best Practices

### Use Win32 Apps for Maximum Flexibility

**Why:** Win32 apps support all Windows desktop apps (.exe, .msi), custom install scripts, dependencies, supersedence, and detection rules. Microsoft Store apps are limited to MSIX-packaged apps.

**Recommendation:** Package all desktop apps as Win32 apps, even if MSI or MSIX versions exist. This provides maximum flexibility and consistent deployment experience.

### Install Apps in Golden Image for Pooled AVD

**Why:** Faster logon, consistent experience, reduced network traffic, lower storage costs.

**Process:**
1. Create golden image VM
2. Install apps manually or via script
3. Sysprep and capture image
4. Deploy session hosts from golden image
5. Update golden image monthly with app updates

**Exception:** Use Intune for emergency patches or user-specific apps.

### Deploy Apps via Intune for Personal AVD

**Why:** Automatic updates, user-specific apps, flexible deployment.

**Process:**
1. Package apps as Win32 apps
2. Upload to Intune
3. Assign as "Required" (core apps) or "Available" (optional apps)
4. Apps install automatically at user logon or on-demand from Company Portal

### Use Detection Rules Based on Version, Not Just Existence

**Why:** Prevents reinstalling same version, but allows updates when new version is deployed.

**Example (Bad):**
- Detection Rule: File exists at `C:\Program Files\Adobe\Acrobat DC\Acrobat\Acrobat.exe`
- Problem: Intune thinks app is installed, even if version is outdated

**Example (Good):**
- Detection Rule: Registry value `HKLM\SOFTWARE\Adobe\Acrobat Reader\DC\Version` equals `23.008.20360`
- Result: Intune reinstalls if version changes (e.g., update to 23.008.20500)

**Best Practice:** Use PowerShell script detection for maximum control (check version, check multiple files, check service status).

### Test App Deployment in Pilot Group First

**Why:** Prevents production outages from installation failures, conflicts, or incorrect detection rules.

**Workflow:**
1. Create "AVD-Devices-Pilot" dynamic group (test session hosts)
2. Assign new app to pilot group as "Required"
3. Monitor installation status for 48 hours
4. If successful, reassign to production group (AVD-Devices-Personal or AVD-Devices-All)

**Best Practice:** Maintain 2-5 test session hosts for pilot deployments.

### Use Dependencies for Apps with Prerequisites

**Why:** Ensures prerequisites (e.g., .NET Framework, Visual C++ Runtime) are installed before main app, reducing installation failures.

**Example:**
- App: Custom Line-of-Business App (requires .NET Framework 4.8)
- Dependency: Microsoft .NET Framework 4.8
- Result: Intune installs .NET Framework 4.8 first, then installs custom app

**Best Practice:** Package common dependencies (.NET Framework, Visual C++ Runtime) as separate Win32 apps, then reference as dependencies in other apps.

### Monitor App Installation Status Weekly

**Why:** Catch installation failures early, ensure apps are deployed to new devices, and identify devices missing required apps.

**Weekly Checklist:**
- Review app installation status (Apps → [App] → Monitor → Device Install Status)
- Investigate devices in "Failed" state
- Remediate issues (fix detection rule, fix install command, update app package)

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **App installation fails with error 0x87D1041C** | Detection rule not met after installation | 1. Verify detection rule is correct (file path, registry key, script logic)<br>2. Manually install app on test device, verify detection rule matches<br>3. Update detection rule in Intune |
| **App installation hangs indefinitely** | Install command not silent (prompts for user input) | 1. Verify install command uses silent flag (e.g., `/quiet`, `/silent`, `/s`)<br>2. Test install command manually on test device<br>3. Update install command in Intune (Apps → [App] → Properties → Program → Install command) |
| **App reinstalls every 8 hours** | Detection rule always returns "not installed" | 1. Verify detection rule logic (file path, registry key, script output)<br>2. Check event logs on session host: Applications and Services → Microsoft → Windows → AppManagement-MSI → Admin<br>3. Fix detection rule (ensure it returns "installed" after installation) |
| **App shows "Not Applicable" for all devices** | Requirements not met (OS version, architecture, disk space) | 1. Review app requirements (Apps → [App] → Properties → Requirements)<br>2. Verify devices meet requirements (OS version, architecture)<br>3. Reduce requirements or update devices |
| **App assignment shows "Available" but not visible in Company Portal** | App assigned to device group instead of user group | 1. Change assignment from device group to user group (Company Portal shows user-assigned apps only)<br>2. Or, instruct users to sync Company Portal (Settings → Sync) |
| **Intune Management Extension not installed on device** | Device did not receive Win32 app assignment or enrollment incomplete | 1. Verify device is enrolled in Intune (Devices → All Devices → [Device])<br>2. Assign at least one Win32 app to device (triggers IME installation)<br>3. Force device sync (Devices → [Device] → Sync)<br>4. Check: `C:\Program Files (x86)\Microsoft Intune Management Extension` exists |

## Summary

Application deployment via Intune is ideal for personal AVD environments where users have persistent desktops and benefit from automatic updates and user-specific app assignments. For pooled AVD, applications should be installed in the golden image for faster logon and consistent user experience. Win32 apps provide maximum flexibility with support for all desktop apps, custom install scripts, dependencies, and detection rules.

**Key Takeaways:**
- Use **Win32 apps** for traditional desktop apps (.exe, .msi)
- Use **Microsoft Store apps** for modern MSIX-packaged apps
- Install apps in **golden image** for pooled AVD (faster logon, consistent experience)
- Deploy apps via **Intune** for personal AVD (automatic updates, user-specific apps)
- Use **detection rules** based on version (not just file existence)
- Test app deployment in **pilot group** before production rollout

## Next Steps

After configuring application deployment:

1. **Windows Update Policies** - Control update deployment with update rings and feature update policies (Page 5)

Proceed to the next page to configure Windows Update Policies for AVD session hosts.