---
title: Backup & Disaster Recovery
description: 
published: true
date: 2025-12-14T04:53:39.692Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:51.491Z
---

# Backup & Disaster Recovery

Backup and disaster recovery (DR) planning is critical for AVD deployments to protect user data, minimize downtime, and meet business continuity requirements. Unlike traditional VDI where everything is backed up, AVD uses a hybrid approach: stateless pooled desktops are rebuilt from images, while user profiles and personal desktops are backed up. This page covers what to back up, Azure Backup configuration, FSLogix-specific strategies, and disaster recovery architecture.

## Azure Backup Overview

Azure Backup is a fully managed service that provides scalable, secure, and cost-effective backup for Azure and on-premises resources. Key features:

**Capabilities**:
- **Application-aware backups**: Consistent backups for VMs, SQL databases, file shares (VSS snapshots on Windows)
- **Incremental backups**: Only changed data after initial full backup (reduces storage and time)
- **Encryption**: Data encrypted at rest (AES-256) and in transit (TLS 1.2)
- **Geo-redundancy**: Store backups in secondary Azure region for disaster recovery
- **Long-term retention**: Keep backups for years (compliance requirements)

**Architecture**:
- **Recovery Services Vault**: Container for backup data. Our vault: `vault588` in RG-Azure-VDI-01.
- **Backup Policies**: Define schedule (daily, weekly), retention (7 days, 30 days, 1 year), and consistency (app-aware vs crash-consistent).
- **Backup Agents**: Microsoft Azure Recovery Services (MARS) agent for VMs, Azure Backup Server for on-premises.

**Cost**: Pay for backup storage (GRS or LRS) + protected instances. Example: 100 GB VM backed up daily = ~$10/month (GRS storage).

## What to Back Up in AVD

AVD environments have different backup needs depending on resource type:

### Personal Desktops: Yes (Back Up)

**What**: Persistent desktops assigned to specific users (1:1 mapping).

**Why**: User customizes desktop (installed apps, desktop files, settings). Loss of desktop = loss of user data and productivity.

**What to Protect**:
- Entire VM (OS disk + data disks)
- User-installed applications
- Local files (if not redirected to OneDrive/FSLogix profile)

**Backup Frequency**: Daily (nightly backups during off-hours).

**Retention**: 30 days (last month of data). Extend to 90 days for compliance-sensitive roles (finance, legal).

**Method**: Azure Backup for VMs (application-consistent backups using VSS).

### Pooled Desktops: No (Usually)

**What**: Non-persistent desktops shared by multiple users (many:many).

**Why**: Stateless design. User logs out, VM returns to clean state (from golden image). No unique data on VM to protect.

**Exception**: If pooled desktop has user-installed apps (breaks stateless model), consider backing up. Better approach: Bake apps into golden image.

**Backup Frequency**: None (rebuild from image if VM fails).

**Recovery**: Deploy new session host from Compute Gallery image.

### FSLogix Profiles: Yes (Critical)

**What**: User profile VHD/VHDX files stored on Azure Files or NetApp Files.

**Why**: Contains user data, app settings, browser history, Outlook cache. Loss of profile = user starts fresh (data loss).

**What to Protect**:
- User profile containers (Profile_username.vhdx)
- Office containers (ODFC_username.vhdx) if using Office Container
- Cloud Cache metadata (if using Cloud Cache)

**Backup Frequency**: Multiple snapshots per day (every 4-6 hours) + daily Azure Backup.

**Retention**: 30 days (frequent snapshots), 1 year (daily backups for compliance).

**Method**: Azure Backup for Azure Files + Azure Files snapshots.

### Golden Images: Stored in Compute Gallery (Versioned)

**What**: Master image used to deploy session hosts.

**Why**: Images are versioned in Azure Compute Gallery (formerly Shared Image Gallery). Each build creates new version. No need for separate backups.

**Protection**: Compute Gallery replicates images across regions (configured per gallery). Keep 3-5 recent versions (last 3 months of builds).

**Recovery**: Deploy session hosts from specific image version.

**Backup Frequency**: None (versioning provides protection).

**Retention**: 3-5 versions per image (delete older versions to save storage costs).

## Recovery Services Vault Configuration

Our Recovery Services Vault: `vault588` in RG-Azure-VDI-01.

### Create Backup Policy for Personal Desktops

```bash
# Create daily backup policy with 30-day retention
az backup policy create \
  --resource-group RG-Azure-VDI-01 \
  --vault-name vault588 \
  --name Policy-AVD-PersonalDesktops \
  --backup-management-type AzureIaasVM \
  --policy '{
    "schedulePolicy": {
      "schedulePolicyType": "SimpleSchedulePolicy",
      "scheduleRunFrequency": "Daily",
      "scheduleRunTimes": ["2025-12-14T02:00:00Z"]
    },
    "retentionPolicy": {
      "retentionPolicyType": "LongTermRetentionPolicy",
      "dailySchedule": {
        "retentionTimes": ["2025-12-14T02:00:00Z"],
        "retentionDuration": {
          "count": 30,
          "durationType": "Days"
        }
      },
      "weeklySchedule": {
        "daysOfTheWeek": ["Sunday"],
        "retentionTimes": ["2025-12-14T02:00:00Z"],
        "retentionDuration": {
          "count": 12,
          "durationType": "Weeks"
        }
      }
    }
  }'
```

**Policy Details**:
- **Daily**: Backup at 2 AM UTC (off-hours)
- **Retention**: 30 daily backups + 12 weekly backups (Sunday)
- **Type**: Application-consistent (VSS snapshot)

### Enable Backup on Personal Desktop VMs

```bash
# Enable backup for specific VM
az backup protection enable-for-vm \
  --resource-group RG-Azure-VDI-01 \
  --vault-name vault588 \
  --vm avd-personal-jdoe \
  --policy-name Policy-AVD-PersonalDesktops
```

**At Scale** (backup all VMs with tag `BackupRequired=true`):

```bash
# Get VMs with backup tag
vms=$(az vm list --resource-group RG-Azure-VDI-01 --query "[?tags.BackupRequired=='true'].name" -o tsv)

# Enable backup for each
for vm in $vms; do
  echo "Enabling backup for $vm"
  az backup protection enable-for-vm \
    --resource-group RG-Azure-VDI-01 \
    --vault-name vault588 \
    --vm $vm \
    --policy-name Policy-AVD-PersonalDesktops
done
```

### Geo-Redundant Storage (GRS)

**Default**: Recovery Services Vault uses Geo-Redundant Storage (GRS) - backups replicated to secondary region (paired region).

**Paired Regions**:
- Central US ↔ East US 2
- East US ↔ West US
- West Europe ↔ North Europe

**Our Vault**: vault588 in Central US → backups replicated to East US 2.

**Cross-Region Restore** (Disaster Recovery):
1. Azure Portal → Recovery Services Vault → vault588
2. Backup items → Azure Virtual Machine
3. Select VM → Restore
4. Choose restore point → **Restore to secondary region (East US 2)**
5. Create new VM in DR resource group

**Cost**: GRS costs 2x LRS (Locally Redundant Storage). GRS: ~$0.10/GB/month. LRS: ~$0.05/GB/month.

**When to Use**:
- **GRS**: Production workloads, multi-region DR strategy
- **LRS**: Dev/test environments, single-region deployments

## Backup Policies

Backup policies define when backups occur and how long they're kept:

### Frequency Options

**Daily**:
- Runs once per day at specified time (e.g., 2 AM)
- Suitable for: User VMs, file shares

**Weekly**:
- Runs on specific days (e.g., Sunday, Wednesday)
- Suitable for: Infrequently-changed data, cost optimization

**Multiple per Day** (Azure Files only):
- Snapshots every 4, 6, or 8 hours
- Suitable for: FSLogix profiles (high change rate)

### Retention Options

**Daily Retention**:
- Keep last 7, 14, 30, or 90 days of daily backups
- Example: 30 daily backups = restore to any day in last month

**Weekly Retention**:
- Keep weekly backups for weeks/months
- Example: 12 weeks = 3 months of Sunday backups

**Monthly Retention**:
- Keep first/last day of month for months/years
- Example: 12 months = 1 year of monthly backups

**Yearly Retention**:
- Keep annual backups for compliance (3, 5, 7, 10 years)
- Example: 7 years for financial records (SOX compliance)

**Our Policy** (FSLogix Profiles):
- **Snapshots**: Every 6 hours, keep 2 days (8 snapshots)
- **Daily**: 2 AM, keep 30 days
- **Weekly**: Sunday, keep 12 weeks
- **Monthly**: First of month, keep 12 months
- **Yearly**: January 1st, keep 3 years

### Geo-Redundancy

**Locally Redundant Storage (LRS)**:
- 3 copies within same datacenter
- Protects against disk/node failures
- Lower cost (~$0.05/GB/month)
- **Risk**: Datacenter-level disaster = data loss

**Geo-Redundant Storage (GRS)**:
- 3 copies in primary region + 3 copies in secondary region (paired region)
- Protects against regional outages
- Higher cost (~$0.10/GB/month)
- **Benefit**: Cross-region restore for DR

**Zone-Redundant Storage (ZRS)** (Preview):
- 3 copies across availability zones in same region
- Protects against zone failures
- Mid-tier cost (~$0.075/GB/month)

**Recommendation**: Use GRS for production AVD (especially FSLogix profiles). Use LRS for dev/test.

## FSLogix Backup Options

FSLogix profiles require special backup strategies due to high change rate and user expectations (near-zero data loss).

### Azure Backup for Azure Files

**How it Works**:
- Azure Backup takes scheduled snapshots of Azure Files share
- Snapshots are incremental (only changed blocks)
- User or admin can restore individual files or entire profile

**Setup**:

```bash
# Enable backup for Azure Files share (FSLogix profiles)
az backup protection enable-for-azurefileshare \
  --resource-group RG-Azure-VDI-01 \
  --vault-name vault588 \
  --storage-account stavdprofiles \
  --azure-file-share profiles \
  --policy-name Policy-FSLogix-Profiles
```

**Restore Profile**:
1. Azure Portal → Recovery Services Vault → vault588
2. Backup items → Azure Storage (Azure Files)
3. Select profiles share → Restore
4. Choose restore point (date/time)
5. **File Recovery**: Restore specific user's VHDX
6. **Full Share Recovery**: Restore entire share to alternate location

**Advantages**:
- Managed by Azure (no scripts to maintain)
- Application-consistent (VSS snapshots)
- Long-term retention (years)

**Limitations**:
- Minimum snapshot interval: 1 per day
- Restore can take minutes to hours (depending on data size)

### Azure Files Snapshots (Self-Managed)

**How it Works**:
- Azure Files native snapshot feature (independent of Azure Backup)
- Snapshots taken on-demand or via scheduled script
- Snapshots stored in same storage account (billed only for changed data)

**Setup** (Automated Snapshots):

```bash
# Create snapshot via Azure CLI (run via Azure Automation schedule)
az storage share snapshot \
  --account-name stavdprofiles \
  --name profiles \
  --quota 1024 \
  --metadata "CreatedBy=AutomationRunbook" "Purpose=HourlyBackup"
```

**Automation** (Azure Automation Runbook):

```powershell
# Runbook: Create-FSLogixSnapshot.ps1
# Schedule: Every 6 hours

$storageAccount = "stavdprofiles"
$shareName = "profiles"
$retentionDays = 2

# Create snapshot
$snapshot = New-AzStorageShareSnapshot -StorageAccountName $storageAccount -Name $shareName

# Clean up old snapshots (older than retention)
$allSnapshots = Get-AzStorageShareSnapshot -StorageAccountName $storageAccount -Name $shareName
$oldSnapshots = $allSnapshots | Where-Object { $_.SnapshotTime -lt (Get-Date).AddDays(-$retentionDays) }

foreach ($old in $oldSnapshots) {
  Remove-AzStorageShareSnapshot -StorageAccountName $storageAccount -Name $shareName -SnapshotTime $old.SnapshotTime
}
```

**Restore from Snapshot**:

```powershell
# Mount snapshot as network drive (read-only)
$snapshotPath = "\\stavdprofiles.file.core.windows.net\profiles\.snapshot\{snapshot-timestamp}\jdoe\Profile_jdoe.vhdx"

# Copy file from snapshot to live share
Copy-Item -Path $snapshotPath -Destination "\\stavdprofiles.file.core.windows.net\profiles\jdoe\Profile_jdoe.vhdx"
```

**Advantages**:
- Frequent snapshots (hourly if needed)
- Fast restore (copy file from snapshot)
- Low cost (incremental storage only)

**Limitations**:
- Self-managed (requires automation)
- Retention limited (Azure Files max 200 snapshots per share)
- No cross-region replication (snapshots in same storage account)

### FSLogix Cloud Cache for High Availability

**What**: FSLogix Cloud Cache writes user profile to multiple storage locations simultaneously (active-active replication).

**How it Works**:
1. User logs in
2. FSLogix mounts profile from **both** storage accounts (primary + secondary)
3. Writes go to both locations in real-time
4. If primary fails, FSLogix seamlessly uses secondary (no user interruption)

**Configuration** (Registry on session hosts):

```powershell
# Cloud Cache with Azure Files (primary) and Azure NetApp Files (secondary)
New-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "CCDLocations" -Value "type=azure,connectionString=|stavdprofiles.file.core.windows.net\profiles;type=smb,connectionString=\\10.0.2.100\profiles" -PropertyType String

New-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "Enabled" -Value 1 -PropertyType DWord
New-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "VHDLocations" -Value "\\stavdprofiles.file.core.windows.net\profiles" -PropertyType String
```

**Use Case**: Mission-critical users (executives, traders) who can't tolerate any downtime. Cloud Cache adds cost (2x storage) but provides active-active HA.

**Backup Strategy with Cloud Cache**:
- Cloud Cache provides HA (one storage fails, other continues)
- Still need backups for user error (accidental deletion) or corruption
- Back up **one** of the Cloud Cache locations (no need to back up both)

## Disaster Recovery Architecture

Disaster recovery (DR) ensures AVD remains available if primary region fails.

### Multi-Region Deployment

**Architecture**:
- **Primary Region**: Central US (production AVD)
  - Session hosts: avd-pool-0, avd-pool-1, ...
  - FSLogix profiles: stavdprofiles (Azure Files)
  - Host pool: HostPool-VDI-01

- **Secondary Region**: East US 2 (DR standby)
  - Session hosts: avd-pool-dr-0, avd-pool-dr-1, ... (powered off until failover)
  - FSLogix profiles: stavdprofilesdr (replicated from Central US)
  - Host pool: HostPool-VDI-01-DR

**Failover Process**:
1. Primary region outage detected
2. Power on session hosts in East US 2
3. Update DNS or AVD workspace to point to DR host pool
4. Users connect to East US 2 session hosts
5. Profiles load from stavdprofilesdr (GRS restore or Azure Files sync)

**Recovery Time Objective (RTO)**: Time to restore service.
- **With pre-deployed hosts (powered off)**: 15-30 minutes (time to power on VMs)
- **Without pre-deployed hosts**: 2-4 hours (deploy VMs from Compute Gallery image)

**Recovery Point Objective (RPO)**: Data loss tolerance.
- **FSLogix profiles with GRS**: ~15 minutes (last snapshot before outage)
- **FSLogix with Cloud Cache across regions**: ~0 minutes (real-time replication)

### Azure Site Recovery (ASR) for VMs

**What**: Azure Site Recovery replicates VMs to secondary region with continuous sync.

**How it Works**:
1. ASR agent installed on VM
2. Disk changes replicated to DR region every 5-15 minutes
3. Failover: Power on replica VM in DR region (latest state)
4. Failback: Reverse replication when primary region recovers

**Cost**: ~$25/VM/month + storage for replica disks.

**Use Case**: Personal desktops (persistent VMs) that need <15 min RPO.

**Not Recommended For**: Pooled session hosts (stateless, cheaper to rebuild from image than replicate).

**Setup**:

```bash
# Enable ASR for personal desktop VM
az backup protection enable-for-vm \
  --resource-group RG-Azure-VDI-01 \
  --vault-name vault588 \
  --vm avd-personal-jdoe \
  --policy-name ASR-DR-Policy \
  --target-resource-group RG-Azure-VDI-DR \
  --target-location eastus2
```

**Failover**:
1. Azure Portal → Recovery Services Vault → vault588
2. Replicated items → avd-personal-jdoe
3. Failover → Select recovery point → Test failover (creates DR VM without affecting prod)
4. Planned failover (for maintenance) or Failover (during outage)

### DNS Failover

**Challenge**: Users connect to AVD workspace URL (e.g., `https://rdweb.wvd.microsoft.com`). How to redirect to DR region?

**Solution 1: Azure Traffic Manager**
- Create Traffic Manager profile with priority routing
- Primary endpoint: HostPool-VDI-01 (Central US)
- Secondary endpoint: HostPool-VDI-01-DR (East US 2)
- Traffic Manager health probes detect primary failure → routes to secondary

**Solution 2: Manual Workspace Update**
- Update AVD workspace to use DR host pool
- Users log out/log in to get new resource list

**Solution 3: Multiple Workspaces** (Recommended)
- Users subscribed to both prod and DR workspaces
- During outage, instruct users to connect to DR workspace
- No DNS changes required

## RTO/RPO Considerations

**RTO (Recovery Time Objective)**: How long can business tolerate downtime?

| Tier | RTO | Strategy | Cost |
|------|-----|----------|------|
| Basic | 4-8 hours | Rebuild session hosts from Compute Gallery, restore profiles from Azure Backup | Low (~$100/month) |
| Standard | 1-2 hours | Pre-deployed DR hosts (powered off), GRS profile backups | Medium (~$500/month) |
| Premium | <15 minutes | Active-active multi-region, Cloud Cache, ASR for personal desktops | High (~$2000/month) |

**RPO (Recovery Point Objective)**: How much data loss is acceptable?

| Tier | RPO | Strategy | Cost |
|------|-----|----------|------|
| Daily | 24 hours | Daily backups of profiles (2 AM) | Low (~$50/month) |
| Frequent | 4-6 hours | Snapshots every 6 hours | Medium (~$100/month) |
| Near-zero | <5 minutes | Cloud Cache active-active replication | High (~$300/month for storage replication) |

**Our Configuration** (Standard Tier):
- **RTO**: 1 hour (DR session hosts powered off in East US 2, 10 minutes to power on)
- **RPO**: 6 hours (FSLogix snapshots every 6 hours)
- **Cost**: ~$600/month (DR VMs powered off = compute cost only when running, GRS storage for profiles)

## Best Practices

- **Test DR Failover Quarterly** - Run test failover to DR region. Verify session hosts boot, users can connect, profiles load. Document any issues. Test failover doesn't impact production.

- **Automate Backup Verification** - Use Azure Automation to query backup status daily. Alert if backup job fails or VM not protected. Don't assume backups work without verification.

- **Tag VMs for Backup Policies** - Use Azure tags (`BackupRequired=true`, `BackupTier=Premium`) to auto-assign backup policies. Prevents missed VMs.

- **Separate Production and DR Workspaces** - Don't mix prod and DR resources in same workspace. Easier to failover and failback.

- **Document Runbooks** - Create step-by-step DR runbook (wiki page) with failover procedures, contact numbers, escalation paths. Assume on-call engineer is new and needs detailed instructions.

- **Monitor Backup Costs** - Backup storage grows over time (retention). Set budget alerts. Review retention policies annually (do you really need 7 years of backups?).

- **Use Immutable Backups for Ransomware Protection** - Enable "Soft Delete" and "Immutable Vault" in Recovery Services Vault. Prevents attacker from deleting backups even with admin credentials.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Backup job fails with "VSS snapshot failed" | VM is under heavy load or VSS writers unhealthy | Retry backup during off-hours. Check VSS writers: `vssadmin list writers`. Restart VM if needed. |
| Restored VM won't boot | Backup was crash-consistent (not app-aware) | Use app-consistent backup policy (requires VM agent). Restore from earlier recovery point. |
| FSLogix profile not in backup | Profile created after last snapshot | Reduce snapshot interval (e.g., 6 hours → 4 hours). Use Cloud Cache for zero data loss. |
| Cross-region restore is slow (hours) | Large VMs (TBs of data) take time to copy across regions | Plan for RTO. Pre-deploy DR VMs (powered off) to avoid restore time. Use ASR for faster failover. |
| Backup cost higher than expected | GRS storage + long retention for all VMs | Use LRS for dev/test. Reduce retention (30 days → 14 days for non-critical VMs). Archive old backups to cool tier. |
| Can't restore file from snapshot | Snapshot deleted (retention expired) | Extend retention for critical data. Use Azure Backup (not just snapshots) for long-term retention. |
| DR failover failed - hosts won't start | DR VMs were deleted or deallocated without snapshot | Keep DR hosts in "Stopped (allocated)" state (retains IP/NIC) or use ASR. Document DR resource dependencies. |

## Related Resources

- Azure Backup Overview: https://learn.microsoft.com/azure/backup/backup-overview
- Backup Azure VMs: https://learn.microsoft.com/azure/backup/backup-azure-vms-introduction
- Backup Azure Files: https://learn.microsoft.com/azure/backup/backup-azure-files
- Azure Site Recovery: https://learn.microsoft.com/azure/site-recovery/site-recovery-overview
- FSLogix Cloud Cache: https://learn.microsoft.com/fslogix/cloud-cache-resiliency-availability-cncpt
- Disaster Recovery Planning: https://learn.microsoft.com/azure/virtual-desktop/disaster-recovery