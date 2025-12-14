---
title: Step 06 - Host Pool Creation
description: Create AVD host pools with proper load balancing and registration token configuration
published: true
date: 2025-12-14
tags: [Quick-Deploy, AVD, host-pool]
---

# Step 06: Host Pool Creation

Create Azure Virtual Desktop host pools to define the container for session hosts. This step configures the host pool type, load balancing algorithm, and max session limits - but does NOT deploy session host VMs (that happens in Step 07).

## Example Scenario

Using naming conventions from [[00-naming-conventions]]:

| Host Pool | Type | Load Balancing | Max Sessions | Purpose |
|-----------|------|----------------|--------------|---------|
| `hp-pooled-prod` | Pooled | Breadth-first | 20 | General/Finance users (150 users) |
| `hp-personal-prod` | Personal | N/A | 1 | Creative/Executives (50 users) |

**Deployment Strategy:**
- Create pooled host pool first (serves majority of users)
- Configure registration token for VM join in Step 07
- Optionally create personal host pool for power users

## Prerequisites

- [ ] Resource group created: `rg-avd-prod-eastus-01`
- [ ] Storage account and file shares configured: `stavdprod01` with `profiles-pooled` and `profiles-personal` (from [[05-storage-fslogix]])
- [ ] VNET and subnet ready for session hosts: `vnet-avd-prod-eastus-01` with `snet-avd-prod-sessionhosts`
- [ ] Contributor or Desktop Virtualization Contributor role on resource group

> **Note:** This step only creates the host pool object. Session host VMs are deployed in [[07-session-host-deployment|Step 07]].

---

## Part 1: Create Pooled Host Pool

Create the pooled host pool for multi-session Windows 11 desktop access.

### Portal: Create Host Pool

**Portal:** Azure Portal → Virtual Desktop → Host pools → + Create

#### Basics Tab

1. **Project details:**
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Host pool name:** `hp-pooled-prod`
   - **Location:** East US
   - **Validation environment:** No

   > **Validation Environment:** Enable this on a separate host pool (e.g., `hp-pooled-validation`) to receive early Windows/AVD updates for testing. Leave disabled for production host pools.

2. **Host pool type:**
   - **Host pool type:** Pooled
   - **Load balancing algorithm:** Breadth-first
   - **Max session limit:** 20

3. Click **Next: Virtual machines**

#### Virtual Machines Tab

> **Important:** Do NOT add virtual machines during host pool creation. We will deploy session hosts separately in Step 07 using Azure Compute Gallery images.

1. **Add Azure virtual machines:** No

2. Click **Next: Workspace**

#### Workspace Tab

Skip workspace assignment for now - we'll create the workspace in [[08-workspace-app-groups|Step 08]] and assign app groups then.

1. **Register desktop app group:** No

2. Click **Next: Advanced**

#### Advanced Tab

1. **Friendly name:** `Pooled Production Desktop`
2. **Description:** `Multi-session pooled desktops for General and Finance users`
3. **Custom RDP properties:** (leave blank for now)
4. **Start VM on Connect:** Enabled

   > **Start VM on Connect:** Automatically starts deallocated session hosts when users connect. Reduces costs during off-peak hours. Requires RBAC permissions (configured automatically).

5. Click **Next: Tags**

#### Tags Tab

1. **Tags:**
   - **Environment:** Production
   - **Purpose:** AVD-HostPool-Pooled
   - **Owner:** IT-Operations
   - **CostCenter:** 1234

2. Click **Next: Review + create**

#### Review + Create

1. Review configuration:
   - Host pool type: Pooled
   - Load balancing: Breadth-first
   - Max session limit: 20
   - Start VM on Connect: Enabled

2. Click **Create**

**Deployment time:** 30-60 seconds

---

## Part 2: Generate Registration Token

Session hosts need a registration token to join the host pool. The token expires after a configured period (default 30 days).

### Portal: Create Registration Token

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-pooled-prod → Properties

1. Scroll to **Registration key** section
2. Click **Generate new key**
3. **Expiration date/time:**
   - Select: 24 hours from now

   > **Why 24 hours?** Short expiration limits security risk if token leaks. Generate new tokens as needed when adding session hosts.

4. Click **Generate**
5. **Copy the registration token** and save securely (you'll need it in Step 07)

**Registration Token Security:**
- Token provides unauthenticated access to join the host pool
- Keep token secret (do not commit to source control)
- Use short expiration (1-7 days max)
- Regenerate before adding new session hosts

> **Note:** Token expiration does NOT affect already-joined session hosts. It only prevents NEW VMs from joining.

### Verify Registration Token

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-pooled-prod → Properties

- [ ] **Registration key expires:** Shows date/time 24 hours from generation
- [ ] **Registration token operation:** Shows "Update"

**Copy token for later:**
```
Registration Token:
________________________________________________________
________________________________________________________
________________________________________________________
(save in secure location - needed in Step 07)
```

---

## Part 3: Configure Load Balancing

Understand and verify the load balancing configuration for pooled host pools.

### Load Balancing Algorithm Selection

| Algorithm | Behavior | When to Use | Cost Impact |
|-----------|----------|-------------|-------------|
| **Breadth-first** | Distributes users evenly across all session hosts | Consistent performance, high availability, peak hours | Higher (all hosts must run) |
| **Depth-first** | Fills first host to capacity before using next | Cost optimization, variable workloads, auto-shutdown | Lower (unused hosts can stop) |

**This deployment uses: Breadth-first**

**Rationale:**
- 150 pooled users during business hours (8am-6pm)
- Prioritize consistent performance over cost savings
- High availability (single host failure impacts fewer users)
- Session hosts sized appropriately (Standard_D4s_v5 = 20 sessions)

> **Alternative:** If cost is primary concern, switch to **Depth-first** and combine with [[../AVD/scaling-plans|Scaling Plans]] to auto-stop unused hosts during off-peak hours.

### Max Session Limit Sizing

The max session limit determines how many concurrent users can connect to each session host.

**Recommended limits by VM size:**

| VM Size | vCPUs | RAM | Max Sessions (Conservative) | Max Sessions (Aggressive) |
|---------|-------|-----|----------------------------|---------------------------|
| Standard_D2s_v5 | 2 | 8 GB | 4-6 | 8-10 |
| Standard_D4s_v5 | 4 | 16 GB | 10-15 | 16-20 |
| Standard_D8s_v5 | 8 | 32 GB | 20-25 | 30-40 |
| Standard_D16s_v5 | 16 | 64 GB | 40-50 | 60-80 |

**This deployment:**
- VM size: Standard_D4s_v5 (4 vCPU, 16 GB RAM)
- Max session limit: **20** (aggressive)
- Workload: Office 365, web apps, light business apps

**Calculation:**
- 150 users ÷ 20 sessions per host = **8 session hosts minimum**
- Recommended: **10 session hosts** (25% overhead for maintenance and peak load)

> **Best Practice:** Start conservative (15 sessions), monitor Azure Monitor CPU/RAM metrics for 2 weeks, then adjust upward if resources underutilized.

### Update Load Balancing (If Needed)

If you need to change load balancing algorithm after creation:

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-pooled-prod → Properties

1. **Load balancing algorithm:** Select Breadth-first or Depth-first
2. **Maximum session limit:** Adjust if needed (1-999999)
3. Click **Save**

> **Note:** Changing load balancing algorithm does NOT disrupt active sessions. New connections use the updated algorithm.

**See:** [[../AVD/host-pools-deep-dive#load-balancing-algorithms|Load Balancing Deep Dive]] for algorithm comparison and scaling strategies.

---

## Part 4: Verification

Confirm host pool created successfully with correct configuration.

### Verify Host Pool Configuration

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-pooled-prod → Properties

- [ ] **Host pool name:** hp-pooled-prod
- [ ] **Host pool type:** Pooled
- [ ] **Load balancing algorithm:** Breadth-first
- [ ] **Max session limit:** 20
- [ ] **Start VM on Connect:** Enabled
- [ ] **Validation environment:** No
- [ ] **Registration key:** Generated and not expired
- [ ] **Resource group:** rg-avd-prod-eastus-01
- [ ] **Location:** East US

### Verify Session Hosts Tab

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-pooled-prod → Session hosts

- [ ] Shows "No session hosts available" (expected - we haven't deployed VMs yet)

This confirms the host pool exists but has no session hosts. Session host deployment occurs in Step 07.

### Verify RBAC for Start VM on Connect

**Portal:** Azure Portal → Resource groups → rg-avd-prod-eastus-01 → Access Control (IAM) → Role assignments

- [ ] "Azure Virtual Desktop" service principal has "Desktop Virtualization Power On Contributor" role

> **Note:** This role assignment happens automatically when "Start VM on Connect" is enabled during host pool creation.

If missing, add manually:

**Portal:** Resource group → Access Control (IAM) → + Add → Add role assignment

1. **Role:** Desktop Virtualization Power On Contributor
2. **Assign access to:** User, group, or service principal
3. **Members:** Search for "Azure Virtual Desktop" (service principal)
4. Click **Review + assign**

---

## Part 5: Personal Host Pool (Optional)

Create a personal host pool for users requiring dedicated, persistent VMs.

> **Decision Point:**
> - **Need personal desktops?** Proceed with this section
> - **Pooled only?** Skip to Verification

### Portal: Create Personal Host Pool

**Portal:** Azure Portal → Virtual Desktop → Host pools → + Create

#### Basics Tab

1. **Project details:**
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Host pool name:** `hp-personal-prod`
   - **Location:** East US
   - **Validation environment:** No

2. **Host pool type:**
   - **Host pool type:** Personal
   - **Assignment type:** Automatic

   > **Assignment Type:**
   > - **Automatic:** First available unassigned VM given to user on first connection (recommended)
   > - **Direct:** Administrator manually assigns specific users to specific VMs

3. Click **Next: Virtual machines**

#### Virtual Machines Tab

1. **Add Azure virtual machines:** No

2. Click **Next: Workspace**

#### Workspace Tab

1. **Register desktop app group:** No

2. Click **Next: Advanced**

#### Advanced Tab

1. **Friendly name:** `Personal Production Desktop`
2. **Description:** `Dedicated single-session desktops for Creative and Executive users`
3. **Start VM on Connect:** Enabled
4. Click **Next: Tags**

#### Tags Tab

1. **Tags:**
   - **Environment:** Production
   - **Purpose:** AVD-HostPool-Personal
   - **Owner:** IT-Operations

2. Click **Review + create**
3. Click **Create**

### Generate Personal Host Pool Registration Token

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-personal-prod → Properties

1. Scroll to **Registration key** section
2. Click **Generate new key**
3. **Expiration date/time:** 24 hours from now
4. Click **Generate**
5. **Copy the registration token** and save securely

### Verify Personal Host Pool

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-personal-prod → Properties

- [ ] **Host pool type:** Personal
- [ ] **Assignment type:** Automatic
- [ ] **Start VM on Connect:** Enabled
- [ ] **Registration key:** Generated and valid

---

## Troubleshooting

### Issue: "Start VM on Connect" Not Working

**Symptom:** Users connect, but deallocated VMs do not start automatically

**Cause:**
- Missing RBAC role on resource group
- Service principal not granted permissions

**Fix:**

1. Verify service principal permissions:

**Portal:** Resource groups → rg-avd-prod-eastus-01 → Access Control (IAM) → Role assignments

2. Search for "Azure Virtual Desktop" service principal
3. Verify it has "Desktop Virtualization Power On Contributor" role
4. If missing, add role assignment:
   - **Role:** Desktop Virtualization Power On Contributor
   - **Members:** Azure Virtual Desktop (service principal)

5. Test by stopping a session host and attempting to connect

### Issue: Cannot Generate Registration Token

**Symptom:** "Generate new key" button grayed out or fails

**Cause:**
- Insufficient RBAC permissions
- Host pool deployment incomplete

**Fix:**

1. Verify permissions:
   - Contributor OR Desktop Virtualization Contributor role required

2. Check host pool status:

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-pooled-prod → Overview

3. Verify "Provisioning state" = Succeeded
4. If failed, delete and recreate host pool

### Issue: Registration Token Expired

**Symptom:** Session hosts fail to join host pool with error "Registration token invalid"

**Cause:**
- Token expiration date passed
- Token regenerated (invalidates old token)

**Fix:**

1. Generate new registration token:

**Portal:** Host pools → hp-pooled-prod → Properties → Generate new key

2. Copy new token
3. Re-run session host deployment with new token (Step 07)

> **Note:** Existing session hosts are NOT affected by token expiration. Only new VMs joining the pool need valid tokens.

### Issue: Wrong Load Balancing Configured

**Symptom:** Users not distributed as expected (all on one host or spread too thin)

**Cause:**
- Wrong load balancing algorithm selected during creation

**Fix:**

1. Update algorithm:

**Portal:** Host pools → hp-pooled-prod → Properties

2. Change **Load balancing algorithm** to desired setting
3. Click **Save**
4. New connections use updated algorithm immediately
5. Existing sessions continue on current hosts

**See:** [[../AVD/host-pools-deep-dive#common-issues|Host Pool Troubleshooting]] for advanced issues.

---

## Next Steps

**Host pool created.** Registration token ready for session host deployment.

**Next:** [[07-session-host-deployment|Step 07: Session Host Deployment]]

In Step 07, you will:
- Deploy session host VMs using Azure Compute Gallery images
- Join VMs to host pool using registration token
- Install AVD agent and FSLogix components
- Configure Entra ID join and Intune enrollment

---

## Variants

### Variant 1: Depth-First Load Balancing with Scaling Plans

If cost optimization is priority over consistent performance:

1. Change load balancing algorithm to **Depth-first**
2. Create scaling plan to auto-stop unused hosts:

**Portal:** Virtual Desktop → Scaling plans → + Create

3. Configure schedule:
   - **Peak hours (8am-6pm):** Minimum 6 hosts running
   - **Off-peak hours (6pm-8am):** Minimum 2 hosts running
   - **Weekends:** Minimum 1 host running

4. Assign scaling plan to host pool

**Cost savings:** 60-70% reduction in compute costs during off-peak hours

**See:** [[../AVD/scaling-plans|Scaling Plans]] for detailed configuration.

### Variant 2: Validation Host Pool

Create a separate validation host pool for testing updates:

1. Create host pool: `hp-pooled-validation`
2. **Validation environment:** Yes
3. Deploy 1-2 session hosts (same config as production)
4. Assign pilot users (IT admins, power users)
5. Monitor for issues before production updates

**Benefits:**
- Early access to Windows/AVD updates (1-2 weeks ahead)
- Test golden image changes without production impact
- Validate application compatibility

### Variant 3: Multi-Region Deployment

For disaster recovery or geo-distributed users:

1. Create host pools in secondary region:
   - `hp-pooled-prod-westus` (West US)
   - `hp-personal-prod-westus` (West US)

2. Replicate Azure Compute Gallery images to West US

3. Configure storage account replication (GRS or ZRS)

4. Assign users to regional workspaces based on location

**See:** [[../Operations/disaster-recovery|Disaster Recovery]] for multi-region architecture.

---

## Related Reference Pages

- [[../AVD/host-pools-deep-dive|Host Pools Deep Dive]] - Load balancing algorithms, max session sizing, validation environments
- [[../AVD/avd-components-overview|AVD Components Overview]] - Architecture, connection flow, Microsoft vs customer responsibilities
- [[../AVD/scaling-plans|Scaling Plans]] - Auto-start/stop session hosts based on schedules or demand
- [[../Operations/capacity-planning|Capacity Planning]] - VM sizing, user density calculations, growth planning
