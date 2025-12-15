---
title: FSLogix Cloud Cache
description: 
published: true
date: 2025-12-14T04:53:46.762Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:30.658Z
---

# FSLogix Cloud Cache

FSLogix Cloud Cache is an advanced feature that provides high availability and disaster recovery for user profiles by maintaining local caches on session hosts and replicating to multiple remote storage providers simultaneously. Cloud Cache solves critical challenges in multi-region deployments and environments requiring zero-downtime profile access during storage outages.

## What is FSLogix Cloud Cache

Cloud Cache fundamentally changes how FSLogix interacts with storage. Instead of directly mounting a VHD from a network share, Cloud Cache maintains a local copy on the session host's disk and asynchronously replicates changes to one or more remote providers (Azure Files, SMB shares, etc.).

### Traditional FSLogix (Without Cloud Cache)

**Architecture:**
- User logs in → FSLogix mounts VHD directly from \\fslogix121025.file.core.windows.net\profiles
- All reads/writes traverse network to Azure Files
- Profile data exists in one location only
- Storage outage = users cannot log in

**Performance Characteristics:**
- Read latency: 1-5ms (network + storage latency)
- Write latency: 1-5ms (synchronous write to network share)
- Bandwidth: Limited by network connection to Azure Files
- Resilience: Single point of failure (storage account down = no access)

### FSLogix with Cloud Cache

**Architecture:**
- User logs in → FSLogix creates/opens local cache file (C:\CCCache\jdoe.vhdx)
- Local cache is primary read/write target (local disk speed)
- Background thread asynchronously replicates to Provider 1 (\\fslogix121025.file.core.windows.net\profiles)
- Second background thread asynchronously replicates to Provider 2 (\\fslogix-dr.file.core.windows.net\profiles)
- Storage outage = users continue working from local cache

**Performance Characteristics:**
- Read latency: 0.1-0.5ms (local NVMe/SSD)
- Write latency: 0.1-0.5ms to local cache (async replication to network)
- Bandwidth: Local disk speed (1-3 GB/s typical)
- Resilience: Continues operation if 1 or both providers unavailable

**Cache File Structure:**
```
C:\CCCache\
├── jdoe.vhdx                        (local cache VHD for user jdoe)
├── jdoe.vhdx.lock                   (lock file)
├── jdoe.vhdx.meta                   (metadata)
├── asmith.vhdx                      (local cache for user asmith)
└── ...
```

**Remote Provider Structure:**
```
\\fslogix121025.file.core.windows.net\profiles\
├── jdoe_jdoe@contoso.com\
│   ├── Profile_jdoe.vhdx.ccc       (Cloud Cache container, replicated from local)
│   └── Profile_jdoe.vhdx.ccc.meta

\\fslogix-dr.file.core.windows.net\profiles\  (Provider 2, same structure)
├── jdoe_jdoe@contoso.com\
│   ├── Profile_jdoe.vhdx.ccc
│   └── Profile_jdoe.vhdx.ccc.meta
```

Note the `.ccc` extension indicating Cloud Cache containers (vs `.vhdx` for traditional FSLogix).

## When to Use Cloud Cache

Cloud Cache adds complexity and resource consumption. Only implement when you have specific requirements that justify the overhead.

### Recommended Use Cases

**Multi-Region Deployments**

**Scenario:** Company has AVD host pools in East US and West Europe, users roam between regions.

**Problem without Cloud Cache:**
- User profile stored in East US (\\fslogix-eastus.file.core.windows.net)
- User logs into West Europe host pool
- Profile access crosses regions (100-150ms latency)
- Terrible user experience (slow logons, app freezes)

**Solution with Cloud Cache:**
- Provider 1: \\fslogix-eastus.file.core.windows.net\profiles
- Provider 2: \\fslogix-westeurope.file.core.windows.net\profiles
- User logs into East US → Local cache built from Provider 1 (low latency)
- User logs into West Europe → Local cache built from Provider 2 (low latency)
- Both providers stay synchronized via async replication

**Disaster Recovery Requirements**

**Scenario:** Business requires <1 hour RTO (Recovery Time Objective) for AVD service.

**Problem without Cloud Cache:**
- Primary storage account fails (regional Azure outage)
- Profiles inaccessible until storage restored or DR procedures invoked
- Manual failover required (update session host registry, remount shares)

**Solution with Cloud Cache:**
- Provider 1: Primary region (East US)
- Provider 2: DR region (West US)
- Primary region fails → Users continue working from local cache
- Replication switches to Provider 2 automatically
- No manual intervention, no user impact

**High Availability for Critical Users**

**Scenario:** Executive team requires zero profile downtime (no tolerance for "wait for IT" scenarios).

**Problem without Cloud Cache:**
- Storage account maintenance window (planned or unplanned)
- Users cannot log in during outage
- Business impact (lost productivity, missed meetings)

**Solution with Cloud Cache:**
- Provider 1: Primary storage account
- Provider 2: Secondary storage account (different availability zone)
- During maintenance, users access local cache seamlessly
- Zero user-visible downtime

**Storage Migration Scenarios**

**Scenario:** Migrating from on-premises file servers to Azure Files without downtime.

**Problem without Cloud Cache:**
- Cut-over migration: All users down during copy
- Sync migration: Complex tooling, risk of data loss

**Solution with Cloud Cache:**
- Provider 1: \\onprem-fileserver\profiles (existing)
- Provider 2: \\fslogix121025.file.core.windows.net\profiles (new)
- Enable Cloud Cache with both providers
- Profiles replicate to Azure Files over days/weeks
- Remove Provider 1 when replication complete
- Zero user downtime, gradual migration

### When NOT to Use Cloud Cache

**Single-Region Deployments with Good Storage**

If you have:
- All session hosts in one Azure region
- Premium v2 Azure Files with private endpoint
- 99.9%+ storage availability (typical)
- No DR requirements beyond backups

**Reason:** Cloud Cache adds complexity (local cache management, replication monitoring) with no tangible benefit. Premium v2 already delivers sub-millisecond latency. Local cache on session host saves 0.5ms vs 1ms (user cannot perceive).

**Cost-Constrained Environments**

Cloud Cache requires:
- Double storage costs (two provider locations)
- Larger session host disks (local cache storage)
- Additional monitoring and management overhead

For small deployments (<50 users), the cost increase (30-50%) often outweighs benefits.

**Simple Pooled Host Pools**

For basic multi-session deployments where:
- Users access same host pool/region always
- Business tolerates 1-hour RTO for storage outages
- No executive/VIP user requirements

Traditional FSLogix is simpler to troubleshoot and maintain.

> **Recommendation:** Start without Cloud Cache. Add it later if you identify specific needs (multi-region expansion, DR audit findings, storage performance issues). Don't deploy Cloud Cache "just in case."

## How Cloud Cache Works

Understanding the replication mechanics helps troubleshoot issues and optimize configuration.

### Logon Process

**Step 1: User Authentication**
- User logs into session host in host pool

**Step 2: Cloud Cache Check**
- FSLogix checks local cache directory: C:\CCCache
- Looks for existing cache file: C:\CCCache\jdoe.vhdx

**Step 3a: Cache Exists (User Previously Logged In)**
- FSLogix opens local cache file (read/write)
- Contacts Provider 1 to check for updates: \\fslogix121025.file.core.windows.net\profiles\jdoe\Profile_jdoe.vhdx.ccc
- If provider has newer data: Syncs changes to local cache (differential, only changed blocks)
- If local cache is newer (shouldn't happen): Uploads changes to provider
- Typical sync time: 5-30 seconds (depends on changes since last logon)

**Step 3b: Cache Does Not Exist (First Logon on This Session Host)**
- FSLogix contacts Provider 1: Check for existing profile
- If exists on Provider 1: Downloads profile to local cache (full copy, 2-10 minutes for 30GB profile)
- If not on Provider 1: Contact Provider 2
- If exists on Provider 2: Downloads from Provider 2
- If not on any provider: Creates new profile (2-5 minutes, initialize Windows user profile)

**Step 4: Mount Profile**
- Local cache mounted at C:\Users\jdoe
- User desktop loads (Windows completes logon)
- Total logon time (cached): 10-20 seconds
- Total logon time (first time): 3-15 minutes (downloading profile)

**Step 5: Background Replication**
- FSLogix starts two replication threads:
  - Thread 1: Monitors local cache, replicates changed blocks to Provider 1
  - Thread 2: Monitors local cache, replicates changed blocks to Provider 2
- Replication interval: Every 30 seconds by default (configurable)
- Only changed blocks uploaded (efficient)

### Active Session

**User Works Normally:**
- All file operations (open Outlook, edit Word doc, save file) write to local cache
- Sub-millisecond latency (local NVMe/SSD)
- No network dependency for reads/writes

**Background Replication Runs:**
- Every 30 seconds, FSLogix identifies blocks changed since last replication
- Uploads changed blocks to Provider 1 and Provider 2 in parallel
- Typical replication lag: 30-60 seconds behind user actions
- If provider unavailable: Queues changes, retries on next interval

**Provider Outage Handling:**
- Provider 1 fails: Replication continues to Provider 2 only, user unaffected
- Provider 2 fails: Replication continues to Provider 1 only, user unaffected
- Both providers fail: User continues working, changes accumulate in local cache, replication resumes when providers return

### Logoff Process

**Step 1: User Initiates Logoff**
- Windows begins logoff (close applications, flush caches)

**Step 2: FSLogix Final Sync**
- FSLogix waits for all replication threads to complete
- Ensures all changes uploaded to at least one provider
- Timeout: 30 seconds (configurable, LogoffTimeout registry value)
- If timeout expires: Logoff continues anyway, unsent changes lost (rare)

**Step 3: Dismount Cache**
- Local cache file dismounted
- Lock file removed
- Cache file remains on disk (C:\CCCache\jdoe.vhdx persists for next logon)

**Step 4: Cleanup (Optional)**
- If ClearCacheOnLogoff enabled: Local cache deleted, next logon re-downloads from provider
- Default: Cache persists (faster subsequent logons on same session host)

**Total Logoff Time:**
- Typical: 10-20 seconds (includes final sync)
- Long: 30+ seconds (large unsent changes, slow provider network)

## Configuration Settings

Cloud Cache is configured via registry on session hosts. All settings are under:
```
HKLM\SOFTWARE\FSLogix\Profiles
```

### Essential Settings

**CCDLocations (Required)**

Defines local cache directory. Must be on local disk (not network share).

```
Name: CCDLocations
Type: REG_MULTI_SZ (multi-string)
Value: type=smb,name=LocalCache;C:\CCCache
```

**Explanation:**
- `type=smb`: Cache type (always SMB for local disk)
- `name=LocalCache`: Friendly name (appears in logs)
- `C:\CCCache`: Local path for cache files

**Disk Requirements:**
- Fast disk (NVMe or SSD recommended, not spinning HDD)
- Sufficient space: (Max Concurrent Users × Profile Size × 1.2)
- Example: 10 users × 30GB profiles × 1.2 = 360GB minimum
- Separate disk from OS (C:\) recommended (dedicated D:\ or E:\)

**VHDLocations (Required)**

Defines remote provider locations (Azure Files, SMB shares, etc.).

```
Name: VHDLocations
Type: REG_MULTI_SZ (multi-string)
Value: type=smb,connectionString=\\fslogix121025.file.core.windows.net\profiles;type=smb,connectionString=\\fslogix-dr.file.core.windows.net\profiles
```

**Explanation:**
- Provider 1: `type=smb,connectionString=\\fslogix121025.file.core.windows.net\profiles`
- Provider 2: `type=smb,connectionString=\\fslogix-dr.file.core.windows.net\profiles`
- Separated by semicolon (;)
- Order matters: Provider 1 is primary, Provider 2 is secondary

**Up to 4 Providers Supported:**
Can define 4 providers for ultimate redundancy (rare, adds overhead).

**Provider Failover Logic:**
- Logon: Try Provider 1, if unavailable try Provider 2, etc.
- Replication: Upload to all available providers simultaneously

**Enabled (Required)**

Enables FSLogix profile management.

```
Name: Enabled
Type: REG_DWORD
Value: 1
```

### Performance Tuning

**HealthyProvidersRequiredForRegister**

Defines minimum providers that must be accessible for profile to register (user can log in).

```
Name: HealthyProvidersRequiredForRegister
Type: REG_DWORD
Value: 0 (allow logon even if all providers down, use local cache only)
Value: 1 (at least 1 provider must be accessible)
Value: 2 (at least 2 providers must be accessible)
```

**Recommendation:**
- 0 for maximum availability (user works from local cache during outages)
- 1 for balanced (ensure changes can replicate to at least one provider)
- 2 for data integrity (ensure changes replicate to both providers before logon allowed)

**ReplicationInterval**

Frequency of background replication (seconds).

```
Name: ReplicationInterval
Type: REG_DWORD
Value: 30 (default, replicate every 30 seconds)
```

**Lower values (15 seconds):** Tighter replication lag, higher network/CPU overhead
**Higher values (60 seconds):** Lower overhead, longer replication lag (more data loss risk if session host crashes)

**ClearCacheOnLogoff**

Whether to delete local cache at user logoff.

```
Name: ClearCacheOnLogoff
Type: REG_DWORD
Value: 0 (default, keep cache for next logon)
Value: 1 (delete cache at logoff)
```

**Keep Cache (0):**
- Faster subsequent logons (no re-download from provider)
- Requires more disk space on session host
- Risk: Stale cache if user logs into different session host and makes changes

**Clear Cache (1):**
- Always fresh profile from provider
- Saves session host disk space
- Slower logons (must download every time)

**Recommendation:** Keep cache (0) for persistent session hosts, clear cache (1) for ephemeral/autoscaled hosts.

### Advanced Settings

**LogoffTimeout**

Maximum time to wait for final replication before forcing logoff (seconds).

```
Name: LogoffTimeout
Type: REG_DWORD
Value: 30 (default)
```

Increase to 60-120 if users have large unsaved changes at logoff and network is slow.

**AccessNetworkAsComputerObject**

Use session host computer account (vs user account) to access network providers.

```
Name: AccessNetworkAsComputerObject
Type: REG_DWORD
Value: 0 (default, use user credentials)
Value: 1 (use computer account credentials)
```

Required when using Entra ID authentication (computer account has RBAC role assigned).

## Provider Setup (Multiple Azure Files Shares)

To implement Cloud Cache with two Azure Files providers in different regions:

### Step 1: Create Secondary Storage Account

```bash
# Primary already exists: fslogix121025 in East US
# Create secondary in West US for DR

az storage account create \
  --name fslogixwestus \
  --resource-group RG-Azure-VDI-DR \
  --location westus \
  --sku PremiumV2_LRS \
  --kind FileStorage \
  --enable-files-aadds true
```

### Step 2: Create File Share on Secondary

```bash
az storage share create \
  --name profiles \
  --account-name fslogixwestus \
  --quota 1024
```

### Step 3: Configure Private Endpoint for Secondary

```bash
az network private-endpoint create \
  --name pe-fslogix-westus \
  --resource-group RG-Azure-VDI-DR \
  --vnet-name vnet-avd-westus \
  --subnet snet-privateendpoints \
  --private-connection-resource-id $(az storage account show -n fslogixwestus -g RG-Azure-VDI-DR --query id -o tsv) \
  --group-id file \
  --connection-name fslogix-westus-connection
```

### Step 4: Assign RBAC to User Groups on Secondary

```bash
# Same groups as primary (avd-users-pooled, avd-users-personal)
az role assignment create \
  --role "Storage File Data SMB Share Contributor" \
  --assignee-object-id $(az ad group show --group "avd-users-pooled" --query id -o tsv) \
  --scope "/subscriptions/<sub-id>/resourceGroups/RG-Azure-VDI-DR/providers/Microsoft.Storage/storageAccounts/fslogixwestus/fileServices/default/fileshares/profiles"
```

### Step 5: Configure Session Hosts with Cloud Cache

```powershell
# Run on all session hosts (or via GPO/Intune)

# Enable FSLogix
New-Item -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "Enabled" -Value 1 -Type DWord

# Configure Cloud Cache local directory
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "CCDLocations" -Value "type=smb,name=LocalCache;C:\CCCache" -Type MultiString

# Configure remote providers (primary and DR)
$providers = @(
  "type=smb,connectionString=\\fslogix121025.file.core.windows.net\profiles",
  "type=smb,connectionString=\\fslogixwestus.file.core.windows.net\profiles"
)
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "VHDLocations" -Value $providers -Type MultiString

# Configure settings
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "SizeInMBs" -Value 30000 -Type DWord
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "IsDynamic" -Value 1 -Type DWord
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "VolumeType" -Value "VHDX" -Type String
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "HealthyProvidersRequiredForRegister" -Value 1 -Type DWord
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "AccessNetworkAsComputerObject" -Value 1 -Type DWord
Set-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "ReplicationInterval" -Value 30 -Type DWord
```

### Step 6: Test Replication

**Logon as test user:**
- Logs into session host in East US region
- FSLogix creates local cache: C:\CCCache\testuser.vhdx
- FSLogix creates profile on Provider 1: \\fslogix121025.file.core.windows.net\profiles\testuser\Profile_testuser.vhdx.ccc
- FSLogix replicates to Provider 2: \\fslogixwestus.file.core.windows.net\profiles\testuser\Profile_testuser.vhdx.ccc

**Verify replication:**
```powershell
# Check both providers have profile
Test-Path "\\fslogix121025.file.core.windows.net\profiles\testuser_testuser@contoso.com\Profile_testuser.vhdx.ccc"
Test-Path "\\fslogixwestus.file.core.windows.net\profiles\testuser_testuser@contoso.com\Profile_testuser.vhdx.ccc"
# Both should return True
```

**Test failover:**
- Disable Provider 1 (simulate outage): Stop storage account or block NSG
- User logs off and logs back in
- FSLogix cannot reach Provider 1, uses Provider 2 from West US
- User profile loads normally (from secondary provider)

## Performance Implications

Cloud Cache changes the performance profile of FSLogix. Understand the tradeoffs.

### Advantages

**Faster User Experience:**
- Reads/writes to local NVMe: 0.1-0.5ms latency (vs 1-5ms network storage)
- No network jitter impact on application performance
- Better responsiveness during network congestion

**Resilience to Network Issues:**
- Packet loss to Azure Files: No impact on user (local cache absorbs)
- Provider downtime: User continues working
- Multi-region deployments: Always local-region cache performance

### Disadvantages

**Initial Logon Penalty:**
- First logon on session host: Must download entire profile from provider (2-10 minutes for 30GB)
- Traditional FSLogix: Mounts VHD immediately (5-15 seconds)
- Mitigation: Pre-cache profiles on session hosts during off-hours

**Disk Space Consumption:**
- Session host must store local caches for all users
- Non-persistent hosts: Cache rebuilt daily (negates speed benefits)
- Example: 10 concurrent users × 30GB profiles = 300GB local disk required

**Replication Lag:**
- User makes change → Replicated to providers after 30 seconds (configurable)
- Session host crashes before replication: Changes lost
- Traditional FSLogix: Writes directly to provider (immediate persistence)

**Double Storage Costs:**
- Provider 1: 100 users × 30GB = 3TB × Premium v2 pricing
- Provider 2: Same (3TB × Premium v2 pricing in secondary region)
- Total: 6TB vs 3TB for single-provider FSLogix

**Complexity:**
- Two storage accounts to manage (permissions, monitoring, costs)
- Replication monitoring required (detect split-brain scenarios)
- More failure modes (local cache corruption, replication stalls)

### Performance Benchmarks

Typical latencies (milliseconds):

| Operation | Traditional FSLogix | Cloud Cache |
|-----------|-------------------|-------------|
| Open Outlook (first time) | 2,500ms | 1,800ms |
| Save Word document | 150ms | 50ms |
| Open File Explorer (My Documents) | 300ms | 100ms |
| Logon (cached profile) | 8,000ms | 12,000ms |
| Logon (first time on host) | 8,000ms | 180,000ms |

**Key Takeaway:** Cloud Cache improves active session performance, but initial logon on new session host is significantly slower.

## Best Practices

**Use Two Providers Only** - More than two providers adds complexity with minimal benefit. Two providers (primary region + DR region) covers 99.9% of scenarios. Four providers is overkill.

**Provision Fast Local Disks** - Cloud Cache performance depends on session host local disk speed. Use NVMe or Premium SSD managed disks for cache directory. Never use Standard HDD (negates all Cloud Cache benefits).

**Set HealthyProvidersRequiredForRegister to 1** - Requires at least one provider accessible for logon. Balances availability (users can log in during single-provider outage) with data integrity (changes replicate somewhere).

**Monitor Replication Lag** - Set up alerts if replication lag exceeds 2 minutes (indicates network issues or insufficient provider IOPS). FSLogix logs replication status to Event Viewer (Application log, source FSLogix-CloudCache).

**Test Failover Quarterly** - Simulate provider outages to verify users can continue working and failover works correctly. Common mistake: Forgetting to assign RBAC to secondary provider (logon fails during failover test).

**Size Session Host Disks Generously** - Allocate at least (Max Users × 40GB) for cache directory. Include 20% overhead for OS, applications, temporary files. Example: 20 users = 800GB cache disk minimum.

**Clear Cache on Autoscale Hosts** - If using autoscale to spin up/down session hosts dynamically, enable ClearCacheOnLogoff to avoid stale caches. Persistent hosts: Keep cache for faster logons.

**Use Same NTFS Permissions on Both Providers** - Configure identical NTFS ACLs on primary and secondary shares. Mismatched permissions cause replication failures (FSLogix cannot write to one provider).

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| First logon takes 15+ minutes | Downloading 30GB profile from provider over slow network | Use ExpressRoute or increase provider throughput, pre-cache profiles |
| Replication lag >5 minutes | Insufficient provider IOPS or throughput | Increase provisioned IOPS/throughput on Premium v2 storage accounts |
| Profile not found on any provider | Local cache corrupt, providers unreachable | Check network connectivity to both providers, verify DNS resolution |
| Changes lost after session host crash | Replication interval too long, changes not sent | Decrease ReplicationInterval to 15 seconds, set HealthyProvidersRequiredForRegister to 1 |
| Disk space full on session host | Too many cached profiles, cache not clearing | Increase cache disk size or enable ClearCacheOnLogoff |
| User cannot log in "No healthy providers" | Both providers down or inaccessible | Check storage account status, NSG rules, private endpoint connectivity |
| Split-brain scenario (different data on two providers) | Network partition during replication | Manually reconcile: Identify newer profile via timestamp, copy to other provider |

> **Warning:** Cloud Cache does NOT replace backups. Local cache and remote providers can all become corrupted simultaneously (ransomware, bugs, misconfigurations). Always maintain separate backup copies of profiles share to separate storage account or Azure Backup.

## Next Steps

- **Page 5:** Troubleshoot FSLogix profile issues using logs, Event Viewer, and diagnostic tools to resolve Cloud Cache replication problems and performance issues