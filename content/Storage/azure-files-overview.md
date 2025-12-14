---
title: Azure Files Overview
description: 
published: true
date: 2025-12-14T04:53:45.004Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:27.491Z
---

# Azure Files Overview

Azure Files is Microsoft's fully managed cloud file share service that provides serverless SMB and NFS file shares accessible from Windows, Linux, and macOS. For Azure Virtual Desktop deployments, Azure Files serves as the critical storage layer for FSLogix profile containers, ensuring users have consistent profile experiences across session hosts.

## What is Azure Files

Azure Files delivers cloud-based file shares accessible via industry-standard protocols (SMB 3.x, NFS 4.1). Unlike blob storage which requires application-level integration, Azure Files presents traditional file shares that can be mounted using standard operating system commands (net use on Windows, mount on Linux).

**Key Characteristics:**
- **Protocol Support:** SMB 2.1, SMB 3.0, SMB 3.1.1 (with encryption), NFS 4.1
- **Access Methods:** REST API, SMB/NFS mount, Azure Portal, Storage Explorer
- **Authentication:** Active Directory Domain Services, Entra ID (Azure AD), Storage account keys
- **Integration:** Native Azure service with RBAC, Private Endpoints, encryption at rest
- **Scale:** Up to 100TB per share (standard), 100TB+ per share (premium)

## Storage Tiers Explained

Azure Files offers multiple performance tiers designed for different workload requirements. Choosing the correct tier is critical for FSLogix performance and cost optimization.

### Standard Tiers (HDD-Based)

Standard file shares use magnetic hard drives and offer three sub-tiers optimized for different access patterns:

**Transaction Optimized (Default)**
- Best for: General-purpose workloads with mixed read/write operations
- IOPS: Up to 10,000 per share (burst), 1,000 baseline
- Throughput: Up to 300 MiB/s ingress, 60 MiB/s egress
- Latency: Higher than premium (typical 10-20ms)
- Cost Model: Pay for storage + transaction costs
- FSLogix Use Case: Development/test environments only

**Hot Tier**
- Best for: Frequently accessed data with predictable access patterns
- IOPS: Same as transaction optimized
- Throughput: Same as transaction optimized
- Cost Model: Higher storage cost, lower transaction costs
- FSLogix Use Case: Not recommended (latency concerns)

**Cool Tier**
- Best for: Archive/backup data accessed infrequently
- IOPS: Same as transaction optimized
- Throughput: Same as transaction optimized
- Cost Model: Lowest storage cost, highest transaction costs
- FSLogix Use Case: Never use for profiles (too slow)

> **Warning:** Standard tier shares are NOT recommended for production FSLogix workloads. The HDD-based storage cannot deliver the IOPS and low latency required for smooth user logon experiences.

### Premium Tiers (SSD-Based)

Premium file shares use solid-state drives and are the only suitable option for production FSLogix deployments.

**Premium FileStorage (Original Premium)**
- Storage Account Kind: FileStorage
- Performance: Provisioned IOPS model
- Base IOPS: 3,000 + (1 IOPS per GiB provisioned)
- Burst IOPS: Up to 100,000 IOPS per share
- Throughput: 100 MiB/s + (0.04 MiB/s per GiB) + (0.06 MiB/s per provisioned IOPS)
- Latency: Single-digit milliseconds (1-2ms typical)
- Minimum Share Size: 100 GiB
- Cost Model: Pay for provisioned capacity (not used space)
- FSLogix Use Case: Production workloads, 50-500 users

**Premium v2 (Newest, Recommended)**
- Storage Account Kind: FileStorage with Premium_LRS or PremiumV2_LRS
- Performance: Pay-as-you-go IOPS and throughput
- Base IOPS: Up to 80,000 IOPS per share
- Burst IOPS: Not applicable (direct provisioning)
- Throughput: Up to 10 GiB/s per share
- Latency: Sub-millisecond (0.5-1ms typical)
- Minimum Share Size: 32 GiB
- Cost Model: Pay for storage used + provisioned IOPS + provisioned throughput
- FSLogix Use Case: **Best choice for all production deployments**

## Our Example: fslogix121025 Configuration

In the RG-Azure-VDI-01 resource group, the storage account **fslogix121025** demonstrates premium v2 configuration:

```
Storage Account: fslogix121025
  - Kind: FileStorage
  - SKU: PremiumV2_LRS
  - Location: Same as session hosts (minimize latency)
  - Redundancy: Locally Redundant Storage (LRS)
  - Large File Shares: Enabled
  - Networking: Private endpoint enabled
  - Authentication: Entra ID enabled
```

**Why PremiumV2_LRS?**
- Predictable low latency for user logons
- Pay only for IOPS/throughput consumed (vs provisioned)
- Better cost efficiency for variable workloads
- Sufficient redundancy for single-region deployments

**When to Use Premium vs Premium v2:**
- Premium v2: Variable user loads, cost optimization priority, single region
- Premium (original): Predictable workloads, need burst capacity, multi-region replication

## FSLogix Sizing Requirements

FSLogix profile containers require specific IOPS and throughput thresholds to deliver acceptable user experiences. Undersizing storage is the most common cause of slow logons and application freezes.

### IOPS Requirements

**Per-User IOPS (Average):**
- Idle session: 5-10 IOPS
- Logon/logoff: 50-200 IOPS (burst)
- Active use (Office apps): 20-50 IOPS
- Power user (multiple apps): 50-100 IOPS

**Total IOPS Calculation:**
```
Peak IOPS = (Concurrent Users × 50 IOPS) + (Logon Users × 150 IOPS)

Example: 100 concurrent users, 20% logging on simultaneously
= (100 × 50) + (20 × 150)
= 5,000 + 3,000
= 8,000 IOPS required
```

For Premium v2 on **fslogix121025**, provision at least 10,000 IOPS for 100 users to handle peak loads.

### Throughput Requirements

**Per-User Throughput:**
- Logon: 10-25 MiB/s (profile read)
- Logoff: 5-15 MiB/s (profile write-back)
- Active session: 1-5 MiB/s

**Total Throughput Calculation:**
```
Peak Throughput = (Simultaneous Logons × 25 MiB/s)

Example: 20 simultaneous logons
= 20 × 25 MiB/s
= 500 MiB/s required
```

For Premium v2, provision at least 600 MiB/s for environments with 20+ simultaneous logons.

### Latency Targets

- **Good:** < 2ms average latency
- **Acceptable:** 2-5ms average latency
- **Poor:** > 5ms average latency (investigate)

Monitor latency using Azure Monitor metrics on the storage account. Consistently high latency indicates network issues, insufficient IOPS, or need for premium tier upgrade.

## Large File Shares (>5TB)

Standard Azure Files shares are limited to 5TB by default. For larger FSLogix deployments, enable Large File Shares:

**Capacity Limits:**
- Standard (without large file shares): 5TB per share
- Standard (with large file shares): 100TB per share
- Premium: 100TB per share (always enabled)

**Enabling Large File Shares:**

Cannot be enabled on existing storage accounts with standard shares. Must create new storage account with feature enabled:

```bash
az storage account create \
  --name fslogixlarge \
  --resource-group RG-Azure-VDI-01 \
  --location eastus \
  --sku Premium_LRS \
  --kind FileStorage \
  --enable-large-file-share
```

**When to Enable:**
- 500+ users with 30GB profiles each = 15TB minimum
- Multi-purpose share (profiles + office containers)
- Long-term growth without migration

**Storage Account Kind Compatibility:**
- Standard: Requires explicit enable flag
- Premium (FileStorage): Automatically enabled, cannot disable

## Performance Characteristics

### Baseline Performance (Premium v2)

| Metric | Premium v2 | Premium (Original) | Standard |
|--------|-----------|-------------------|----------|
| Max IOPS/Share | 80,000 | 100,000 (burst) | 10,000 (burst) |
| Latency (avg) | 0.5-1ms | 1-2ms | 10-20ms |
| Throughput/Share | 10 GiB/s | Variable | 300 MiB/s |
| Minimum Size | 32 GiB | 100 GiB | 1 GiB |
| Billing Model | Pay-per-use | Provisioned | Storage + transactions |

### Factors Affecting Performance

**Network Connectivity:**
- Public endpoint: Subject to internet latency and bandwidth
- Private endpoint: Recommended, uses Azure backbone network
- Service endpoint: Good alternative if private endpoint not feasible

**SMB Multichannel:**
- Enabled by default on Premium v2 and Premium
- Requires multiple NICs or RSS-capable NIC on session hosts
- Can increase throughput by 2-4x
- Verify with: `Get-SmbClientConfiguration | Select EnableMultichannel`

**Concurrent Users:**
- More users = more IOPS demand
- Consider staggered logon schedules to avoid IOPS throttling
- Monitor storage account metrics during peak hours

## Protocol Support Details

### SMB (Server Message Block)

**Supported Versions:**
- SMB 2.1: Windows 7, Windows Server 2008 R2+ (not encrypted)
- SMB 3.0: Windows 8, Windows Server 2012+ (encrypted channel support)
- SMB 3.1.1: Windows 10, Windows Server 2016+ (AES-256 encryption)

**SMB Security Features:**
- Encryption in transit (SMB 3.0+)
- Signing (integrity verification)
- Channel binding (prevents relay attacks)
- Pre-authentication integrity (SMB 3.1.1)

**For AVD/FSLogix:**
Always use SMB 3.1.1 with encryption enabled (default on Windows 10/11 session hosts).

### NFS (Network File System)

NFS 4.1 support is available for Linux-based workloads:
- Not applicable for FSLogix (Windows-only)
- Useful for Linux session hosts storing user data
- Requires VNet integration (no public endpoint support)
- No authentication (uses network security only)

## Best Practices

**Use Premium v2 for Production** - The cost difference between standard and premium is negligible compared to the productivity loss from slow logons. Premium v2 offers the best balance of performance and cost.

**Enable Private Endpoints** - Public endpoints expose storage accounts to the internet. Private endpoints (like pe-fslogix-files in our setup) keep traffic on the Azure backbone network and reduce latency by 30-50%.

**Right-Size Provisioning** - Start with conservative IOPS/throughput estimates and monitor actual usage. Premium v2's pay-per-use model makes it safe to over-provision initially and scale down based on metrics.

**Monitor Storage Metrics** - Set up Azure Monitor alerts for:
  - Availability < 99.9%
  - Average latency > 5ms
  - IOPS throttling events
  - Capacity > 80% of provisioned

**Plan for Growth** - FSLogix profile containers grow over time. Allocate 30GB per user minimum, 50GB for power users. Monitor weekly and adjust share quotas proactively.

**Use Separate Storage Accounts** - Isolate FSLogix storage from other workloads to prevent noisy neighbor issues. Create dedicated storage accounts per AVD host pool for large deployments (500+ users).

**Enable Large File Shares** - Always enable on new storage accounts to avoid future migration. The feature has no cost impact and prevents capacity constraints.

**Test Performance Before Production** - Use tools like DiskSpd or LoginVSI to simulate user loads and validate IOPS/latency targets. Identify bottlenecks during pilot phase, not production rollout.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Slow logons (>30 sec) | Insufficient IOPS or high latency | Upgrade to Premium v2, add provisioned IOPS, check network path |
| Intermittent disconnects | Storage account throttling | Monitor metrics, increase IOPS/throughput quotas |
| Cannot mount share | Network/firewall blocking SMB | Enable private endpoint, check NSG rules for port 445 |
| High storage costs | Over-provisioned Premium (original) | Switch to Premium v2 pay-per-use model |
| Profile corruption | IOPS throttling during write | Increase IOPS, enable Cloud Cache for redundancy |
| 5TB limit reached | Large file shares not enabled | Migrate to new storage account with feature enabled |

> **Warning:** Never use storage account keys in production FSLogix configurations. Always use Entra ID authentication with RBAC roles. Storage account keys provide unrestricted access and cannot be audited per-user.

## Next Steps

- **Page 2:** Learn how FSLogix Profile Containers work and how they use Azure Files
- **Page 3:** Configure Entra ID authentication and RBAC permissions for user access
- **Page 4:** Implement FSLogix Cloud Cache for high availability scenarios
- **Page 5:** Troubleshoot common FSLogix profile issues using logs and diagnostic tools