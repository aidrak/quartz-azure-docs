---
title: Step 07 - FSLogix Configuration via Intune
description: Configure FSLogix profile containers on session hosts using Intune Settings Catalog
published: true
date: 2025-12-14T00:00:00.000Z
tags:
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 07: FSLogix Configuration via Intune

Configure FSLogix profile container policies in Intune **before deploying session hosts**. This ensures VMs receive FSLogix policies immediately upon enrollment, eliminating post-deployment configuration delays. For architecture and sizing details, see [[../Storage/fslogix-profile-containers|FSLogix Profile Containers]].

## Prerequisites

- [ ] Storage account with profiles-pooled file share created (from [[05-storage-fslogix]])
- [ ] Device group created: `avd-devices-pooled` (from [[02-identity-setup]])
- [ ] Intune Policy Administrator role

> **Note:** FSLogix agent pre-installed on Windows 11 multi-session/single-session images. No manual installation required. Policies deploy to enrolled devices automatically upon creation.

---

## Create FSLogix Configuration Profile

**Portal:** Intune Admin Center → Devices → Configuration profiles → + Create profile

1. **Name:** `AVD - FSLogix - Pooled Profile Containers`
2. **Platform:** Windows 10 and later
3. **Profile type:** Settings catalog
4. Click **Create**

### Configure Settings

5. **Configuration settings → + Add settings**
   - Search: `FSLogix`
   - Expand: **FSLogix → Profiles**
   - Add the following settings:

| Setting | Value |
|---------|-------|
| **Enabled** | Enabled |
| **VHDLocations** | `\\stavdprod01.file.core.windows.net\profiles-pooled` |
| **DeleteLocalProfileWhenVHDShouldApply** | Enabled |
| **FlipFlopProfileDirectoryName** | Enabled |
| **SizeInMBs** | 30000 |
| **VolumeType** | VHDX |
| **IsDynamic** | Enabled |
| **LockedRetryCount** | 3 |
| **LockedRetryInterval** | 15 |
| **ProfileType** | 0 |
| **PreventLoginWithFailure** | Enabled |
| **PreventLoginWithTempProfile** | Enabled |

6. Click **Next**

### Assign Profile

7. **Assignments:**
   - Select groups: `avd-devices-pooled`
   - Click **Next**

8. **Review + create:**
   - Verify VHDLocations: `\\stavdprod01.file.core.windows.net\profiles-pooled`
   - Click **Create**

---

## Force Policy Sync

**Portal:** Intune Admin Center → Devices → All devices

1. Search and select pooled/personal session hosts
2. Click **Sync** (top toolbar)
3. Wait 5-10 minutes for policy application

```powershell
# On the device itself
Restart-Service -Name "IntuneManagementExtension" -Force
Get-ScheduledTask | Where-Object { $_.TaskName -like "*pushlaunch*" } | Start-ScheduledTask
```

```powershell
# Query Intune App Logs
Get-Content "C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log" -Wait -Tail 50 | Select-String "Win32App"
```

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
   Test-NetConnection stavdprod01.file.core.windows.net -Port 445
   net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled
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
- User not member of avd-users-pooled group

**Fix:**

1. **Verify RBAC role assignment:**
   - **Portal:** Storage account → File share → Access Control (IAM) → Role assignments
   - Verify `avd-users-pooled` has "Storage File Data SMB Share Contributor"

2. **Verify user group membership:**
   - **Portal:** Entra ID → Groups → avd-users-pooled → Members
   - Confirm user appears in list

3. **Verify NTFS permissions:**
   ```powershell
   # Mount share as admin (must be in avd-users-admin group)
   net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled

   # Check root folder ACL
   Get-Acl "Z:\" | Format-List

   # Expected: avd-users-pooled has Modify on "This folder only"
   #           CREATOR OWNER has Full Control on "Subfolders and files only"
   ```

4. **Re-apply NTFS permissions if incorrect:**
   - See [[05-storage-fslogix#configure-ntfs-permissions|NTFS Permissions Configuration]]

---

## Next Steps

**FSLogix configured for pooled session hosts.** User profiles will persist across pooled hosts upon VM deployment.

**Next:** [[08-host-pool-creation|Step 08: Host Pool Creation]]

Personal session hosts do not require FSLogix configuration.

---

## Related Reference Pages

- [[../Storage/fslogix-profile-containers|FSLogix Profile Containers]] - Architecture, sizing, concurrent access, best practices
- [[../Storage/fslogix-cloud-cache|FSLogix Cloud Cache]] - High availability and multi-region scenarios
- [[../Storage/troubleshooting-fslogix-profiles|Troubleshooting FSLogix Profiles]] - Common issues, log analysis, error codes
- [[../Storage/storage-permissions-for-fslogix|Storage Permissions for FSLogix]] - RBAC roles, NTFS permissions, Entra ID authentication
- [[../Intune/device-configuration-profiles|Intune Device Configuration Profiles]] - Settings Catalog, profile types, assignment strategies
