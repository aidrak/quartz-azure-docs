---
title: Azure Monitor Alerting
description: 
published: true
date: 2025-12-14T04:53:37.888Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:49.768Z
---

# Azure Monitor Alerting

Azure Monitor alerts proactively notify you when critical conditions are detected in your infrastructure, applications, or user experience. For AVD deployments, alerting is essential to maintain SLA commitments, prevent downtime, and respond to issues before users report them. This page covers alert types, action groups, recommended AVD-specific alerts, and alert processing rules for intelligent routing and suppression.

## Azure Monitor Alerts Overview

Azure Monitor alerts consist of three components:

1. **Alert Rule**: Defines the condition to monitor (e.g., "CPU >80% for 5 minutes") and when to fire. Configured per resource or log query.

2. **Action Group**: Defines what happens when alert fires (email, SMS, run automation, call webhook). Reusable across multiple alert rules.

3. **Alert Instance**: A specific occurrence of an alert firing. Tracked in Azure Monitor with state (New, Acknowledged, Closed).

**Workflow**:
```
Monitor Resource → Condition Met → Alert Rule Fires → Action Group Triggered → Notification/Automation
```

**Example**: Session host CPU >90% for 10 minutes → Alert "High-CPU-AVD-Pool-0" fires → Action group "AVD-Ops-Team" sends email to ops@contoso.com and creates ServiceNow ticket.

**Cost**: Alerts are charged per evaluation and notification. Typical AVD deployment: ~$5-20/month for 10-20 alert rules.

## Alert Types

Azure Monitor supports three types of alerts, each suited for different monitoring scenarios:

### Metric Alerts

**Purpose**: Monitor numeric values from Azure resources (CPU, memory, disk, network).

**How it Works**:
- Azure collects metrics every 1 minute (platform metrics) or custom interval
- Alert rule evaluates condition (e.g., "Average CPU >80%") over time window
- Fires when threshold crossed for specified duration

**Example: High CPU Alert**

```bash
az monitor metrics alert create \
  --name "High-CPU-AVD-Pool-0" \
  --resource-group RG-Azure-VDI-01 \
  --scopes /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Compute/virtualMachines/avd-pool-0 \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --description "Session host avd-pool-0 CPU usage exceeded 80% for 5 minutes" \
  --action /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Insights/actionGroups/AVD-Ops-Team
```

**Use Case**: Monitor session host resource utilization, storage account throttling, network bandwidth.

**Advantages**:
- Fast (1-minute evaluation)
- Low latency (alerts fire within 1-2 minutes of threshold breach)
- Built-in for Azure resources (no configuration needed)

**Limitations**:
- Only works with numeric metrics
- Can't query logs or correlate multiple resources

### Log Alerts (KQL Queries)

**Purpose**: Query Log Analytics using KQL to detect complex conditions across multiple resources.

**How it Works**:
- Alert rule runs KQL query on schedule (every 5 minutes, hourly, etc.)
- Fires when query returns results or result count exceeds threshold
- Can aggregate data, join tables, use advanced logic

**Example: Failed Connection Alert**

```bash
az monitor scheduled-query alert create \
  --name "AVD-Failed-Connections" \
  --resource-group RG-Azure-VDI-01 \
  --scopes /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.OperationalInsights/workspaces/log-avd-prod \
  --condition "count > 10" \
  --condition-query "WVDConnections | where TimeGenerated > ago(5m) | where State == 'Failed' | summarize count()" \
  --window-size 5m \
  --evaluation-frequency 5m \
  --severity 2 \
  --description "More than 10 failed AVD connections in 5 minutes" \
  --action /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Insights/actionGroups/AVD-Ops-Team
```

**Advanced Example: Slow Connection Alert**

```kql
// Alert when >5 users have RTT >150ms in last 10 minutes
WVDConnections
| where TimeGenerated > ago(10m)
| where State == "Connected" and UdpRoundTripTimeInMS > 150
| summarize AffectedUsers = dcount(UserName), AvgRTT = avg(UdpRoundTripTimeInMS)
| where AffectedUsers > 5
```

**Use Case**: Detect patterns not visible in single metric (connection failures, profile load errors, security events).

**Advantages**:
- Flexible (any KQL query)
- Correlate multiple resources (users + session hosts + network)
- Historical analysis (query last 24 hours to detect trends)

**Limitations**:
- Higher latency (5-15 minute delay from event to alert)
- More expensive (charges per query execution and data scanned)

### Activity Log Alerts

**Purpose**: Monitor Azure control plane operations (resource creation, deletion, configuration changes).

**How it Works**:
- Azure Activity Log records all Azure Resource Manager operations
- Alert fires when specific operation occurs (e.g., "VM deleted", "NSG rule modified")
- Useful for auditing and security

**Example: Session Host Deleted Alert**

```bash
az monitor activity-log alert create \
  --name "AVD-Session-Host-Deleted" \
  --resource-group RG-Azure-VDI-01 \
  --scope /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01 \
  --condition category=Administrative and operationName=Microsoft.Compute/virtualMachines/delete \
  --action-group /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Insights/actionGroups/AVD-Security-Team \
  --description "Alert when any VM in RG-Azure-VDI-01 is deleted"
```

**Example: NSG Rule Changed Alert**

```bash
az monitor activity-log alert create \
  --name "AVD-NSG-Modified" \
  --resource-group RG-Azure-VDI-01 \
  --scope /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Network/networkSecurityGroups/NSG-AVD \
  --condition category=Administrative and operationName=Microsoft.Network/networkSecurityGroups/securityRules/write \
  --action-group /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Insights/actionGroups/AVD-Security-Team \
  --description "Alert when NSG rules are created or modified"
```

**Use Case**: Security monitoring (unauthorized changes), compliance auditing, change tracking.

**Advantages**:
- No additional configuration (Activity Log always enabled)
- Instant alerts (fires within seconds of operation)
- No cost (Activity Log alerts are free)

**Limitations**:
- Only monitors ARM operations (not guest OS events or application logs)
- Can't detect performance issues or user experience problems

## Action Groups

Action groups define who gets notified and what automation runs when an alert fires. One action group can be reused by multiple alert rules.

**Supported Actions**:

### Email

**Use Case**: Primary notification method for human operators.

**Configuration**:
- Email address (supports distribution lists)
- Optional: Enable common alert schema (standardized JSON format)

**Limitations**:
- Rate limited: Max 100 emails per hour
- Use email for critical alerts only; secondary alerts should go to ticketing system

### SMS

**Use Case**: Critical alerts requiring immediate attention (e.g., production outage).

**Configuration**:
- Phone number (include country code: +1-555-1234)
- SMS limited to 1 per 5 minutes per phone number

**Cost**: SMS charges apply (~$0.003 per SMS in US).

**Best Practice**: Reserve SMS for severity 0-1 alerts only.

### Voice Call

**Use Case**: Escalation path when email/SMS ignored.

**Configuration**:
- Phone number
- Voice call plays automated message: "Alert [alert name] has fired in Azure subscription..."

**Cost**: ~$0.10 per call.

### Webhook (HTTP/HTTPS)

**Use Case**: Integrate with third-party systems (ServiceNow, PagerDuty, Slack).

**Configuration**:
- URL endpoint
- Optional: Custom headers, authentication

**Example: Post to Slack**

```json
{
  "actionGroupName": "AVD-Ops-Team",
  "actions": [
    {
      "actionType": "Webhook",
      "name": "Slack-Notifications",
      "webhookUri": "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX"
    }
  ]
}
```

**Common Schema**: Enable "Use common alert schema" to get standardized JSON (easier to parse).

### Logic App

**Use Case**: Complex workflows (e.g., "Create ServiceNow ticket, check on-call schedule, escalate if not acked in 15 minutes").

**Configuration**:
- Select existing Logic App
- Logic App receives alert context as JSON input

**Example Workflow**:
1. Alert fires
2. Logic App queries on-call schedule (PagerDuty API)
3. Sends SMS to on-call engineer
4. Creates ticket in ServiceNow with alert details
5. If ticket not acknowledged in 15 minutes, escalate to manager

**Cost**: Logic App execution charges apply (~$0.000025 per action).

### Azure Automation Runbook

**Use Case**: Automated remediation (e.g., "Restart session host if unhealthy", "Add host to pool if capacity low").

**Configuration**:
- Select Automation Account and Runbook
- Runbook receives alert context as parameters

**Example: Auto-Scale Host Pool**

```powershell
# Runbook: Add-AVDSessionHost.ps1
param(
  [string]$HostPoolName,
  [int]$TargetCapacity
)

# Get current host count
$hosts = Get-AzWvdSessionHost -HostPoolName $HostPoolName
$currentCount = $hosts.Count

# Add hosts if below target
if ($currentCount -lt $TargetCapacity) {
  $hostsToAdd = $TargetCapacity - $currentCount
  # Deploy new VMs using ARM template
  New-AzResourceGroupDeployment -ResourceGroupName "RG-Azure-VDI-01" -TemplateFile "add-session-hosts.json" -Count $hostsToAdd
}
```

**Trigger**: Alert "AVD-Host-Pool-Capacity-Low" fires when available sessions <10% of total.

### ITSM Connector

**Use Case**: Create tickets in ServiceNow, BMC Remedy, Provance.

**Configuration**:
- Configure ITSM connector in Azure (one-time setup)
- Select connector in action group

**Ticket Fields**:
- Title: Alert name
- Description: Alert details, affected resources
- Severity: Maps from Azure alert severity (0-4)
- Assignment group: Based on resource tags or alert category

## Recommended AVD Alerts

These alerts cover the most critical AVD monitoring scenarios:

### 1. Session Host Unavailable

**Condition**: Session host is offline or not reporting to AVD control plane.

**Impact**: Users can't connect to that host; capacity reduced.

**Alert Rule**:
```kql
WVDAgentHealthStatus
| where TimeGenerated > ago(10m)
| where Status != "Available"
| summarize UnavailableHosts = dcount(SessionHostName) by SessionHostName, Status
| where UnavailableHosts > 0
```

**Action**: Email ops team, create ServiceNow ticket.

**Severity**: 2 (Warning) if 1 host down, 1 (Error) if >25% of hosts down.

### 2. High Round-Trip Time (RTT)

**Condition**: >10% of connections have RTT >150ms in last 15 minutes.

**Impact**: Degraded user experience (laggy desktop).

**Alert Rule**:
```kql
WVDConnections
| where TimeGenerated > ago(15m)
| where State == "Connected" and isnotnull(UdpRoundTripTimeInMS)
| summarize
    TotalConnections = count(),
    HighRTTConnections = countif(UdpRoundTripTimeInMS > 150),
    AvgRTT = avg(UdpRoundTripTimeInMS)
| extend PercentHighRTT = (HighRTTConnections * 100.0) / TotalConnections
| where PercentHighRTT > 10
```

**Action**: Email network team, check ExpressRoute/VPN health.

**Severity**: 2 (Warning).

### 3. FSLogix Profile Load Failures

**Condition**: Users unable to load FSLogix profiles (can't access their data).

**Impact**: Critical - user can't log in or sees default profile.

**Alert Rule**:
```kql
Event
| where TimeGenerated > ago(5m)
| where Source == "FSLogix-Profile" and EventLevelName == "Error"
| where RenderedDescription contains "Failed to attach VHD" or RenderedDescription contains "Profile load failed"
| summarize Failures = count() by Computer
| where Failures > 0
```

**Action**: Page on-call engineer (SMS), create P1 ticket.

**Severity**: 1 (Error).

### 4. Low Disk Space on Session Host

**Condition**: C: drive <10% free space on session host.

**Impact**: Profile writes fail, temp files can't be created, system instability.

**Alert Rule** (Metric):
```bash
az monitor metrics alert create \
  --name "AVD-Low-Disk-Space" \
  --resource-group RG-Azure-VDI-01 \
  --scopes /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Compute/virtualMachines/avd-pool-0 \
  --condition "avg OS Disk Free Space Percentage < 10" \
  --window-size 15m \
  --evaluation-frequency 5m \
  --severity 2
```

**Action**: Run cleanup script (Automation Runbook), alert ops team.

**Severity**: 2 (Warning) at 10% free, 1 (Error) at 5% free.

### 5. Host Pool at Capacity

**Condition**: >90% of max sessions in use (no available sessions for new users).

**Impact**: Users get "No resources available" error when connecting.

**Alert Rule**:
```kql
WVDCheckpoints
| where TimeGenerated > ago(5m)
| where Name == "SessionHostChange"
| summarize ActiveSessions = dcount(CorrelationId) by HostPoolName = tostring(Parameters.HostPoolName)
| join kind=inner (
    // Assuming max 20 sessions per host, 8 hosts = 160 max sessions
    datatable(HostPoolName:string, MaxSessions:int) [
      "HostPool-VDI-01", 160
    ]
) on HostPoolName
| extend CapacityPercent = (ActiveSessions * 100.0) / MaxSessions
| where CapacityPercent > 90
```

**Action**: Trigger auto-scale (add session hosts via Automation Runbook).

**Severity**: 2 (Warning).

### 6. Session Host High CPU

**Condition**: CPU >80% for 10 minutes.

**Impact**: Slow desktop performance, application timeouts.

**Alert Rule** (Metric):
```bash
az monitor metrics alert create \
  --name "AVD-High-CPU" \
  --resource-group RG-Azure-VDI-01 \
  --scopes /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Compute/virtualMachines/avd-pool-0 \
  --condition "avg Percentage CPU > 80" \
  --window-size 10m \
  --evaluation-frequency 5m \
  --severity 2 \
  --description "Session host CPU usage sustained above 80%"
```

**Action**: Email ops team, check for runaway processes.

**Severity**: 2 (Warning) at 80%, 1 (Error) at 95%.

### 7. Azure Service Health Issues

**Condition**: Azure reports service degradation or outage in AVD.

**Impact**: May explain widespread connection failures (not your fault).

**Alert Rule** (Service Health):
1. Azure Portal → Monitor → Service Health → Health alerts
2. Create alert rule:
   - Services: Azure Virtual Desktop
   - Regions: Central US (or your deployment region)
   - Event types: Service issues, Planned maintenance

**Action**: Email ops team, post status update to internal communications.

**Severity**: Matches Azure severity (Informational to Critical).

## Alert Processing Rules

Alert processing rules (formerly "action rules") modify alert behavior without changing the underlying alert rule. Use cases:

### Suppression (Maintenance Windows)

**Problem**: Alerts fire during planned maintenance (e.g., patching session hosts).

**Solution**: Suppress alerts during maintenance window.

**Example**:
```bash
az monitor alert-processing-rule create \
  --name "Suppress-Alerts-During-Patching" \
  --resource-group RG-Azure-VDI-01 \
  --scopes /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01 \
  --filter-alert-rule-name "AVD-High-CPU" "AVD-Session-Host-Unavailable" \
  --schedule-type OneTime \
  --schedule-start "2025-12-15T02:00:00Z" \
  --schedule-end "2025-12-15T06:00:00Z" \
  --actions Suppress
```

**Use Case**: Sunday 2 AM - 6 AM monthly patching (suppresses CPU and availability alerts).

### Routing (Escalation)

**Problem**: Different teams handle different resource types.

**Solution**: Route AVD connection alerts to app team, infrastructure alerts to ops team.

**Example**:
```bash
# Route connection errors to app team
az monitor alert-processing-rule create \
  --name "Route-Connection-Errors-AppTeam" \
  --resource-group RG-Azure-VDI-01 \
  --scopes /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01 \
  --filter-alert-rule-name "AVD-Failed-Connections" \
  --actions AddActionGroups \
  --action-groups /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Insights/actionGroups/App-Team

# Route infrastructure alerts to ops team
az monitor alert-processing-rule create \
  --name "Route-Infra-Errors-OpsTeam" \
  --resource-group RG-Azure-VDI-01 \
  --scopes /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01 \
  --filter-alert-rule-name "AVD-High-CPU" "AVD-Low-Disk-Space" \
  --actions AddActionGroups \
  --action-groups /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Insights/actionGroups/Ops-Team
```

### Enrichment (Add Context)

**Problem**: Alert doesn't include business context (which customer, which application).

**Solution**: Use processing rule to add custom dimensions (via Azure tags).

**Example**:
1. Tag AVD resources: `Customer: Contoso`, `Environment: Production`
2. Processing rule includes tags in alert payload
3. ITSM connector creates ticket with customer and environment fields populated

## Best Practices

- **Start with Critical Alerts Only** - Avoid alert fatigue. Focus on alerts that require immediate action (service down, capacity exhausted). Add informational alerts later.

- **Use Action Groups for Teams, Not Individuals** - Create action groups by function (Ops-Team, Security-Team), not person (john@contoso.com). Easier to manage when people change roles.

- **Test Alerts Before Production** - Fire test alert: Azure Portal → Alert rule → Test → Fire. Verify notifications arrive and automation works.

- **Set Severity Appropriately** - Severity 0-1: Page on-call (SMS/call). Severity 2-3: Email. Severity 4: Log only (no notification). Don't cry wolf with high severity for informational alerts.

- **Use Common Alert Schema** - Standardized JSON format makes webhook/Logic App integrations easier. Enable in action group settings.

- **Monitor Alert Volume** - If alert fires >10 times/day, either raise threshold or fix underlying issue. Alerts should be exceptional, not routine.

- **Document Runbooks** - For each alert, create runbook (wiki page) with troubleshooting steps. Example: "AVD-High-CPU alert runbook: 1. Check top processes, 2. Check user count, 3. Resize VM if sustained."

- **Review Fired Alerts Weekly** - Look at Alert History in Azure Monitor. Identify noisy alerts (fire frequently but false positives) and tune thresholds.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Alert fires but no email received | Email address typo in action group | Verify email: Action group → Notifications tab → Check spelling. Test: Fire test alert. |
| Alert fires repeatedly (flapping) | Threshold too sensitive or metric oscillating | Increase evaluation window (e.g., 5m → 15m). Add "must exceed threshold for X consecutive periods" logic. |
| Log alert doesn't fire despite query returning results | Query result format doesn't match condition | Log alerts expect specific result format: `count()`, `AggregatedValue`, etc. Test query in Log Analytics first. |
| Action group webhook fails | Endpoint down or authentication invalid | Check webhook URL. Test: `curl -X POST [webhook-url] -d '{"test":"data"}'`. Verify SSL cert valid. |
| SMS not delivered | Phone number incorrect format | Use E.164 format: `+1-555-123-4567` (country code required). Verify number receives SMS normally. |
| Alert suppression not working | Processing rule scope doesn't include alert resource | Check processing rule scope: Must include resource or resource group of alert. Verify time window. |
| Logic App not triggered | Logic App connection expired or disabled | Logic App → API connections → Check status. Reauthorize if expired. Verify Logic App enabled (not disabled). |

## Related Resources

- Azure Monitor Alerts Overview: https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-overview
- Action Groups: https://learn.microsoft.com/azure/azure-monitor/alerts/action-groups
- Alert Processing Rules: https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-processing-rules
- Common Alert Schema: https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-common-schema
- AVD Monitoring Best Practices: https://learn.microsoft.com/azure/virtual-desktop/diagnostics-log-analytics