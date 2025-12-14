---
title: Step 03 - Azure Compute Gallery & Golden Images
description: Create Azure Compute Gallery and build golden images for AVD deployment
published: true
date: 2025-12-14T00:00:00.000Z
tags: [Quick-Deploy, images, gallery, golden-image]
---

# Step 03: Azure Compute Gallery & Golden Images

Create the Azure Compute Gallery, define image templates, and build golden images for both multi-session (pooled) and single-session (personal) AVD deployments. This is a foundational but time-intensive step that establishes the base OS configuration for all session hosts.

> **Time Warning:** Golden image creation is a ~4 hour manual process per image type. Plan accordingly and consider building both images in parallel if you have two administrators available.

## Example Scenario

From our 200-user deployment (see [[00-naming-conventions]]):
- **Pooled desktops (150 users):** Windows 11 multi-session image
- **Personal desktops (50 users):** Windows 11 single-session image

Both images will include:
- Base Windows 11 Enterprise 23H2
- Latest Windows updates
- FSLogix agent (profile management)
- **Placeholder for apps** - Office 365 and Adobe will be installed later via Intune (not baked into image)

## Prerequisites

- [ ] Completed [[01-prerequisites-licensing]]
- [ ] Completed [[02-identity-setup]]
- [ ] Resource group created: `rg-avd-prod-eastus-01`
- [ ] Virtual network created (networking setup - if not complete, can use temporary build network)
- [ ] 8-10 hours of uninterrupted time (4 hours per image, can overlap)

**Duration:** 8-10 hours total (4 hours per image type)
- Azure Compute Gallery creation: ~5 minutes
- Base VM deployment: ~10 minutes
- Windows updates: ~1 hour
- Software installation: ~1 hour
- Optimizations: ~30 minutes
- Sysprep and capture: ~30 minutes
- Replication: ~30 minutes (background, can proceed to next image)

---

## Part 1: Create Azure Compute Gallery

The Azure Compute Gallery stores versioned images and replicates them across regions.

**Portal:** Azure Portal → Create a resource → Search "Azure Compute Gallery"

1. Click **+ Create**
2. **Basics tab:**
   - **Subscription:** Your Azure subscription
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Name:** `gal-avd-prod-01`
   - **Region:** East US (match your primary AVD region)
   - **Description:** `Production AVD golden images for pooled and personal deployments`

3. **Sharing tab:**
   - **Sharing:** RBAC (default - keeps sharing within your organization)

4. **Tags:**
   - `Environment: Production`
   - `Workload: AVD`
   - `ManagedBy: IT-Operations`

5. Click **Review + create**
6. Click **Create**

**Deployment time:** ~30 seconds

**Validation:**
- Navigate to: Azure Portal → Azure Compute Gallery → `gal-avd-prod-01`
- Verify **Provisioning state: Succeeded**

**See:** [[azure-compute-gallery|Azure Compute Gallery Reference]] for advanced sharing and replication strategies.

---

## Part 2: Create Image Definitions

Image definitions are logical containers that hold versioned snapshots. Create two definitions: one for multi-session, one for single-session.

### Multi-Session Image Definition

**Portal:** Azure Portal → Compute galleries → `gal-avd-prod-01` → + Create → VM image definition

1. **Basics tab:**
   - **VM image definition name:** `win11-multisession-23h2`
   - **Region:** East US (inherited from gallery)
   - **Publisher:** `Internal`
   - **Offer:** `Windows-11-AVD`
   - **SKU:** `23H2-MultiSession`

2. **Operating system tab:**
   - **OS type:** Windows
   - **OS state:** Generalized
   - **VM generation:** Gen 2 (required for modern VMs)
   - **Security type:** Trusted launch virtual machines (recommended)

3. **Recommended configuration:**
   - **vCPUs:** 4-8
   - **Memory (GB):** 16-32
   - **Description:** `Windows 11 Enterprise multi-session 23H2 with FSLogix. Apps deployed via Intune.`

4. **Lifecycle policy:**
   - Leave default (no expiration)

5. Click **Review + create**
6. Click **Create**

### Single-Session Image Definition

Repeat the process with these changes:

1. **Basics tab:**
   - **VM image definition name:** `win11-singlesession-23h2`
   - **SKU:** `23H2-SingleSession`

2. **Recommended configuration:**
   - **Description:** `Windows 11 Enterprise single-session 23H2 with FSLogix. Apps deployed via Intune.`

3. Click **Review + create** and **Create**

**Validation:**
- Navigate to `gal-avd-prod-01` → Image definitions
- Verify both `win11-multisession-23h2` and `win11-singlesession-23h2` appear

---

## Part 3: Build Multi-Session Golden Image

This section walks through creating the Windows 11 multi-session golden image. The process is manual and takes approximately 4 hours.

> **Tip:** Have a second administrator build the single-session image in parallel following Part 4 to save time.

### Step 3.1: Create Base VM from Marketplace

**Portal:** Azure Portal → Create a resource → Virtual machines → + Create

1. **Basics tab:**
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Virtual machine name:** `vm-image-build-multi`
   - **Region:** East US
   - **Availability options:** No infrastructure redundancy required
   - **Security type:** Trusted launch virtual machines
   - **Image:** Search marketplace for:
     - `Windows 11 Enterprise multi-session, Version 23H2 - x64 Gen2`
   - **VM size:** Standard_D4s_v5 (4 vCPU, 16GB RAM)
     - Click "See all sizes" if not visible
     - Larger than production to speed up builds

2. **Administrator account:**
   - **Username:** `localadmin`
   - **Password:** Create strong password (20+ chars, save to password manager)

3. **Inbound port rules:**
   - **Public inbound ports:** Allow selected ports
   - **Select inbound ports:** RDP (3389)

4. **Licensing:**
   - Check: "I confirm I have an eligible Windows 11 license with multi-tenant hosting rights"

5. **Disks tab:**
   - **OS disk type:** Premium SSD (faster for builds)
   - **Delete with VM:** Yes (cleanup after capture)

6. **Networking tab:**
   - **Virtual network:** Create new or use existing
   - **Subnet:** Any available subnet (temporary)
   - **Public IP:** Create new (for RDP access)
   - **NIC network security group:** Basic
   - **Public inbound ports:** Allow RDP from your IP only
     - Click "Advanced" → Create new NSG
     - Edit inbound rules → RDP → Source: My IP address

7. **Management tab:**
   - **Enable auto-shutdown:** On
   - **Shutdown time:** 7:00 PM (prevents accidental overnight costs)
   - **Boot diagnostics:** Enable with managed storage account

8. **Monitoring tab:**
   - **Enable OS guest diagnostics:** Off (not needed for build VM)

9. **Tags:**
   - `Purpose: ImageBuild`
   - `DeleteAfter: 2025-01-30` (reminder to clean up)

10. Click **Review + create**
11. Click **Create**

**Deployment time:** ~5 minutes

**First boot:** Once deployment completes:
1. Click **Go to resource**
2. Click **Connect → RDP**
3. Download RDP file
4. Open RDP file and connect with `localadmin` credentials
5. Windows OOBE (Out-of-Box Experience) will complete automatically

### Step 3.2: Install Windows Updates

> **Critical:** Apply ALL Windows updates before installing applications. Incomplete updates cause sysprep failures.

**Connect to VM via RDP, then:**

1. Open **Settings** (Win + I)
2. Navigate to **Windows Update**
3. Click **Check for updates**
4. Install all available updates
5. **Restart** when prompted
6. **Repeat** steps 2-5 until "You're up to date" appears

**Verification:**

```powershell
# Run in PowerShell as Administrator
Get-WindowsUpdate

# Expected output: "No updates available" or "You're up to date"
```

**Expected time:** 45-90 minutes (varies based on patch cycle)

> **Note:** If near Patch Tuesday (second Tuesday of month), updates may take longer. Budget extra time.

### Step 3.3: Install FSLogix Agent

FSLogix is mandatory for AVD profile management.

**Download FSLogix:**

1. Open browser in VM
2. Navigate to: https://aka.ms/fslogix-latest
3. Download installer (currently version 2.9.9+)
4. Save to `C:\Temp\`

**Install FSLogix:**

```powershell
# Run in PowerShell as Administrator
New-Item -Path "C:\Temp" -ItemType Directory -Force

# Extract downloaded ZIP (if downloaded manually, extract via GUI)
# Assuming FSLogixAppsSetup.zip is in Downloads folder
Expand-Archive -Path "$env:USERPROFILE\Downloads\FSLogix-Apps-*.zip" -DestinationPath "C:\Temp\FSLogix" -Force

# Install FSLogix
Start-Process -FilePath "C:\Temp\FSLogix\x64\Release\FSLogixAppsSetup.exe" -ArgumentList "/install", "/quiet", "/norestart" -Wait

# Verify installation
Get-Service frxsvc, frxccds
# Expected: Both services Running
```

**Post-install verification:**

```powershell
# Check registry for version
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Apps" | Select-Object Version

# Expected output: Version 2.9.9xxxxx or later
```

> **Important:** Do NOT configure FSLogix profile paths in the image. Profile container paths are set via GPO or Intune policy after deployment.

**See:** [[fslogix-profile-containers|FSLogix Configuration Reference]] for post-deployment configuration.

### Step 3.4: Placeholder - Apps Installed via Intune

**Microsoft 365 Apps, Adobe Acrobat, and other applications will be deployed via Intune AFTER session hosts are live.**

This "thin image" approach:
- **Reduces image build time** from 4 hours to 2 hours
- **Simplifies app updates** (no image rebuild for app patches)
- **Improves flexibility** (different apps for different departments via Intune targeting)

If you must bake apps into the image (legacy requirement or offline scenario), see [[golden-image-process|Golden Image Process]] for detailed app installation steps.

### Step 3.5: Apply Optimizations

Use Microsoft's Virtual Desktop Optimization Tool (VDOT) to improve performance.

**Download VDOT:**

```powershell
# Download VDOT from GitHub
$vdotUrl = "https://github.com/The-Virtual-Desktop-Team/Virtual-Desktop-Optimization-Tool/archive/refs/heads/main.zip"
$vdotPath = "C:\Temp\VDOT.zip"
Invoke-WebRequest -Uri $vdotUrl -OutFile $vdotPath

# Extract
Expand-Archive -Path $vdotPath -DestinationPath "C:\Temp\VDOT" -Force
```

**Run VDOT:**

```powershell
# Navigate to VDOT directory
cd "C:\Temp\VDOT\Virtual-Desktop-Optimization-Tool-main"

# Run with recommended settings for Windows 11
.\Windows_VDOT.ps1 -WindowsVersion "11" -Verbose -AcceptEULA

# Review output in terminal for applied optimizations
```

**What VDOT does:**
- Disables unnecessary Windows services (Xbox, Windows Search indexing)
- Removes built-in apps (Candy Crush, Xbox Game Bar)
- Configures Windows Update to notify-only mode
- Optimizes visual effects and animations
- Disables scheduled tasks that impact performance

**Manual optimizations (optional but recommended):**

```powershell
# Disable Windows Search indexing (reduces disk I/O)
Stop-Service -Name "WSearch" -Force
Set-Service -Name "WSearch" -StartupType Disabled

# Disable hibernation (saves disk space)
powercfg /hibernate off

# Set time zone to match AVD region
Set-TimeZone -Id "Eastern Standard Time"

# Configure high performance power plan
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
```

**Verification:**

```powershell
# Check Windows Search service
Get-Service WSearch
# Expected: Status = Stopped, StartType = Disabled

# Check time zone
Get-TimeZone
# Expected: Eastern Standard Time (or your region)
```

**Expected time:** 20-30 minutes

### Step 3.6: Clean Up Before Sysprep

Remove temporary files and clear logs to reduce image size.

```powershell
# Clear temp files
Remove-Item -Path "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "C:\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue

# Clear event logs (optional, for cleaner captures)
wevtutil el | ForEach-Object { wevtutil cl $_ }

# Clear browser cache and user temp files
Remove-Item -Path "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue
```

**Pre-sysprep checklist:**

```powershell
# 1. Verify no pending Windows updates
# Settings → Windows Update → Check for updates
# Must show: "You're up to date"

# 2. Verify no pending reboots
# Check: No yellow warning in Windows Update

# 3. Verify FSLogix installed
Get-Service frxsvc
# Expected: Running

# 4. Verify no logged-in users (except current admin)
query user
# Expected: Only localadmin shown
```

### Step 3.7: Run Sysprep

Sysprep generalizes the Windows installation, removing computer-specific data to prepare for cloning.

> **Critical:** Do NOT skip any pre-sysprep checks. Sysprep failures are difficult to troubleshoot.

**Run sysprep:**

```powershell
# Navigate to sysprep directory
cd C:\Windows\System32\Sysprep

# Execute sysprep with generalize and shutdown
.\sysprep.exe /generalize /oobe /shutdown /mode:vm
```

**Sysprep arguments explained:**
- `/generalize` - Removes computer-specific info (SID, hostname, event logs)
- `/oobe` - Triggers Out-of-Box Experience on next boot (Azure handles this)
- `/shutdown` - Powers down VM after sysprep completes (required for capture)
- `/mode:vm` - VM mode (skips hardware re-detection)

**Expected behavior:**
1. Sysprep window appears
2. Progress bar shows "Generalizing..."
3. After 3-5 minutes, VM shuts down automatically
4. In Azure Portal, VM status changes to "Stopped"

**If sysprep fails:**

1. Check error log:
   ```
   C:\Windows\System32\Sysprep\Panther\setuperr.log
   ```

2. Common issues:
   - **Pending updates:** Install all updates, reboot until "You're up to date"
   - **AppX package errors:** Run VDOT again or manually remove problematic apps
   - **User profile corruption:** Delete all user profiles except localadmin

**See:** [[golden-image-process#sysprep-best-practices|Sysprep Best Practices]] for detailed troubleshooting.

**Expected time:** 5-10 minutes

### Step 3.8: Deallocate VM

After sysprep shuts down the VM, you must deallocate it (not just stop it) before capturing.

**Portal:** Azure Portal → Virtual machines → `vm-image-build-multi`

1. Click **Stop** (if not already stopped)
2. Wait for status: "Stopped (deallocated)"
   - "Stopped" alone is NOT sufficient
   - Must be "Stopped (deallocated)" to capture

**Azure CLI alternative:**

```bash
az vm deallocate \
  --resource-group rg-avd-prod-eastus-01 \
  --name vm-image-build-multi
```

**Verification:**
- VM status shows: **Stopped (deallocated)**
- This usually takes 1-2 minutes

### Step 3.9: Capture Image to Gallery

Now capture the generalized VM as a versioned image in the Compute Gallery.

**Portal:** Azure Portal → Virtual machines → `vm-image-build-multi` → Capture

1. Click **Capture** (top menu bar)

2. **Instance details:**
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Share image to Azure Compute Gallery:** Yes
   - **Operating system state:** Generalized (already set)
   - **Automatically delete this virtual machine after creating the image:** No (keep for validation)

3. **Target Azure Compute Gallery:**
   - **Azure Compute Gallery:** `gal-avd-prod-01`

4. **Target VM image definition:**
   - **Operating system:** Windows
   - **VM image definition:** `win11-multisession-23h2` (select existing)
   - **Version number:** `1.0.0`
   - **Exclude from latest:** Unchecked (this becomes the default version)
   - **End of life date:** Leave blank (or set to +90 days for testing)

5. **Replication:**
   - **Default replica count:** 3 (supports ~30 concurrent deployments)
   - **Target regions:**
     - **East US:** 3 replicas (primary region)
     - Add additional regions if multi-region deployment planned
   - **Storage account type:** Premium SSD (faster replication)

6. **Encryption:**
   - **Encryption type:** Platform-managed keys (default)

7. Click **Review + create**
8. Click **Create**

**Capture time:** 15-30 minutes for initial snapshot

**Background replication:** 30-60 minutes (happens after capture completes)

**Validation:**

1. Navigate to: `gal-avd-prod-01` → `win11-multisession-23h2` → Versions
2. Verify version `1.0.0` shows **Provisioning state: Succeeded**
3. Click on version `1.0.0` → **Replication status** tab
4. Wait for all target regions to show **Completed**

> **Note:** You can proceed to build the single-session image while replication completes in the background.

**See:** [[image-versioning|Image Versioning Reference]] for version numbering strategies and update workflows.

---

## Part 4: Build Single-Session Golden Image

Repeat the process from Part 3 with these key differences:

### Step 4.1: Create Base VM

Same as 3.1, but:
- **VM name:** `vm-image-build-single`
- **Image:** `Windows 11 Enterprise, Version 23H2 - x64 Gen2` (single-session, NOT multi-session)
- **Licensing:** "I confirm I have an eligible Windows 11 license" (no multi-tenant hosting checkbox)

### Step 4.2-4.7: Follow Identical Process

- Install Windows Updates (same as 3.2)
- Install FSLogix Agent (same as 3.3)
- Skip app installation (same as 3.4 - apps via Intune)
- Apply Optimizations (same as 3.5)
- Clean up (same as 3.6)
- Run Sysprep (same as 3.7)

### Step 4.8: Deallocate VM

Same as 3.8:
- Stop VM: `vm-image-build-single`
- Verify: "Stopped (deallocated)"

### Step 4.9: Capture Image

Same as 3.9, but:
- **VM image definition:** `win11-singlesession-23h2` (NOT multi-session)
- **Version number:** `1.0.0`
- All other settings identical

**Expected total time for single-session image:** 4 hours (same as multi-session)

---

## Verification Checklist

Confirm both images are ready for deployment:

### Azure Compute Gallery

**Portal:** Azure Portal → Compute galleries → `gal-avd-prod-01`

- [ ] Gallery `gal-avd-prod-01` created in `rg-avd-prod-eastus-01`
- [ ] Gallery provisioning state: Succeeded
- [ ] Two image definitions exist:
  - [ ] `win11-multisession-23h2`
  - [ ] `win11-singlesession-23h2`

### Image Definitions

**Portal:** `gal-avd-prod-01` → Image definitions

For each image definition:

- [ ] **win11-multisession-23h2:**
  - [ ] OS type: Windows
  - [ ] OS state: Generalized
  - [ ] VM generation: Gen 2
  - [ ] Security type: Trusted launch
  - [ ] Version `1.0.0` exists

- [ ] **win11-singlesession-23h2:**
  - [ ] OS type: Windows
  - [ ] OS state: Generalized
  - [ ] VM generation: Gen 2
  - [ ] Security type: Trusted launch
  - [ ] Version `1.0.0` exists

### Image Versions

**Portal:** Each image definition → Versions → `1.0.0`

- [ ] Multi-session v1.0.0:
  - [ ] Provisioning state: Succeeded
  - [ ] Replication status: All regions show "Completed"
  - [ ] Excluded from latest: No

- [ ] Single-session v1.0.0:
  - [ ] Provisioning state: Succeeded
  - [ ] Replication status: All regions show "Completed"
  - [ ] Excluded from latest: No

### Build VMs (Cleanup)

- [ ] `vm-image-build-multi` deallocated (can delete after validation)
- [ ] `vm-image-build-single` deallocated (can delete after validation)

> **Recommendation:** Keep build VMs for 24-48 hours in case you need to recapture. Delete after confirming session hosts deploy successfully from images.

---

## Cleanup Build Resources

After confirming images work (test deployment in next step):

**Portal:** Azure Portal → Resource groups → `rg-avd-prod-eastus-01`

1. Select both build VMs:
   - `vm-image-build-multi`
   - `vm-image-build-single`

2. Click **Delete**
3. Confirm deletion by typing VM name
4. Check: "Delete associated disks, NICs, and public IPs"
5. Click **Delete**

**Cost savings:** Deleting build VMs saves ~$200/month in compute costs. Image versions remain in gallery indefinitely.

---

## Troubleshooting

### Issue: Sysprep fails with "AppX package error"

**Symptom:** Sysprep stops with error mentioning "Package_for_CandyCrush" or similar

**Cause:** Built-in Windows Store apps prevent generalization

**Fix:**

```powershell
# Run VDOT to remove AppX packages
cd "C:\Temp\VDOT\Virtual-Desktop-Optimization-Tool-main"
.\Windows_VDOT.ps1 -WindowsVersion "11" -Verbose -AcceptEULA

# Manually remove remaining problematic apps
Get-AppxPackage -AllUsers | Where-Object {$_.Name -like "*Xbox*"} | Remove-AppxPackage -AllUsers
Get-AppxPackage -AllUsers | Where-Object {$_.Name -like "*CandyCrush*"} | Remove-AppxPackage -AllUsers

# Retry sysprep
cd C:\Windows\System32\Sysprep
.\sysprep.exe /generalize /oobe /shutdown /mode:vm
```

### Issue: Image capture fails with "Disk not found"

**Symptom:** Azure Portal error when clicking "Capture"

**Cause:** VM not fully deallocated

**Fix:**

```bash
# Force deallocate via CLI
az vm deallocate \
  --resource-group rg-avd-prod-eastus-01 \
  --name vm-image-build-multi

# Verify status
az vm show \
  --resource-group rg-avd-prod-eastus-01 \
  --name vm-image-build-multi \
  --query "provisioningState"

# Expected output: "Succeeded" and VM status "Stopped (deallocated)"
```

### Issue: Replication stuck at "In Progress" for hours

**Symptom:** Image version replication doesn't complete after 2 hours

**Cause:** Regional capacity issues or network throttling

**Fix:**

1. Check Azure Service Health:
   - Portal → Service Health → Service issues
   - Look for outages in target regions

2. Reduce target regions:
   - Edit image version → Replication
   - Remove problematic regions temporarily
   - Re-add after initial replication completes

3. Wait longer:
   - Premium SSD: ~30-60 minutes for 127GB image
   - Standard HDD: up to 4 hours

### Issue: FSLogix service not running after image deploy

**Symptom:** Session hosts deployed from image don't have FSLogix running

**Cause:** Service startup type set to Disabled before sysprep

**Fix (rebuild image):**

```powershell
# Before running sysprep, verify FSLogix service startup type
Get-Service frxsvc | Select-Object Name, StartType, Status

# Should be: StartType = Automatic

# If Disabled, fix with:
Set-Service -Name frxsvc -StartupType Automatic
Set-Service -Name frxccds -StartupType Automatic

# Then run sysprep
```

**See:** [[golden-image-process#common-sysprep-failures-and-fixes|Common Sysprep Failures]] for more troubleshooting scenarios.

---

## Time Optimization Tips

### Parallel Building

- Have two administrators build multi-session and single-session images simultaneously
- Reduces total time from 8 hours to 4 hours

### Use Azure Bastion Instead of Public IP

- Faster RDP connection
- No need to configure NSG rules for your IP
- See [[azure-bastion|Azure Bastion]] for setup

### Schedule Builds Overnight

1. Start Windows Updates at 5:00 PM
2. Let updates install overnight (automated via script)
3. Continue manual steps next morning

### Automate Future Updates

After initial manual build, consider:
- **Azure Image Builder:** Automates monthly patch versions
- See [[azure-image-builder|Azure Image Builder]] for automation templates

---

## Next Steps

**Images are ready.** You can now deploy session hosts from these golden images.

**Immediate next step:** [[04-host-pool-deployment|Step 04: Host Pool Deployment]]

**Future image updates:**
- **Monthly:** Rebuild with Windows Updates (version 1.0.1, 1.0.2, etc.)
- **Quarterly:** Add new applications via Intune (or rebuild image if needed)
- **Annually:** Upgrade to next Windows version (e.g., 24H2 → 25H2)

**See:** [[image-versioning|Image Versioning]] for update strategies and rollout procedures.

---

## Related Reference Pages

- [[golden-image-process|Golden Image Process]] - Detailed walkthrough with app installation
- [[azure-compute-gallery|Azure Compute Gallery]] - Gallery architecture and sharing
- [[image-versioning|Image Versioning]] - Version numbering and rollout strategies
- [[azure-image-builder|Azure Image Builder]] - Automate future image builds
- [[fslogix-profile-containers|FSLogix Configuration]] - Profile container setup (post-deployment)
