---
title: Step 10 - Application Deployment
description: Deploy Win32 applications to AVD session hosts using Intune for personal deployments
published: true
date: 2025-12-14T00:00:00.000Z
tags: [Quick-Deploy, Intune, applications, Win32]
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 10: Application Deployment

Deploy applications to AVD session hosts using Intune Win32 app packages. This step covers Microsoft 365 Apps, Adobe Acrobat Reader, and Adobe Creative Cloud deployment strategies for both pooled and personal host pools.

## Example Scenario

Deploying three core applications to AVD environment:

| Application | Target Users | Deployment Method | Assignment Group |
|-------------|--------------|-------------------|------------------|
| Microsoft 365 Apps | All users (200) | Intune Win32 | AVD-Pooled-Users, AVD-Personal-Users |
| Adobe Acrobat Reader | All users (200) | Intune Win32 | AVD-Pooled-Users, AVD-Personal-Users |
| Adobe Creative Cloud | Creative users (30) | Intune Win32 | AVD-Personal-Users |

**Deployment Strategy:**

| Host Pool Type | Application Source | Reason |
|----------------|-------------------|---------|
| **Pooled** (`hp-pooled-prod`) | Golden image ONLY | Faster logon, consistent experience, no per-session installation overhead |
| **Personal** (`hp-personal-prod`) | Intune Win32 apps | Flexible deployment, automatic updates, user-specific app assignments |

> **Critical:** For pooled host pools, install ALL applications in the golden image. Intune deployment to pooled hosts creates inconsistent experiences and slow logon times.

## Prerequisites

- [ ] Intune subscription active (included with Microsoft 365 E3/E5)
- [ ] Session hosts enrolled in Intune (from [[07-session-hosts]])
- [ ] Device groups created: `AVD-Devices-Personal` (from [[02-identity-setup]])
- [ ] User groups created: `AVD-Pooled-Users`, `AVD-Personal-Users` (from [[02-identity-setup]])
- [ ] IntuneWinAppUtil.exe downloaded from https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool
- [ ] Application installers downloaded (Microsoft 365, Adobe Reader, etc.)

> **Note:** This step focuses on PERSONAL host pool deployments. Pooled host pool applications should be installed during golden image creation (Step 03).

---

## Part 1: Understand Deployment Strategy

### Pooled vs Personal Application Deployment

**Pooled Host Pools (`hp-pooled-prod`):**

| Method | Speed | Consistency | Use Case |
|--------|-------|-------------|----------|
| **Golden Image** | Fast (apps pre-installed) | 100% consistent | Recommended for ALL pooled apps |
| Intune Win32 | Slow (installs at logon) | Variable (install failures possible) | Emergency patches ONLY |

**Personal Host Pools (`hp-personal-prod`):**

| Method | Speed | Consistency | Use Case |
|--------|-------|-------------|----------|
| Golden Image | Fast (apps pre-installed) | 100% consistent | Base apps only (Office, FSLogix) |
| **Intune Win32** | First logon slow, subsequent fast | High (retry on failure) | **Recommended for all apps** |

### AVD-Specific Considerations

**Shared Computer Activation (Office 365):**
- Required for pooled host pools (multiple users per VM)
- NOT required for personal host pools (1:1 user-to-VM)
- Enable via Intune configuration: `SharedComputerLicensing=1`

**Multi-Session Compatibility:**
- Verify apps support Windows multi-session (check vendor documentation)
- Most Microsoft apps support multi-session (Office, Edge, Teams)
- Third-party apps may require special licensing (Adobe Creative Cloud requires Named User Licensing)

**Storage Optimization:**
- Use MSIX app attach for large apps (reduces storage per session host)
- Not covered in this guide; see [[../Images/msix-app-attach|MSIX App Attach]]

**See:** [[../Intune/application-deployment-with-intune|Application Deployment with Intune]] for detailed app deployment strategies.

---

## Part 2: Prepare Packaging Environment

Set up local workstation for creating Win32 app packages.

### Download Required Tools

**IntuneWinAppUtil.exe:**

```powershell
# Download IntuneWinAppUtil from GitHub
Invoke-WebRequest -Uri "https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool/raw/master/IntuneWinAppUtil.exe" -OutFile "C:\Tools\IntuneWinAppUtil.exe"
```

**Verify download:**

```powershell
Get-Item C:\Tools\IntuneWinAppUtil.exe
```

Expected: File size ~150 KB, last modified within current year.

### Create Packaging Directory Structure

```powershell
# Create packaging workspace
New-Item -Path "C:\AppPackaging" -ItemType Directory -Force
New-Item -Path "C:\AppPackaging\Office365" -ItemType Directory
New-Item -Path "C:\AppPackaging\AdobeReader" -ItemType Directory
New-Item -Path "C:\AppPackaging\AdobeCreativeCloud" -ItemType Directory
New-Item -Path "C:\AppPackaging\Output" -ItemType Directory
```

**Directory Structure:**

```
C:\AppPackaging\
├── Office365\                    # Office 365 source files
│   ├── setup.exe                 # Office Deployment Tool
│   └── configuration.xml         # Office installation config
├── AdobeReader\                  # Adobe Reader source files
│   ├── AcroRdrDC2300820360_en_US.exe
│   └── install.ps1               # Custom install script
├── AdobeCreativeCloud\           # Creative Cloud source files
│   ├── CreativeCloudInstaller.exe
│   └── install.ps1
└── Output\                       # Packaged .intunewin files
    ├── Office365.intunewin
    ├── AdobeReader.intunewin
    └── AdobeCreativeCloud.intunewin
```

---

## Part 3: Package and Deploy Microsoft 365 Apps

Deploy Office suite (Word, Excel, PowerPoint, Outlook) with Shared Computer Activation for personal host pools.

### Download Office Deployment Tool

**Office Deployment Tool (ODT):**

```powershell
# Download Office Deployment Tool
Invoke-WebRequest -Uri "https://download.microsoft.com/download/2/7/A/27AF1BE6-DD20-4CB4-B154-EBAB8A7D4A7E/officedeploymenttool_17830-20162.exe" -OutFile "C:\AppPackaging\Office365\ODT.exe"

# Extract ODT
Start-Process -FilePath "C:\AppPackaging\Office365\ODT.exe" -ArgumentList "/quiet /extract:C:\AppPackaging\Office365" -Wait
```

**Verify extraction:**

```powershell
Get-ChildItem C:\AppPackaging\Office365
```

Expected: `setup.exe`, `configuration-Office2021Enterprise.xml`, `configuration-Office365-x64.xml`

### Create Office Configuration File

Create custom configuration for AVD deployment.

**Create `C:\AppPackaging\Office365\configuration.xml`:**

```xml
<Configuration>
  <Add OfficeClientEdition="64" Channel="MonthlyEnterprise">
    <Product ID="O365ProPlusRetail">
      <Language ID="en-us" />
      <ExcludeApp ID="Groove" />
      <ExcludeApp ID="Lync" />
      <ExcludeApp ID="OneDrive" />
      <ExcludeApp ID="OneNote" />
      <ExcludeApp ID="Publisher" />
    </Product>
  </Add>
  <Property Name="SharedComputerLicensing" Value="1" />
  <Property Name="FORCEAPPSHUTDOWN" Value="TRUE" />
  <Property Name="DeviceBasedLicensing" Value="0" />
  <Property Name="PinIconsToTaskbar" Value="TRUE" />
  <Updates Enabled="TRUE" Channel="MonthlyEnterprise" />
  <RemoveMSI />
  <Display Level="None" AcceptEULA="TRUE" />
</Configuration>
```

**Key Settings Explained:**

| Setting | Value | Purpose |
|---------|-------|---------|
| `OfficeClientEdition` | 64 | 64-bit Office (recommended for AVD) |
| `Channel` | MonthlyEnterprise | Monthly feature updates, predictable schedule |
| `SharedComputerLicensing` | 1 | **Required for pooled AVD** (activates per user, not per device) |
| `FORCEAPPSHUTDOWN` | TRUE | Close Office apps before installation |
| `ExcludeApp` | Groove, Lync, OneDrive, OneNote, Publisher | Exclude unnecessary apps (reduces image size) |
| `RemoveMSI` | (empty) | Uninstall old Office MSI versions before installing |

> **Important:** `SharedComputerLicensing=1` is REQUIRED for pooled host pools. Without this, Office activates for the device (not user), consuming extra licenses.

**See:** [[../Intune/application-deployment-with-intune#microsoft-365-apps-deployment|Microsoft 365 Apps]] for configuration options.

### Package Office 365 with IntuneWinAppUtil

```powershell
# Package Office 365
C:\Tools\IntuneWinAppUtil.exe `
  -c "C:\AppPackaging\Office365" `
  -s "setup.exe" `
  -o "C:\AppPackaging\Output"
```

**Output:** `setup.intunewin` in `C:\AppPackaging\Output\`

### Upload to Intune

**Portal:** Intune Admin Center (https://intune.microsoft.com) → Apps → Windows → + Add

1. **App type:**
   - Select: **Windows app (Win32)**
   - Click **Select**

2. **App package file:**
   - Click **Select app package file**
   - Browse to: `C:\AppPackaging\Output\setup.intunewin`
   - Click **OK**

3. **App information:**
   - **Name:** `Microsoft 365 Apps (AVD)`
   - **Description:** `Office suite with Shared Computer Activation for AVD personal hosts`
   - **Publisher:** `Microsoft Corporation`
   - **Category:** Productivity
   - **Show this as featured app in Company Portal:** No
   - **Information URL:** `https://www.microsoft.com/microsoft-365`
   - **Privacy URL:** `https://privacy.microsoft.com`
   - Click **Next**

4. **Program:**
   - **Install command:**
     ```
     setup.exe /configure configuration.xml
     ```
   - **Uninstall command:**
     ```
     setup.exe /configure uninstall.xml
     ```
   - **Install behavior:** System
   - **Device restart behavior:** Determine behavior based on return codes
   - **Return codes:** (leave default)
   - Click **Next**

5. **Requirements:**
   - **Operating system architecture:** 64-bit
   - **Minimum operating system:** Windows 10 21H2
   - **Disk space required (MB):** 4000 (4 GB)
   - **Physical memory required (MB):** 4096 (4 GB)
   - **Logical processors required:** 2
   - Click **Next**

6. **Detection rules:**
   - **Rules format:** Use a custom detection script
   - **Script file:** Upload PowerShell detection script (see below)
   - Click **Next**

7. **Dependencies:**
   - None
   - Click **Next**

8. **Supersedence:**
   - None
   - Click **Next**

9. **Assignments:**
   - **Required:**
     - Click **+ Add group**
     - Search: `AVD-Personal-Users`
     - Select group
     - Click **Select**
   - Click **Next**

10. **Review + create:**
    - Review settings
    - Click **Create**

**Deployment time:** ~5 minutes to upload, 15-30 minutes to deploy to devices

### Create Detection Script for Office 365

**Create `Detect-Office365.ps1`:**

```powershell
# Detection script for Microsoft 365 Apps
# Returns exit code 0 if Office installed, exit code 1 if not installed

$officePath = "C:\Program Files\Microsoft Office\root\Office16"
$excelPath = Join-Path $officePath "EXCEL.EXE"
$wordPath = Join-Path $officePath "WINWORD.EXE"

if ((Test-Path $excelPath) -and (Test-Path $wordPath)) {
    # Office installed
    Write-Output "Microsoft 365 Apps installed"
    exit 0
} else {
    # Office not installed
    exit 1
}
```

**Upload in Intune:**
- Detection rules → Use a custom detection script → Upload `Detect-Office365.ps1`
- **Run script as 32-bit process:** No
- **Enforce script signature check:** No

### Create Uninstall Configuration (Optional)

**Create `C:\AppPackaging\Office365\uninstall.xml`:**

```xml
<Configuration>
  <Remove All="TRUE" />
  <Display Level="None" AcceptEULA="TRUE" />
</Configuration>
```

> **Note:** Package `uninstall.xml` together with `configuration.xml` in same folder before running IntuneWinAppUtil.

### Monitor Office 365 Deployment

**Portal:** Intune Admin Center → Apps → Windows → Microsoft 365 Apps (AVD) → Monitor → Device install status

**Expected States:**

| State | Count | Meaning |
|-------|-------|---------|
| **Installed** | 40 | Office installed successfully on personal hosts |
| In progress | 5 | Installation in progress |
| Failed | 0 | Installation failures (investigate errors) |
| Not applicable | 10 | Pooled hosts (not assigned) |

**Troubleshooting Failed Installations:**

```powershell
# On session host, check Intune Management Extension logs
Get-Content "C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log" | Select-String -Pattern "Microsoft 365"
```

**See:** [[../Intune/application-deployment-with-intune#common-issues|Common Issues]] for error resolution.

---

## Part 4: Package and Deploy Adobe Acrobat Reader

Deploy Adobe Acrobat Reader DC with customizations (disable auto-update, set default PDF viewer).

### Download Adobe Acrobat Reader

**Download installer:**

```powershell
# Download Adobe Reader DC (replace with latest version URL from Adobe FTP)
Invoke-WebRequest -Uri "https://ardownload2.adobe.com/pub/adobe/reader/win/AcrobatDC/2300820360/AcroRdrDC2300820360_en_US.exe" -OutFile "C:\AppPackaging\AdobeReader\AcroRdrDC2300820360_en_US.exe"
```

> **Note:** Adobe Reader download URLs change frequently. Get latest URL from: https://get.adobe.com/reader/enterprise/

**Verify download:**

```powershell
Get-Item C:\AppPackaging\AdobeReader\AcroRdrDC2300820360_en_US.exe
```

Expected: File size ~200 MB

### Create Custom Install Script

Adobe Reader requires custom script to suppress auto-update prompts.

**Create `C:\AppPackaging\AdobeReader\install.ps1`:**

```powershell
# Install Adobe Acrobat Reader DC with customizations
# Disable auto-update and set as default PDF viewer

# Silent install
Start-Process -FilePath "AcroRdrDC2300820360_en_US.exe" -ArgumentList "/sAll /msi EULA_ACCEPT=YES" -Wait -NoNewWindow

# Disable auto-update via registry
$regPath = "HKLM:\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown"
if (-not (Test-Path $regPath)) {
    New-Item -Path $regPath -Force | Out-Null
}
Set-ItemProperty -Path $regPath -Name "bUpdater" -Value 0 -Type DWord

# Set Adobe Reader as default PDF viewer
$progId = "AcroExch.Document.DC"
cmd /c "assoc .pdf=$progId"
cmd /c "ftype $progId=`"C:\Program Files\Adobe\Acrobat DC\Acrobat\Acrobat.exe`" `"%1`""

Write-Output "Adobe Acrobat Reader DC installed and configured"
exit 0
```

**Key Customizations:**

| Customization | Registry/Command | Purpose |
|---------------|------------------|---------|
| Disable auto-update | `bUpdater=0` | Prevent users from seeing update prompts (updates managed via Intune) |
| Set default PDF viewer | `assoc .pdf` | Open PDFs with Reader by default (not Edge) |
| Silent install | `/sAll /msi EULA_ACCEPT=YES` | No user interaction required |

### Package Adobe Reader with IntuneWinAppUtil

```powershell
# Package Adobe Reader
C:\Tools\IntuneWinAppUtil.exe `
  -c "C:\AppPackaging\AdobeReader" `
  -s "AcroRdrDC2300820360_en_US.exe" `
  -o "C:\AppPackaging\Output"
```

**Output:** `AcroRdrDC2300820360_en_US.intunewin` in `C:\AppPackaging\Output\`

### Upload to Intune

**Portal:** Intune Admin Center → Apps → Windows → + Add

1. **App type:** Windows app (Win32)

2. **App package file:** Upload `AcroRdrDC2300820360_en_US.intunewin`

3. **App information:**
   - **Name:** `Adobe Acrobat Reader DC`
   - **Description:** `PDF reader with auto-update disabled for AVD`
   - **Publisher:** `Adobe Inc.`
   - **Category:** Productivity

4. **Program:**
   - **Install command:**
     ```powershell
     powershell.exe -ExecutionPolicy Bypass -File "install.ps1"
     ```
   - **Uninstall command:**
     ```
     msiexec /x {AC76BA86-7AD7-1033-7B44-AC0F074E4100} /quiet
     ```
   - **Install behavior:** System
   - **Device restart behavior:** No specific action

5. **Requirements:**
   - **Operating system architecture:** 64-bit
   - **Minimum operating system:** Windows 10 21H2
   - **Disk space required (MB):** 500

6. **Detection rules:**
   - **Rules format:** Manually configure detection rules
   - **Rule type:** File
   - **Path:** `C:\Program Files\Adobe\Acrobat DC\Acrobat`
   - **File or folder:** `Acrobat.exe`
   - **Detection method:** File or folder exists
   - **Associated with a 32-bit app:** No

7. **Assignments:**
   - **Required:**
     - `AVD-Personal-Users`

8. Click **Create**

### Monitor Adobe Reader Deployment

**Portal:** Intune Admin Center → Apps → Adobe Acrobat Reader DC → Device install status

**Verify on Session Host:**

```powershell
# Verify Adobe Reader installed
Get-ItemProperty "HKLM:\SOFTWARE\Adobe\Acrobat Reader\DC" -Name "Version"

# Verify auto-update disabled
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown" -Name "bUpdater"
```

Expected: Version shows `23.008.20360`, bUpdater shows `0`

---

## Part 5: Package and Deploy Adobe Creative Cloud

Deploy Adobe Creative Cloud for creative users (personal host pools only).

### Adobe Creative Cloud Licensing Requirements

**Named User Licensing (NUL):**
- Each user requires individual Adobe Creative Cloud license
- User authenticates with Adobe ID when launching Creative Cloud apps
- License follows user across devices (not tied to session host)

**AVD Requirements:**
- Personal host pools: Standard NUL licensing (recommended)
- Pooled host pools: Requires special Adobe licensing agreement (not recommended for AVD)

> **Important:** Adobe Creative Cloud is NOT recommended for pooled host pools due to licensing complexity. Deploy ONLY to personal host pools.

**See:** Adobe VDI documentation for licensing details: https://helpx.adobe.com/enterprise/kb/deploy-creative-cloud-vdi.html

### Download Creative Cloud Installer

**Adobe Admin Console:**

1. Login to Adobe Admin Console (https://adminconsole.adobe.com)
2. Navigate to **Packages** → **Create a Package**
3. **Package Type:** Managed Package
4. **Select Apps:** Creative Cloud Desktop (includes Photoshop, Illustrator, etc.)
5. **Options:**
   - Enable self-service install: Yes
   - Allow non-admins to update apps: No (managed via Intune)
6. **Build Package** → Download

**Save to:**

```powershell
# Example (actual filename varies based on Adobe package)
# Save downloaded package to:
C:\AppPackaging\AdobeCreativeCloud\CreativeCloudInstaller.exe
```

### Create Install Script

**Create `C:\AppPackaging\AdobeCreativeCloud\install.ps1`:**

```powershell
# Install Adobe Creative Cloud Desktop (Named User Licensing)
# Requires user to sign in with Adobe ID on first launch

# Silent install
Start-Process -FilePath "CreativeCloudInstaller.exe" -ArgumentList "--silent" -Wait -NoNewWindow

Write-Output "Adobe Creative Cloud Desktop installed"
exit 0
```

### Package Creative Cloud with IntuneWinAppUtil

```powershell
# Package Adobe Creative Cloud
C:\Tools\IntuneWinAppUtil.exe `
  -c "C:\AppPackaging\AdobeCreativeCloud" `
  -s "CreativeCloudInstaller.exe" `
  -o "C:\AppPackaging\Output"
```

**Output:** `CreativeCloudInstaller.intunewin`

### Upload to Intune

**Portal:** Intune Admin Center → Apps → Windows → + Add

1. **App type:** Windows app (Win32)

2. **App package file:** Upload `CreativeCloudInstaller.intunewin`

3. **App information:**
   - **Name:** `Adobe Creative Cloud Desktop`
   - **Description:** `Creative Cloud app management (requires Adobe ID sign-in). Personal hosts only.`
   - **Publisher:** `Adobe Inc.`
   - **Category:** Productivity

4. **Program:**
   - **Install command:**
     ```powershell
     powershell.exe -ExecutionPolicy Bypass -File "install.ps1"
     ```
   - **Uninstall command:**
     ```
     "C:\Program Files\Adobe\Adobe Creative Cloud\ACC\Creative Cloud Uninstaller.exe" --uninstall=1
     ```
   - **Install behavior:** System
   - **Device restart behavior:** No specific action

5. **Requirements:**
   - **Operating system architecture:** 64-bit
   - **Minimum operating system:** Windows 10 21H2
   - **Disk space required (MB):** 2000 (Creative Cloud apps are large)

6. **Detection rules:**
   - **Rules format:** File
   - **Path:** `C:\Program Files\Adobe\Adobe Creative Cloud\ACC`
   - **File or folder:** `Creative Cloud.exe`
   - **Detection method:** File or folder exists

7. **Assignments:**
   - **Required:**
     - `AVD-Personal-Users` (or subset: Creative department users only)

8. Click **Create**

### User First-Run Experience

**After Intune installation:**

1. User logs into personal AVD session
2. Creative Cloud Desktop installed automatically (first logon)
3. Creative Cloud icon appears in system tray
4. User clicks Creative Cloud icon
5. User prompted to sign in with Adobe ID
6. User enters Adobe credentials (must have assigned Creative Cloud license)
7. Creative Cloud shows available apps (Photoshop, Illustrator, etc.)
8. User installs desired apps on-demand

> **Note:** Creative Cloud apps (Photoshop, etc.) install to `C:\Program Files\Adobe\` and persist across sessions (personal host pool).

### Monitor Creative Cloud Deployment

**Portal:** Intune Admin Center → Apps → Adobe Creative Cloud Desktop → Device install status

**Verify on Session Host:**

```powershell
# Verify Creative Cloud Desktop installed
Get-Item "C:\Program Files\Adobe\Adobe Creative Cloud\ACC\Creative Cloud.exe"

# Check Creative Cloud service running
Get-Service -Name "Adobe*" | Where-Object {$_.Status -eq "Running"}
```

---

## Part 6: Deployment Monitoring and Validation

Monitor app deployment status and troubleshoot failures.

### Monitor Overall Deployment Status

**Portal:** Intune Admin Center → Apps → Overview → Deployment status

**Key Metrics:**

| Metric | Target | Meaning |
|--------|--------|---------|
| **App installation success rate** | >95% | Percentage of successful installations |
| **Devices with app installation failures** | <5% | Devices with at least one failed app |
| **Pending app installations** | Decreasing | Apps queued for installation |

**Best Practice:** Review deployment status daily during rollout, weekly after stabilization.

### Per-App Deployment Status

**Portal:** Intune Admin Center → Apps → Windows → [App Name] → Monitor → Device install status

**Filter by Status:**

| Status | Action Required |
|--------|-----------------|
| **Installed** | None (success) |
| In progress | Wait 24 hours, retry if stuck |
| Failed | Investigate error code |
| Not installed | Device not in scope |
| Not applicable | Requirements not met (OS version, architecture) |

### Common Installation Error Codes

| Error Code | Meaning | Solution |
|------------|---------|----------|
| **0x87D1041C** | Detection rule not met after installation | Verify detection rule (file path, registry key) matches actual installation |
| **0x80070643** | MSI installation failed | Check event logs on device: `eventvwr.msc` → Windows Logs → Application |
| **0x87D1FDE8** | Installation timeout (exceeds 60 min default) | Increase timeout in app settings or split into smaller packages |
| **0x80070005** | Access denied | Install command requires admin privileges (ensure Install behavior: System) |
| **0x87D12906** | Content download failed | Check network connectivity, retry deployment |

**See:** [[../Intune/application-deployment-with-intune#common-issues|Common Issues]] for full error code reference.

### Verify App Installation on Session Host

**RDP to personal session host as administrator:**

```powershell
# List installed Win32 apps
Get-WmiObject -Class Win32_Product | Select-Object Name, Version | Sort-Object Name

# Check Intune Management Extension logs
Get-Content "C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log" -Tail 50

# Check specific app installation
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | Where-Object {$_.DisplayName -like "*Adobe*"}
```

**Verify app functionality:**

1. Launch Office app (Word, Excel)
2. Check activation status: File → Account → Product Information
3. Verify Shared Computer Activation enabled (if pooled host)
4. Launch Adobe Reader, open PDF
5. Launch Creative Cloud Desktop, verify user signed in

---

## Part 7: App Updates and Maintenance

Manage app updates and version control.

### Office 365 Updates

**Update Channel:** Monthly Enterprise Channel (configured in `configuration.xml`)

**Update Behavior:**
- Office checks for updates daily (managed by Office CDN, not Intune)
- Updates install automatically during low-usage periods
- Users prompted to close apps before update (if apps open during update window)

**Force Update (if needed):**

```powershell
# On session host, force Office update check
cd "C:\Program Files\Common Files\Microsoft Shared\ClickToRun"
.\OfficeC2RClient.exe /update user updatetoversion=16.0.16827.20166
```

**Monitor Update Compliance:**

**Portal:** Intune Admin Center → Apps → Monitor → App update compliance

### Adobe Reader Updates

**Update Strategy:** Disable auto-update (configured via registry), deploy new versions via Intune supersedence.

**Process:**

1. Download new Adobe Reader version (e.g., v23.009)
2. Package as new Win32 app: `Adobe Acrobat Reader DC v23.009`
3. Configure supersedence relationship:
   - New app supersedes old app (`Adobe Acrobat Reader DC v23.008`)
   - Uninstall old version before installing new version
4. Assign to `AVD-Personal-Users`
5. Intune automatically uninstalls v23.008 and installs v23.009

**See:** [[../Intune/application-deployment-with-intune#dependencies-and-supersedence|Supersedence]] for version management.

### Creative Cloud Updates

**Update Strategy:** Adobe manages updates via Creative Cloud Desktop (users install app updates on-demand).

**Disable Automatic Updates (Optional):**

**Registry:**

```powershell
# Disable Creative Cloud automatic updates (requires admin)
$regPath = "HKLM:\SOFTWARE\Policies\Adobe\Adobe Desktop Common\AutoUpdate"
New-Item -Path $regPath -Force
Set-ItemProperty -Path $regPath -Name "AutoUpdateDisabled" -Value 1 -Type DWord
```

**Recommendation:** Allow users to update Creative Cloud apps on-demand (creative users need latest features).

---

## Variant: Pooled Host Pool Application Deployment

Applications for pooled host pools should be installed in golden image, NOT via Intune.

### Golden Image Application List

**Install during image creation (Step 03):**

| Application | Version | Install Method |
|-------------|---------|----------------|
| Microsoft 365 Apps | Latest | Office Deployment Tool with Shared Computer Activation |
| Adobe Acrobat Reader | Latest | Silent install with auto-update disabled |
| Google Chrome | Latest | MSI package from https://chromeenterprise.google/browser/download/ |
| Microsoft Teams | Latest | VDI-optimized installer from https://aka.ms/msteams |
| FSLogix Agent | Latest | From https://aka.ms/fslogix_download |

**Process:**

1. Create VM from marketplace image (Windows 11 multi-session)
2. Install applications (same installers as Intune packages)
3. Run Sysprep: `C:\Windows\System32\Sysprep\sysprep.exe /oobe /generalize /shutdown`
4. Capture image to Azure Compute Gallery
5. Deploy session hosts from golden image

**Benefits:**
- Apps pre-installed (no installation at user logon)
- Consistent experience across all session hosts
- Faster logon times (15-30 seconds vs 2-5 minutes with Intune deployment)

**See:** [[03-image-gallery|Step 03: Image Gallery]] for golden image creation.

### Emergency Patching via Intune

**Use Case:** Critical security update needed before next image update cycle.

**Process:**

1. Package update as Win32 app
2. Assign to `AVD-Devices-Pooled` (device group, not user group)
3. Set assignment intent: Required
4. App installs at next device check-in (within 8 hours)
5. Update golden image with same patch
6. Remove Intune app assignment after next session host redeployment

> **Important:** Emergency patching should be rare. Plan monthly golden image updates for routine app updates.

---

## Verification Checklist

Confirm all applications deployed successfully.

### Intune App Configuration

**Portal:** Intune Admin Center → Apps → Windows

- [ ] Microsoft 365 Apps (AVD) exists with detection rules configured
- [ ] Adobe Acrobat Reader DC exists with install script uploaded
- [ ] Adobe Creative Cloud Desktop exists (if deploying to creative users)
- [ ] All apps assigned to `AVD-Personal-Users` (or subset)

### Deployment Status

**Portal:** Intune Admin Center → Apps → [App] → Monitor → Device install status

- [ ] Microsoft 365 Apps: Installed on 40/40 personal hosts (100%)
- [ ] Adobe Acrobat Reader: Installed on 40/40 personal hosts (100%)
- [ ] Adobe Creative Cloud: Installed on creative user hosts only (subset)
- [ ] Failed installations: 0 (or <5%)

### Functionality Testing

**Test on personal session host:**

- [ ] Office apps launch successfully (Word, Excel, PowerPoint, Outlook)
- [ ] Office activation status: "Product Activated" (File → Account)
- [ ] Adobe Reader opens PDFs, is set as default PDF viewer
- [ ] Adobe Creative Cloud Desktop appears in system tray (creative users only)
- [ ] Creative Cloud user signed in with Adobe ID

**Test on pooled session host (golden image apps):**

- [ ] Office apps pre-installed and activated with Shared Computer Activation
- [ ] Adobe Reader pre-installed with auto-update disabled
- [ ] No Intune apps deployed (apps installed during image creation)

---

## Troubleshooting

### Issue: Office 365 installation fails with error 0x87D1041C

**Symptom:** Intune reports "Detection rule not met after installation"

**Cause:**
- Detection script looking for wrong file path
- Office installed to non-default location
- Detection script running before installation completes

**Fix:**

1. Verify detection script path matches actual installation:
   ```powershell
   # On session host, check Office installation path
   Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | Where-Object {$_.DisplayName -like "*Microsoft 365*"}
   ```
2. Update detection script to check registry instead of file path:
   ```powershell
   # Detection script (registry-based)
   $officePath = Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Office\ClickToRun" -Name "InstallPath" -ErrorAction SilentlyContinue
   if ($officePath) { exit 0 } else { exit 1 }
   ```
3. Increase installation timeout (if slow network):
   - Portal: App → Properties → Program → Maximum allowed run time: 120 minutes

### Issue: Adobe Reader auto-update prompts appear

**Symptom:** Users see "A new version of Adobe Reader is available" prompts

**Cause:**
- Registry key `bUpdater=0` not set during installation
- Install script failed to apply registry customizations

**Fix:**

1. Verify registry key on session host:
   ```powershell
   Get-ItemProperty "HKLM:\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown" -Name "bUpdater"
   ```
2. If missing, manually set via Intune PowerShell script:
   - Portal: Devices → Scripts → + Add → Windows 10 and later
   - Upload remediation script:
     ```powershell
     $regPath = "HKLM:\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown"
     Set-ItemProperty -Path $regPath -Name "bUpdater" -Value 0 -Type DWord
     ```
   - Assign to `AVD-Devices-Personal`

### Issue: Creative Cloud requires sign-in every session

**Symptom:** User prompted for Adobe ID credentials on every logon

**Cause:**
- Creative Cloud credentials not cached in FSLogix profile
- Adobe credential cache excluded from profile

**Fix:**

1. Verify FSLogix profile includes Adobe cache directory:
   ```powershell
   # On session host, check FSLogix exclusions
   Get-Content "C:\Program Files\FSLogix\Apps\frx.exe" | Select-String -Pattern "Adobe"
   ```
2. Add Adobe cache to FSLogix inclusions (if excluded):
   - Registry: `HKLM\SOFTWARE\FSLogix\Profiles\IncludedFolders`
   - Add: `%LOCALAPPDATA%\Adobe`
3. Instruct user to check "Keep me signed in" when signing into Creative Cloud

### Issue: Apps install on pooled hosts (should be in golden image only)

**Symptom:** Intune deploying apps to pooled session hosts, causing slow logon

**Cause:**
- Apps assigned to user group (follows user to pooled hosts)
- Incorrect assignment scope

**Fix:**

1. Change app assignment from user group to device group:
   - Portal: App → Assignments → Remove `AVD-Personal-Users`
   - Add: `AVD-Devices-Personal` (device group, not user group)
2. Verify device group membership:
   - Portal: Devices → All devices → Filter: `vm-personal-prod-*`
   - Ensure only personal hosts in `AVD-Devices-Personal` group

---

## Next Steps

**Applications deployed successfully.** Users have access to Office, Adobe Reader, and Creative Cloud on personal hosts.

**Next:** Step 11: Windows Updates and Patch Management (coming soon)

In Step 11, you will:
- Configure Windows Update policies for session hosts
- Set up update rings (test vs production)
- Configure feature update deferral
- Implement maintenance windows

---

## Related Reference Pages

- [[../Intune/application-deployment-with-intune|Application Deployment with Intune]] - Win32 apps, detection rules, dependencies, supersedence
- [[../Images/golden-image-creation|Golden Image Creation]] - Installing apps in pooled host pool images
- [[../Images/msix-app-attach|MSIX App Attach]] - Dynamic app delivery for pooled hosts
- [[../Operations/capacity-planning|Capacity Planning]] - Storage and performance impact of applications
- [[../Security/data-loss-prevention|Data Loss Prevention]] - Securing app data (clipboard, drive redirection)
