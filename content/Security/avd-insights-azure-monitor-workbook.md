---
title: AVD Insights (Azure Monitor Workbook)
description: 
published: true
date: 2025-12-14T04:53:34.465Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:46.384Z
---

# AVD Insights (Azure Monitor Workbook)

AVD Insights is a pre-built Azure Monitor workbook that provides comprehensive monitoring and diagnostics for Azure Virtual Desktop deployments. It consolidates connection data, performance metrics, and user activity into interactive dashboards, enabling rapid troubleshooting and capacity planning. AVD Insights is the recommended monitoring solution for production AVD environments and integrates seamlessly with Log Analytics and Azure Monitor Agent.

## What is AVD Insights

AVD Insights is an Azure Monitor workbook template specifically designed for AVD. Unlike generic monitoring tools, it understands AVD-specific concepts like connection brokers, session hosts, feed discovery, and RDP stack telemetry.

**Key Capabilities**:

1. **Connection Diagnostics**: Tracks user connection attempts from feed subscription through session establishment. Identifies failures at each stage (DNS resolution, gateway authentication, broker assignment, session host connection).

2. **Performance Monitoring**: Visualizes CPU, memory, disk, and network utilization across session hosts. Correlates resource consumption with user sessions (e.g., "Host avd-pool-0 has 15 active users and 85% CPU").

3. **User Experience Metrics**: Measures round-trip time (RTT), input delay, and frame rate. Alerts on degraded user experience (RTT >150ms).

4. **Historical Trending**: Compares current week vs previous weeks for capacity planning. Identifies usage patterns (peak hours, seasonal trends).

5. **Resource Health**: Shows session host agent status, VM power state, and availability. Detects hosts that are offline or unhealthy.

**Access AVD Insights**:
1. Azure Portal → Azure Virtual Desktop → Insights
2. Select workspace, host pool, or view all resources
3. Choose time range (Last 24 hours, Last 7 days, custom)

**Our Configuration**: AVD Insights enabled for all resources in RG-Azure-VDI-01, sending data to `log-avd-prod` workspace using DCR `microsoft-avdi-centralus`.

## Prerequisites

AVD Insights requires specific configuration to function properly:

### 1. Log Analytics Workspace

**Requirement**: One or more Log Analytics workspaces to store AVD telemetry.

**Setup**:
- Already deployed: `log-avd-prod` in RG-Azure-VDI-01
- Ensure 30+ days retention for trending (recommended: 90 days for WVDConnections)

**Verification**:
```bash
az monitor log-analytics workspace show \
  --resource-group RG-Azure-VDI-01 \
  --name log-avd-prod \
  --query "{Name:name, Location:location, RetentionDays:retentionInDays}" \
  --output table
```

### 2. Diagnostic Settings on AVD Resources

**Requirement**: All AVD resources must send logs to Log Analytics.

**Resources to Configure**:
- Host pools (HostPool-VDI-01)
- Application groups (AppGroup-Desktop, AppGroup-RemoteApp)
- Workspaces (Workspace-AVD-Prod)

**Enable Diagnostics**:

```bash
# Host pool
az monitor diagnostic-settings create \
  --name SendToLogAnalytics \
  --resource /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/hostpools/HostPool-VDI-01 \
  --workspace /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.OperationalInsights/workspaces/log-avd-prod \
  --logs '[
    {"category":"Connection","enabled":true,"retentionPolicy":{"enabled":false,"days":0}},
    {"category":"Error","enabled":true,"retentionPolicy":{"enabled":false,"days":0}},
    {"category":"Checkpoint","enabled":true,"retentionPolicy":{"enabled":false,"days":0}},
    {"category":"Management","enabled":true,"retentionPolicy":{"enabled":false,"days":0}},
    {"category":"AgentHealthStatus","enabled":true,"retentionPolicy":{"enabled":false,"days":0}}
  ]'

# Application group
az monitor diagnostic-settings create \
  --name SendToLogAnalytics \
  --resource /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/applicationgroups/AppGroup-Desktop \
  --workspace /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.OperationalInsights/workspaces/log-avd-prod \
  --logs '[
    {"category":"Checkpoint","enabled":true},
    {"category":"Error","enabled":true},
    {"category":"Management","enabled":true}
  ]'
```

**Validation**:
```kql
// Check if data is flowing
WVDConnections
| where TimeGenerated > ago(1h)
| summarize count() by _ResourceId
| order by count_ desc
```

If no results, diagnostics are not configured or no users have connected recently.

### 3. Azure Monitor Agent on Session Hosts

**Requirement**: Azure Monitor Agent (AMA) deployed to all session host VMs to collect performance counters and Windows Event Logs.

**Install AMA Extension**:

```bash
az vm extension set \
  --resource-group RG-Azure-VDI-01 \
  --vm-name avd-pool-0 \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --enable-auto-upgrade true
```

**Deploy to All Session Hosts** (at scale):

```bash
# Get all VMs in host pool
vms=$(az vm list --resource-group RG-Azure-VDI-01 --query "[?tags.HostPool=='HostPool-VDI-01'].name" -o tsv)

# Install AMA on each
for vm in $vms; do
  echo "Installing AMA on $vm"
  az vm extension set \
    --resource-group RG-Azure-VDI-01 \
    --vm-name $vm \
    --name AzureMonitorWindowsAgent \
    --publisher Microsoft.Azure.Monitor \
    --enable-auto-upgrade true \
    --no-wait  # Parallel deployment
done
```

**Associate with Data Collection Rule**:

```bash
# Link VM to DCR
az monitor data-collection rule association create \
  --name AVD-Insights-Association \
  --rule-id /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Insights/dataCollectionRules/microsoft-avdi-centralus \
  --resource /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Compute/virtualMachines/avd-pool-0
```

**Verification**:
```kql
// Check performance data
Perf
| where TimeGenerated > ago(30m)
| where Computer == "avd-pool-0"
| summarize count() by ObjectName, CounterName
```

Expected counters: Processor, Memory, LogicalDisk, Network Interface, User Input Delay.

## Workbook Sections

AVD Insights organizes data into five main sections:

### Overview Section

**Purpose**: High-level health status and usage metrics.

**Key Visualizations**:
- **Connection Success Rate**: Percentage of successful connections vs total attempts (target: >95%)
- **Active Users**: Current number of connected users across all host pools
- **Session Host Availability**: How many hosts are online/offline
- **Top Errors**: Most frequent error codes in last 24 hours

**Sample Metrics**:
```
Connection Success Rate: 97.2% (423 successful / 435 total attempts)
Active Users: 142 across 8 session hosts
Session Hosts: 7 healthy, 1 offline (avd-pool-3)
Top Error: ConnectionFailedClientDisconnect (12 occurrences)
```

**Use Case**: Daily health check. Share with management to demonstrate SLA compliance.

### Connection Diagnostics

**Purpose**: Troubleshoot failed connections by analyzing each connection stage.

**Connection Stages**:
1. **Feed Discovery**: User retrieves list of available desktops/apps from AVD workspace
2. **Authentication**: User authenticates to Azure AD
3. **Gateway**: RDP traffic routed through AVD gateway
4. **Broker**: AVD broker selects session host (based on load balancing)
5. **Session Host**: User connects to assigned VM

**Failure Analysis**:
- Filter by user: "Show all connection attempts for user jdoe@contoso.com"
- Filter by time: "Connections during incident window (10:00-10:15 AM)"
- Group by error code: "ServiceError == 'NoCandidateSessionHostFound'" (no available hosts)

**Example Query** (built into workbook):
```kql
WVDConnections
| where TimeGenerated > ago(24h)
| where State == "Failed"
| summarize FailureCount = count() by ServiceError, Code
| order by FailureCount desc
| take 10
```

**Common Errors**:
- `ConnectionFailedClientDisconnect`: User closed client before connection completed (user error, not system issue)
- `NoCandidateSessionHostFound`: All session hosts at capacity or offline (add more hosts)
- `SessionHostRegisteredWrongWorkspace`: Host pool misconfiguration (re-register session host)

### Connection Performance

**Purpose**: Monitor user experience metrics (latency, frame rate, bandwidth).

**Key Metrics**:

**Round-Trip Time (RTT)**:
- Measures network latency between user's device and AVD gateway
- **Good**: <50ms
- **Acceptable**: 50-150ms
- **Poor**: >150ms (noticeable lag)

**Input Delay**:
- Time from user keyboard/mouse input to screen update
- Measured per session on session host
- **Target**: <50ms (imperceptible to user)
- **Alert**: >100ms (impacts productivity)

**Bandwidth Utilization**:
- RDP traffic volume (MB/s)
- Higher for video content, lower for text-based apps
- Helps size ExpressRoute circuits or VPN connections

**Visualization**: Heat map showing RTT by user location and time of day. Identifies if specific geographies have poor connectivity.

**Example**:
```kql
WVDConnections
| where TimeGenerated > ago(7d)
| where State == "Connected"
| summarize AvgRTT = avg(UdpRoundTripTimeInMS), P95RTT = percentile(UdpRoundTripTimeInMS, 95) by ClientOS, ClientCountryOrRegion
| order by P95RTT desc
```

**Output**:
| ClientOS | ClientCountryOrRegion | AvgRTT | P95RTT |
|----------|----------------------|--------|--------|
| Windows | Philippines | 245 | 312 |
| macOS | United States | 35 | 68 |

**Action**: Philippines users have high latency. Consider deploying AVD in Southeast Asia region or optimizing WAN.

### Host Diagnostics

**Purpose**: Monitor session host health and resource utilization.

**Key Metrics**:

**CPU Utilization**:
- Average and peak CPU per session host
- Identify overloaded hosts (sustained >80%)
- Trigger: Add hosts or resize to larger VM SKU

**Memory Pressure**:
- Available memory in MB
- Alert: <10% free memory (risk of Out of Memory errors)

**Disk Performance**:
- IOPS and latency on OS and data disks
- Detect storage bottlenecks (e.g., slow FSLogix profile loads)

**Network Throughput**:
- Bytes sent/received per session host
- Identify bandwidth saturation

**Visualization**: Stacked area chart showing CPU usage per host over time. Correlates spikes with user session count.

**Example**:
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize AvgCPU = avg(CounterValue), MaxCPU = max(CounterValue) by Computer
| join kind=inner (
    WVDCheckpoints
    | where TimeGenerated > ago(1h)
    | where Name == "SessionHostChange"
    | summarize ActiveSessions = dcount(CorrelationId) by SessionHostName = tostring(Parameters.SessionHostName)
) on $left.Computer == $right.SessionHostName
| project Computer, AvgCPU, MaxCPU, ActiveSessions
| order by MaxCPU desc
```

**Output**:
| Computer | AvgCPU | MaxCPU | ActiveSessions |
|----------|--------|--------|----------------|
| avd-pool-0 | 78.2 | 95.1 | 18 |
| avd-pool-1 | 52.3 | 68.7 | 12 |

**Action**: avd-pool-0 is overloaded. Drain sessions to other hosts or add capacity.

### User Reports

**Purpose**: Track user activity and identify heavy users or unusual patterns.

**Metrics**:

**User Session Duration**:
- Average time users stay connected
- Helps plan session timeout policies

**Connection Frequency**:
- How many times per day users connect
- Detects disconnection issues (users reconnecting frequently = network problems)

**Application Usage** (if using RemoteApp):
- Which applications are most used
- Optimize host pool capacity for popular apps

**Geographic Distribution**:
- Where users are connecting from
- Plan regional deployments

**Example**:
```kql
WVDConnections
| where TimeGenerated > ago(30d)
| where State == "Connected"
| extend SessionDurationMinutes = datetime_diff('minute', DisconnectTime, ConnectTime)
| summarize
    TotalConnections = count(),
    AvgSessionMinutes = avg(SessionDurationMinutes),
    TotalSessionHours = sum(SessionDurationMinutes) / 60
  by UserName
| order by TotalSessionHours desc
| take 20
```

**Output**:
| UserName | TotalConnections | AvgSessionMinutes | TotalSessionHours |
|----------|------------------|-------------------|-------------------|
| alice@contoso.com | 45 | 378 | 283.5 |
| bob@contoso.com | 102 | 125 | 212.5 |

**Insight**: Alice has fewer but longer sessions (full-day work). Bob reconnects frequently (mobile user, network issues?).

## Key Metrics to Monitor

Focus on these metrics for proactive monitoring:

### Round-Trip Time (RTT)

**What**: Network latency between client and AVD gateway.

**Why**: High RTT degrades user experience (laggy mouse, delayed typing).

**Threshold**:
- Good: <50ms
- Acceptable: 50-150ms
- Poor: >150ms (requires investigation)

**Query**:
```kql
WVDConnections
| where TimeGenerated > ago(24h)
| where State == "Connected" and isnotnull(UdpRoundTripTimeInMS)
| summarize
    AvgRTT = avg(UdpRoundTripTimeInMS),
    P50RTT = percentile(UdpRoundTripTimeInMS, 50),
    P95RTT = percentile(UdpRoundTripTimeInMS, 95)
  by bin(TimeGenerated, 1h)
| render timechart
```

**Remediation**:
- High RTT for specific users: Network issue on user side (ISP, Wi-Fi)
- High RTT globally: Deploy AVD in region closer to users
- High RTT during specific hours: Network congestion (upgrade WAN link)

### Session Success Rate

**What**: Percentage of connection attempts that succeed.

**Why**: Low success rate = users can't work = business impact.

**Threshold**:
- Target: >98%
- Acceptable: 95-98%
- Critical: <95%

**Query**:
```kql
WVDConnections
| where TimeGenerated > ago(24h)
| summarize
    TotalAttempts = count(),
    SuccessfulConnections = countif(State == "Connected"),
    FailedConnections = countif(State == "Failed")
| extend SuccessRate = (SuccessfulConnections * 100.0) / TotalAttempts
| project SuccessRate, SuccessfulConnections, FailedConnections, TotalAttempts
```

**Remediation**:
- Low success rate: Check top errors in Connection Diagnostics
- Sudden drop: Check Azure service health, session host availability

### Input Delay

**What**: Time from user input (keystroke, mouse click) to screen update.

**Why**: High input delay makes desktop feel unresponsive.

**Threshold**:
- Good: <50ms
- Acceptable: 50-100ms
- Poor: >100ms

**Query**:
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "User Input Delay per Session" and CounterName == "Max Input Delay"
| summarize AvgInputDelay = avg(CounterValue), MaxInputDelay = max(CounterValue) by Computer
| order by MaxInputDelay desc
```

**Remediation**:
- High input delay: Session host resource constraint (CPU, memory, disk)
- Check concurrent users per host (lower user density)
- Upgrade VM size

## Our Data Collection Rules (DCRs)

We use these DCRs to configure data collection:

### microsoft-avdi-centralus

**Purpose**: AVD-specific metrics and events (auto-created by AVD Insights setup wizard).

**Data Collected**:
- Performance counters: User Input Delay, RemoteFX Network
- Windows Event Logs: AVD operational events, application errors
- Custom metrics: Session host health

**Associated Resources**: All session hosts in centralus region.

**Location**: Managed by AVD Insights; don't modify manually.

### MSVMI-centralus-avd-pool-0

**Purpose**: VM Insights performance data for detailed VM monitoring.

**Data Collected**:
- Processor, Memory, Disk, Network counters (sampled every 1 minute)
- Process-level metrics (top CPU/memory consumers)
- Network connections (ServiceMap dependency mapping)

**Associated Resources**: Specific to avd-pool-0 (created when VM Insights enabled).

**Note**: Each session host gets its own DCR for VM Insights. Consolidate if managing many hosts (use single DCR for all hosts in host pool).

## Alerting Based on AVD Insights Data

AVD Insights data in Log Analytics powers alert rules:

**Example: Alert on Low Success Rate**

```bash
az monitor metrics alert create \
  --name "AVD-Low-Success-Rate" \
  --resource-group RG-Azure-VDI-01 \
  --scopes /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.OperationalInsights/workspaces/log-avd-prod \
  --condition "count 'WVDConnections' where State == 'Failed'" \
  --window-size 5m \
  --evaluation-frequency 5m \
  --severity 2 \
  --description "Fires when connection failure rate exceeds 10% in 5 minutes"
```

**More examples in "Alerting" page.**

## Best Practices

- **Check AVD Insights Daily** - Spend 5 minutes each morning reviewing Overview and Connection Diagnostics. Catch issues before users complain.

- **Set Baseline Metrics** - Record typical RTT, CPU usage, and success rate during normal operations. Alert on deviations (>20% increase).

- **Pin Frequently-Used Queries** - Save custom queries as workbook tabs or Azure Dashboard widgets. Example: "Top 10 users by session hours this month."

- **Enable VM Insights for Deep Troubleshooting** - VM Insights (separate from AVD Insights) provides process-level data. Use when AVD Insights shows high CPU but you need to know which process.

- **Export Reports for Management** - Use workbook's "Export to Excel" feature to generate monthly reports (uptime, user counts, top errors).

- **Correlate with Azure Service Health** - If AVD Insights shows widespread issues, check Azure status page (https://status.azure.com/status). May be platform issue, not your config.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No data in AVD Insights workbook | Diagnostics not enabled on AVD resources | Enable diagnostics on host pools, app groups, workspaces. Wait 15 minutes for data flow. |
| Performance counters missing (CPU, memory) | Azure Monitor Agent not installed or DCR not associated | Install AMA extension on VMs. Associate VMs with DCR `microsoft-avdi-centralus`. |
| Connection data exists but no performance data | AMA installed but collecting wrong counters | Verify DCR includes Processor, Memory, Disk, Network counters. Edit DCR: Azure Portal → Data Collection Rules. |
| AVD Insights shows old data (not real-time) | Log Analytics ingestion delay | Normal: 2-5 minute delay from event to queryable data. If delay >15 minutes, check AMA agent status on VMs. |
| Can't see AVD Insights workbook | Insufficient RBAC | Require "Monitoring Reader" on subscription or "Log Analytics Reader" on workspace. Check: Portal → Subscriptions → Access Control (IAM). |
| Workbook shows "No resources found" | Wrong subscription or resource group filter | Change filters at top of workbook: Select correct subscription, RG, host pool. |
| RTT always shows 0 or null | Client doesn't support UDP telemetry (old RDP client) | Update Remote Desktop client: Windows Store (Windows), App Store (macOS), or download from aka.ms/rdclients. |

## Related Resources

- AVD Insights Setup Guide: https://learn.microsoft.com/azure/virtual-desktop/insights
- Data Collection Rules Documentation: https://learn.microsoft.com/azure/azure-monitor/essentials/data-collection-rule-overview
- Azure Monitor Workbooks: https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-overview
- VM Insights: https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-overview
- AVD Metrics Reference: https://learn.microsoft.com/azure/virtual-desktop/insights-use-cases