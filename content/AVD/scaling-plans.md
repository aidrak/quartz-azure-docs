---
title: Scaling Plans
description: 
published: true
date: 2025-12-14T04:52:23.118Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:10.977Z
---

# Scaling Plans

Scaling plans (also called Autoscale) are the primary cost optimization tool for Azure Virtual Desktop. They automatically start and stop session hosts based on schedules and capacity thresholds, ensuring you only pay for compute resources when users actually need them. A well-designed scaling plan can reduce AVD compute costs by 40-60% without impacting user experience.

## What are Scaling Plans

A scaling plan is an automated schedule that controls when session host VMs in a host pool are powered on or off. It defines different operational phases throughout the day (ramp-up, peak, ramp-down, off-peak) and adjusts capacity and load balancing behavior for each phase.

**Key Concepts:**

- **Schedule-Based:** Actions trigger at specific times (e.g., start VMs at 7:00 AM, stop at 7:00 PM)
- **Capacity Thresholds:** Number or percentage of session hosts to keep running during each phase
- **Load Balancing Changes:** Can switch between breadth-first and depth-first based on demand
- **Drain Mode:** Gracefully stops accepting new sessions before shutting down VMs
- **Applies to Pooled Host Pools Only:** Personal host pools use Start VM on Connect instead

**Example from Our Environment:**

- **Scaling Plan:** sp-hp1-business-hours
- **Host Pool:** hp-pooled-prod1 (5 session hosts, 10 max sessions each)
- **Schedule:**
  - **Ramp-Up (7:00-9:00 AM):** Start 60% of hosts (3 VMs), breadth-first load balancing
  - **Peak (9:00 AM-5:00 PM):** Keep 80% of hosts running (4 VMs), depth-first load balancing
  - **Ramp-Down (5:00-7:00 PM):** Drain and stop VMs when sessions end, down to 40% (2 VMs)
  - **Off-Peak (7:00 PM-7:00 AM):** Keep only 20% running (1 VM), depth-first load balancing
- **Cost Savings:** $2,000/month by deallocating 60% of VMs during off-peak (16 hours/day × 5 VMs × $0.096/hour)

## Schedule Phases

Scaling plans divide the day into four phases, each with different capacity and load balancing behavior.

### Ramp-Up Phase

**When:** Morning hours when users start logging in (e.g., 7:00-9:00 AM)

**Purpose:** Gradually increase capacity to accommodate arriving users without over-provisioning

**Configuration Options:**

- **Start Time:** When phase begins (e.g., 7:00 AM)
- **Load Balancing Algorithm:** Breadth-first (distribute users across hosts) or Depth-first (fill hosts sequentially)
- **Minimum % Hosts:** Percentage of total session hosts to keep running (e.g., 60%)
- **Capacity Threshold:** Session limit before starting additional VMs (e.g., 75%)

**Example:**

Host Pool: 10 session hosts, 10 max sessions each (100 total capacity)

Ramp-Up Schedule:
- **Start Time:** 7:00 AM
- **Minimum % Hosts:** 50% (5 VMs)
- **Capacity Threshold:** 70% (35 sessions out of 50 available)
- **Load Balancing:** Breadth-first

**What Happens:**

- 7:00 AM: Scaling plan starts 5 VMs (50% of 10)
- Users log in, distributed evenly across 5 VMs (breadth-first)
- When 35 sessions are active (70% of 50 capacity), plan starts 6th VM
- Continues starting VMs as threshold is exceeded

**Reasoning:** Breadth-first during ramp-up ensures no single VM is overloaded as users arrive simultaneously.

### Peak Phase

**When:** Core business hours when most users are active (e.g., 9:00 AM-5:00 PM)

**Purpose:** Maintain maximum capacity and performance for full user load

**Configuration Options:**

- **Start Time:** When phase begins (e.g., 9:00 AM)
- **Load Balancing Algorithm:** Breadth-first or Depth-first
- **Minimum % Hosts:** Typically 80-100% during peak

**Example:**

Peak Schedule:
- **Start Time:** 9:00 AM
- **Minimum % Hosts:** 100% (all 10 VMs)
- **Load Balancing:** Depth-first

**What Happens:**

- 9:00 AM: All 10 VMs are running (100% capacity)
- New connections use depth-first (fill first VM to max, then next)
- If any VM is stopped manually, scaling plan restarts it within 5 minutes

**Reasoning:** Depth-first during peak consolidates users onto fewer VMs, allowing unused VMs to be identified for cost optimization later.

### Ramp-Down Phase

**When:** Evening hours when users start logging off (e.g., 5:00-7:00 PM)

**Purpose:** Gracefully reduce capacity as demand decreases, stopping VMs when sessions end

**Configuration Options:**

- **Start Time:** When phase begins (e.g., 5:00 PM)
- **Load Balancing Algorithm:** Typically depth-first to drain VMs
- **Minimum % Hosts:** Minimum to keep running after ramp-down (e.g., 20%)
- **Capacity Threshold:** Session limit before stopping VMs (e.g., 50%)
- **Stop Hosts When:** "Users log off" (wait for sessions to end) or "Force log off" (disconnect users after warning)

**Example:**

Ramp-Down Schedule:
- **Start Time:** 5:00 PM
- **Minimum % Hosts:** 20% (2 VMs)
- **Capacity Threshold:** 50%
- **Load Balancing:** Depth-first
- **Stop Hosts When:** Users log off (graceful drain)

**What Happens:**

- 5:00 PM: Scaling plan sets 8 VMs to drain mode (no new sessions)
- As users log off, sessions consolidate onto remaining 2 VMs (depth-first)
- When a VM reaches 0 sessions, scaling plan deallocates it
- By 7:00 PM, only 2 VMs remain running

**Reasoning:** Graceful drain prevents disconnecting active users while still reducing costs as demand drops.

### Off-Peak Phase

**When:** Overnight and early morning when minimal users are active (e.g., 7:00 PM-7:00 AM)

**Purpose:** Maintain minimal capacity for 24/7 access while maximizing cost savings

**Configuration Options:**

- **Start Time:** When phase begins (e.g., 7:00 PM)
- **Load Balancing Algorithm:** Depth-first (consolidate few users onto minimal VMs)
- **Minimum % Hosts:** Typically 10-20% (1-2 VMs)

**Example:**

Off-Peak Schedule:
- **Start Time:** 7:00 PM
- **Minimum % Hosts:** 10% (1 VM)
- **Load Balancing:** Depth-first

**What Happens:**

- 7:00 PM: Only 1 VM remains running
- If a late-night user connects and VM reaches max sessions, scaling plan starts 2nd VM
- 7:00 AM: Ramp-up phase begins, starting additional VMs

**Reasoning:** Minimal capacity reduces costs overnight while still allowing 24/7 access via Start VM on Connect.

## Load Balancing Changes Per Phase

Scaling plans can dynamically change load balancing algorithms throughout the day to optimize for performance or cost.

| Phase | Recommended Algorithm | Reasoning |
|-------|----------------------|-----------|
| **Ramp-Up** | Breadth-First | Distribute morning surge evenly, avoid overloading single VM |
| **Peak** | Depth-First | Consolidate users, identify underutilized VMs |
| **Ramp-Down** | Depth-First | Drain VMs sequentially, stop when empty |
| **Off-Peak** | Depth-First | Consolidate minimal users onto minimal VMs |

**Example Configuration:**

```
Ramp-Up (7:00-9:00 AM): Breadth-First
Peak (9:00 AM-5:00 PM): Depth-First
Ramp-Down (5:00-7:00 PM): Depth-First
Off-Peak (7:00 PM-7:00 AM): Depth-First
```

**Why This Works:**

- Morning: Users arrive in clusters, breadth-first prevents bottlenecks
- Midday: Depth-first consolidates steady-state load, unused VMs can be stopped manually if needed
- Evening: Depth-first drains VMs in order, stopping them as users leave
- Overnight: Depth-first keeps minimal footprint

## Capacity Thresholds

Capacity thresholds determine when scaling plans start additional VMs during ramp-up and peak phases.

**Threshold Types:**

1. **Percentage of Available Capacity:** Start new VM when X% of running sessions are in use
2. **Absolute Session Count:** Start new VM when X sessions are active

**Example:**

Host Pool: 5 VMs, 10 max sessions each
Ramp-Up: 60% minimum hosts (3 VMs), 75% capacity threshold

**Math:**

- 3 VMs × 10 sessions = 30 available sessions
- 75% of 30 = 22.5 sessions (round down to 22)
- When 23rd user connects, scaling plan starts 4th VM

**Best Practice:**

Set capacity threshold to 60-80%:
- **Too Low (e.g., 30%):** VMs start prematurely, increasing costs
- **Too High (e.g., 95%):** Performance degrades before new VM starts, impacting user experience
- **Sweet Spot (70-80%):** Balance between cost and performance

## Cost Savings Potential

Scaling plans deliver significant cost savings by deallocating VMs during low-demand periods.

**Example Calculation:**

Host Pool: 10 session hosts, Standard_D4s_v4 ($0.192/hour)

**Without Scaling Plan:**
- 10 VMs × 24 hours × 30 days × $0.192/hour = $1,382.40/month

**With Scaling Plan:**
- **Peak (9 AM-5 PM, 8 hours):** 10 VMs running = 10 × 8 = 80 VM-hours/day
- **Ramp-Up/Down (7-9 AM, 5-7 PM, 4 hours):** 6 VMs running average = 6 × 4 = 24 VM-hours/day
- **Off-Peak (7 PM-7 AM, 12 hours):** 2 VMs running = 2 × 12 = 24 VM-hours/day
- **Total Daily:** 80 + 24 + 24 = 128 VM-hours/day
- **Monthly Cost:** 128 × 30 days × $0.192/hour = $737.28/month

**Savings:** $1,382.40 - $737.28 = $645.12/month (47% reduction)

**Annual Savings:** $7,741.44/year

**Our Environment:**

- **Host Pool:** hp-pooled-prod1 (5 VMs, Standard_D2s_v4, $0.096/hour)
- **Monthly Cost Without Scaling:** 5 × 24 × 30 × $0.096 = $345.60
- **Monthly Cost With Scaling:** 128 VM-hours × $0.096 = $147.46
- **Savings:** $198.14/month ($2,377.68/year, 57% reduction)

## Configuration Options

| Setting | Options | Impact |
|---------|---------|--------|
| **Time Zone** | Any Windows time zone | Determines when schedule phases trigger |
| **Ramp-Up Start Time** | HH:MM (e.g., 07:00) | When morning phase begins |
| **Peak Start Time** | HH:MM (e.g., 09:00) | When peak phase begins |
| **Ramp-Down Start Time** | HH:MM (e.g., 17:00) | When evening phase begins |
| **Off-Peak Start Time** | HH:MM (e.g., 19:00) | When overnight phase begins |
| **Minimum % Hosts (per phase)** | 0-100% | Percentage of total session hosts to keep running |
| **Capacity Threshold (ramp-up/peak)** | 0-100% | Session load before starting additional VMs |
| **Load Balancing (per phase)** | Breadth-First, Depth-First | How users are distributed |
| **Stop Hosts When (ramp-down)** | Users log off, Force log off | Graceful drain vs forced disconnect |

## How to Configure

### Portal: Create Scaling Plan

**Path:** Azure Portal → Virtual Desktops → Scaling Plans → Create

1. **Basics:**
   - Subscription: (your subscription)
   - Resource Group: RG-Azure-VDI-01
   - Name: sp-hp1-business-hours
   - Location: East US
   - Friendly Name: "Business Hours Autoscale"
   - Time Zone: (UTC-05:00) Eastern Time (US & Canada)

2. **Schedule:**
   - **Ramp-Up:**
     - Start Time: 07:00
     - Load Balancing Algorithm: Breadth-First
     - Minimum % Hosts: 60%
     - Capacity Threshold: 75%
   - **Peak:**
     - Start Time: 09:00
     - Load Balancing Algorithm: Depth-First
     - Minimum % Hosts: 80%
   - **Ramp-Down:**
     - Start Time: 17:00
     - Load Balancing Algorithm: Depth-First
     - Minimum % Hosts: 20%
     - Capacity Threshold: 50%
     - Stop Hosts When: Users log off
   - **Off-Peak:**
     - Start Time: 19:00
     - Load Balancing Algorithm: Depth-First
     - Minimum % Hosts: 20%

3. **Host Pool Assignments:**
   - Add host pools: hp-pooled-prod1
   - Enable autoscale: Yes

4. **Review + Create**

### CLI: Create Scaling Plan

```bash
# Create scaling plan
az desktopvirtualization scaling-plan create \
  --name sp-hp1-business-hours \
  --resource-group RG-Azure-VDI-01 \
  --location eastus \
  --friendly-name "Business Hours Autoscale" \
  --time-zone "Eastern Standard Time" \
  --host-pool-type Pooled

# Add schedule (ramp-up phase)
az desktopvirtualization scaling-plan create \
  --name sp-hp1-business-hours \
  --resource-group RG-Azure-VDI-01 \
  --schedules '[{
    "name": "weekday-schedule",
    "daysOfWeek": ["Monday","Tuesday","Wednesday","Thursday","Friday"],
    "rampUpStartTime": {"hour": 7, "minute": 0},
    "rampUpLoadBalancingAlgorithm": "BreadthFirst",
    "rampUpMinimumHostsPct": 60,
    "rampUpCapacityThresholdPct": 75,
    "peakStartTime": {"hour": 9, "minute": 0},
    "peakLoadBalancingAlgorithm": "DepthFirst",
    "rampDownStartTime": {"hour": 17, "minute": 0},
    "rampDownLoadBalancingAlgorithm": "DepthFirst",
    "rampDownMinimumHostsPct": 20,
    "rampDownCapacityThresholdPct": 50,
    "rampDownStopHostsWhen": "ZeroSessions",
    "offPeakStartTime": {"hour": 19, "minute": 0},
    "offPeakLoadBalancingAlgorithm": "DepthFirst"
  }]'

# Assign to host pool
az desktopvirtualization scaling-plan create \
  --name sp-hp1-business-hours \
  --resource-group RG-Azure-VDI-01 \
  --host-pool-references '[{
    "hostPoolArmPath": "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/hostPools/hp-pooled-prod1",
    "scalingPlanEnabled": true
  }]'
```

## Best Practices

- **Start Conservative** - Begin with higher minimum % hosts (e.g., 40-60%) and gradually reduce as you understand usage patterns; abrupt capacity drops frustrate users
- **Align with Business Hours** - Survey users to understand actual login/logout times, don't assume 9-5; adjust phases to match real demand peaks
- **Weekend Schedules** - Create separate weekend schedules with lower minimum % hosts if weekend usage is minimal
- **Capacity Threshold Tuning** - Monitor session counts during ramp-up; if VMs start too early, increase threshold; if users experience slowness, decrease threshold
- **Graceful Ramp-Down** - Always use "Users log off" during ramp-down to avoid forcibly disconnecting active users; only use force log off with advance warning
- **Combine with Start VM on Connect** - Enable Start VM on Connect as a safety net; if scaling plan underestimates demand, users can still trigger additional VMs
- **Monitor Scaling Events** - Use Log Analytics to track scaling actions (VM starts/stops), correlate with user complaints, and adjust schedules

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **VMs not starting at scheduled time** | Scaling plan not assigned to host pool or disabled | Verify scaling plan shows host pool in assignments, check "Enable autoscale" is Yes |
| **VMs starting too early or too late** | Time zone mismatch between scaling plan and local time | Verify scaling plan time zone matches your business location (Azure Portal → Scaling Plan → Schedule) |
| **VMs not stopping during off-peak** | Users still logged in or minimum % hosts too high | Check active sessions via Azure Portal, reduce minimum % hosts, or enable force log off with warning |
| **Users report "No resources available" during ramp-up** | Capacity threshold too high, VMs not starting fast enough | Lower capacity threshold to 60-70%, increase minimum % hosts during ramp-up |
| **Costs not decreasing as expected** | Scaling plan not deallocating VMs (just stopping) | Verify "Stop Hosts When" is set to "Users log off" and VMs show "Deallocated" status (not "Stopped") |
| **Personal host pool not autoscaling** | Scaling plans only apply to pooled host pools | Use "Start VM on Connect" for personal host pools instead of scaling plans |

## Monitoring Scaling Plan Performance

**Key Metrics to Track:**

1. **Active Sessions per Phase:** Compare to capacity thresholds
2. **VM Start/Stop Events:** Ensure scaling actions align with schedule
3. **User Logon Time:** Detect if users wait for VMs to start during ramp-up
4. **Cost Reduction:** Compare monthly compute costs before/after scaling plan

**Log Analytics Query (Scaling Events):**

```kusto
WVDAutoscaleEvaluationPooled
| where TimeGenerated > ago(7d)
| where HostPoolName == "hp-pooled-prod1"
| summarize VMsStarted=countif(ScalingAction == "Start"), VMsStopped=countif(ScalingAction == "Deallocate") by bin(TimeGenerated, 1h)
| render timechart
```

---

**Next:** Proceed to "Session Host Sizing" to learn how to choose the right VM sizes and optimize capacity for your workloads.