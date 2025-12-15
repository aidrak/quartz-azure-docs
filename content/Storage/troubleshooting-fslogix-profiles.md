---
title: Troubleshooting FSLogix Profiles
description: 
published: true
date: 2025-12-14T04:53:51.937Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:35.907Z
---

# Troubleshooting FSLogix Profiles

FSLogix profile issues manifest as slow logons, missing user data, or complete logon failures. Effective troubleshooting requires understanding FSLogix's diagnostic tools, log formats, and common failure patterns. This guide provides systematic approaches to identify root causes and implement solutions quickly.

## FSLogix Logs Location

FSLogix maintains detailed logs of all operations. Knowing where to find logs and how to interpret them is the foundation of troubleshooting.

### Primary Log Directory

**Path:** `C:\ProgramData\FSLogix\Logs`

**Subdirectories:**

**Profile Logs (Most Important):**
```
C:\ProgramData\FSLogix\Logs\Profile\
├── Profile-<YYYYMMDD-HHMMSS>-<PID>.log     (Current session logs)
├── Profile-<YYYYMMDD-HHMMSS>-<PID>.log.1   (Previous session)
└── Profile-<YYYYMMDD-HHMMSS>-<PID>.log.2   (Older session)
```

**What's Logged:**
- User logon/logoff events
- VHD mount/dismount operations
- Registry redirection actions
- Error conditions (permissions, network, corruption)
- Performance timings (mount duration, replication lag)

**Example Log Entry:**
```
[2025-12-14 08:15:23.456] [INFO] User: CONTOSO\jdoe (S-1-5-21-xxx-1001)
[2025-12-14 08:15:23.567] [INFO] Searching for profile in: \\fslogix121025.file.core.windows.net\profiles
[2025-12-14 08:15:24.123] [INFO] Found existing profile: Profile_jdoe.vhdx (Size: 15.3 GB)
[2025-12-14 08:15:24.234] [INFO] Mounting VHD...
[2025-12-14 08:15:26.789] [INFO] VHD mounted successfully at C:\Users\jdoe (Duration: 2.5 seconds)
```

**Cloud Cache Logs:**
```
C:\ProgramData\FSLogix\Logs\CloudCache\
```

Contains Cloud Cache-specific events (replication status, provider health checks, failover events).

**ODFC Logs (Office Container):**
```
C:\ProgramData\FSLogix\Logs\ODFC\
```

Only present if Office Container enabled (rare in modern deployments).

### Log Retention

**Default Behavior:**
- FSLogix keeps last 10 log files per component
- Rotates logs daily or when file size exceeds 10MB
- Oldest logs automatically deleted

**Adjust Retention (Registry):**
```
HKLM\SOFTWARE\FSLogix\Logging
  LogFileKeepingPeriod: 30 (days)
  MaxLogFileSize: 20971520 (20MB in bytes)
```

**Long-Term Archival:**
Set up scheduled task to copy logs to centralized logging (Azure Monitor, Splunk, etc.) weekly for compliance and trend analysis.

### Reading Logs Effectively

**Identify User Session:**
Logs are per-process. Find relevant log file by timestamp or search for username:

```powershell
# Find logs for specific user
Get-ChildItem "C:\ProgramData\FSLogix\Logs\Profile" -Recurse |
  Select-String "CONTOSO\\jdoe" |
  Select-Object -ExpandProperty Path -Unique
```

**Common Patterns to Search:**

**ERROR entries (all failures):**
```powershell
Select-String -Path "C:\ProgramData\FSLogix\Logs\Profile\*.log" -Pattern "\[ERROR\]"
```

**Permission issues:**
```
[ERROR] Access denied to \\fslogix121025.file.core.windows.net\profiles
[ERROR] Cannot create directory: Permission denied
[ERROR] User lacks required RBAC role: Storage File Data SMB Share Contributor
```

**Network issues:**
```
[ERROR] Cannot reach provider: \\fslogix121025.file.core.windows.net\profiles
[ERROR] Network path not found
[ERROR] DNS resolution failed for fslogix121025.file.core.windows.net
```

**VHD corruption:**
```
[ERROR] VHD header invalid
[ERROR] Cannot mount VHD: Corrupted file system
[ERROR] VHDX read error at offset 0x12345678
```

**Timeout issues:**
```
[WARN] Mount operation exceeded timeout (30 seconds)
[ERROR] Logoff replication incomplete after 60 seconds
```

## Common Issues and Solutions

Systematic troubleshooting by symptom.

### Issue: Profile Not Loading

**Symptom:** User logs in but gets temporary profile. Desktop is empty, settings reset.

**Causes and Solutions:**

**Cause 1: FSLogix Agent Not Running**

**Diagnosis:**
```powershell
Get-Service | Where-Object {$_.Name -like "*FSLogix*"}
```

Expected output:
```
Status   Name               DisplayName
------   ----               -----------
Running  frxsvc             FSLogix Service
Running  frxdrv             FSLogix Driver
```

**Solution:**
```powershell
Start-Service frxsvc
Set-Service frxsvc -StartupType Automatic
```

**Cause 2: VHDLocations Registry Incorrect**

**Diagnosis:**
```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name VHDLocations
```

**Common Mistakes:**
- Missing double backslashes: `\fslogix121025...` (should be `\\fslogix121025...`)
- Typo in storage account name
- Path includes file name (should be folder only)

**Solution:**
```powershell
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "VHDLocations" -Value "\\fslogix121025.file.core.windows.net\profiles" -Type MultiString
```

**Cause 3: Permission Denied**

**Diagnosis:**
Check FSLogix log for:
```
[ERROR] Access denied to \\fslogix121025.file.core.windows.net\profiles
```

**Solution:**
- Verify user is member of avd-users-pooled or avd-users-personal group
- Verify group has RBAC role "Storage File Data SMB Share Contributor"
- Verify NTFS permissions on share root allow "Modify" for user group

```bash
# Check RBAC assignment
az role assignment list \
  --scope "/subscriptions/<sub-id>/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Storage/storageAccounts/fslogix121025/fileServices/default/fileshares/profiles" \
  --query "[?principalName=='avd-users-pooled']"
```

**Cause 4: Network Connectivity Issue**

**Diagnosis:**
Test from session host:
```powershell
Test-NetConnection fslogix121025.file.core.windows.net -Port 445

# Expected output:
# TcpTestSucceeded : True
```

If False:
- Private endpoint DNS not resolving correctly
- NSG blocking port 445
- Service endpoint not configured on session host subnet

**Solution:**
```powershell
# Verify DNS resolution
Resolve-DnsName fslogix121025.file.core.windows.net

# Should resolve to private IP (10.x.x.x) if using private endpoint
# Should resolve to public IP if using public endpoint (not recommended)

# Test SMB connectivity
net use Z: \\fslogix121025.file.core.windows.net\profiles
```

If DNS resolves but net use fails: Check NSG rules, verify private endpoint connection state.

### Issue: Slow Logons (>30 Seconds)

**Symptom:** User credentials accepted, but desktop takes 30+ seconds to load.

**Causes and Solutions:**

**Cause 1: Insufficient Storage IOPS**

**Diagnosis:**
Check Azure Monitor metrics for storage account:
- Navigate to Azure Portal → Storage Account fslogix121025 → Monitoring → Metrics
- Metric: "Transactions" filtered by "Response Type: Success" and "Response Type: ClientThrottlingError"
- If throttling errors present during logon times: Insufficient IOPS

**Solution:**
Increase provisioned IOPS on Premium v2 storage account:
```bash
az storage account update \
  --name fslogix121025 \
  --resource-group RG-Azure-VDI-01 \
  --sku PremiumV2_LRS \
  --enable-large-file-share
```

Or migrate to Premium v2 if still on Standard tier.

**Cause 2: High Network Latency**

**Diagnosis:**
Measure latency from session host:
```powershell
# Install PSPing from Sysinternals
psping \\fslogix121025.file.core.windows.net:445

# Expected: <5ms average
# Concerning: >10ms average
```

**Solution:**
- Verify private endpoint configured (reduces latency 30-50% vs public endpoint)
- Check session hosts and storage account in same region
- Enable SMB Multichannel:
```powershell
Set-SmbClientConfiguration -EnableMultiChannel $true -Force
```

**Cause 3: Large Profile Size**

**Diagnosis:**
Check profile VHD size:
```powershell
Get-ChildItem "\\fslogix121025.file.core.windows.net\profiles" -Recurse -Filter "*.vhdx" |
  Where-Object {$_.Name -like "*jdoe*"} |
  Select-Object Name, @{N='SizeGB';E={[math]::Round($_.Length/1GB,2)}}
```

If profile >40GB: Logons will be slow (especially first logon).

**Solution:**
- Implement folder redirection (move Documents, Desktop to OneDrive)
- Enable OneDrive Files On-Demand (reduce local cache)
- Clean up profile:
```powershell
# Mount VHD offline, delete temp files, browser cache
# Or use Disk Cleanup tool while user logged in
```

**Cause 4: Antivirus Scanning VHD**

**Diagnosis:**
Check antivirus real-time protection logs for:
- Scanning C:\ProgramData\FSLogix\
- Scanning \\fslogix121025.file.core.windows.net\profiles\

**Solution:**
Add exclusions to Windows Defender or third-party AV:

**Process exclusions:**
- `C:\Program Files\FSLogix\Apps\frxsvc.exe`
- `C:\Program Files\FSLogix\Apps\frxdrv.sys`

**Path exclusions:**
- `C:\ProgramData\FSLogix\`
- `\\fslogix121025.file.core.windows.net\profiles\`
- `C:\CCCache\` (if using Cloud Cache)

**File extension exclusions:**
- `*.vhd`
- `*.vhdx`
- `*.ccc`

```powershell
# Add Windows Defender exclusions
Add-MpPreference -ExclusionPath "C:\ProgramData\FSLogix"
Add-MpPreference -ExclusionPath "\\fslogix121025.file.core.windows.net\profiles"
Add-MpPreference -ExclusionExtension "vhdx"
```

### Issue: Profile Corruption

**Symptom:** User logs in, but applications crash, settings reset randomly, or "profile cannot be loaded" error.

**Causes and Solutions:**

**Cause 1: VHD File System Corruption**

**Diagnosis:**
```powershell
# Download profile VHD to local machine
Copy-Item "\\fslogix121025.file.core.windows.net\profiles\jdoe_jdoe@contoso.com\Profile_jdoe.vhdx" -Destination "C:\Temp\"

# Mount VHD
Mount-DiskImage -ImagePath "C:\Temp\Profile_jdoe.vhdx"

# Get drive letter
$DriveLetter = (Get-DiskImage -ImagePath "C:\Temp\Profile_jdoe.vhdx" | Get-Disk | Get-Partition | Get-Volume).DriveLetter

# Run CHKDSK
chkdsk "${DriveLetter}:" /F /R
```

**Solution:**
If CHKDSK finds errors:
- Repair automatically (CHKDSK /F /R)
- If unrepairable: Restore from backup or create new profile (user loses data)

**Cause 2: Abrupt Shutdown During Write**

**Diagnosis:**
Check Windows Event Log on session host:
```powershell
Get-WinEvent -LogName System |
  Where-Object {$_.Id -eq 41} |  # Kernel-Power event (unexpected shutdown)
  Select-Object TimeCreated, Message
```

If unexpected shutdowns during user sessions: Risk of profile corruption.

**Solution:**
- Enable FSLogix Cloud Cache (redundant copies, resilience to crashes)
- Increase logoff timeout:
```powershell
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "LogoffTimeout" -Value 60 -Type DWord
```
- Ensure users save work before logoff (don't force-close sessions)

**Cause 3: Concurrent Access Conflict**

**Diagnosis:**
Check for multiple .lock files or lock file timestamp from past session:
```powershell
Get-ChildItem "\\fslogix121025.file.core.windows.net\profiles" -Recurse -Filter "*.lock" |
  Select-Object Name, LastWriteTime
```

If .lock file from hours/days ago: Session host crashed without releasing lock, potential concurrent access corruption.

**Solution:**
- Delete orphaned .lock files:
```powershell
Remove-Item "\\fslogix121025.file.core.windows.net\profiles\jdoe_jdoe@contoso.com\Profile_jdoe.vhdx.lock" -Force
```
- Enable PreventLoginWithFailure to block concurrent sessions:
```powershell
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "PreventLoginWithFailure" -Value 1 -Type DWord
```

### Issue: Duplicate Profiles

**Symptom:** User has two profile folders, FSLogix loads wrong one (missing recent data).

**Causes and Solutions:**

**Cause: Username vs UPN Folder Naming**

**Example:**
- Profile 1: `jdoe_jdoe@contoso.com` (UPN-based)
- Profile 2: `jdoe` (username-only)

**Diagnosis:**
```powershell
Get-ChildItem "\\fslogix121025.file.core.windows.net\profiles" |
  Where-Object {$_.Name -like "*jdoe*"}
```

**Solution:**
1. Identify active profile (most recent LastWriteTime)
2. Delete duplicate profile (or rename to .old for safety)
3. Standardize folder naming via registry:

```powershell
# Use UPN-based naming (recommended)
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "FlipFlopProfileDirectoryName" -Value 1 -Type DWord

# Use SID-based naming (most unique)
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "SIDDirNamePattern" -Value "%sid%" -Type String
```

**Best Practice:** Use `FlipFlopProfileDirectoryName: 1` (creates `username_upn` folders). Prevents collisions across domains.

## Status Codes and Meanings

FSLogix logs include numeric status codes. Understanding them accelerates troubleshooting.

### Common Status Codes

**0x00000000 - Success**
- Operation completed successfully
- No action required

**0x80070005 - Access Denied (E_ACCESSDENIED)**
- User lacks permissions to share or profile folder
- Check RBAC role assignment and NTFS permissions

**0x80070035 - Network Path Not Found**
- Cannot reach storage account
- Check DNS resolution, NSG rules, private endpoint connectivity

**0x8007003B - Network Path Invalid**
- Incorrect VHDLocations registry path
- Verify UNC path format: `\\server\share` (not `\\server\share\folder\file.vhdx`)

**0x80070091 - Directory Not Empty**
- FSLogix cannot delete or rename folder (files in use)
- Check for locked files, antivirus scanning

**0x800703EE - Volume Not Mounted**
- VHD mount failed (corruption, insufficient disk space)
- Run CHKDSK on VHD, verify session host has disk space

**0x80070070 - Disk Full**
- Profile VHD reached SizeInMBs limit
- Increase SizeInMBs registry value, or clean up profile

**0xC03A0014 - VHD_E_INVALID_STATE**
- VHD file corruption
- Restore from backup or recreate profile

**0xC03A001A - VHD_E_FILE_SYSTEM_CORRUPT**
- VHD file system requires repair
- Run CHKDSK /F /R on mounted VHD

**Example Log with Status Code:**
```
[2025-12-14 08:15:24.567] [ERROR] Cannot mount VHD: Status 0x80070005
[2025-12-14 08:15:24.578] [ERROR] Access denied to \\fslogix121025.file.core.windows.net\profiles\jdoe_jdoe@contoso.com\Profile_jdoe.vhdx
```

**Lookup Status Code:**
```powershell
# Convert hex to decimal, search Microsoft docs
[System.Convert]::ToInt32("80070005",16)  # Returns 2147942405

# Or use built-in error lookup
net helpmsg 5  # Returns "Access is denied."
```

## Event Log Locations

FSLogix also logs to Windows Event Viewer for integration with centralized monitoring.

### Event Viewer Logs

**Path:** Event Viewer → Applications and Services Logs → FSLogix

**Log Categories:**

**FSLogix-Apps/Operational**
- All FSLogix component events
- Info, Warning, Error severities
- User logon/logoff, mount/dismount, errors

**FSLogix-Apps/Admin**
- Administrative events (service start/stop, configuration changes)

**Key Event IDs:**

| Event ID | Severity | Description |
|----------|----------|-------------|
| 1 | Info | User logon initiated |
| 2 | Info | Profile loaded successfully |
| 3 | Info | User logoff initiated |
| 4 | Info | Profile unloaded successfully |
| 50 | Warning | Mount operation took longer than expected |
| 51 | Warning | Provider unreachable, using fallback |
| 100 | Error | Cannot load profile (general failure) |
| 101 | Error | Access denied to profile location |
| 102 | Error | VHD corruption detected |
| 103 | Error | Network connectivity failure |

**Query Events (PowerShell):**

```powershell
# Get all FSLogix errors from last 24 hours
Get-WinEvent -LogName "FSLogix-Apps/Operational" -MaxEvents 100 |
  Where-Object {$_.LevelDisplayName -eq "Error" -and $_.TimeCreated -gt (Get-Date).AddDays(-1)} |
  Select-Object TimeCreated, Id, Message
```

**Filter for Specific User:**
```powershell
Get-WinEvent -LogName "FSLogix-Apps/Operational" -MaxEvents 500 |
  Where-Object {$_.Message -like "*jdoe*"}
```

## frx Command-Line Tool

FSLogix includes a command-line utility for diagnostics and management.

### Location

```
C:\Program Files\FSLogix\Apps\frx.exe
```

### Common Commands

**List Active Sessions:**
```cmd
frx list
```

Output:
```
User: CONTOSO\jdoe
  Profile VHD: \\fslogix121025.file.core.windows.net\profiles\jdoe_jdoe@contoso.com\Profile_jdoe.vhdx
  Mount Point: C:\Users\jdoe
  Status: Mounted
  Session ID: 2
```

**Check Profile Status:**
```cmd
frx status /username:jdoe
```

**Force Unmount Profile (Caution):**
```cmd
frx unmount /username:jdoe /force
```

**Warning:** Forced unmount can cause data loss (unsaved changes). Only use when user logged off but profile stuck mounted.

**Verify Configuration:**
```cmd
frx config
```

Output:
```
FSLogix Version: 2.9.8884.27471
Enabled: 1
VHDLocations: \\fslogix121025.file.core.windows.net\profiles
SizeInMBs: 30000
IsDynamic: 1
VolumeType: VHDX
```

**Rebuild Profile (Nuclear Option):**
```cmd
# Delete user's profile VHD (user gets fresh profile next logon)
# Ensure user logged off first!
Remove-Item "\\fslogix121025.file.core.windows.net\profiles\jdoe_jdoe@contoso.com\Profile_jdoe.vhdx" -Force
```

## Testing Access: net use to Share

Basic connectivity testing before FSLogix troubleshooting.

### Test from Session Host

**As User:**
```powershell
# Authenticate as user (not admin)
runas /user:CONTOSO\jdoe "powershell.exe"

# Attempt to mount share
net use Z: \\fslogix121025.file.core.windows.net\profiles
```

**Expected Output (Success):**
```
The command completed successfully.
```

**Failure Examples:**

**Access Denied:**
```
System error 5 has occurred.
Access is denied.
```

**Cause:** User lacks RBAC role or NTFS permissions.

**Network Path Not Found:**
```
System error 53 has occurred.
The network path was not found.
```

**Cause:** DNS resolution failure, NSG blocking port 445, private endpoint misconfigured.

**Logon Failure:**
```
System error 1326 has occurred.
The user name or password is incorrect.
```

**Cause:** Entra ID authentication not enabled on storage account (falling back to key auth, which user doesn't have).

### Verify Permissions After Mount

```powershell
# Create test folder
New-Item "Z:\testuser_testuser@contoso.com" -ItemType Directory

# Expected: Success (user can create folder via NTFS permissions)

# Try to create another user's folder
New-Item "Z:\otheruser_otheruser@contoso.com" -ItemType Directory

# Expected: Access Denied (user cannot create folders for other users)
```

## Profile Health Monitoring

Proactive monitoring prevents user-impacting issues.

### Key Metrics to Monitor

**Logon Duration:**
- Target: <15 seconds from credential entry to desktop
- Alert if >30 seconds (indicates storage performance or network issues)

**Profile VHD Size:**
- Target: <80% of SizeInMBs limit
- Alert if any user exceeds 25GB (default 30GB limit)

**Storage Account Health:**
- Availability: >99.9%
- Average latency: <5ms
- Throttling errors: 0

**FSLogix Service Status:**
- frxsvc service: Running on all session hosts
- frxdrv driver: Running on all session hosts
- Alert if service stops on any host

### Monitoring Solutions

**Azure Monitor Workbook:**

Create custom workbook with:
- Storage account IOPS utilization
- Average file operation latency
- Transaction success rate
- Throttling event count

**Log Analytics Query (FSLogix Errors):**
```kusto
Event
| where Source == "FSLogix-Apps"
| where EventLevelName == "Error"
| summarize Count=count() by Computer, EventID, RenderedDescription
| order by Count desc
```

**PowerShell Health Check Script:**
```powershell
# Run on each session host via scheduled task

# Check service status
$services = @("frxsvc", "frxdrv")
foreach ($svc in $services) {
  $status = (Get-Service $svc).Status
  if ($status -ne "Running") {
    Write-EventLog -LogName Application -Source "FSLogix-Monitor" -EntryType Error -EventId 1001 -Message "Service $svc is $status"
  }
}

# Check registry configuration
$vhdloc = (Get-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name VHDLocations).VHDLocations
if (-not (Test-Path $vhdloc.Replace("\\","\\"))) {
  Write-EventLog -LogName Application -Source "FSLogix-Monitor" -EntryType Error -EventId 1002 -Message "Cannot reach VHDLocations: $vhdloc"
}

# Check recent errors
$errors = Get-WinEvent -LogName "FSLogix-Apps/Operational" -MaxEvents 50 |
  Where-Object {$_.LevelDisplayName -eq "Error" -and $_.TimeCreated -gt (Get-Date).AddHours(-1)}
if ($errors.Count -gt 5) {
  Write-EventLog -LogName Application -Source "FSLogix-Monitor" -EntryType Warning -EventId 1003 -Message "High error rate: $($errors.Count) errors in last hour"
}
```

Schedule via Task Scheduler to run every 15 minutes, send alerts to monitoring system.

## Best Practices

**Enable Verbose Logging During Troubleshooting** - Increase FSLogix log verbosity temporarily to capture detailed operation timings and intermediate steps. Revert to default after issue resolved (verbose logging consumes disk space).

```powershell
# Enable verbose logging
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "LogLevel" -Value 3 -Type DWord  # 0=Disabled, 1=Errors, 2=Warnings, 3=Info (default), 4=Debug

# After troubleshooting, revert
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "LogLevel" -Value 3 -Type DWord
```

**Test with Known-Good User Account** - When troubleshooting permissions, create a test user account, assign to avd-users group, attempt logon. Eliminates user-specific Entra ID issues (stale cached credentials, group membership propagation delays).

**Always Check Both RBAC and NTFS** - Profile access requires RBAC role (share-level) AND NTFS permissions (file-level). Common mistake: Fixing one but not the other. Verify both layers when troubleshooting access denied errors.

**Monitor Orphaned Lock Files** - Set up weekly scheduled task to identify .lock files older than 24 hours (indicates crashed session hosts). Delete orphaned locks automatically or alert administrators for manual review.

**Implement Profile Backups** - FSLogix profiles are critical user data. Configure Azure Backup or file share snapshots daily. Test restore procedures quarterly to ensure RTO <4 hours for profile recovery.

**Document Troubleshooting Steps** - Maintain runbook documenting common issues (access denied, slow logons, corruption) with step-by-step resolution procedures. Include screenshots, PowerShell commands, escalation paths.

**Centralize Logs** - Forward FSLogix logs from all session hosts to Azure Monitor Logs or SIEM. Enables cross-host correlation (identify widespread issues vs single-host problems) and long-term trend analysis.

## Common Issues Summary Table

| Symptom | First Check | Most Likely Cause | Quick Fix |
|---------|-------------|-------------------|-----------|
| Temporary profile loaded | FSLogix service status | Service not running | Start frxsvc service |
| Access denied error | RBAC role assignment | User lacks Storage File Data SMB Share Contributor | Assign RBAC role to user's group |
| Network path not found | DNS resolution | Private endpoint DNS not configured | Verify privatelink.file.core.windows.net zone |
| Logon >30 seconds | Storage account metrics | Insufficient IOPS | Increase provisioned IOPS (Premium v2) |
| Profile corruption | CHKDSK on VHD | File system corruption | Run CHKDSK /F /R, restore from backup |
| Duplicate profiles | Folder naming pattern | FlipFlopProfileDirectoryName mismatch | Standardize folder naming, merge profiles |
| Slow application performance | Antivirus exclusions | AV scanning VHD files | Add FSLogix paths/extensions to exclusions |
| Cannot create profile folder | NTFS permissions on root | Missing Modify permission | Grant Modify to avd-users on share root |

> **Warning:** Never delete a user's profile VHD without confirming backup exists or user approval. Profile contains all user data (Desktop files, Documents, browser favorites, application settings). Deletion is irreversible without backup.

## Next Steps

- Review **Page 1: Azure Files Overview** to understand storage performance characteristics affecting FSLogix
- Review **Page 3: Storage Permissions** to verify RBAC and NTFS configuration
- If using Cloud Cache, review **Page 4** for Cloud Cache-specific troubleshooting (replication lag, split-brain scenarios)
- Implement proactive monitoring using Azure Monitor workbooks and Log Analytics queries
- Document recurring issues in team knowledge base with resolution procedures