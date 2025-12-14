---
title: Golden Image Process
description: 
published: true
date: 2025-12-14T04:53:05.441Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:22.473Z
---

# Golden Image Process

Creating a golden image is the foundation of a consistent Azure Virtual Desktop deployment. A golden image is a fully configured, generalized Windows installation that serves as the template for all session hosts in a host pool. This page covers the manual process of building, customizing, and capturing golden images—the essential skill every AVD administrator must master before moving to automated solutions like Azure Image Builder.

## What is a Golden Image

A golden image is a pristine, fully patched, and pre-configured Windows virtual machine that has been generalized using Microsoft's System Preparation Tool (sysprep). "Generalized" means that all computer-specific information (computer name, security identifiers, drivers, user profiles) has been stripped out, allowing the image to be cloned to create multiple unique VMs.

For Azure Virtual Desktop, a golden image typically includes:
- Base Windows OS (Windows 11 Enterprise multi-session or Windows 11 Enterprise single-session)
- Latest Windows security updates and feature updates
- FSLogix Profile Container agent for user profile management
- Microsoft 365 Apps for enterprise (formerly Office 365 ProPlus)
- Line-of-business (LOB) applications specific to your organization
- Monitoring and management agents (Azure Monitor, Defender for Endpoint, SCCM/Intune)
- Corporate security policies, certificates, and GPO templates
- Performance and network optimizations (Windows Virtual Desktop Optimization Tool)

The goal is to install everything once, capture it as an image, and deploy hundreds of identical session hosts from that image—ensuring consistency, reducing deployment time, and minimizing configuration drift.

## Manual Golden Image Process

### 1. Create Base VM from Marketplace

**Azure Portal Path:** Azure Portal → Create a resource → Virtual machine

**Configuration:**
- **Resource Group:** Use a dedicated "Image Management" RG (e.g., `RG-ImageBuild-Prod`)
- **VM Name:** `AVD-GoldenImage-Win11-Multi-01` (descriptive naming for build VMs)
- **Region:** Your primary AVD region (e.g., East US 2)
- **Availability Options:** No infrastructure redundancy required (this is a temporary build VM)
- **Security Type:** Trusted launch virtual machines (enables Secure Boot and vTPM)
- **Image:** Windows 11 Enterprise multi-session, Version 25H2 - Gen2 (from Azure Marketplace)
  - Alternative: Windows 11 Enterprise (single-session) for personal host pools
  - Always select Gen2 (not Gen1) for modern security features
- **VM Size:** Standard_D4s_v5 (4 vCPU, 16GB RAM)
  - Larger than production session hosts to speed up software installation
  - Can downsize production hosts—image captures disk content, not VM size
- **Administrator Account:**
  - **Username:** `localadmin` (avoid "Administrator" as it's a default target for attacks)
  - **Password:** Strong password (16+ characters, stored in Azure Key Vault)
- **Inbound Port Rules:** RDP (3389) from your IP only (restrict in NSG after creation)
- **OS Disk:**
  - **Type:** Premium SSD (faster for builds and capturing)
  - **Size:** 127GB default (auto-expands from 30GB on first boot)
  - **Encryption:** Platform-managed keys (default)
- **Networking:**
  - **Virtual Network:** Dedicated image build subnet (not production AVD subnet)
  - **Public IP:** Yes (temporary, for build access—delete after capture)
  - **NIC NSG:** Create with RDP restricted to your public IP
- **Management:**
  - **Enable auto-shutdown:** 7:00 PM (prevents accidental overnight costs)
  - **Boot diagnostics:** Enabled (helps troubleshoot boot issues)
- **Monitoring:**
  - **Disable** Azure Monitor agent for build VM (add to image later if needed)
- **Tags:**
  - `Environment: ImageBuild`
  - `Purpose: GoldenImage`
  - `DeleteAfter: 2025-01-30` (reminder to clean up after capture)

**Deployment Time:** 3-5 minutes

**Initial Boot:** Windows first-run setup (OOBE) will start. Log in with local admin credentials.

### 2. Install Applications

Connect to the VM via RDP and install all required software. Install applications in a logical order to minimize conflicts and reboots.

#### Phase 1: Windows Updates

**Critical:** Apply all Windows updates before installing applications. Partially patched systems can cause sysprep failures.

1. Open **Settings → Windows Update → Check for updates**
2. Install all available updates (may require multiple restarts)
3. Continue checking until "You're up to date" appears
4. Verify no pending updates: `Get-WindowsUpdate` (if using PSWindowsUpdate module)

**Average Time:** 30-60 minutes (varies by patch Tuesday proximity)

#### Phase 2: FSLogix Agent

FSLogix is mandatory for AVD to enable user profile containers and Office 365 container redirection.

**Download:** [FSLogix Download Page](https://aka.ms/fslogix-latest) (currently 2.9.9000+)

**Installation (PowerShell as Administrator):**

```powershell
# Download FSLogix installer
$fslogixUrl = "https://aka.ms/fslogix-latest"
$installerPath = "C:\Temp\FSLogixAppsSetup.zip"
New-Item -Path "C:\Temp" -ItemType Directory -Force
Invoke-WebRequest -Uri $fslogixUrl -OutFile $installerPath

# Extract and install
Expand-Archive -Path $installerPath -DestinationPath "C:\Temp\FSLogix" -Force
Start-Process -FilePath "C:\Temp\FSLogix\x64\Release\FSLogixAppsSetup.exe" -ArgumentList "/install", "/quiet", "/norestart" -Wait

# Verify installation
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Apps" | Select-Object -Property Version
```

**Post-Install Configuration:**
- **Do NOT configure profile paths yet** - this is done via GPO or Intune policy after deployment
- Verify service status: `Get-Service frxsvc, frxccds` (should be Running)

#### Phase 3: Microsoft 365 Apps for Enterprise

**Download:** Use Office Deployment Tool (ODT) for customized installs

**Configuration (Configuration.xml):**

```xml
<Configuration>
  <Add OfficeClientEdition="64" Channel="MonthlyEnterprise">
    <Product ID="O365ProPlusRetail">
      <Language ID="en-us" />
      <ExcludeApp ID="Groove" />  <!-- Exclude OneDrive (managed separately) -->
      <ExcludeApp ID="Lync" />    <!-- Exclude Skype (Teams used instead) -->
    </Product>
  </Add>
  <Property Name="SharedComputerLicensing" Value="1" /> <!-- Required for AVD -->
  <Property Name="FORCEAPPSHUTDOWN" Value="TRUE" />
  <Updates Enabled="TRUE" Channel="MonthlyEnterprise" />
  <Display Level="None" AcceptEULA="TRUE" />
</Configuration>
```

**Installation (PowerShell):**

```powershell
# Download ODT
$odtUrl = "https://www.microsoft.com/en-us/download/confirmation.aspx?id=49117"
# (Manual download required due to redirect complexity)

# Place Configuration.xml in C:\Temp\ODT\
cd C:\Temp\ODT
.\setup.exe /configure Configuration.xml

# Verify installation
$m365Path = "C:\Program Files\Microsoft Office\root\Office16\WINWORD.EXE"
if (Test-Path $m365Path) {
    Write-Host "[SUCCESS] Microsoft 365 Apps installed"
    # Check version
    (Get-Item $m365Path).VersionInfo.ProductVersion
}
```

**Critical Setting:** `SharedComputerLicensing="1"` is mandatory for multi-session AVD environments.

#### Phase 4: Line-of-Business Applications

Install company-specific applications. Document every installation for reproducibility.

**Best Practices:**
- **Silent installations only:** Use `/quiet`, `/silent`, or MSI switches (`/qn`)
- **Install to default paths:** Avoid custom paths like `D:\Apps` (session hosts may not have D:)
- **Disable auto-updates:** Applications should update via image rebuilds, not individual session host updates (prevents configuration drift)
- **Test application compatibility:** Verify each application works in multi-session mode (not all apps support concurrent users)

**Example: Adobe Acrobat Reader DC (Silent Install):**

```powershell
$adobeUrl = "https://ardownload2.adobe.com/pub/adobe/reader/win/AcrobatDC/2300820470/AcroRdrDC2300820470_en_US.exe"
$adobeInstaller = "C:\Temp\AdobeReader.exe"
Invoke-WebRequest -Uri $adobeUrl -OutFile $adobeInstaller
Start-Process -FilePath $adobeInstaller -ArgumentList "/sAll", "/rs", "/msi", "EULA_ACCEPT=YES" -Wait
```

**Testing:** Log in as a test user and launch each application to verify it runs without errors.

#### Phase 5: Monitoring and Management Agents

**Azure Monitor Agent (AMA):**

```powershell
# Install via Azure VM extension (recommended over manual install)
# This will be done via ARM template during session host deployment
# For golden image, skip to avoid duplicate installations
```

**Microsoft Defender for Endpoint:**
- Typically deployed via Intune or GPO post-deployment
- If required in image, use onboarding script from Microsoft 365 Defender portal

**SCCM/Intune Client:**
- Pre-install Configuration Manager client if using SCCM for patch management
- Intune enrollment happens post-deployment (device-based, not image-based)

#### Phase 6: Optimizations

**Virtual Desktop Optimization Tool (VDOT):**

Download from [GitHub - VDOT](https://github.com/The-Virtual-Desktop-Team/Virtual-Desktop-Optimization-Tool)

```powershell
# Download and extract VDOT
$vdotUrl = "https://github.com/The-Virtual-Desktop-Team/Virtual-Desktop-Optimization-Tool/archive/refs/heads/main.zip"
$vdotPath = "C:\Temp\VDOT.zip"
Invoke-WebRequest -Uri $vdotUrl -OutFile $vdotPath
Expand-Archive -Path $vdotPath -DestinationPath "C:\Temp\VDOT" -Force

# Run VDOT (select recommended optimizations)
cd "C:\Temp\VDOT\Virtual-Desktop-Optimization-Tool-main"
.\Windows_VDOT.ps1 -WindowsVersion "11" -Optimizations "All" -AdvancedOptimizations "None" -AcceptEULA

# VDOT disables unnecessary services, scheduled tasks, and Windows features
# Review C:\Temp\VDOT\VDOT_*.log for applied changes
```

**Manual Optimizations:**

```powershell
# Disable Windows Search indexing (reduces CPU/disk usage)
Set-Service -Name "WSearch" -StartupType Disabled
Stop-Service -Name "WSearch" -Force

# Configure pagefile (set to system-managed on C:)
$cs = Get-WmiObject -Class Win32_ComputerSystem
$cs.AutomaticManagedPagefile = $true
$cs.Put()

# Disable hibernation (saves disk space)
powercfg /hibernate off

# Set time zone (match your AVD region)
Set-TimeZone -Id "Eastern Standard Time"

# Enable RDP optimization registry keys
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
    -Name "MaxInstanceCount" -Value 4294967295 -PropertyType DWord -Force
```

### 3. Configure Settings

**Windows Settings:**
- **Privacy:** Disable telemetry and diagnostics where company policy allows
- **Display:** Set default scaling to 100% (users will adjust per-device)
- **Power Plan:** High Performance (for VMs, prevents CPU throttling)
- **Default Apps:** Set corporate-approved defaults (Edge for browser, Adobe for PDFs)

**Group Policy Configurations (Local GPO):**
- Configure FSLogix redirections (only if not using domain GPO)
- Disable Windows Store auto-updates (control updates via image versioning)
- Configure Windows Update to notify only (don't auto-install on session hosts)

**Certificates:**
- Import root CA certificates for internal PKI
- Import code-signing certificates for internal applications

**Network Settings:**
- Disable IPv6 if not used
- Configure DNS suffixes for corporate domains

### 4. Run Sysprep

Sysprep generalizes the Windows installation, removing computer-specific data and preparing the image for capture.

**Pre-Sysprep Checklist:**

```powershell
# 1. Verify no pending Windows updates
Get-WindowsUpdate

# 2. Check for pending reboots
Test-PendingReboot  # (if using PendingReboot module)

# 3. Clear temp files
Remove-Item -Path "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "C:\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue

# 4. Clear event logs (optional, for cleaner captures)
wevtutil el | ForEach-Object { wevtutil cl $_ }

# 5. Verify no users logged in (except current admin)
query user
```

**Run Sysprep:**

```powershell
# Navigate to sysprep directory
cd C:\Windows\System32\Sysprep

# Execute sysprep with generalize and shutdown
.\sysprep.exe /generalize /oobe /shutdown /mode:vm
```

**Sysprep Arguments Explained:**
- `/generalize` - Removes computer-specific information (SID, computer name, event logs)
- `/oobe` - Triggers Out-of-Box Experience on next boot (Azure handles this automatically)
- `/shutdown` - Powers down VM after sysprep completes (required for image capture)
- `/mode:vm` - Virtual machine mode (skips hardware-specific redetection)

**Expected Behavior:**
- Sysprep window appears, shows progress bars
- After 3-5 minutes, VM shuts down automatically
- In Azure Portal, VM status changes to "Stopped"

**If Sysprep Fails:**
- Check `C:\Windows\System32\Sysprep\Panther\setuperr.log` for errors
- Common causes: pending updates, Modern Apps (AppX) errors, corrupted user profiles
- See [Common Sysprep Failures](#common-sysprep-failures-and-fixes) below

### 5. Capture to Gallery

Once the VM is shut down (not just stopped, but deallocated), you can capture it to the Azure Compute Gallery.

**Azure Portal Path:** Azure Portal → Virtual Machines → [Your VM] → Capture

**Capture Configuration:**

1. **Resource Group:** Select your Compute Gallery's resource group (e.g., `RG-Azure-VDI-01`)
2. **Share image to Azure Compute Gallery:** Yes (required for versioning)
3. **Target Azure Compute Gallery:** `avd_gallery`
4. **Operating System State:** Generalized (mandatory—VM has been sysprepped)
5. **Gallery Image Definition:**
   - **Use existing:** Select `Win11_Multi_25H2_Gen2` (or create new if first capture)
   - **Create new:** Fill in Publisher, Offer, SKU, OS Type (Windows), OS State (Generalized), Gen (Gen 2)
6. **Version Number:** `1.0.0` (first version), `1.0.1` (subsequent patches), etc.
7. **Exclude from latest:** Unchecked (this version becomes default for deployments)
8. **Replication:**
   - **Default replica count:** 3
   - **Target regions:** East US 2 (3 replicas), West US 2 (2 replicas)
   - **Storage account type:** Premium SSD (faster replication, higher cost)
9. **Lifecycle:**
   - **End of life date:** Optional (e.g., +90 days for auto-expiration)
10. **Encryption:** Platform-managed keys
11. **Delete VM after capture:** Optional (recommended after validation)
    - **Warning:** If you delete the VM, you cannot recapture—test the image first!

**Capture Time:** 15-30 minutes for OS disk snapshot and metadata creation. Replication to additional regions happens in the background (30-60 minutes).

**Validation:**
1. Navigate to `avd_gallery` → `Win11_Multi_25H2_Gen2` → version `1.0.0`
2. Verify "Provisioning state: Succeeded"
3. Check "Replication state" tab—all target regions should show "Completed"
4. Test deployment: Create a test VM from the image version to verify it boots and applications work

## Software to Include

### Mandatory Components

**FSLogix Agent (2.9.9000 or later):**
- Enables profile containers and Office 365 container redirection
- Critical for multi-user session hosts to prevent profile corruption
- Without FSLogix, each user gets a temporary profile that's deleted at logoff

**Microsoft 365 Apps for Enterprise:**
- Word, Excel, PowerPoint, Outlook, Teams
- Must enable Shared Computer Licensing (`SharedComputerLicensing=1`)
- Use Monthly Enterprise Channel for stability (Semi-Annual for ultra-conservative environments)

**Microsoft Edge (Chromium):**
- Pre-installed on Windows 11, but verify it's updated
- Configure enterprise policies via GPO (homepage, proxy, extensions)

### Recommended Components

**Microsoft Teams (VDI optimized):**
- Use the VDI-optimized installer (enables media offload to local device)
- Download from [Teams VDI](https://aka.ms/teams-vdi)
- Install machine-wide: `msiexec /i Teams_windows_x64.msi ALLUSER=1 /qn`

**OneDrive for Business (VDI mode):**
- Install per-machine: `OneDriveSetup.exe /allusers`
- Enable Files On-Demand to minimize storage usage
- Configure Known Folder Move (Desktop, Documents, Pictures) via GPO

**Company VPN Client:**
- If users need VPN access from AVD session hosts (not common—usually VNet-to-VNet)
- Cisco AnyConnect, Palo Alto GlobalProtect, etc.

**PDF Reader:**
- Adobe Acrobat Reader DC (enterprise version, disable auto-updates)
- Alternative: Microsoft Edge (built-in PDF viewer)

**Monitoring Agents:**
- Azure Monitor Agent (AMA) - typically deployed post-deployment via policy
- Microsoft Defender for Endpoint - can pre-install or deploy via Intune
- SCCM client - if using ConfigMgr for patch management

**Security Tools:**
- Root CA certificates for internal PKI
- Company code-signing certificates
- Trusted SSL certificates for intranet sites

### Line-of-Business (LOB) Applications

**Industry-Specific:**
- **Healthcare:** Epic, Cerner, MEDITECH
- **Finance:** Bloomberg Terminal, Thomson Reuters Eikon
- **Legal:** Westlaw, LexisNexis
- **Engineering:** AutoCAD, SolidWorks (verify GPU requirements)

**General Business:**
- **CRM:** Salesforce (browser-based, no install), Dynamics 365
- **ERP:** SAP GUI, Oracle E-Business Suite
- **Accounting:** QuickBooks Desktop, Sage 50
- **Project Management:** Microsoft Project, Smartsheet

**Compatibility Considerations:**
- **Multi-session limitations:** Some apps don't support concurrent users (e.g., older versions of QuickBooks)
- **Licensing:** Verify app licenses allow VDI/terminal server deployment (many require additional licensing)
- **GPU requirements:** CAD/graphics apps may require NVv4 or NCasT4 VM series

## Sysprep Best Practices

**Always apply all Windows updates first** - Pending updates are the #1 cause of sysprep failures. Run `Get-WindowsUpdate` and verify "You're up to date" before sysprepping.

**Remove provisioned AppX packages** - Built-in Windows apps (Xbox, Candy Crush, etc.) can interfere with sysprep. Use VDOT or manually remove with `Get-AppxProvisionedPackage` and `Remove-AppxProvisionedPackage`.

**Avoid custom user profiles** - Do not log in with test users before sysprep. If you must test, delete the user profile before sysprepping: `Remove-LocalUser -Name "testuser"` and manually delete `C:\Users\testuser`.

**Use /mode:vm flag** - This tells sysprep it's running in a VM, skipping hardware re-detection that can cause issues.

**Don't run sysprep more than 3 times on the same VM** - Windows has a sysprep counter that blocks after 3 runs. If you need to re-sysprep, reset the counter: `Set-ItemProperty -Path "HKLM:\SYSTEM\Setup\Status\SysprepStatus" -Name "GeneralizationState" -Value 7`.

**Monitor sysprep logs in real-time** - Open `C:\Windows\System32\Sysprep\Panther\setuperr.log` in a text editor before running sysprep. Errors appear here immediately if sysprep fails.

**Disable antivirus during sysprep** - Third-party AV can interfere with file operations. Temporarily disable Defender or other AV before sysprepping.

**Clear temp files and event logs** - Reduces image size and removes potentially sensitive data from captured images.

**Test sysprep in a snapshot** - Before running sysprep on your final build VM, take a snapshot. If sysprep fails, you can revert and troubleshoot without rebuilding from scratch.

**Document all installed software** - Maintain a build checklist or script log of every application and version number. This is critical for reproducing images or troubleshooting deployment issues.

## Common Sysprep Failures and Fixes

| Issue | Cause | Solution |
|-------|-------|----------|
| **Sysprep fails with "AppX package error"** | Built-in Windows Store apps (Xbox, Candy Crush) prevent generalization | Remove AppX packages: `Get-AppxPackage -AllUsers \| Remove-AppxPackage -AllUsers`. Use VDOT to automate this. |
| **Sysprep fails with "Pending Windows updates"** | Windows Update has staged updates that haven't been installed | Install all updates: `Settings → Windows Update → Check for updates`. Reboot until "You're up to date" appears. |
| **Sysprep hangs at "Running sysprep"** | Corrupted user profiles or third-party software conflicts | Check `setuperr.log` for specific errors. Delete all user profiles except Administrator. Disable antivirus during sysprep. |
| **Sysprep error "A fatal error occurred while trying to sysprep"** | Registry corruption or invalid system state | Run `sfc /scannow` and `DISM /Online /Cleanup-Image /RestoreHealth` to repair system files. |
| **Sysprep completes but image won't boot** | Incorrect sysprep arguments (missing `/mode:vm`) or bootloader corruption | Use `/mode:vm` flag. Verify VM is Gen2 and Secure Boot is enabled. Recreate VM from marketplace and rebuild. |
| **Sysprep fails with "Maximum number of generalizations exceeded"** | Windows limits sysprep to 3 runs per installation | Reset counter: `Set-ItemProperty -Path "HKLM:\SYSTEM\Setup\Status\SysprepStatus" -Name "GeneralizationState" -Value 7`. Or rebuild VM from scratch. |
| **Sysprep runs but VM fails to deallocate** | Azure VM agent conflict or incorrect shutdown | Manually deallocate VM: `az vm deallocate --name <vm> --resource-group <rg>`. Wait for "Stopped (deallocated)" status. |
| **Capture fails with "OS disk is not generalized"** | Sysprep didn't complete successfully or VM was restarted | Verify `C:\Windows\System32\Sysprep\Panther\setupact.log` shows "Sysprep succeeded". If VM was restarted after sysprep, it's no longer generalized—must re-sysprep. |
| **New VMs from image get duplicate SIDs** | Sysprep was run without `/generalize` flag | Cannot fix deployed VMs—must recapture image with correct sysprep command. Use NewSID tool as temporary workaround (not recommended for production). |
| **FSLogix agent missing after deploy** | FSLogix service set to Disabled before sysprep | Verify service startup type is Automatic: `Set-Service -Name frxsvc -StartupType Automatic`. Re-run sysprep and recapture. |

## Alternative: Automated Builds

While manual image creation is essential to understand, it's time-consuming and error-prone. For production environments, consider:

- **Azure Image Builder:** Automates the entire process with YAML/JSON templates. See [[azure-image-builder]].
- **Packer by HashiCorp:** Infrastructure-as-code tool for building images across Azure, AWS, VMware.
- **SCCM/Intune Task Sequences:** Traditional enterprise imaging workflows adapted for Azure.

Manual builds are best for:
- Initial image creation and testing
- One-off custom images
- Troubleshooting and learning the process
- Small environments (<10 session hosts)

Automated builds are best for:
- Production environments
- Frequent image updates (monthly patch cycles)
- Multi-region deployments
- Compliance and audit requirements (full build log traceability)

## Next Steps

- **[[azure-image-builder]]** - Automate image builds with repeatable templates
- **[[image-versioning]]** - Strategies for managing multiple image versions
- **[[azure-compute-gallery]]** - Understanding the gallery structure for storing images