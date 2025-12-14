---
title: FSLogix Profile Containers
description: 
published: true
date: 2025-12-14T04:53:48.460Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:32.433Z
---

# FSLogix Profile Containers

FSLogix Profile Container is a profile management solution that addresses the limitations of traditional Windows roaming profiles in virtual desktop environments. By mounting user profiles as virtual hard disks (VHD/VHDX) stored on network shares, FSLogix delivers native Windows profile performance while enabling profile roaming across non-persistent session hosts.

## What is FSLogix

FSLogix is a profile management technology acquired by Microsoft in 2018 and now included free with Windows 10/11 Enterprise multi-session licenses and Microsoft 365 subscriptions. It solves the critical challenge of providing consistent user experiences in Azure Virtual Desktop environments where users may connect to different session hosts on each login.

**Core Problem FSLogix Solves:**

Traditional Windows roaming profiles copy the entire profile to/from the session host at logon/logoff. This process:
- Takes 5-20 minutes for profiles with large data
- Causes "last write wins" conflicts across simultaneous sessions
- Fails with AppData files locked by applications
- Consumes local disk space on session hosts

**FSLogix Solution:**

Instead of copying, FSLogix mounts the user's profile as a virtual disk over the network:
- Profile stays on the network share (Azure Files in our case)
- Session host reads/writes directly to VHD/VHDX
- Logon completes in seconds, regardless of profile size
- No local disk consumption on session hosts
- Supports concurrent sessions (with limitations)

## How FSLogix Works

FSLogix uses a filter driver installed on the session host to redirect profile locations to a virtual hard disk mounted at user logon.

### Logon Process

1. **User authenticates** to Azure Virtual Desktop host pool
2. **Session broker assigns** user to available session host
3. **FSLogix agent detects** user logon event
4. **Agent queries** configured VHDLocations registry setting for share path
5. **Agent searches** for existing profile container: `\\fslogix121025.file.core.windows.net\profiles\jdoe_jdoe@contoso.com`
6. **If found:** Mounts existing VHDX at C:\Users\jdoe (typically in 2-5 seconds)
7. **If not found:** Creates new 30GB dynamically expanding VHDX and initializes profile
8. **Windows logon continues** with profile now available at C:\Users\jdoe
9. **User desktop loads** with all settings, documents, and AppData intact

### Active Session

While the user is logged on:
- Profile VHD remains mounted and locked to the session host
- All file operations (AppData, Desktop, Documents) write directly to VHD
- No file copying between session host and share
- VHD grows dynamically as user saves data (up to maximum size)
- Other session hosts cannot mount the VHD (exclusive lock)

### Logoff Process

1. **User initiates logoff** or session times out
2. **Windows closes** all user processes
3. **FSLogix agent waits** for file handles to release (configurable timeout)
4. **Agent flushes** any cached writes to VHD
5. **VHD is dismounted** and file lock released
6. **Profile container returns** to available state
7. **Total logoff time:** 5-15 seconds typically

## Profile Container vs Office Container

FSLogix offers two container types designed for different data scopes. Understanding the distinction is critical for proper deployment.

### Profile Container (Recommended)

**What it Contains:**
- Entire user profile: C:\Users\%USERNAME%
- All AppData (Local, LocalLow, Roaming)
- Desktop, Documents, Downloads, Pictures, etc.
- Registry hive: HKEY_CURRENT_USER
- All application settings and cached data

**File Naming Convention:**
```
Profile_jdoe.vhdx              (single session, most common)
Profile_jdoe_S-1-5-21-xxx.vhdx (with SID for disambiguation)
```

**When to Use:**
- All AVD deployments (default choice)
- Non-persistent session hosts
- Pooled host pools where users get different VMs
- Personal host pools where you want profile portability

**Configuration:**
```
Registry: HKLM\SOFTWARE\FSLogix\Profiles
  - Enabled: 1
  - VHDLocations: \\fslogix121025.file.core.windows.net\profiles
  - SizeInMBs: 30000 (30GB default)
  - IsDynamic: 1 (dynamic expansion)
  - VolumeType: VHDX (modern format)
```

### Office Container (ODFC)

**What it Contains:**
- OneDrive cache: %localappdata%\Microsoft\OneDrive
- Outlook cache (OST): %localappdata%\Microsoft\Outlook
- Teams cache: %appdata%\Microsoft\Teams
- OneNote cache
- Outlook search index

**File Naming Convention:**
```
ODFC_jdoe.vhdx
ODFC_jdoe_S-1-5-21-xxx.vhdx
```

**When to Use (Rare):**
- You need to separate Office data from profile (legacy requirement)
- Using Citrix Profile Management for base profile but want FSLogix for Office
- Compliance requires Office data on separate storage tier

**Why We Don't Use It:**
- Profile Container already includes all Office data
- Managing two containers adds complexity
- No performance benefit in AVD scenarios
- Microsoft recommends Profile Container only for modern deployments

> **Best Practice:** Use Profile Container only. Office Container is a legacy option from pre-Microsoft FSLogix days when third-party profile solutions couldn't handle Office 365 data effectively.

## Folder Structure on Azure Files Share

On the Azure Files share (fslogix121025 in our example), FSLogix creates a specific folder hierarchy:

```
\\fslogix121025.file.core.windows.net\profiles\
├── jdoe_jdoe@contoso.com\
│   ├── Profile_jdoe.vhdx              (user's profile VHD)
│   ├── Profile_jdoe.vhdx.lock         (lock file, present when mounted)
│   └── Profile_jdoe.vhdx.meta         (metadata, last write time, etc.)
├── asmith_asmith@contoso.com\
│   ├── Profile_asmith.vhdx
│   ├── Profile_asmith.vhdx.lock
│   └── Profile_asmith.vhdx.meta
└── bwilliams_bwilliams@contoso.com\
    ├── Profile_bwilliams.vhdx
    ├── Profile_bwilliams.vhdx.lock
    └── Profile_bwilliams.vhdx.meta
```

**Folder Naming:**
- Default format: `username_upn` (e.g., jdoe_jdoe@contoso.com)
- Controlled by: `FlipFlopProfileDirectoryName` registry setting
- Alternative: `%username%` only (simpler, but risks collisions with same username in different domains)

**File Types:**

**.vhdx (Profile Container)**
- Virtual hard disk containing entire user profile
- Dynamically expanding (only consumes space for actual data)
- Maximum size set by SizeInMBs registry value (30GB default)
- Format: VHDX (newer, supports >2TB, resilient to power loss)
- Alternative: VHD (legacy, 2TB limit, use only for compatibility)

**.vhdx.lock (Lock File)**
- Created when VHD is mounted on session host
- Contains hostname of session host with active mount
- Prevents concurrent mounts by other session hosts
- Deleted automatically at logoff
- Orphaned locks (session host crash): Auto-release after timeout period

**.vhdx.meta (Metadata File)**
- JSON format containing VHD metadata
- Last access time, last write time
- VHD version information
- Not critical for operation (safe to delete if corrupted)

## Container Sizing

Proper container sizing balances user needs against storage costs and performance.

### Default Size: 30GB

Microsoft's recommended default for general-purpose users:

```
Registry: HKLM\SOFTWARE\FSLogix\Profiles\SizeInMBs
Value: 30000 (30GB)
```

**What Fits in 30GB:**
- Windows user profile structure: 2GB
- AppData for Office 365, Teams, Edge: 5-10GB
- Desktop files, Documents: 10GB
- OneDrive local cache: 5GB (if not using Files On-Demand)
- Outlook OST cache: 5GB
- Remaining space: User data, downloads, etc.

### Power User Size: 50GB

For CAD, development, or data analysis users:

```
SizeInMBs: 51200 (50GB)
```

**Additional Space Used:**
- Visual Studio cache: 10GB
- AutoCAD support files: 5GB
- Local datasets: 15GB
- Development tools (Node modules, Docker images): 10GB

### Task Worker Size: 20GB

For task workers (call center, data entry) with minimal local data:

```
SizeInMBs: 20480 (20GB)
```

**Why Smaller Works:**
- No local file storage (documents in SharePoint)
- OneDrive Files On-Demand (no local cache)
- Web-based applications (no AppData footprint)
- Thin profile reduces backup and sync overhead

### Dynamic Expansion Explained

**IsDynamic: 1 (Recommended)**

The VHDX file grows as users add data:
- Newly created 30GB VHDX uses only 2GB initially (profile structure only)
- User saves 5GB of documents → VHDX grows to 7GB on-disk
- User installs application → VHDX grows as AppData increases
- VHDX never shrinks automatically (even if user deletes files inside)

**Advantages:**
- Storage efficient: Pay only for used space on Azure Files
- No pre-allocation delay during profile creation
- Easy to monitor actual consumption vs limit

**Disadvantages:**
- VHD fragmentation over time (minimal impact on SSDs)
- User can fill VHD and hit size limit unexpectedly

**IsDynamic: 0 (Fixed)**

The VHDX file allocates full size immediately:
- Newly created 30GB VHDX uses 30GB on-disk (even if empty)
- Faster write performance (no expansion overhead)
- No fragmentation issues

**When to Use Fixed:**
- Predictable storage billing requirements
- Maximum write performance needed
- Storage space not a concern

> **Recommendation:** Use dynamic expansion (IsDynamic: 1) unless you have specific reasons for fixed. The performance difference is negligible on Premium v2 storage, and the storage savings are substantial.

### Monitoring and Alerts

**Track Container Sizes:**

```powershell
# Get all profile containers and their sizes
Get-ChildItem "\\fslogix121025.file.core.windows.net\profiles" -Recurse -Filter "*.vhdx" |
  Select-Object Name, @{N='SizeGB';E={[math]::Round($_.Length/1GB,2)}} |
  Sort-Object SizeGB -Descending
```

**Set Up Alerts:**
- Alert when any user exceeds 80% of SizeInMBs limit
- Weekly report of top 10 largest profiles
- Trend analysis: profiles growing >5GB per month

**User Communication:**
When user approaches limit:
1. Notify user via email/Teams
2. Suggest moving large files to OneDrive/SharePoint
3. Offer to increase SizeInMBs if justified
4. Provide guidance on Disk Cleanup and profile hygiene

## Concurrent Access Considerations

FSLogix supports multiple simultaneous sessions for the same user, but with important limitations.

### Single Session (Default)

**Behavior:**
- User logs into first session host → Profile VHD mounts with exclusive lock
- User attempts second session → FSLogix detects existing mount, denies second session
- User sees error: "The profile is already in use"

**Registry Setting:**
```
PreventLoginWithFailure: 1 (default)
```

**When This Occurs:**
- User force-closes RDP session without logging off properly
- Session host crashes while user logged in (orphaned lock)
- User attempts simultaneous logon to two different host pools

**Resolution:**
Wait for lock timeout (default 30 minutes) or manually delete .lock file after verifying session is truly orphaned.

### Concurrent Sessions (Advanced)

**Enabling Concurrent Access:**

```
Registry: HKLM\SOFTWARE\FSLogix\Profiles
  - ConcurrentUserSessions: 1 (enable multi-session per user)
  - PreventLoginWithTempProfile: 0 (allow temp profiles as fallback)
```

**How It Works:**
- First session: Mounts Profile_user.vhdx with read/write access
- Second session: Mounts same VHD in read-write mode (SMB 3.x oplocks coordinate access)
- File-level locking prevents corruption
- Both sessions see profile changes in near-real-time

**Limitations:**
- Registry changes may not sync between sessions (last write wins)
- Application state conflicts (Outlook, Teams) if both sessions active
- Higher IOPS demand (two session hosts accessing same VHD)
- Not officially supported by Microsoft (use at own risk)

**When to Use:**
- Users frequently need simultaneous desktop and published app sessions
- Testing/troubleshooting scenarios
- Temporary workaround for orphaned lock issues

> **Warning:** Concurrent sessions can cause profile corruption if both sessions modify the same registry keys or application data simultaneously. Recommended for testing only. For production, use separate profiles or configure applications to handle multi-session gracefully.

### Orphaned Lock Handling

**Scenario:** Session host crashes while user logged in, leaving .lock file on share.

**FSLogix Behavior:**
1. User attempts new logon
2. FSLogix detects existing .lock file
3. FSLogix reads hostname from .lock file
4. FSLogix attempts to ping hostname
5. If hostname unreachable for 30 minutes (default): Auto-delete .lock and mount VHD
6. If hostname reachable: Deny logon (assume session still active)

**Manual Resolution:**

```powershell
# Verify session truly orphaned (check session host)
Get-RdsUserSession -TenantName contoso -HostPoolName hp-pooled |
  Where-Object {$_.UserPrincipalName -eq "jdoe@contoso.com"}

# If no active session, delete lock file
Remove-Item "\\fslogix121025.file.core.windows.net\profiles\jdoe_jdoe@contoso.com\Profile_jdoe.vhdx.lock" -Force
```

**Prevent Orphaned Locks:**
- Enable session host health monitoring
- Configure automatic session host restarts during maintenance windows
- Use Azure Virtual Desktop drain mode before reboots
- Set reasonable session timeout policies (disconnect after 30 min idle)

## Best Practices

**Use Profile Container Only** - Skip Office Container unless you have a legacy dependency on third-party profile solutions. Profile Container handles all data types effectively in modern AVD deployments.

**Set Appropriate Size Limits** - Start with 30GB for general users. Monitor actual consumption monthly and adjust per-user or per-group as needed. Avoid setting limits too high (wastes storage) or too low (user frustration).

**Enable Dynamic Expansion** - Set IsDynamic: 1 to save storage costs. Premium v2 Azure Files handles dynamic expansion without performance impact. Fixed VHDs waste storage on day-one profiles that consume only 2GB.

**Use VHDX Format** - Always use VolumeType: VHDX (not VHD). VHDX supports larger sizes, better resilience to power loss, and improved performance on modern storage. VHD format is legacy only.

**Implement Folder Redirection** - Redirect Desktop, Documents, Pictures to OneDrive to keep profile containers small. Users get cloud backup and cross-device access while reducing FSLogix storage load.

**Monitor Container Growth** - Set up weekly reports showing profiles over 80% capacity. Proactively contact users to clean up or request size increases. Avoid surprise failures when users hit limits.

**Test Logon Performance** - Measure time from credential entry to desktop ready state. Target <15 seconds for cached profiles, <30 seconds for first logon. Investigate if times exceed thresholds.

**Plan for Failures** - Implement Cloud Cache or regular backups of the profiles share. Profile corruption or share outages can render entire host pools unusable. Have recovery procedures documented and tested.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| New profile every logon | FSLogix not finding existing VHD | Check VHDLocations path, verify network connectivity to share |
| Logon takes >2 minutes | Insufficient IOPS or high latency | Upgrade Azure Files to Premium v2, verify private endpoint configured |
| Profile not available | Concurrent session lock conflict | Check for orphaned .lock files, verify ConcurrentUserSessions setting |
| User hits size limit | Container too small for user needs | Increase SizeInMBs, implement folder redirection to reduce profile size |
| VHD corruption | Abrupt session host shutdown during writes | Enable Cloud Cache, ensure graceful shutdowns, use VHDX format |
| Lost profile after migration | Folder naming mismatch (username vs UPN) | Verify FlipFlopProfileDirectoryName matches old and new environments |

> **Warning:** Never manually edit files inside a mounted profile VHD from another machine. Always let FSLogix handle VHD mounting and dismounting. Manual edits can corrupt the VHD and require profile rebuild.

## Next Steps

- **Page 3:** Configure Entra ID authentication and set RBAC permissions for user access to the profiles share
- **Page 4:** Implement FSLogix Cloud Cache for high availability and multi-region scenarios
- **Page 5:** Learn troubleshooting techniques for profile issues using logs and diagnostic tools