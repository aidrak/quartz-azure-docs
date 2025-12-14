---
title: Step 09 - FSLogix Configuration via Intune
description: Configure FSLogix profile containers on session hosts using Intune Settings Catalog
published: true
date: 2025-12-14T00:00:00.000Z
tags:
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 09: FSLogix Configuration via Intune

Deploy FSLogix profile container settings to AVD session hosts using Intune Settings Catalog. FSLogix provides persistent user profiles across non-persistent pooled session hosts, ensuring users retain their settings, desktop files, and application data between sessions.

## Example Scenario

Using naming conventions from [[00-naming-conventions]]:

| Resource | Name | Target Group | Purpose |
|----------|------|--------------|---------|
| Intune Config Profile (Pooled) | `AVD - FSLogix - Pooled Profile Containers` | AVD-Devices-Pooled | FSLogix settings for pooled hosts |
| Intune Config Profile (Personal) | `AVD - FSLogix - Personal Profile Containers` | AVD-Devices-Personal | FSLogix settings for personal hosts |
| Storage Account | `stavdprodeus01` | N/A | Azure Files storage for profiles |
| File Share (Pooled) | `profiles-pooled` | N/A | Profile storage for pooled users |
| File Share (Personal) | `profiles-personal` | N/A | Profile storage for personal users |

**Target Session Hosts:**
- **Pooled:** `vm-pooled-prod-001` through `vm-pooled-prod-010` (10 VMs)
- **Personal:** `vm-personal-prod-011` through `vm-personal-prod-050` (40 VMs)

## Prerequisites

- [ ] Storage account created: `stavdprodeus01` with Azure Files Premium (from [[05-storage-fslogix]])
- [ ] File shares created: `profiles-pooled`, `profiles-personal` (from [[05-storage-fslogix]])
- [ ] RBAC permissions configured on file shares (from [[05-storage-fslogix]])
- [ ] NTFS permissions set on share root folders (from [[05-storage-fslogix]])
- [ ] Session hosts deployed and Intune-enrolled (from [[07-session-hosts]])
- [ ] Device groups created: `AVD-Devices-Pooled`, `AVD-Devices-Personal` (from [[02-identity-setup]])
- [ ] Intune Policy Administrator or Endpoint Security Manager role

> **Note:** FSLogix agent is pre-installed on Windows 11 multi-session and single-session images from Azure Marketplace. No manual installation required.

---

## Part 1: Understanding FSLogix for AVD

### Why FSLogix is Critical for Pooled Desktops

**Without FSLogix (Pooled Host Pool):**

| Session 1 | Session 2 | Problem |
|-----------|-----------|---------|
| User logs into `vm-pooled-prod-001` | User logs into `vm-pooled-prod-003` (different VM) | New profile created each session |
| Sets desktop wallpaper, installs apps | Desktop defaults, apps missing | Settings lost |
| 20-minute login (new profile creation) | 20-minute login (new profile creation) | Poor user experience |

**With FSLogix (Pooled Host Pool):**

| Session 1 | Session 2 | Benefit |
|-----------|-----------|---------|
| User logs into `vm-pooled-prod-001` | User logs into `vm-pooled-prod-003` (different VM) | Same profile VHD mounts |
| Profile VHD created: `\\stavdprodeus01\profiles-pooled\jdoe_jdoe@contoso.com\Profile_jdoe.vhdx` | Same VHD mounts at `C:\Users\jdoe` | Profile persists |
| 5-second login (VHD mount) | 5-second login (VHD mount) | Fast, consistent experience |

**See:** [[../Storage/fslogix-profile-containers|FSLogix Profile Containers]] for detailed architecture and how FSLogix works.

### FSLogix on Personal vs Pooled Hosts

**Personal Host Pools:**
- Users connect to the SAME session host every session (1:1 assignment)
- Local Windows profile persists automatically (no roaming needed)
- FSLogix is OPTIONAL for personal hosts (used only for profile portability if user reassigned to different VM)

**Pooled Host Pools:**
- Users connect to DIFFERENT session hosts each session (load balanced)
- Local Windows profile destroyed on logoff (non-persistent VMs)
- FSLogix is REQUIRED for pooled hosts (ensures profile follows user)

> **Decision:** Configure FSLogix on BOTH pooled and personal hosts. Personal hosts benefit from profile portability and easier troubleshooting (profiles centralized on Azure Files).

---

## Part 2: Create FSLogix Configuration Profile (Pooled)

Create Intune Settings Catalog profile to configure FSLogix on pooled session hosts.

### Create Settings Catalog Profile

**Portal:** Intune Admin Center → Devices → Configuration profiles → + Create profile

1. **Platform:** Windows 10 and later
2. **Profile type:** Settings catalog
3. Click **Create**

### Configure Profile Basics

1. **Basics:**
   - **Name:** `AVD - FSLogix - Pooled Profile Containers`
   - **Description:** `FSLogix profile container settings for pooled multi-session hosts. Directs profiles to \\stavdprodeus01\profiles-pooled`
   - Click **Next**

### Add FSLogix Settings

2. **Configuration settings:**
   - Click **+ Add settings**
   - In search box, type: `FSLogix`
   - Expand: **FSLogix → Profiles**
   - Select the following settings (checkboxes):

**Core Settings:**

| Setting | Configure As | Purpose |
|---------|-------------|---------|
| **Enabled** | Enabled | Activates FSLogix profile containers |
| **VHDLocations** | `\\stavdprodeus01.file.core.windows.net\profiles-pooled` | Path to Azure Files share for pooled profiles |
| **DeleteLocalProfileWhenVHDShouldApply** | Enabled | Clean up local profiles (prevents disk bloat on session hosts) |
| **FlipFlopProfileDirectoryName** | Enabled | Use `username_upn` format for folders (e.g., `jdoe_jdoe@contoso.com`) |
| **SizeInMBs** | 30000 | 30GB max profile size |
| **VolumeType** | VHDX | Modern VHD format (more resilient than VHD) |
| **IsDynamic** | Enabled | Dynamic disk expansion (saves storage, only consumes space actually used) |

**Advanced Settings (Optional but Recommended):**

| Setting | Configure As | Purpose |
|---------|-------------|---------|
| **LockedRetryCount** | 3 | Retry attempts if profile VHD is locked (handles orphaned locks) |
| **LockedRetryInterval** | 15 | Wait 15 seconds between retry attempts |
| **ProfileType** | 0 | Use profile containers (not redirected profiles) |
| **PreventLoginWithFailure** | Enabled | Block login if profile fails to load (prevents temp profile confusion) |
| **PreventLoginWithTempProfile** | Enabled | Block login with temporary profile (forces profile troubleshooting) |

**Exclusions (Reduces Profile Size):**

Add these settings to exclude folders from profile container (saves storage, improves performance):

| Setting | Value | Purpose |
|---------|-------|---------|
| **RedirXMLSourceFolder** | `C:\Program Files\FSLogix\Apps\Redirections.xml` | Path to folder exclusion config file |

> **Note:** `Redirections.xml` excludes temp files, browser caches, and OneDrive sync folders from profile. Default exclusions work well for most deployments.

3. Click **Next**

### Assign to Device Group

3. **Assignments:**
   - **Assign to:** Selected groups
   - Click **+ Select groups to include**
   - Search for: `AVD-Devices-Pooled`
   - Select the group
   - Click **Select**
   - **Exclude groups:** None
   - Click **Next**

4. **Applicability Rules:** (None required)
   - Click **Next**

5. **Review + create:**
   - Review all settings
   - Verify VHDLocations path is correct: `\\stavdprodeus01.file.core.windows.net\profiles-pooled`
   - Click **Create**

**Deployment time:** Profile created instantly, applies to devices within 8 hours (or on-demand sync)

### Verify Profile Creation

**Portal:** Intune Admin Center → Devices → Configuration profiles

- [ ] `AVD - FSLogix - Pooled Profile Containers` appears in list
- [ ] Profile type: Settings catalog
- [ ] Platform: Windows 10 and later

---

## Part 3: Create FSLogix Configuration Profile (Personal)

Repeat profile creation for personal session hosts with different VHDLocations path.

### Create Settings Catalog Profile

**Portal:** Intune Admin Center → Devices → Configuration profiles → + Create profile

1. **Platform:** Windows 10 and later
2. **Profile type:** Settings catalog
3. Click **Create**

### Configure Profile Basics

1. **Basics:**
   - **Name:** `AVD - FSLogix - Personal Profile Containers`
   - **Description:** `FSLogix profile container settings for personal single-session hosts. Directs profiles to \\stavdprodeus01\profiles-personal`
   - Click **Next**

### Add FSLogix Settings

2. **Configuration settings:**
   - Click **+ Add settings**
   - Search: `FSLogix`
   - Expand: **FSLogix → Profiles**
   - Add SAME settings as pooled profile with ONE change:

**Key Difference:**

| Setting | Pooled Value | Personal Value |
|---------|--------------|----------------|
| **VHDLocations** | `\\stavdprodeus01.file.core.windows.net\profiles-pooled` | `\\stavdprodeus01.file.core.windows.net\profiles-personal` |

All other settings identical to pooled profile.

3. Click **Next**

### Assign to Device Group

3. **Assignments:**
   - **Assign to:** Selected groups
   - Search for: `AVD-Devices-Personal`
   - Select the group
   - Click **Next**

4. **Applicability Rules:** (None)
   - Click **Next**

5. **Review + create:**
   - Verify VHDLocations: `\\stavdprodeus01.file.core.windows.net\profiles-personal`
   - Click **Create**

---

## Part 4: Force Policy Sync to Session Hosts

Intune policies sync every 8 hours by default. Force immediate sync for testing.

### Option 1: Sync via Intune Portal

**Portal:** Intune Admin Center → Devices → All devices

1. Filter devices: Search for `vm-pooled-prod-` or `vm-personal-prod-`
2. Select one or more session hosts
3. Click **Sync** (top toolbar)
4. Confirmation: "Sync command sent to X devices"
5. Wait 5-10 minutes for policy to apply

### Option 2: Sync via Session Host (PowerShell)

RDP to session host as administrator:

**PowerShell (Run as Administrator):**

```powershell
# Trigger Intune policy sync
Get-ScheduledTask | Where-Object {$_.TaskName -eq 'PushLaunch'} | Start-ScheduledTask

# Wait 2 minutes for sync to complete
Start-Sleep -Seconds 120

# Verify FSLogix registry keys created
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles" | Format-List

# Expected output:
# Enabled              : 1
# VHDLocations         : \\stavdprodeus01.file.core.windows.net\profiles-pooled
# SizeInMBs            : 30000
# IsDynamic            : 1
# VolumeType           : VHDX
```

**Expected Result:** Registry keys populated with FSLogix settings from Intune profile.

### Option 3: Sync via Company Portal App

RDP to session host as administrator:

1. Open Start menu
2. Search for: `Company Portal`
3. Open Company Portal app
4. Click **Settings** (gear icon, top-right)
5. Click **Sync**
6. Wait for "Sync completed successfully" message

---

## Part 5: Verify FSLogix Configuration

Confirm FSLogix settings applied correctly to session hosts.

### Verify Registry Settings (Session Host)

RDP to session host as administrator:

**PowerShell:**

```powershell
# Check FSLogix Profiles registry
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles"

# Expected output:
# Enabled                              : 1
# VHDLocations                         : \\stavdprodeus01.file.core.windows.net\profiles-pooled
# DeleteLocalProfileWhenVHDShouldApply : 1
# FlipFlopProfileDirectoryName         : 1
# SizeInMBs                            : 30000
# VolumeType                           : VHDX
# IsDynamic                            : 1
# LockedRetryCount                     : 3
# LockedRetryInterval                  : 15
# PreventLoginWithFailure              : 1
# PreventLoginWithTempProfile          : 1
```

**If registry empty:**
- Policy not yet synced (wait or force sync)
- Session host not in device group (check dynamic group rules)
- Policy assignment error (check Intune portal for errors)

### Verify FSLogix Service Running

**PowerShell:**

```powershell
# Check FSLogix service status
Get-Service frxsvc | Format-List

# Expected output:
# Name        : frxsvc
# DisplayName : FSLogix Apps Services
# Status      : Running
# StartType   : Automatic
```

**If service not running:**

```powershell
# Start FSLogix service
Start-Service frxsvc

# Set to automatic startup
Set-Service frxsvc -StartupType Automatic
```

### Verify Storage Connectivity

**PowerShell (as administrator on session host):**

```powershell
# Test SMB connectivity to storage account
Test-NetConnection stavdprodeus01.file.core.windows.net -Port 445

# Expected output:
# TcpTestSucceeded : True

# Attempt to mount file share (uses Entra ID authentication)
net use Z: \\stavdprodeus01.file.core.windows.net\profiles-pooled

# Expected output:
# "The command completed successfully."
# No credential prompt (Entra ID auth automatic for domain-joined VMs)

# Verify write permissions
New-Item "Z:\test-permissions" -ItemType Directory

# Expected: Folder created (confirms RBAC and NTFS permissions correct)

# Clean up test
Remove-Item "Z:\test-permissions" -Force
net use Z: /delete
```

**If credential prompt appears:**
- Entra ID authentication not configured on storage account (check [[05-storage-fslogix#enable-entra-id-authentication]])
- Session host not joined to Entra ID (check `dsregcmd /status`)
- RBAC role missing for device group (check storage account IAM)

---

## Part 6: Test User Profile Creation

Verify FSLogix creates profile containers on first user logon.

### Prerequisites for Testing

- [ ] Test user account: `testuser@contoso.com` (member of AVD-Pooled-Users)
- [ ] Test user can access workspace and connect to pooled desktop (from [[08-app-groups-workspace]])
- [ ] Session hosts have FSLogix configured (registry verified above)
- [ ] Storage account accessible from session hosts (connectivity test passed)

### Test First Logon (Profile Creation)

1. **Connect as Test User:**
   - Open Remote Desktop client
   - Connect to **"Pooled Production Desktop"** (from workspace)
   - Authenticate as `testuser@contoso.com`

2. **Monitor Logon Process:**
   - First logon takes 30-60 seconds (profile VHD creation)
   - Windows desktop loads
   - User profile folder: `C:\Users\testuser`

3. **Verify Profile VHD Created:**

**From session host (PowerShell as admin):**

```powershell
# Check mounted volumes for FSLogix profiles
Get-Volume | Where-Object {$_.FileSystemLabel -like "*Profile*"}

# Expected output shows mounted VHD for logged-in user
```

**From storage account (Azure Portal):**

**Portal:** Azure Portal → Storage accounts → stavdprodeus01 → File shares → profiles-pooled → Browse

Expected folder structure:
```
profiles-pooled\
└── testuser_testuser@contoso.com\
    ├── Profile_testuser.vhdx           (VHD file, ~2GB initial size)
    ├── Profile_testuser.vhdx.lock      (lock file, contains session host name)
    └── Profile_testuser.vhdx.meta      (metadata JSON)
```

4. **Verify Profile Persistence:**
   - On session host: Create test file on desktop (`New-Item "$env:USERPROFILE\Desktop\test.txt"`)
   - Log off from session
   - Reconnect to pooled desktop (may connect to different session host)
   - Verify test file still on desktop (profile persisted)

**Expected Result:** Profile VHD created on first logon, mounts on subsequent logons, desktop files persist across sessions.

### Test Logon Performance

**PowerShell (on session host, as administrator):**

```powershell
# Monitor FSLogix logs during user logon
Get-WinEvent -LogName 'Microsoft-FSLogix-Apps/Operational' -MaxEvents 20 |
  Where-Object {$_.Id -eq 2} |
  Format-Table TimeCreated, Message -AutoSize

# Event ID 2: Profile attached successfully
# Look for "Profile attached in X seconds" message
```

**Target Performance:**
- First logon (new profile creation): 30-60 seconds
- Subsequent logons (existing profile mount): 5-15 seconds

**If logon exceeds 60 seconds:**
- Check storage IOPS (Premium tier provides <1ms latency)
- Verify private endpoint configured (reduces network latency)
- Check for profile size issues (run `Get-ChildItem \\stavdprodeus01\...` to check VHD size)

---

## Part 7: Registry-Based Configuration (Group Policy Alternative)

For hybrid-joined session hosts using Group Policy instead of Intune, configure FSLogix via registry or GPO.

> **Skip this section if using Intune (cloud-only Entra Join).** This is for reference only.

### GPO Configuration Path

**GPO Editor:** Computer Configuration → Administrative Templates → FSLogix → Profiles

**Settings (same as Intune):**

| GPO Setting | Value |
|-------------|-------|
| **Enabled** | Enabled |
| **VHD Locations** | `\\stavdprodeus01.file.core.windows.net\profiles-pooled` |
| **Size in MBs** | 30000 |
| **VHD Type** | VHDX |
| **Dynamic VHD** | Enabled |
| **Delete local profile when FSLogix Profile should apply** | Enabled |

**Link GPO to:** OU containing AVD session hosts

### Manual Registry Configuration (Testing Only)

**PowerShell (on session host, as administrator):**

```powershell
# Create FSLogix registry keys (for testing only, use Intune/GPO for production)
New-Item -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Force

# Enable FSLogix
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "Enabled" -Value 1 -Type DWord

# Set VHD location
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "VHDLocations" -Value "\\stavdprodeus01.file.core.windows.net\profiles-pooled" -Type MultiString

# Set profile size (30GB)
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "SizeInMBs" -Value 30000 -Type DWord

# Enable dynamic VHD
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "IsDynamic" -Value 1 -Type DWord

# Set VHD type (VHDX)
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "VolumeType" -Value "VHDX" -Type String

# Delete local profile when FSLogix applies
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "DeleteLocalProfileWhenVHDShouldApply" -Value 1 -Type DWord

# Enable FlipFlopProfileDirectoryName (username_upn format)
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "FlipFlopProfileDirectoryName" -Value 1 -Type DWord

# Restart FSLogix service
Restart-Service frxsvc
```

**See:** [[../Storage/fslogix-profile-containers#registry-settings-reference|FSLogix Registry Reference]] for complete registry key documentation.

---

## Part 8: Monitor Intune Policy Deployment

Verify FSLogix configuration profile deployed successfully to all session hosts.

### Check Profile Deployment Status

**Portal:** Intune Admin Center → Devices → Configuration profiles → AVD - FSLogix - Pooled Profile Containers → Monitor → Device status

**Expected Status:**

| Status | Count | Meaning |
|--------|-------|---------|
| **Succeeded** | 10 | All 10 pooled session hosts successfully applied profile |
| **Error** | 0 | No errors |
| **Conflict** | 0 | No conflicting profiles |
| **Not Applicable** | 0 | Profile applies to all assigned devices |
| **Pending** | 0 | All devices synced (if >0, devices haven't synced yet) |

**If errors present:**
- Click on "Error" count to see device list
- Click on device name to view error details
- Common errors:
  - **Setting not applicable:** OS version mismatch (FSLogix requires Windows 10/11)
  - **Remediation failed:** Registry permission issue (rare, reboot session host)
  - **Device not compliant:** Device not in assigned group (check dynamic group rules)

### Check Individual Device Configuration

**Portal:** Intune Admin Center → Devices → All devices → [Select Session Host] → Device configuration

**Expected:**
- [ ] `AVD - FSLogix - Pooled Profile Containers` shows **State: Succeeded**
- [ ] **Last check-in:** Within last 8 hours
- [ ] No conflicts with other profiles

**If state is "Pending":**
- Force sync (Device → Sync button)
- Wait 10 minutes, refresh page
- If still pending after 1 hour, check device can reach `*.manage.microsoft.com`

### Export Deployment Report

**Portal:** Intune Admin Center → Devices → Configuration profiles → AVD - FSLogix - Pooled Profile Containers → Monitor → Device status

1. Click **Export** (top toolbar)
2. Download CSV file
3. Review in Excel:
   - Filter for "Error" or "Conflict" status
   - Identify devices needing remediation

**Weekly Monitoring:** Schedule weekly review of FSLogix profile deployment status to catch configuration drift or new devices not receiving policy.

---

## Troubleshooting

### Issue: Profile not loading (new profile created each session)

**Symptom:** User sees fresh desktop every logon, settings reset, files missing

**Cause:**
- FSLogix registry not configured on session host
- VHDLocations path incorrect or unreachable
- Storage permissions incorrect (RBAC or NTFS)
- FSLogix service not running

**Fix:**

1. **Verify registry on session host:**
   ```powershell
   Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles"
   ```
   - If empty: Force Intune sync, verify device in correct group
   - If VHDLocations wrong: Correct Intune profile, force sync

2. **Verify storage connectivity:**
   ```powershell
   Test-NetConnection stavdprodeus01.file.core.windows.net -Port 445
   net use Z: \\stavdprodeus01.file.core.windows.net\profiles-pooled
   ```
   - If fails: Check NSG allows outbound SMB (445), verify private endpoint DNS resolution
   - If credential prompt: Entra ID auth not working (check [[05-storage-fslogix#enable-entra-id-authentication]])

3. **Verify FSLogix service:**
   ```powershell
   Get-Service frxsvc | Restart-Service
   ```

4. **Check FSLogix logs:**
   ```powershell
   Get-WinEvent -LogName 'Microsoft-FSLogix-Apps/Operational' -MaxEvents 50 |
     Where-Object {$_.LevelDisplayName -eq 'Error'} |
     Format-Table TimeCreated, Message -Wrap
   ```

**See:** [[../Storage/troubleshooting-fslogix-profiles|Troubleshooting FSLogix Profiles]] for detailed error resolution.

### Issue: User gets "Access Denied" during profile creation

**Symptom:** FSLogix logs show "Access Denied" when creating profile folder

**Cause:**
- User's group missing "Storage File Data SMB Share Contributor" RBAC role
- NTFS permissions on share root folder incorrect
- User not member of AVD-Pooled-Users group

**Fix:**

1. **Verify RBAC role assignment:**
   - **Portal:** Storage account → File share → Access Control (IAM) → Role assignments
   - Verify `AVD-Pooled-Users` has "Storage File Data SMB Share Contributor"

2. **Verify user group membership:**
   - **Portal:** Entra ID → Groups → AVD-Pooled-Users → Members
   - Confirm user appears in list

3. **Verify NTFS permissions:**
   ```powershell
   # Mount share as admin (must be in AVD-Admins group)
   net use Z: \\stavdprodeus01.file.core.windows.net\profiles-pooled

   # Check root folder ACL
   Get-Acl "Z:\" | Format-List

   # Expected: AVD-Pooled-Users has Modify on "This folder only"
   #           CREATOR OWNER has Full Control on "Subfolders and files only"
   ```

4. **Re-apply NTFS permissions if incorrect:**
   - See [[05-storage-fslogix#configure-ntfs-permissions|NTFS Permissions Configuration]]

### Issue: Intune profile shows "Error" on some devices

**Symptom:** Device status page shows errors for specific session hosts

**Cause:**
- OS version incompatible (FSLogix settings require Windows 10 1809+)
- Device not communicating with Intune
- Registry corruption on session host

**Fix:**

1. **Check OS version:**
   ```powershell
   # On session host
   [System.Environment]::OSVersion.Version
   # Minimum: 10.0.17763 (Windows 10 1809)
   ```

2. **Force Intune sync:**
   - **Portal:** Intune Admin Center → Devices → [Device] → Sync
   - Wait 10 minutes, check status again

3. **Check Intune connectivity:**
   ```powershell
   # On session host
   Test-NetConnection manage.microsoft.com -Port 443
   # Must succeed for Intune to work
   ```

4. **Reboot session host:**
   - If other fixes fail, reboot session host to clear registry issues
   - Force sync after reboot

### Issue: Profile VHD growing too large (approaching 30GB limit)

**Symptom:** User profile approaching SizeInMBs limit (30GB), logon performance degrading

**Cause:**
- User storing large files in profile (Desktop, Documents, Downloads)
- OneDrive local cache consuming space
- Application data bloat (browser caches, temp files)

**Fix:**

1. **Check VHD size:**
   ```powershell
   # On Azure Files share
   Get-ChildItem "\\stavdprodeus01.file.core.windows.net\profiles-pooled" -Recurse -Filter "*.vhdx" |
     Select-Object Name, @{N='SizeGB';E={[math]::Round($_.Length/1GB,2)}} |
     Sort-Object SizeGB -Descending
   ```

2. **Enable OneDrive Known Folder Move:**
   - Redirect Desktop, Documents, Pictures to OneDrive (keeps profile small)
   - See [[../Intune/device-configuration-profiles#onedrive-known-folder-move|OneDrive KFM Configuration]]

3. **Implement folder exclusions:**
   - Update FSLogix Redirections.xml to exclude large folders
   - Exclude: Browser caches, temp folders, OneDrive sync folders

4. **Increase SizeInMBs (if justified):**
   - Edit Intune profile: Change SizeInMBs from 30000 to 51200 (50GB)
   - Force sync to session hosts
   - Communicate with user about profile hygiene

**See:** [[../Storage/fslogix-profile-containers#container-sizing|FSLogix Container Sizing]] for size planning guidance.

---

## Verification Checklist

Confirm FSLogix configured correctly.

### Intune Profiles Created

**Portal:** Intune Admin Center → Devices → Configuration profiles

- [ ] `AVD - FSLogix - Pooled Profile Containers` exists (Settings Catalog)
- [ ] `AVD - FSLogix - Personal Profile Containers` exists (Settings Catalog)
- [ ] Both profiles assigned to correct device groups
- [ ] Deployment status: All devices "Succeeded"

### Session Host Configuration

**PowerShell (on session host, as admin):**

```powershell
# FSLogix registry configured
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles" | Select Enabled, VHDLocations, SizeInMBs

# FSLogix service running
Get-Service frxsvc | Select Status, StartType

# Storage accessible
Test-NetConnection stavdprodeus01.file.core.windows.net -Port 445
```

- [ ] Registry keys populated with correct values
- [ ] FSLogix service running (Status: Running, StartType: Automatic)
- [ ] Storage account reachable (TcpTestSucceeded: True)

### User Profile Testing

**Test with real user account:**

- [ ] User connects to pooled desktop via workspace
- [ ] First logon creates profile VHD on Azure Files (check storage account)
- [ ] Profile folder format: `username_upn` (e.g., `jdoe_jdoe@contoso.com`)
- [ ] VHD file exists: `Profile_username.vhdx`
- [ ] Lock file created during session: `Profile_username.vhdx.lock`
- [ ] User creates file on desktop, logs off, reconnects → file persists
- [ ] Logon time <15 seconds for existing profiles

### Storage Account Verification

**Portal:** Azure Portal → Storage accounts → stavdprodeus01 → File shares → profiles-pooled → Browse

- [ ] User profile folders created (one per user)
- [ ] VHD files present (*.vhdx)
- [ ] Lock files present during active sessions (*.vhdx.lock)
- [ ] VHD sizes reasonable (~2-5GB for new profiles)

---

## Next Steps

**FSLogix configured and tested.** User profiles now persist across pooled session hosts.

**Next:** [[10-monitoring-insights|Step 10: Azure Monitor Insights for AVD]] (coming soon)

In Step 10, you will:
- Configure Azure Monitor Insights for AVD
- Set up Log Analytics workspace
- Deploy diagnostics settings to host pools
- Create alerts for session host health and user connection issues

---

## Related Reference Pages

- [[../Storage/fslogix-profile-containers|FSLogix Profile Containers]] - Architecture, sizing, concurrent access, best practices
- [[../Storage/fslogix-cloud-cache|FSLogix Cloud Cache]] - High availability and multi-region scenarios
- [[../Storage/troubleshooting-fslogix-profiles|Troubleshooting FSLogix Profiles]] - Common issues, log analysis, error codes
- [[../Storage/storage-permissions-for-fslogix|Storage Permissions for FSLogix]] - RBAC roles, NTFS permissions, Entra ID authentication
- [[../Intune/device-configuration-profiles|Intune Device Configuration Profiles]] - Settings Catalog, profile types, assignment strategies
