---
title: Step 11 - Go-Live & Monitoring
description: Configure monitoring, alerts, and scaling plans before production go-live
published: true
date: 2025-12-14
tags:
editor: markdown
---

# Step 11: Go-Live & Monitoring Setup

Configure monitoring before production go-live. This step creates a Log Analytics workspace, enables diagnostic settings on AVD resources, configures critical alerts, and deploys scaling plans.

## Prerequisites

- [ ] Host pools deployed: `hp-pooled-prod`, `hp-personal-prod` (from [[08-host-pool-creation]])
- [ ] Session hosts running and joined to host pools (from [[09-session-hosts]])
- [ ] Application groups and workspace configured (from [[10-app-groups-workspace]])
- [ ] Users able to connect and access desktops
- [ ] Contributor role on resource group

---

## Part 1: Create Log Analytics Workspace

**Portal:** Azure Portal → Log Analytics workspaces → + Create

1. **Basics:**
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-01`
   - **Name:** `law-avd-prod-01`
   - **Region:** East US

2. **Pricing tier:** Pay-as-you-go

3. **Tags:**
   - **Environment:** Production
   - **Purpose:** AVD-Monitoring

4. Click **Review + create** → **Create**

**Set retention:** law-avd-prod-01 → Usage and estimated costs → Data Retention → 90 days

---

## Part 2: Enable Diagnostic Settings

Configure AVD resources to send logs to Log Analytics.

| Resource | Diagnostic Setting Name | Log Categories |
|----------|------------------------|-----------------|
| Host pools (both) | SendToLogAnalytics | Connection, Error, Checkpoint, Management, AgentHealthStatus |
| Application groups (both) | SendToLogAnalytics | Checkpoint, Error, Management |
| Workspace | SendToLogAnalytics | Checkpoint, Error, Management, Feed |

**For each resource:**
- **Portal:** [Resource] → Diagnostic settings → + Add
- **Name:** SendToLogAnalytics
- **Destination:** law-avd-prod-01
- **Log categories:** See table above
- Click **Save**

---

## Part 3: Deploy Azure Monitor Agent & Data Collection Rule

**Portal:** Virtual machines → Select all session hosts (pooled + personal) → Extensions → + Add

- Search: `AzureMonitorWindowsAgent`
- Enable automatic upgrade: Yes
- Click **Create**

**Create Data Collection Rule:**

**Portal:** Monitor → Data collection rules → + Create

1. **Basics:**
   - **Name:** `dcr-avd-prod-sessionhosts`
   - **Resource group:** `rg-avd-prod-01`
   - **Region:** East US

2. **Resources:**
   - Scope: `rg-avd-prod-01` (all VMs)

3. **Collect and deliver:**
   - **Data source 1: Performance Counters**
     - Sampling: 60 seconds
     - Counters: Processor, Memory, Disk space, Network, User input delay
     - Destination: law-avd-prod-01

   - **Data source 2: Windows Event Logs**
     - Application/System: Critical, Error, Warning
     - Destination: law-avd-prod-01

4. Click **Review + create** → **Create**

---

## Part 4: Enable AVD Insights

**Portal:** Azure Virtual Desktop → Insights → Open configuration workbook

1. Select subscription, host pool, workspace (law-avd-prod-01)
2. Click **Configure**

Wait 15-30 minutes for data ingestion. Monitor Overview tab for connection success rates, active users, and session host availability.

---

## Part 5: Create Action Groups & Alerts

**Action Group:**

**Portal:** Monitor → Alerts → Action groups → + Create

- **Name:** `ag-avd-ops-team`
- **Notifications:** Email (ops@contoso.com)
- Click **Create**

**Critical Alerts (Create 3 essential alerts):**

| Alert Name | Query | Threshold | Severity |
|-----------|-------|-----------|----------|
| AVD-SessionHost-Unavailable | `WVDAgentHealthStatus \| where Status != "Available" \| dcount(SessionHostName)` | >0 in 10m | Warning |
| AVD-HighConnectionFailure | `WVDConnections \| where State == "Failed" \| count()` | >10 in 5m | Warning |
| AVD-LowDiskSpace | `Perf \| where ObjectName == "LogicalDisk" and CounterName == "% Free Space" and InstanceName == "C:" \| where CounterValue < 10` | >0 | Warning |

**For each alert:**
- **Portal:** Monitor → Alerts → + Create → Alert rule
- **Scope:** law-avd-prod-01 (or select VMs for disk space)
- **Condition:** [Use query/signal above]
- **Actions:** ag-avd-ops-team
- Click **Create**

---

## Part 6: Configure Scaling Plan (Pooled Only)

**Portal:** Virtual Desktop → Scaling plans → + Create

1. **Basics:**
   - **Name:** `sp-pooled-prod`
   - **Time zone:** Eastern Time

2. **Schedules:**
   - **Weekday:** 7 AM (60% hosts) → 9 AM (80%) → 5 PM (20%) → 7 PM (20%)
   - **Weekend:** 9 AM (20%) → 11 AM (20%) → 5 PM (10%) → 7 PM (10%)

3. **Host pool:** hp-pooled-prod, Enable autoscale: Yes

4. Click **Create**

**See:** [[../AVD/scaling-plans|Scaling Plans]] for detailed configuration.

---

## Critical Troubleshooting

### Issue: No Data in Log Analytics

**Fix:**
1. Verify diagnostic settings enabled: **Portal** → [Resource] → Diagnostic settings
2. Wait 15-30 minutes for ingestion
3. Check users have connected: `WVDConnections | where TimeGenerated > ago(24h) | count()`

### Issue: AVD Insights Shows "No Data"

**Fix:**
1. Check Azure Monitor Agent installed: **Portal** → VM → Extensions
2. Verify DCR resources: **Portal** → Monitor → Data collection rules → dcr-avd-prod-sessionhosts → Resources
3. Ensure at least one active session exists

---

## Next Steps

**Monitoring configured. AVD deployment ready for production rollout.**

**Daily:**
- Review AVD Insights Overview (5 min)
- Check fired alerts

**Weekly:**
- Review capacity trends
- Plan host additions if needed

**See operational guides:**
- [[../Operations/user-acceptance-testing|User Acceptance Testing]] - Complete UAT procedures
- [[../Operations/avd-runbooks|AVD Runbooks]] - Common operational tasks
- [[../Operations/go-live-checklist|Go-Live Checklist]] - Full pre-launch verification
- [[../Security/log-analytics-workspace|Log Analytics Workspace]] - KQL queries, cost optimization
- [[../Security/azure-monitor-alerting|Azure Monitor Alerting]] - Advanced alert configuration
- [[../Operations/troubleshooting-avd-monitoring|Monitoring Troubleshooting]] - Extended issue resolution

---

## Related Reference Pages

- [[../AVD/scaling-plans|Scaling Plans]] - Advanced configuration
- [[../Operations/capacity-planning|Capacity Planning]] - Growth planning
- [[../Images/golden-image-process|Golden Image Updates]] - Image management
