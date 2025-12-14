---
title: Session Host Sizing
description: 
published: true
date: 2025-12-14T04:52:24.671Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:12.629Z
---

# Session Host Sizing

Choosing the right VM size for AVD session hosts is one of the most important decisions for balancing performance, user experience, and cost. Undersized VMs lead to poor performance and user complaints; oversized VMs waste budget. This page provides practical guidance on VM families, sizing formulas, disk types, and right-sizing based on real-world metrics.

## VM Families for AVD

Azure offers dozens of VM families, but only a few are well-suited for AVD workloads. The right choice depends on your user workload profile (light, medium, heavy) and specific application requirements.

### D-Series (General Purpose)

**Best For:** Standard office workers, web apps, light LOB applications

**Characteristics:**
- Balanced CPU-to-RAM ratio (1 vCPU : 4 GB RAM)
- Cost-effective for most AVD deployments
- Good for Windows 10/11 Multi-Session pooled desktops

**Common Sizes:**

| Size | vCPU | RAM | Temp Storage | Cost (East US, approx) | Recommended Users (Pooled) |
|------|------|-----|--------------|------------------------|----------------------------|
| **Standard_D2s_v5** | 2 | 8 GB | 75 GB | $0.096/hour | 4-8 light users |
| **Standard_D4s_v5** | 4 | 16 GB | 150 GB | $0.192/hour | 10-16 light users, 6-10 medium users |
| **Standard_D8s_v5** | 8 | 32 GB | 300 GB | $0.384/hour | 20-32 light users, 12-16 medium users |
| **Standard_D16s_v5** | 16 | 64 GB | 600 GB | $0.768/hour | 40-64 light users, 24-32 medium users |

**Our Environment:**
- **Session Hosts:** avd-pool-0, avd-pool-1, avd-pool-2
- **Size:** Standard_D2s_v4 (2 vCPU, 8 GB RAM)
- **Workload:** 10 light users (Office 365, web apps, email)
- **Max Sessions:** 10 per host
- **Performance:** CPU averages 40-50%, RAM 60-70%

**When to Use:**
- **Light Users:** Office 365, web browsers, email, PDF viewing
- **Medium Users:** Light multitasking (Office + Teams + web apps)
- **Pooled Desktops:** Windows 10/11 Multi-Session

### E-Series (Memory-Optimized)

**Best For:** Heavy Office users, data analysis, large Excel/database apps

**Characteristics:**
- High RAM-to-CPU ratio (1 vCPU : 8 GB RAM)
- 2x the memory of D-series for the same vCPU count
- More expensive, but necessary for memory-intensive workloads

**Common Sizes:**

| Size | vCPU | RAM | Temp Storage | Cost (East US, approx) | Recommended Users (Pooled) |
|------|------|-----|--------------|------------------------|----------------------------|
| **Standard_E2s_v5** | 2 | 16 GB | 75 GB | $0.126/hour | 3-6 medium users |
| **Standard_E4s_v5** | 4 | 32 GB | 150 GB | $0.252/hour | 8-12 medium users, 4-6 heavy users |
| **Standard_E8s_v5** | 8 | 64 GB | 300 GB | $0.504/hour | 16-24 medium users, 8-12 heavy users |

**When to Use:**
- **Heavy Office Users:** Large Excel files (100+ MB), Outlook with 10+ GB mailboxes
- **Data Analysis:** Power BI, SQL Server Management Studio, data visualization tools
- **Memory-Intensive Apps:** GIS software, scientific applications

**Example Scenario:**

Finance team runs complex Excel models with 200 MB files, multiple pivot tables, and linked workbooks. Standard_D4s_v5 (16 GB RAM) shows 90% memory usage with 6 users, causing slowdowns. Upgrade to Standard_E4s_v5 (32 GB RAM) reduces memory to 60% and improves responsiveness.

### F-Series (Compute-Optimized)

**Best For:** CPU-intensive apps, rendering, compilation

**Characteristics:**
- High CPU-to-RAM ratio (1 vCPU : 2 GB RAM)
- Optimized for single-threaded performance
- Less common for AVD (most workloads are memory-bound, not CPU-bound)

**When to Use:**
- **Development Workloads:** Visual Studio compilation, code building
- **Rendering:** 3D rendering, video transcoding (without GPU)

**Caution:** Most AVD workloads hit RAM limits before CPU limits. F-series is rarely cost-effective unless you have proven CPU bottlenecks.

### NV-Series (GPU-Accelerated)

**Best For:** CAD, 3D design, graphics-intensive applications

**Characteristics:**
- Includes NVIDIA GPU for hardware acceleration
- Supports DirectX, OpenGL, CUDA workloads
- Significantly more expensive (3-5x cost of D-series)

**Common Sizes:**

| Size | vCPU | RAM | GPU | Cost (East US, approx) | Use Case |
|------|------|-----|-----|------------------------|----------|
| **Standard_NV6ads_A10_v5** | 6 | 55 GB | 1/6 NVIDIA A10 | $0.722/hour | AutoCAD, SolidWorks, Adobe Creative Cloud |
| **Standard_NV12ads_A10_v5** | 12 | 110 GB | 1/3 NVIDIA A10 | $1.444/hour | Heavy CAD, 3D rendering |

**When to Use:**
- **CAD/CAM:** AutoCAD, Revit, SolidWorks, CATIA
- **Graphics Design:** Adobe Photoshop, Illustrator, Premiere Pro
- **3D Visualization:** Unreal Engine, Unity, 3D modeling

**Our Environment:**
We do NOT use GPU VMs currently. All users are light/medium office workers. If we onboard engineering team needing CAD, we'll create a separate personal host pool with NV-series VMs.

## Sizing Guidelines

The most common question: "How many users can I fit per session host?"

**Answer:** It depends on your workload profile.

### Light Users (Office Productivity)

**Workload:**
- Office 365 (Word, Excel, PowerPoint)
- Web browsers (Edge, Chrome)
- Email (Outlook)
- PDF viewers

**Characteristics:**
- Low CPU usage (5-10% per user)
- Moderate RAM usage (1.5-2 GB per user)
- Minimal disk I/O

**Sizing Formula:**

**vCPU:** 4-6 users per vCPU
**RAM:** 2 GB per user + 4 GB OS overhead

**Example:**

Standard_D4s_v5 (4 vCPU, 16 GB RAM)
- **vCPU Capacity:** 4 vCPU × 6 users = 24 users
- **RAM Capacity:** (16 GB - 4 GB OS) / 2 GB = 6 users
- **Bottleneck:** RAM (limits to 6 users)
- **Recommended Max Sessions:** 12 users (leave 20% headroom)

**Our Environment:**
- **VM Size:** Standard_D2s_v4 (2 vCPU, 8 GB RAM)
- **Formula:** (8 GB - 2 GB OS) / 1.5 GB = 4 users (RAM-bound)
- **Configured Max Sessions:** 10 (we're slightly oversubscribed, but monitoring shows it works)

### Medium Users (Multitasking)

**Workload:**
- Office 365 + Teams + Power BI
- Multiple browser tabs (10-20+)
- CRM/ERP applications (Salesforce, Dynamics)
- Light data analysis

**Characteristics:**
- Moderate CPU usage (10-20% per user)
- Higher RAM usage (2.5-3 GB per user)
- Moderate disk I/O

**Sizing Formula:**

**vCPU:** 2-4 users per vCPU
**RAM:** 3 GB per user + 4 GB OS overhead

**Example:**

Standard_D8s_v5 (8 vCPU, 32 GB RAM)
- **vCPU Capacity:** 8 vCPU × 4 users = 32 users
- **RAM Capacity:** (32 GB - 4 GB OS) / 3 GB = 9 users
- **Bottleneck:** RAM (limits to 9 users)
- **Recommended Max Sessions:** 12 users (with monitoring)

### Heavy Users (Power Users, Developers)

**Workload:**
- Visual Studio, SQL Server Management Studio
- Large Excel files (100+ MB)
- Virtual machines (nested Hyper-V, Docker)
- Data analysis (R, Python, MATLAB)

**Characteristics:**
- High CPU usage (30-50% per user)
- Very high RAM usage (4-8 GB per user)
- Heavy disk I/O

**Sizing Formula:**

**vCPU:** 1-2 users per vCPU
**RAM:** 6-8 GB per user + 4 GB OS overhead

**Example:**

Standard_E8s_v5 (8 vCPU, 64 GB RAM)
- **vCPU Capacity:** 8 vCPU × 2 users = 16 users
- **RAM Capacity:** (64 GB - 4 GB OS) / 6 GB = 10 users
- **Recommended Max Sessions:** 8 users (conservative for developers)

**Recommendation for Heavy Users:**
Use **Personal Host Pools** (1:1 user-to-VM ratio) instead of pooled. Developers and power users benefit from dedicated resources and persistence.

## Disk Types

Session host disk performance impacts boot time, logon speed, and application responsiveness.

### Premium SSD (Recommended)

**Characteristics:**
- Low latency (sub-10ms)
- High IOPS (up to 20,000 per disk)
- SLA-backed performance

**Sizes:**

| Size | IOPS | Throughput | Cost (East US, approx) |
|------|------|------------|------------------------|
| **P10 (128 GB)** | 500 | 100 MB/s | $19.71/month |
| **P20 (512 GB)** | 2,300 | 150 MB/s | $73.22/month |
| **P30 (1 TB)** | 5,000 | 200 MB/s | $135.17/month |

**When to Use:**
- All production AVD session hosts
- Faster boot times (30-45 seconds vs 60-90 seconds with Standard SSD)
- Better logon performance (10-15 seconds vs 20-30 seconds with Standard SSD)

**Our Environment:**
- **OS Disk:** Premium SSD P10 (128 GB)
- **Cost:** $19.71/month per VM
- **Performance:** Boot time 40 seconds, logon time 12 seconds

### Standard SSD

**Characteristics:**
- Moderate latency (sub-20ms)
- Lower IOPS (up to 6,000 per disk)
- 30-40% cheaper than Premium SSD

**When to Use:**
- Test/validation environments
- Cost-sensitive deployments with light workloads

**Not Recommended For:**
- Production AVD with >5 users per session host
- Workloads with heavy disk I/O (databases, video editing)

### Standard HDD

**Do NOT Use for AVD**

Standard HDD has unacceptable performance for AVD session hosts. Boot times exceed 2-3 minutes, logon times exceed 60 seconds, and users experience constant disk latency.

## Right-Sizing with Azure Monitor Metrics

Don't guess—use real metrics to optimize VM sizes.

### Key Metrics to Monitor

**1. CPU Utilization**

- **Target:** 40-70% average during peak hours
- **Underutilized:** <30% average = consider downsizing
- **Overutilized:** >80% sustained = consider upsizing or reducing max sessions

**2. Memory Utilization**

- **Target:** 50-75% average during peak hours
- **Underutilized:** <40% average = consider downsizing or D-series instead of E-series
- **Overutilized:** >85% sustained = consider upsizing or E-series for more RAM

**3. Disk IOPS**

- **Target:** <50% of disk IOPS limit
- **Overutilized:** Frequently hitting disk IOPS limit = upgrade to larger Premium SSD or add data disk

**4. Logon Duration**

- **Target:** <15 seconds for profile load + desktop ready
- **Slow:** >30 seconds = investigate FSLogix, disk I/O, or network latency

### Azure Monitor Query (CPU/RAM by Session Host)

```kusto
Perf
| where TimeGenerated > ago(7d)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where Computer startswith "avd-pool"
| summarize AvgCPU=avg(CounterValue) by Computer, bin(TimeGenerated, 1h)
| render timechart
```

### Rightsizing Decision Matrix

| Metric | Current | Action |
|--------|---------|--------|
| **CPU <30%, RAM <40%** | Underutilized | Downsize VM (e.g., D8s_v5 → D4s_v5) or increase max sessions |
| **CPU 40-70%, RAM 50-75%** | Optimal | No change needed |
| **CPU >80%, RAM <60%** | CPU-bound | Upsize VM (e.g., D4s_v5 → D8s_v5) or reduce max sessions |
| **CPU <60%, RAM >85%** | RAM-bound | Switch to E-series (e.g., D4s_v5 → E4s_v5) |
| **CPU >80%, RAM >85%** | Overloaded | Upsize VM urgently or reduce max sessions by 50% |

**Our Environment Example:**

Monitoring hp-pooled-prod1 for 2 weeks:
- **CPU Average:** 48% (peak 65%)
- **RAM Average:** 68% (peak 78%)
- **Logon Time:** 12 seconds
- **Decision:** Standard_D2s_v4 is appropriately sized, no changes needed

If we saw RAM consistently >80%, we'd either:
1. Reduce max sessions from 10 to 8
2. Upgrade to Standard_D4s_v4 (double RAM and vCPU)

## Configuration Options

| Setting | Options | Impact |
|---------|---------|--------|
| **VM Size** | D-series, E-series, F-series, NV-series | Determines vCPU, RAM, cost, and performance |
| **OS Disk Type** | Premium SSD, Standard SSD | Affects boot time, logon time, and responsiveness |
| **OS Disk Size** | 128 GB, 256 GB, 512 GB | Larger disks = higher IOPS (for Premium SSD) |
| **Temp Disk** | Included with VM size | Used for page file, not persistent (lost on reboot) |
| **Max Sessions** | 1-999999 | Limits concurrent users per session host |

## How to Configure

### Portal: Change VM Size

**Path:** Azure Portal → Virtual Machines → avd-pool-0 → Size → Resize

1. Stop VM (deallocate)
2. Select new size (e.g., Standard_D4s_v5)
3. Resize
4. Start VM
5. Update host pool max sessions if needed

**Downtime:** 5-10 minutes per VM

### CLI: Resize VM

```bash
# Stop VM
az vm deallocate --resource-group RG-Azure-VDI-01 --name avd-pool-0

# Resize
az vm resize --resource-group RG-Azure-VDI-01 --name avd-pool-0 --size Standard_D4s_v5

# Start VM
az vm start --resource-group RG-Azure-VDI-01 --name avd-pool-0
```

### Change OS Disk Type

```bash
# Stop VM
az vm deallocate --resource-group RG-Azure-VDI-01 --name avd-pool-0

# Get disk ID
DISK_ID=$(az vm show --resource-group RG-Azure-VDI-01 --name avd-pool-0 --query "storageProfile.osDisk.managedDisk.id" -o tsv)

# Update disk to Premium SSD
az disk update --ids $DISK_ID --sku Premium_LRS

# Start VM
az vm start --resource-group RG-Azure-VDI-01 --name avd-pool-0
```

## Best Practices

- **Start Small, Monitor, Adjust** - Begin with D4s_v5 or D2s_v5, monitor for 1-2 weeks, then right-size based on actual metrics; don't over-provision upfront
- **Consistent VM Sizes in Host Pool** - All session hosts in a host pool should be the same size to ensure predictable performance and avoid user complaints about inconsistent experience
- **Premium SSD for Production** - Always use Premium SSD for OS disks in production; the $20/month cost difference is negligible compared to poor user experience
- **Personal Host Pools for Power Users** - Don't try to fit developers and heavy users into pooled desktops; give them dedicated VMs (1:1 ratio) with larger sizes
- **Monitor Before Resizing** - Never resize based on complaints alone; collect 1-2 weeks of Azure Monitor data to confirm bottleneck (CPU vs RAM vs disk)
- **Test in Validation Host Pool** - Before changing production VM sizes, test new size in validation host pool with pilot users

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Slow logon times (>30 seconds)** | Disk I/O bottleneck (Standard SSD or HDD) | Upgrade to Premium SSD, check FSLogix profile disk performance |
| **Applications freezing or lagging** | CPU or RAM oversubscription | Check Azure Monitor for CPU >80% or RAM >90%, reduce max sessions or upsize VM |
| **Users report "out of memory" errors** | RAM exhausted (>95% utilization) | Reduce max sessions immediately, plan migration to E-series or larger VM |
| **Uneven performance across session hosts** | Mixed VM sizes in host pool | Standardize all session hosts to same size, redeploy mismatched VMs |
| **High costs despite scaling plan** | VM size too large for workload | Monitor CPU/RAM usage, downsize if consistently <30% utilization |
| **Cannot resize VM (grayed out)** | VM is running or size unavailable in region | Deallocate VM first, check available sizes with: az vm list-sizes --location eastus |

---

**Summary:** AVD session host sizing is a balance between performance and cost. Start with D-series for general users, monitor metrics, and adjust based on real-world data. Use Premium SSD for production, and remember: it's easier to upsize than downsize (downsizing requires user notification and downtime).