---
title: Monitoring Setup
description: 
published: true
date: 2025-12-14T04:52:35.858Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:29.938Z
---

# Monitoring Setup

Set up Log Analytics, AVD Insights, and diagnostics to monitor connection health, performance, and user experience.

## Pre-Deployment Checklist

- [ ] Log Analytics workspace created
- [ ] AVD Insights enabled
- [ ] Diagnostic settings configured on all AVD resources
- [ ] Data Collection Rules associated with session hosts
- [ ] Alert rules defined for critical events

## Step 1: Create Log Analytics Workspace

If you don't have an existing workspace, create one dedicated to AVD monitoring.

**Portal Path:** Azure Portal → Log Analytics workspaces → Create

**Configuration:**
| Setting | Value | Rationale |
|---------|-------|-----------|
| **Name** | `log-avd-prod` | Matches our naming convention |
| **Resource Group** | `RG-Azure-VDI-01` | Same RG as AVD resources |
| **Region** | `Central US` | Same region as session hosts |
| **Pricing Tier** | Pay-as-you-go | Switch to commitment tier if >100GB/day |

**Azure CLI:**
```bash
az monitor log-analytics workspace create \
  --resource-group RG-Azure-VDI-01 \
  --workspace-name log-avd-prod \
  --location centralus \
  --retention-time 30
```

## Step 2: Enable Diagnostic Settings on Host Pool

Diagnostic settings stream AVD telemetry (connections, errors, checkpoints) to Log Analytics.

**Portal Path:** Azure Portal → Host Pools → hp-pooled-prod1 → Diagnostic settings → Add diagnostic setting

**Configuration:**
| Setting | Value |
|---------|-------|
| **Diagnostic setting name** | `DiagToLogAnalytics` |
| **Log Categories** | Select ALL: Checkpoint, Connection, Error, Management, HostRegistration, AgentHealthStatus |
| **Destination** | Send to Log Analytics workspace: `log-avd-prod` |

**Repeat for:**
- hp-personal-prod1 (personal host pool)
- ws-avd-prod (workspace)
- hp-pooled-prod1-DAG (desktop app group)
- hp-personal-prod1-DAG (desktop app group)

**Azure CLI (Host Pool):**
```bash
az monitor diagnostic-settings create \
  --name DiagToLogAnalytics \
  --resource "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/hostpools/hp-pooled-prod1" \
  --workspace "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.OperationalInsights/workspaces/log-avd-prod" \
  --logs '[{"category":"Checkpoint","enabled":true},{"category":"Connection","enabled":true},{"category":"Error","enabled":true},{"category":"Management","enabled":true},{"category":"HostRegistration","enabled":true},{"category":"AgentHealthStatus","enabled":true}]'
```

## Step 3: Enable AVD Insights

AVD Insights is a pre-built workbook that provides dashboards for connection health, session host performance, and user experience metrics.

**Portal Path:** Azure Portal → Monitor → Insights → Azure Virtual Desktop

**First-Time Setup:**
1. Click "Open Insights Configuration Workbook"
2. Select subscription and resource group
3. Configure session hosts:
   - Select host pool: hp-pooled-prod1
   - Select Log Analytics workspace: log-avd-prod
   - Click "Configure" to deploy Data Collection Rules
4. Wait 10-15 minutes for initial data collection

**What AVD Insights Shows:**
- **Connection Reliability:** Success rate, failure reasons
- **Connection Performance:** Round-trip time, bandwidth
- **Session Host Health:** CPU, memory, disk, agent status
- **User Experience:** Input delay, frames per second

## Step 4: Deploy Azure Monitor Agent to Session Hosts

The Azure Monitor Agent (AMA) collects performance counters and Windows events from session hosts.

**Option A: Auto-Deploy via AVD Insights**
When you click "Configure" in AVD Insights (Step 3), it automatically deploys AMA and creates Data Collection Rules.

**Option B: Manual Deployment**

**Azure CLI:**
```bash
# Deploy AMA extension to each session host
az vm extension set \
  --resource-group RG-Azure-VDI-01 \
  --vm-name avd-pool-0 \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --enable-auto-upgrade true

# Associate with AVD Insights DCR
az monitor data-collection rule association create \
  --name avd-insights-association \
  --rule-id "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Insights/dataCollectionRules/microsoft-avdi-centralus" \
  --resource "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Compute/virtualMachines/avd-pool-0"
```

## Step 5: Create Essential Alert Rules

Proactive alerting prevents issues from impacting users.

**Recommended Alerts:**

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| **Connection Failures High** | WVDErrors > 10 in 15 min | Sev 2 | Email + Teams |
| **Session Host Unhealthy** | AgentHealthStatus = Unhealthy | Sev 1 | Email + PagerDuty |
| **High CPU on Session Host** | CPU > 90% for 10 min | Sev 3 | Email |
| **Profile Mount Failures** | FSLogix Error 30 | Sev 2 | Email + Ticket |

**Create Alert (Connection Failures):**

**Portal Path:** Azure Portal → Monitor → Alerts → Create alert rule

**Configuration:**
1. **Scope:** Select `log-avd-prod` workspace
2. **Condition:** Custom log search
   ```kql
   WVDErrors
   | where TimeGenerated > ago(15m)
   | summarize ErrorCount = count()
   | where ErrorCount > 10
   ```
3. **Actions:** Select action group (email, Teams webhook)
4. **Alert rule name:** `AVD-ConnectionFailures-High`

## Step 6: Verify Data Flow

Wait 15-30 minutes after configuration, then verify data is flowing.

**KQL Query - Check WVDConnections:**
```kql
WVDConnections
| where TimeGenerated > ago(1h)
| summarize count() by State
| render piechart
```

Expected: Shows "Connected", "Completed", and potentially "Failed" states.

**KQL Query - Check Performance Counters:**
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor"
| summarize avg(CounterValue) by Computer
```

Expected: Shows average CPU for each session host.

## Validation

- [ ] Log Analytics workspace shows data in WVDConnections table
- [ ] AVD Insights dashboard loads with connection and performance data
- [ ] Performance counters (CPU, memory) visible in Perf table
- [ ] Alert rule test email received
- [ ] Agent health status shows "Healthy" for all session hosts

## Post-Deployment Tasks

1. **Tune Alert Thresholds:** Adjust based on baseline after 7 days of data
2. **Create Custom Dashboards:** Pin useful queries to Azure Dashboard
3. **Configure Log Retention:** Extend to 90 days for compliance if required
4. **Document Runbooks:** Create troubleshooting procedures for common alerts

## Reference

- **Concept:** [[Log Analytics Workspace|Log Analytics]] - Workspace setup, KQL queries
- **Concept:** [[AVD Insights Azure Monitor Workbook|AVD Insights]] - Workbook configuration
- **Concept:** [[Azure Monitor Alerting|Alerting]] - Alert rules, action groups

## Next Step

→ Proceed to [Scaling & Optimization](scaling-optimization) to configure autoscaling and cost optimization.