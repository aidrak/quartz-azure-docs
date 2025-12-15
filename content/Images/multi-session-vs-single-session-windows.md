---
title: Multi-Session vs Single-Session Windows
description: 
published: true
date: 2025-12-14T04:53:08.228Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:25.843Z
---

# Multi-Session vs Single-Session Windows

Choosing between Windows 11 multi-session and single-session is one of the most critical architectural decisions in Azure Virtual Desktop planning. This choice impacts licensing costs, user experience, application compatibility, and infrastructure design. This page explains the technical differences, licensing requirements, performance characteristics, and decision criteria for selecting the right Windows edition for your AVD environment.

## Multi-Session Windows (Enterprise Multi-Session)

Windows 11 Enterprise multi-session is a unique SKU available exclusively in Azure (not available for on-premises or other cloud providers). It allows multiple users to log into a single Windows instance simultaneously, similar to Windows Server Remote Desktop Services, but with the full Windows 11 desktop experience and application compatibility.

### What It Is

Multi-session Windows is a modified version of Windows 11 Enterprise that removes the traditional single-user limitation. In standard Windows, only one interactive user can be logged in at a time (local console or RDP, not both). Multi-session Windows allows dozens of users to share the same OS instance, each with their own isolated session, user profile, and desktop environment.

**Technical Implementation:**
- Based on Windows 11 Enterprise (not Server)
- Kernel modifications enable concurrent interactive sessions
- Each user session runs in a separate Windows session (Session 1, 2, 3, etc.)
- Session 0 remains the system services session
- User sessions are isolated via Windows session management, not Hyper-V containers

**Marketplace Image:** `MicrosoftWindowsDesktop:Windows-11:win11-25h2-avd:latest`

**Our Image Definition:** `Win11_Multi_25H2_Gen2` (versions 1.0.0, 1.0.1, 1.0.2)

### Licensing Requirements

Multi-session Windows requires **per-user** licensing, not per-device. This is fundamentally different from traditional Windows licensing.

**Valid License Options:**
- **Microsoft 365 E3** or **E5** (includes Windows 11 Enterprise + Office + Teams)
- **Microsoft 365 F3** (for frontline workers, limited feature set)
- **Windows 11 Enterprise E3** or **E5** (without M365 Apps)
- **Windows 365** (includes AVD rights)

**Per-User, Not Per-Device:**
- A user with an M365 E3 license can access AVD from any device (corporate PC, home laptop, iPad, etc.)
- Unlimited concurrent devices per user (can be logged into office desktop + AVD simultaneously)
- Each user requires a license; the session host VM itself does not need a Windows license

**Cost Example:**
- 100 users each with M365 E3 ($36/user/month) = $3,600/month total licensing cost
- Session hosts run license-free (Azure just charges for VM compute/storage)
- If 100 users share 10 session hosts (avg 10 users per host), the Windows licensing is $0 for the VMs

**Validation:** Azure validates licensing via Azure Active Directory (Entra ID). Users must authenticate with Entra ID credentials that have an active M365/Windows E3+ license assigned.

### Use Case: Pooled Host Pools

Multi-session Windows is designed for **pooled host pools** where users do not have dedicated desktops.

**Pooled Host Pool Characteristics:**
- **Non-persistent:** Users are assigned to any available session host at login (load balancing)
- **Shared resources:** Multiple users share CPU, RAM, and disk on the same VM
- **Stateless sessions:** User profiles are stored externally (FSLogix on Azure Files/NetApp), not locally
- **Cost-optimized:** High user density (10-20 users per session host) reduces per-user infrastructure cost

**Ideal User Personas:**
- **Task workers:** Call center agents, retail staff, data entry clerks
- **Shift workers:** Nurses, factory floor supervisors (multiple shifts sharing resources)
- **General office workers:** Standard productivity apps (M365, web browsers, light LOB apps)
- **Temporary/seasonal workers:** No need for dedicated desktops

**Not Suitable For:**
- **Power users:** Developers, video editors (need dedicated resources)
- **Persistent state requirements:** Applications that store data locally instead of centralized storage
- **Incompatible apps:** Some legacy apps don't support multi-user mode (see Application Compatibility section)

### Performance Characteristics

Multi-session Windows performance depends heavily on **user density** (users per session host) and **workload intensity**.

**Typical User Density:**
- **Light workload** (email, web, Office): 15-20 users per D4s_v5 (4 vCPU, 16GB RAM)
- **Medium workload** (CRM, ERP, moderate Excel/PowerBI): 10-12 users per D4s_v5
- **Heavy workload** (graphics, data analysis, development): 5-8 users per D8s_v5 (8 vCPU, 32GB RAM)

**Resource Contention:**
- **CPU:** Most critical resource. High contention causes slowdowns, UI lag, and poor user experience. Monitor `\Processor(_Total)\% Processor Time` in Azure Monitor.
- **RAM:** Windows multi-session aggressively shares memory via page file. 16GB can support 15 users if apps are memory-efficient. Heavy apps (Chrome with 50 tabs, PowerBI) require more RAM.
- **Disk IOPS:** Shared OS disk can bottleneck during "login storms" (many users logging in simultaneously at 8:00 AM). Use Premium SSD or Ultra Disk for OS drives.
- **Network:** 100+ users on a single VNet subnet can saturate network bandwidth. Use accelerated networking on session hosts.

**Optimization Strategies:**
- **Autoscaling:** Scale session hosts based on active sessions (e.g., 2 hosts during off-hours, 10 hosts during peak)
- **VDOT (Virtual Desktop Optimization Tool):** Disable unnecessary services, scheduled tasks, and Windows features to reduce resource usage
- **FSLogix Profile Containers:** Offload profile storage to Azure Files (ZRS for redundancy)
- **Separate user data from OS disk:** Use D: (temporary disk) for pagefile, C: for OS only, Azure Files for profiles

**Performance Metrics (Per Session Host):**
- **CPU:** Aim for <70% average, <90% peak (sustained 100% causes user complaints)
- **RAM:** <80% utilization (above 80%, Windows aggressively pages to disk, slowing logins)
- **Disk Queue Length:** <10 (higher values indicate IOPS bottleneck)
- **Active Sessions:** Monitor with `quser` or AVD Insights dashboard

## Single-Session Windows (Enterprise)

Windows 11 Enterprise single-session is the standard desktop OS—only one interactive user can be logged in at a time. For AVD, this means each user gets a dedicated VM.

### What It Is

Single-session Windows is identical to the Windows 11 Enterprise that runs on corporate laptops and desktops. The only difference is that it's running in Azure as a virtual machine instead of on physical hardware.

**Key Difference from Multi-Session:**
- **One user per VM:** User A logs into VM-01, User B logs into VM-02 (no sharing)
- **Persistent or non-persistent:** Can be configured as either (personal host pools are usually persistent)
- **Full local admin rights possible:** Since users don't share the VM, granting admin rights is lower risk
- **Better application compatibility:** All Windows apps work (no multi-user concerns)

**Marketplace Image:** `MicrosoftWindowsDesktop:Windows-11:win11-25h2-ent:latest`

**Our Image Definition:** `Win11_Single_25H2_Gen2` (version 1.0.0)

### Licensing Requirements

Single-session Windows in AVD uses the same per-user licensing as multi-session.

**Valid License Options:**
- Microsoft 365 E3/E5
- Windows 11 Enterprise E3/E5
- Windows 365

**No Per-Device Licensing:** Unlike traditional VDI solutions (VMware Horizon, Citrix), AVD does not support per-device licensing. Users must have per-user licenses even for single-session desktops.

**Cost Impact:**
- Same licensing cost as multi-session ($36/user/month for M365 E3)
- Higher infrastructure cost (1 VM per user vs. 1 VM per 10-15 users)

**Example:**
- 100 users on single-session AVD:
  - Licensing: $3,600/month (same as multi-session)
  - Infrastructure: 100 VMs x D4s_v5 (~$120/month each) = $12,000/month
- 100 users on multi-session AVD:
  - Licensing: $3,600/month (same)
  - Infrastructure: 10 VMs x D4s_v5 = $1,200/month
- **Cost Difference:** 10x higher infrastructure cost for single-session

### Use Case: Personal Host Pools

Single-session Windows is used for **personal host pools** where each user is assigned a specific VM.

**Personal Host Pool Characteristics:**
- **Persistent:** User A always logs into the same VM (VM-01)
- **Dedicated resources:** All CPU/RAM/disk resources belong to one user
- **Local data storage:** Users can save files locally (though cloud storage is recommended)
- **Customizable:** Users can install applications, change settings (if admin rights granted)

**Ideal User Personas:**
- **Executives/VIPs:** Require consistent experience, no shared resources, highest performance
- **Developers:** Need admin rights, install SDKs, run VMs/containers locally
- **Power users:** Heavy Excel, AutoCAD, video editing (require dedicated GPU via NVv4/NCasT4)
- **Regulatory/Compliance:** Users who cannot share VMs due to data isolation requirements (HIPAA, PCI-DSS)
- **Persistent local state:** Applications that don't support profile redirection (legacy CAD, engineering software)

**Not Suitable For:**
- **Cost-sensitive deployments:** 10x infrastructure cost vs. multi-session
- **Highly scalable environments:** Managing 1000 personal VMs is complex
- **Task workers:** No need for dedicated resources

### Performance Characteristics

Single-session performance is predictable—each user gets 100% of the VM's resources.

**VM Sizing:**
- **Light users (email, Office):** D2s_v5 (2 vCPU, 8GB RAM) = ~$60/month
- **Standard users (Office, CRM, web):** D4s_v5 (4 vCPU, 16GB RAM) = ~$120/month
- **Power users (Excel, development):** D8s_v5 (8 vCPU, 32GB RAM) = ~$240/month
- **Graphics/CAD users:** NV6ads_A10_v5 (6 vCPU, 55GB RAM, 1/6 A10 GPU) = ~$500/month

**No Resource Contention:**
- User A's CPU usage does not affect User B (separate VMs)
- Predictable performance: if the VM has 4 vCPU, the user always has access to 4 vCPU
- Disk IOPS: Each VM has dedicated OS disk (Premium SSD recommended for best performance)

**Autoscaling Limitations:**
- **Start/stop on demand:** VMs can be deallocated when users log off (save 80% of compute cost)
- **Cannot oversubscribe:** With multi-session, 100 users might only use 50 active sessions at a time (natural scaling). Single-session requires 100 VMs ready at peak usage.

**Performance Metrics:**
- Same as physical PC: CPU, RAM, disk, network
- Easier to troubleshoot: high CPU = user's apps, not other users

## Comparison: Win11_Multi_25H2_Gen2 vs Win11_Single_25H2_Gen2

### Our Environment Images

**Win11_Multi_25H2_Gen2:**
- **OS:** Windows 11 Enterprise multi-session, version 25H2 (build 26100)
- **Versions:** 1.0.0, 1.0.1, 1.0.2 (monthly updates)
- **Deployed To:** hp-pooled-prod (pooled host pool, breadth-first load balancing)
- **User Count:** 150 users across 12 session hosts (avg 12-13 users per host)
- **VM Size:** Standard_D4s_v5 (4 vCPU, 16GB RAM)
- **Software:** FSLogix, M365 Apps (SharedComputerLicensing=1), Adobe Reader, VPN client
- **Use Case:** General office workers, call center agents, shift workers

**Win11_Single_25H2_Gen2:**
- **OS:** Windows 11 Enterprise, version 25H2 (build 26100)
- **Versions:** 1.0.0 (initial deployment)
- **Deployed To:** hp-personal-prod (personal host pool, direct assignment)
- **User Count:** 20 users (executives, developers)
- **VM Size:** D8s_v5 (8 vCPU, 32GB RAM) for most, NV6ads_A10_v5 for CAD users
- **Software:** Same base image as multi-session, but M365 Apps with SharedComputerLicensing=0
- **Use Case:** Executives needing dedicated resources, developers with admin rights

### Configuration Differences

**Image Template Differences:**

| Configuration | Multi-Session | Single-Session |
|---------------|---------------|----------------|
| **Marketplace SKU** | `win11-25h2-avd` | `win11-25h2-ent` |
| **M365 Apps Licensing** | `SharedComputerLicensing=1` | `SharedComputerLicensing=0` |
| **FSLogix Profile Containers** | Enabled (mandatory) | Optional (can use local profiles) |
| **VDOT Optimizations** | Aggressive (disable search, superfetch, etc.) | Moderate (preserve user features) |
| **Local Admin Rights** | Denied (shared VM security) | Granted (for developers/power users) |
| **Pagefile Location** | D: drive (temporary disk) | C: drive (OS disk) or none (RAM-only) |
| **Windows Update Policy** | Disabled (image updates only) | Enabled (WSUS/Intune managed) |

**Host Pool Differences:**

| Setting | Pooled (Multi-Session) | Personal (Single-Session) |
|---------|------------------------|---------------------------|
| **Host pool type** | Pooled | Personal |
| **Load balancing** | Breadth-first (spread users) | N/A (direct assignment) |
| **Max session limit** | 15 (per session host) | 1 (one user per VM) |
| **Assignment type** | Automatic (any available host) | Direct (User A → VM-01 always) |
| **Drain mode** | Yes (prevent new sessions) | N/A (dedicated VM) |
| **Autoscaling** | Based on active sessions | Based on user login/logoff |

## Application Compatibility Considerations

### Multi-Session Compatibility Challenges

Not all Windows applications were designed for multi-user environments. Some apps assume they are the only instance running and fail when multiple users launch them simultaneously.

**Common Issues:**

**Shared Temp Files:**
- **Problem:** App writes to `C:\Temp\app.lock` and crashes if file exists (created by another user)
- **Solution:** Configure app to use `%TEMP%` (user-specific temp folder) or `%LOCALAPPDATA%`

**Single-Instance Checks:**
- **Problem:** App uses a named mutex/semaphore to prevent multiple instances (e.g., "MyApp_Running")
- **Solution:** Use App-V or MSIX to containerize the app per-user session, or contact vendor for multi-user license

**Registry Conflicts:**
- **Problem:** App writes settings to `HKEY_LOCAL_MACHINE` (shared across users) instead of `HKEY_CURRENT_USER`
- **Solution:** Use FSLogix App Masking to redirect registry writes, or App-V to virtualize registry

**Hardcoded Paths:**
- **Problem:** App saves data to `C:\ProgramData\MyApp\userdata.db` (shared), causing data corruption
- **Solution:** Configure app to use `%APPDATA%\MyApp\` or contact vendor for fix

**Licensing Restrictions:**
- **Problem:** App license allows 1 concurrent user per Windows installation (terminal server licensing required)
- **Solution:** Purchase terminal server/VDI license from vendor (often 2-5x more expensive)

**Known Problematic Apps:**
- **QuickBooks Desktop:** Older versions (pre-2019) do not support multi-user mode. Use QuickBooks Online or purchase Enterprise license.
- **Sage 50 Accounting:** Requires separate license for each concurrent user on multi-session Windows.
- **AutoCAD (non-subscription):** Perpetual licenses don't support multi-session. Subscription licenses work via Autodesk licensing server.
- **Legacy LOB Apps:** Custom .NET/VB6 apps often hardcode paths or use shared resources improperly.

**Testing Procedure:**
1. Deploy app to multi-session test environment
2. Log in with 2 test users simultaneously
3. Both users launch the app and perform typical tasks
4. Check for errors, crashes, data conflicts
5. If issues occur, review app logs and contact vendor

### Single-Session Compatibility

Single-session Windows has 100% application compatibility—if it runs on a physical Windows 11 PC, it runs on AVD single-session.

**Use Single-Session For:**
- Apps that explicitly fail multi-user testing
- Vendor-unsupported scenarios (vendor says "only supported on single-user Windows")
- Legacy apps with no vendor support (can't fix compatibility issues)

**Trade-off:** Higher cost (dedicated VM per user) vs. guaranteed compatibility.

## Performance Differences in Practice

### Login Time Comparison

**Multi-Session:**
- **Cold boot (no active sessions):** 15-20 seconds (OS already running, just starting new session)
- **Warm login (other users active):** 10-15 seconds (session infrastructure already initialized)
- **Profile load (FSLogix):** 5-10 seconds (Azure Files SMB mount)
- **Total login time:** 20-30 seconds

**Single-Session:**
- **Cold boot (VM deallocated):** 60-90 seconds (VM start + OS boot + agent registration)
- **Warm login (VM running):** 15-20 seconds (RDP connection + profile load)
- **Profile load (local profile):** 2-5 seconds (no network mount)
- **Total login time (deallocated):** 75-95 seconds
- **Total login time (running):** 17-25 seconds

**Optimization:** Keep single-session VMs running during business hours (higher cost but better UX).

### Resource Utilization Under Load

**Multi-Session (12 users on D4s_v5):**
- **CPU:** 60-70% average, 85-90% during login storms
- **RAM:** 14GB / 16GB used (87%)
- **Disk IOPS:** 300-500 IOPS (Premium SSD P10 = 500 IOPS cap)
- **Network:** 50-80 Mbps (RDP protocol + FSLogix traffic)

**Single-Session (1 user on D4s_v5):**
- **CPU:** 10-20% average (lots of headroom)
- **RAM:** 6GB / 16GB used (37%)
- **Disk IOPS:** 50-100 IOPS (well below Premium SSD limits)
- **Network:** 5-10 Mbps (just RDP protocol)

**Conclusion:** Multi-session maximizes resource efficiency but requires careful capacity planning. Single-session is wasteful from a utilization perspective but provides guaranteed performance.

## Decision Matrix

| Criteria | Choose Multi-Session | Choose Single-Session |
|----------|----------------------|----------------------|
| **User Type** | Task workers, general office | Developers, power users, executives |
| **App Compatibility** | All apps tested and multi-user compatible | Legacy apps or vendor requires single-user |
| **Budget** | Cost-sensitive (optimize per-user infra cost) | Premium experience justified |
| **User Density** | High (100+ users) | Low (<50 users) |
| **Performance Requirements** | Acceptable to share resources | Dedicated resources required |
| **Customization** | Standardized environment (no local admin) | Users need admin rights, install apps |
| **Regulatory** | Standard compliance | Data isolation mandates (HIPAA, PCI) |
| **Scalability** | Highly scalable (autoscaling pools) | Limited scale (1 VM per user) |
| **Login Speed** | Fast (15-30 sec) | Moderate (20-95 sec depending on VM state) |
| **Management Complexity** | Lower (manage 10 VMs for 150 users) | Higher (manage 150 VMs for 150 users) |

## Hybrid Approach (Recommended)

Most organizations use **both** multi-session and single-session host pools to optimize cost and user experience.

**Example Architecture:**
- **Pooled Host Pool (Multi-Session):** 150 general office workers, 12 VMs, $1,200/month infra
- **Personal Host Pool (Single-Session):** 20 developers/executives, 20 VMs, $2,400/month infra
- **Total:** 170 users, $3,600/month infra (vs. $20,400 if all single-session)

**Assignment Logic:**
- Default: Assign all users to pooled host pool
- Exception requests: Users who demonstrate need (app incompatibility, performance issues) get escalated to personal pool
- Review quarterly: Users who no longer need personal desktops get migrated back to pooled

**Benefits:**
- 80% cost savings for majority of users
- Premium experience for users who need it
- Flexibility to adjust as requirements change

## Next Steps

- **[[azure-compute-gallery]]** - Managing separate image definitions for multi and single-session
- **[[golden-image-process]]** - Building both image types with appropriate configurations
- **[[image-versioning]]** - Version strategies for maintaining both image definitions