---
title: Host Pools Deep Dive
description: 
published: true
date: 2025-12-14T04:52:21.505Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:09.267Z
---

# Host Pools Deep Dive

A host pool is the foundational building block of Azure Virtual Desktop. It's a collection of one or more identical session hosts (VMs) that serve desktops or applications to users. Understanding host pool types, load balancing algorithms, and configuration options is essential for designing AVD environments that meet performance, cost, and user experience requirements.

## What is a Host Pool

A host pool is a logical grouping of session hosts that share the same configuration, including:

- **OS Image:** All session hosts in a host pool must run the same operating system image (Windows 10/11 Multi-Session or Windows Server 2019/2022)
- **VM Size:** All session hosts should be identically sized (same SKU) to ensure consistent performance
- **Network Configuration:** All session hosts are typically in the same VNET/subnet for simplicity
- **Load Balancing Algorithm:** Determines how users are distributed across session hosts
- **Max Session Limit:** Maximum number of concurrent user sessions allowed per session host

When a user connects to AVD, the Connection Broker assigns them to a session host within the appropriate host pool based on the load balancing algorithm and current session counts.

**Example from Our Environment:**

- **Host Pool:** hp-pooled-prod1
- **Type:** Pooled
- **Load Balancing:** DepthFirst
- **Max Sessions:** 10 per session host
- **Session Hosts:** avd-pool-0, avd-pool-1, avd-pool-2 (Standard_D2s_v4, Entra Joined)

This configuration means users connecting to hp-pooled-prod1 will be assigned to the first available session host until it reaches 10 sessions, then the broker moves to the next host.

## Pooled vs Personal Host Pools

Choosing between pooled and personal host pools is the most important architectural decision in AVD.

### Pooled Host Pools

**Use Case:** Task workers, call centers, shift workers, general office productivity

**Characteristics:**
- **Shared Resources:** Multiple users share the same session host VMs
- **Stateless Sessions:** Users do not get the same session host each time (unless using drain mode for maintenance)
- **Cost-Effective:** Maximizes VM utilization by sharing compute resources
- **Non-Persistent:** Local changes (outside profile) are lost at logoff unless captured in FSLogix profile or redirected to network storage
- **Profile Management Required:** FSLogix or similar to preserve user settings and data across sessions

**Pros:**
- Lower cost per user (10-20 users per VM depending on workload)
- Easier to scale (add/remove VMs as needed)
- Simplified patching (update golden image, redeploy VMs)

**Cons:**
- Performance impact if users run resource-intensive apps
- Requires profile management solution
- Limited user customization (no admin rights typically)

**Our Example:**
- **Host Pool:** hp-pooled-prod1
- **Scenario:** 50 office workers accessing Office 365, web apps, and shared line-of-business applications
- **Configuration:** 5 session hosts × 10 max sessions = 50 concurrent user capacity
- **Load Balancing:** DepthFirst to consolidate users and allow auto-shutdown of unused hosts during off-peak

### Personal Host Pools

**Use Case:** Developers, power users, executives, persistent workstations

**Characteristics:**
- **Dedicated VMs:** Each user gets their own session host
- **Persistent Assignment:** User always connects to the same VM (like a traditional VDI desktop)
- **Stateful:** Users can install software, customize settings, and maintain local data
- **No Profile Management Required:** All data and settings are stored on the VM itself (though FSLogix can still be used for Office 365 cache)

**Assignment Types:**
- **Automatic:** Broker assigns the first available unassigned VM to a user on first connection
- **Direct:** Administrator manually assigns specific users to specific VMs

**Pros:**
- Full Windows experience with admin rights (if granted)
- Users can install applications and maintain local state
- Predictable performance (no resource sharing)

**Cons:**
- Higher cost (1:1 user-to-VM ratio)
- More VMs to manage, patch, and monitor
- Underutilization risk (VM runs even when user is not connected)

**Our Example:**
- **Host Pool:** hp-personal-prod1
- **Scenario:** 10 developers needing admin rights, Visual Studio, Docker, and local development environments
- **Configuration:** 10 session hosts, Automatic assignment (first-come, first-served)
- **Load Balancing:** Not applicable (users assigned to specific VMs)

## Load Balancing Algorithms

Load balancing only applies to **pooled** host pools. It determines how the Connection Broker distributes users across available session hosts.

### Breadth-First (Horizontal Scaling)

**Behavior:** Broker spreads user connections evenly across all available session hosts

**Example Scenario:**
- 3 session hosts, max 10 sessions each
- First 3 users: Each gets their own session host (1, 1, 1)
- Next 3 users: Round-robin continues (2, 2, 2)
- Result: All hosts have equal load

**When to Use:**
- **Consistent Performance:** Users get more resources per session because load is distributed
- **High Availability:** Failure of one host impacts fewer users
- **Peak Hours:** All hosts are active and serving users

**Cost Implication:**
All session hosts must be running during business hours, so auto-shutdown is less effective.

**Our Environment:**
We do NOT use breadth-first in production because it prevents cost savings through auto-shutdown.

### Depth-First (Vertical Scaling)

**Behavior:** Broker fills the first session host to max capacity before moving to the next

**Example Scenario:**
- 3 session hosts, max 10 sessions each
- First 10 users: All on session host 1 (10, 0, 0)
- Next 10 users: All on session host 2 (10, 10, 0)
- Result: Hosts filled sequentially

**When to Use:**
- **Cost Optimization:** Unused session hosts can be deallocated during off-peak hours
- **Scaling Plans:** Combine with autoscale to start/stop VMs based on demand
- **Variable Workloads:** Not all users are active simultaneously

**Cost Implication:**
If you have 50 users but only 20 are logged in, depth-first might only need 2 session hosts running, allowing you to stop the other 3.

**Our Environment:**
- **Host Pool:** hp-pooled-prod1 uses DepthFirst
- **Reasoning:** Combined with scaling plan sp-hp1-business-hours, we shut down 60% of session hosts during off-peak hours, saving $2000/month in compute costs

### Choosing the Right Algorithm

| Factor | Breadth-First | Depth-First |
|--------|---------------|-------------|
| **Performance** | Better (lower per-host load) | Variable (first host heavily loaded) |
| **Cost** | Higher (all hosts run) | Lower (unused hosts can be stopped) |
| **Scaling** | Horizontal (add more hosts) | Vertical (fill existing hosts first) |
| **Failure Impact** | Lower (fewer users per host) | Higher (losing first host impacts more users) |
| **Best For** | Predictable workloads, peak performance | Variable workloads, cost optimization |

**Decision Matrix:**
- **Call Center (200 users, 8am-5pm):** Breadth-first with all hosts running during shift
- **Office Workers (100 users, flexible hours):** Depth-first with scaling plan to stop unused hosts
- **Developers (20 users, inconsistent schedules):** Personal host pool (no load balancing)

## Validation Host Pools

A validation host pool is a special configuration used for testing Windows Updates, AVD agent updates, or golden image changes before deploying to production.

**How it Works:**
- Enable "Validation Environment" property on a host pool
- Microsoft releases updates to validation host pools 1-2 weeks before general availability
- Assign a small group of pilot users to test the environment

**Configuration:**

```bash
az desktopvirtualization hostpool update \
  --name hp-pooled-validation \
  --resource-group RG-Azure-VDI-01 \
  --validation-environment true
```

**Best Practice:**
Maintain a separate validation host pool with 1-2 session hosts mirroring your production configuration. Assign IT staff and power users to this pool to catch issues before they impact production.

**Our Environment:**
We do not currently have a validation host pool, but plan to create hp-pooled-validation with 2 session hosts once we scale beyond 100 users.

## Configuration Options

| Setting | Options | Impact |
|---------|---------|--------|
| **Host Pool Type** | Pooled, Personal | Determines if VMs are shared or dedicated |
| **Load Balancing** | Breadth-First, Depth-First | (Pooled only) Affects cost and performance |
| **Max Session Limit** | 1-999999 | (Pooled only) Limits users per session host |
| **Assignment Type** | Automatic, Direct | (Personal only) How users are assigned to VMs |
| **Validation Environment** | True, False | Receives early updates for testing |
| **Preferred App Group Type** | Desktop, RailApplications | Hint for what type of app groups will be assigned |
| **Start VM on Connect** | Enabled, Disabled | Automatically starts deallocated VMs when user connects |

## How to Configure

### Portal: Create a Host Pool

**Path:** Azure Portal → Virtual Desktops → Host Pools → Create

1. **Basics:**
   - Subscription: (your subscription)
   - Resource Group: RG-Azure-VDI-01
   - Host Pool Name: hp-pooled-prod1
   - Location: East US
   - Validation Environment: No
   - Host Pool Type: Pooled
   - Load Balancing Algorithm: Depth-First
   - Max Session Limit: 10

2. **Virtual Machines:**
   - Add virtual machines: Yes
   - Resource Group: RG-Azure-VDI-01
   - Name Prefix: avd-pool
   - Virtual Machine Location: East US
   - Virtual Machine Size: Standard_D2s_v4
   - Number of VMs: 3
   - OS Disk Type: Premium SSD
   - Boot Diagnostics: Enable with managed storage account

3. **Networking:**
   - Virtual Network: vnet-avd-prod
   - Subnet: snet-sessionhosts
   - Public IP: No

4. **Domain to Join:**
   - Select which directory to join: Microsoft Entra ID
   - Enroll VM with Intune: Yes
   - (If using AD DS, specify domain, OU, and credentials)

5. **Workspace:**
   - Register desktop app group: Yes
   - To this workspace: ws-avd-prod

6. **Review + Create**

### CLI: Create Host Pool

```bash
# Create host pool
az desktopvirtualization hostpool create \
  --name hp-pooled-prod1 \
  --resource-group RG-Azure-VDI-01 \
  --location eastus \
  --host-pool-type Pooled \
  --load-balancer-type DepthFirst \
  --max-session-limit 10 \
  --preferred-app-group-type Desktop \
  --start-vm-on-connect true

# Generate registration token (valid 30 days)
az desktopvirtualization hostpool update \
  --name hp-pooled-prod1 \
  --resource-group RG-Azure-VDI-01 \
  --registration-info expiration-time="$(date -u -d '+30 days' '+%Y-%m-%dT%H:%M:%S.000Z')" registration-token-operation="Update"
```

## Best Practices

- **Max Session Limit** - Set conservatively based on VM size: 2 vCPU = 4-8 sessions, 4 vCPU = 10-16 sessions, 8 vCPU = 20-32 sessions; monitor CPU/RAM and adjust
- **Start VM on Connect** - Enable this feature to allow deallocated VMs to auto-start when users connect, reducing costs while maintaining availability (adds 2-3 minute wait for first user)
- **Consistent VM Sizes** - Use identical VM SKUs within a host pool to avoid performance variability; if you need different sizes, create separate host pools
- **Registration Token Expiration** - Tokens expire after the configured period (default 30 days); generate new tokens before adding session hosts or joining VMs to the host pool
- **Naming Conventions** - Use descriptive names: hp-{workload}-{type}-{environment} (e.g., hp-pooled-prod1, hp-developers-personal-prod, hp-callcenter-pooled-val)
- **Drain Mode** - Use drain mode (prevent new sessions) when patching or troubleshooting a session host without disconnecting existing users

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Session hosts not appearing in host pool** | Registration token expired or invalid | Generate new token in Azure Portal → Host Pool → Properties → Registration Token, re-run AVD agent installer on session hosts |
| **"No available resources" despite VMs running** | All session hosts at max session limit or in drain mode | Increase max session limit, disable drain mode, or add more session hosts |
| **Users getting disconnected frequently** | Session host at capacity (CPU/RAM exhausted) | Reduce max session limit per host or scale up to larger VM size (e.g., D2s_v4 → D4s_v4) |
| **Load balancing not working as expected** | Host pool set to personal instead of pooled | Verify host pool type in Azure Portal, cannot change type after creation (must recreate) |
| **Personal host pool not assigning VMs** | Assignment type set to Direct instead of Automatic | Change to Automatic in host pool settings, or manually assign users via session host properties |
| **Start VM on Connect not working** | Missing RBAC permissions for AVD service principal | Grant "Desktop Virtualization Power On Contributor" role to AVD service principal on resource group |

---

**Next:** Proceed to "Application Groups" to learn how to publish desktops and apps to users.