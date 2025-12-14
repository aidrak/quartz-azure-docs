---
title: Scaling & Optimization
description: 
published: true
date: 2025-12-14T04:52:42.370Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:36.373Z
---

# Scaling & Optimization

Configure autoscaling plans to automatically start/stop session hosts based on demand, reducing costs while maintaining user experience.

## Pre-Deployment Checklist

- [ ] Host pool deployed with multiple session hosts (minimum 2 for meaningful scaling)
- [ ] Session hosts registered and healthy
- [ ] Azure RBAC permissions for scaling (Desktop Virtualization Power On Off Contributor)
- [ ] Baseline usage patterns understood (peak hours, typical user count)

## Step 1: Create Scaling Plan

Scaling plans define when session hosts start/stop based on schedules and capacity thresholds.

**Portal Path:** Azure Portal → Azure Virtual Desktop → Scaling Plans → Create

**Configuration:**
| Setting | Value | Rationale |
|---------|-------|-----------|
| **Name** | `sp-hp1-business-hours` | Descriptive name |
| **Resource Group** | `RG-Azure-VDI-01` | Same RG as host pool |
| **Location** | `Central US` | Same region as host pool |
| **Friendly Name** | `Business Hours Autoscale` | User-friendly display name |
| **Time Zone** | `(UTC-06:00) Central Time` | Match your business timezone |
| **Host Pool Type** | `Pooled` | Scaling only works for pooled |

## Step 2: Configure Schedule Phases

A scaling schedule has four phases: Ramp-Up, Peak, Ramp-Down, Off-Peak.

**Our Business Hours Schedule:**

### Ramp-Up Phase (7:00 AM - 9:00 AM)
Users start logging in. Gradually increase capacity.

| Setting | Value |
|---------|-------|
| **Start Time** | 07:00 |
| **Load Balancing** | Breadth-First |
| **Minimum % Hosts** | 60% |
| **Capacity Threshold** | 75% |

### Peak Hours Phase (9:00 AM - 5:00 PM)
Maximum usage. Keep full capacity available.

| Setting | Value |
|---------|-------|
| **Start Time** | 09:00 |
| **Load Balancing** | Depth-First |
| **Minimum % Hosts** | 80% |

### Ramp-Down Phase (5:00 PM - 7:00 PM)
Users logging off. Gracefully reduce capacity.

| Setting | Value |
|---------|-------|
| **Start Time** | 17:00 |
| **Load Balancing** | Depth-First |
| **Minimum % Hosts** | 20% |
| **Capacity Threshold** | 50% |
| **Stop Hosts When** | Sessions = 0 (Users log off) |
| **Wait Time Before Stopping** | 15 minutes |

### Off-Peak Phase (7:00 PM - 7:00 AM)
Minimal usage. Keep minimum hosts for 24/7 access.

| Setting | Value |
|---------|-------|
| **Start Time** | 19:00 |
| **Load Balancing** | Depth-First |
| **Minimum % Hosts** | 20% |

## Step 3: Assign Scaling Plan to Host Pool

**Portal Path:** Scaling Plan → Host Pool Assignments → Add

**Configuration:**
| Setting | Value |
|---------|-------|
| **Host Pool** | hp-pooled-prod1 |
| **Enable Autoscale** | Yes |

**Azure CLI:**
```bash
# Create scaling plan
az desktopvirtualization scaling-plan create \
  --name sp-hp1-business-hours \
  --resource-group RG-Azure-VDI-01 \
  --location centralus \
  --friendly-name "Business Hours Autoscale" \
  --time-zone "Central Standard Time" \
  --host-pool-type Pooled

# Add schedule (simplified example)
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
az desktopvirtualization scaling-plan update \
  --name sp-hp1-business-hours \
  --resource-group RG-Azure-VDI-01 \
  --host-pool-references '[{
    "hostPoolArmPath": "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/hostPools/hp-pooled-prod1",
    "scalingPlanEnabled": true
  }]'
```

## Step 4: Configure RBAC for Scaling

The AVD scaling service needs permission to start/stop VMs. Assign built-in role to the AVD first-party app.

**Identify AVD First-Party App:**
- App ID: `9cdead84-a844-4324-93f2-b2e6bb768d07` (Windows Virtual Desktop)

**Portal Path:** Resource Group → Access Control (IAM) → Add role assignment

**Configuration:**
| Setting | Value |
|---------|-------|
| **Role** | Desktop Virtualization Power On Off Contributor |
| **Assign Access To** | User, group, or service principal |
| **Select** | Search for `9cdead84-a844-4324-93f2-b2e6bb768d07` |
| **Scope** | Resource Group: RG-Azure-VDI-01 |

**Azure CLI:**
```bash
az role assignment create \
  --assignee 9cdead84-a844-4324-93f2-b2e6bb768d07 \
  --role "Desktop Virtualization Power On Off Contributor" \
  --scope "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01"
```

## Step 5: Enable Start VM on Connect (Safety Net)

Start VM on Connect allows users to trigger VM startup when all session hosts are stopped.

**Portal Path:** Host Pool → Properties → Start VM on connect → Enable

**How It Works:**
1. User attempts to connect
2. All session hosts are deallocated (off-peak, scaling plan stopped them)
3. AVD service starts a session host on-demand
4. User waits 1-2 minutes while VM boots
5. Connection completes

**Best Practice:** Enable as safety net even with scaling plan. Prevents "No resources available" errors during unexpected off-hours access.

## Step 6: Create Weekend Schedule (Optional)

If weekend usage differs significantly, create a separate schedule.

**Weekend Schedule:**
| Phase | Start Time | Minimum % Hosts |
|-------|------------|-----------------|
| **Ramp-Up** | 09:00 | 20% |
| **Peak** | 10:00 | 40% |
| **Ramp-Down** | 16:00 | 20% |
| **Off-Peak** | 18:00 | 10% |

**Days of Week:** Saturday, Sunday

## Step 7: Monitor Scaling Events

Track scaling plan actions to validate behavior and troubleshoot issues.

**KQL Query - Scaling Events:**
```kql
WVDAutoscaleEvaluationPooled
| where TimeGenerated > ago(7d)
| where HostPoolName == "hp-pooled-prod1"
| project TimeGenerated, ScalingAction, SessionHostsStarted, SessionHostsStopped, ActiveSessionCount
| order by TimeGenerated desc
```

**KQL Query - Cost Savings Estimate:**
```kql
WVDAutoscaleEvaluationPooled
| where TimeGenerated > ago(30d)
| where HostPoolName == "hp-pooled-prod1"
| summarize TotalStopped = sum(SessionHostsStopped), TotalStarted = sum(SessionHostsStarted)
| extend EstimatedSavingsHours = TotalStopped * 8  // Assume 8 hours stopped per stop event
```

## Cost Optimization Summary

**Without Scaling (24/7 Running):**
- 5 VMs × 24 hours × 30 days × $0.096/hour = $345.60/month

**With Scaling (Business Hours):**
- Peak: 4 VMs × 8 hours × 22 days = 704 VM-hours
- Off-Peak: 1 VM × 16 hours × 22 days = 352 VM-hours
- Weekends: 1 VM × 48 hours × 8 days = 384 VM-hours
- Total: 1,440 VM-hours × $0.096 = $138.24/month

**Savings:** $207.36/month (60% reduction)

## Validation

- [ ] Scaling plan shows "Enabled" status
- [ ] RBAC role assigned to AVD first-party app
- [ ] Session hosts start/stop at scheduled times (verify next business day)
- [ ] Start VM on Connect works during off-peak hours
- [ ] AVD Insights shows scaling events in "Autoscale" tab

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| VMs not starting at scheduled time | RBAC role missing | Assign Desktop Virtualization Power On Off Contributor to AVD app |
| VMs not stopping during off-peak | Users still logged in | Verify "Stop hosts when: Sessions = 0" is configured |
| Users report "No resources available" | All VMs stopped, Start VM on Connect disabled | Enable Start VM on Connect on host pool |
| Scaling plan shows "Error" | Time zone mismatch or invalid schedule | Verify time zone matches region, check schedule for overlaps |

## Reference

- **Concept:** [[scaling-plans]] - Detailed configuration options
- **Concept:** [[session-host-sizing]] - VM sizing for cost optimization

## Variant Paths

If your deployment requires hybrid connectivity (VPN/ExpressRoute) or Intune management, proceed to the variant pages:

→ [Variant: Hybrid Connectivity](variant-hybrid-connectivity) - VPN Gateway, ExpressRoute, DNS
→ [Variant: Intune Integration](variant-intune-integration) - Device enrollment, configuration profiles