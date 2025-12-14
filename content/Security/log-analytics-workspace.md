---
title: Log Analytics Workspace
description: 
published: true
date: 2025-12-14T04:53:41.459Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:53.256Z
---

# Log Analytics Workspace

Azure Log Analytics is the centralized logging and analytics engine for Azure Monitor. It collects telemetry from virtual machines, applications, and Azure services into a workspace where you can query, analyze, and alert on the data using Kusto Query Language (KQL). For AVD deployments, Log Analytics is essential for troubleshooting connection issues, tracking user activity, monitoring performance, and meeting compliance requirements.

## What is Log Analytics

Log Analytics is a service within Azure Monitor that provides:

1. **Data Ingestion**: Collects logs and metrics from diverse sources (Azure resources, VMs, applications, custom sources) into a centralized repository.

2. **Storage & Indexing**: Stores data in compressed, columnar format optimized for fast queries. Automatically indexes key fields (TimeGenerated, ResourceId, etc.).

3. **Query Engine**: Executes KQL queries across billions of rows in seconds. Supports aggregations, joins, time-series analysis, and machine learning functions.

4. **Visualization**: Powers Azure Monitor workbooks, Grafana dashboards, and Power BI reports. Provides prebuilt dashboards for common scenarios (VM performance, AVD diagnostics).

5. **Alerting**: Triggers alerts when query results meet thresholds. Integrates with action groups (email, SMS, Logic Apps, webhooks).

**Architecture**: A Log Analytics workspace is a logical container with its own:
- Data retention settings (30-730 days)
- Access controls (RBAC at workspace or table level)
- Query scope (data from all connected resources)
- Pricing tier (Pay-as-you-go, Commitment, or Capacit Reservation)

**Our Workspace**: `log-avd-prod` (Resource Group: RG-Azure-VDI-01)

## Workspace Architecture

A well-designed workspace strategy balances centralization (easier querying) with isolation (security, compliance).

### Single Workspace vs Multiple Workspaces

**Single Workspace (Recommended for Most)**:
- All AVD resources send logs to one workspace
- Simplifies cross-resource queries (e.g., "Which users had slow connections on session host X?")
- Easier RBAC management
- Lower cost (volume discounts, commitment tiers)

**Use Case**: Single Azure tenant, one IT team, uniform compliance requirements.

**Multiple Workspaces (Advanced Scenarios)**:
- Separate workspaces per environment (prod, dev, test)
- Separate workspaces per customer (MSP multi-tenancy)
- Separate workspaces per region (data residency requirements)

**Use Case**: MSP managing multiple customers with data isolation requirements, or multinational org with GDPR regional restrictions.

**Our Design**: Single workspace (`log-avd-prod`) collecting data from:
- Session hosts (avd-pool-0, avd-pool-1, ...)
- Host pools (HostPool-VDI-01)
- Application groups (AppGroup-Desktop, AppGroup-RemoteApp)
- Storage accounts (FSLogix profiles)
- Virtual networks (flow logs to stavdflowlogs1523)

### Workspace Tables

Log Analytics organizes data into tables. Key tables for AVD:

| Table | Description | Sample Query Use Case |
|-------|-------------|----------------------|
| `WVDConnections` | User connection attempts, success/failure, client info | "How many connections failed today?" |
| `WVDCheckpoints` | Connection lifecycle events (connecting, connected, disconnected) | "Average time from connecting to connected" |
| `WVDErrors` | AVD service errors (authentication, gateway, broker) | "Why is user X unable to connect?" |
| `WVDAgentHealthStatus` | Session host agent status (heartbeats, version, upgrade status) | "Which hosts have outdated agents?" |
| `WVDManagement` | Administrative operations (create/delete resources) | "Who deleted the host pool?" |
| `Perf` | Performance counters (CPU, memory, disk, network) | "Which session hosts are CPU-constrained?" |
| `Event` | Windows Event Log entries | "Find application crashes" |
| `Syslog` | Linux syslog (if using Linux VMs) | N/A for Windows AVD |
| `AzureDiagnostics` | Resource logs from Azure services (NSG, Key Vault, etc.) | "Which IPs were blocked by NSG?" |

**Custom Tables**: You can create custom tables for application-specific logs using Data Collection Rules (DCRs).

## Data Sources

Log Analytics ingests data from multiple sources:

### 1. Azure Diagnostics

Azure resources send diagnostic logs and metrics directly to Log Analytics.

**Enable Diagnostics for AVD Resources**:

```bash
# Host pool diagnostics
az monitor diagnostic-settings create \
  --name DiagToLogAnalytics \
  --resource /subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/hostpools/HostPool-VDI-01 \
  --workspace /subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.OperationalInsights/workspaces/log-avd-prod \
  --logs '[{"category":"Connection","enabled":true},{"category":"Error","enabled":true},{"category":"Checkpoint","enabled":true},{"category":"Management","enabled":true}]'

# Application group diagnostics
az monitor diagnostic-settings create \
  --name DiagToLogAnalytics \
  --resource /subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/applicationgroups/AppGroup-Desktop \
  --workspace /subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.OperationalInsights/workspaces/log-avd-prod \
  --logs '[{"category":"Checkpoint","enabled":true},{"category":"Error","enabled":true}]'
```

**Important**: Enable diagnostics on **all** AVD resources (host pools, app groups, workspaces) for complete visibility.

### 2. Azure Monitor Agent (AMA)

The Azure Monitor Agent (successor to Log Analytics Agent/MMA) collects data from VM guest OS:
- Performance counters (CPU, memory, disk)
- Windows Event Logs (Application, Security, System)
- IIS logs, custom text logs
- Syslog (Linux)

**Install AMA on Session Hosts**:

```bash
# Deploy AMA extension
az vm extension set \
  --resource-group RG-Azure-VDI-01 \
  --vm-name avd-pool-0 \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --enable-auto-upgrade true

# Associate with Data Collection Rule (DCR)
az monitor data-collection rule association create \
  --name AVD-DCR-Association \
  --rule-id /subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Insights/dataCollectionRules/microsoft-avdi-centralus \
  --resource /subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Compute/virtualMachines/avd-pool-0
```

**Data Collection Rules (DCRs)**: Define what data to collect and where to send it. Our DCRs:
- `microsoft-avdi-centralus`: AVD-specific metrics and events (auto-created by AVD Insights)
- `MSVMI-centralus-avd-pool-0`: VM Insights performance counters

**Performance Counters to Collect**:
```yaml
# Example DCR configuration (JSON snippet)
"performanceCounters": [
  {"samplingFrequency": "PT1M", "counterSpecifiers": ["\\Processor(_Total)\\% Processor Time"]},
  {"samplingFrequency": "PT1M", "counterSpecifiers": ["\\Memory\\Available MBytes"]},
  {"samplingFrequency": "PT1M", "counterSpecifiers": ["\\LogicalDisk(C:)\\% Free Space"]},
  {"samplingFrequency": "PT1M", "counterSpecifiers": ["\\Network Interface(*)\\Bytes Total/sec"]},
  {"samplingFrequency": "PT1M", "counterSpecifiers": ["\\User Input Delay per Session(*)\\Max Input Delay"]}
]
```

### 3. Custom Logs

Send application-specific logs to Log Analytics using:

- **HTTP Data Collector API**: POST JSON data to workspace REST endpoint
- **Log Analytics Agent custom logs**: Monitor text log files (e.g., `C:\AppLogs\myapp.log`)
- **Azure Functions**: Parse logs and send to workspace via API

**Example: Custom Log Table**:
```bash
# Create custom table via REST API
POST https://log-avd-prod.ods.opinsights.azure.com/api/logs?api-version=2016-04-01
Headers:
  Content-Type: application/json
  Log-Type: MyCustomAppLog
  x-ms-date: [RFC1123 timestamp]
  Authorization: SharedKey [WorkspaceId]:[Signature]

Body:
[
  {"Timestamp": "2025-12-14T10:30:00Z", "Severity": "Error", "Message": "Database timeout"}
]
```

Data appears in table `MyCustomAppLog_CL` (suffix `_CL` denotes custom log).

## Retention Settings

Data retention balances compliance needs with cost. Log Analytics charges for:
1. **Data ingestion**: Per GB ingested
2. **Data retention beyond default**: Per GB per month

**Default Retention**: 30 days (included in ingestion cost)

**Extended Retention**: 31-730 days (additional cost)

**Configure Retention**:
1. Azure Portal → Log Analytics workspaces → log-avd-prod
2. Usage and estimated costs → Data Retention
3. Set slider: 30-730 days

**Table-Specific Retention** (Advanced):
```bash
# Keep WVDConnections for 90 days, Perf for 30 days
az monitor log-analytics workspace table update \
  --resource-group RG-Azure-VDI-01 \
  --workspace-name log-avd-prod \
  --name WVDConnections \
  --retention-time 90

az monitor log-analytics workspace table update \
  --resource-group RG-Azure-VDI-01 \
  --workspace-name log-avd-prod \
  --name Perf \
  --retention-time 30
```

**Archive to Storage** (Cost Optimization):
- Export old logs to Azure Storage (cheap long-term storage)
- Use Log Analytics export rules or Logic App scheduled queries
- Compliance scenario: Keep 7 years in cold storage for audits

## Cost Optimization

Log Analytics pricing is primarily ingestion-based (~$2.50/GB in Pay-as-you-go). High-volume environments can reduce costs through:

### 1. Commitment Tiers

Commit to minimum daily ingestion for discounted rate:

| Tier | Daily Commitment | Price per GB | Savings vs Pay-as-you-go |
|------|------------------|--------------|--------------------------|
| Pay-as-you-go | None | $2.50 | Baseline |
| 100 GB/day | 100 GB | $2.30 | 8% |
| 200 GB/day | 200 GB | $2.10 | 16% |
| 500 GB/day | 500 GB | $1.75 | 30% |

**When to Use**: Predictable, high-volume ingestion. If you consistently ingest >100 GB/day, commitment tier saves money.

**Calculation**:
```bash
# Estimate current ingestion
workspace=$(az monitor log-analytics workspace show --resource-group RG-Azure-VDI-01 --name log-avd-prod --query id -o tsv)
az monitor metrics list \
  --resource $workspace \
  --metric "Ingestion" \
  --start-time 2025-12-07T00:00:00Z \
  --end-time 2025-12-14T00:00:00Z \
  --interval PT24H \
  --aggregation Total \
  --output table
# If average > 100 GB/day, consider commitment tier
```

**Change Pricing Tier**:
1. Azure Portal → Log Analytics workspaces → log-avd-prod
2. Usage and estimated costs → Pricing tier
3. Select tier → Save

### 2. Data Collection Rules (DCRs) Filtering

DCRs can filter data **before** ingestion, avoiding charges for unwanted data.

**Example: Exclude verbose IIS logs**:
```json
{
  "dataSources": {
    "iisLogs": [
      {
        "streams": ["Microsoft-W3CIISLog"],
        "logDirectories": ["C:\\inetpub\\logs\\LogFiles"],
        "filter": "sc-status != 200"  // Only collect non-success responses
      }
    ]
  }
}
```

**AVD Use Case**: Exclude successful connection events, keep only errors and slow connections.

### 3. Sample Data Collection

For high-frequency performance counters, sample instead of collecting every interval:

```json
// Instead of every 10 seconds (expensive):
"samplingFrequency": "PT10S"  // 8,640 samples/day per counter

// Use 1-minute sampling:
"samplingFrequency": "PT1M"  // 1,440 samples/day (83% reduction)
```

**Trade-off**: Lower granularity. Acceptable for capacity planning, not for real-time troubleshooting.

### 4. Table-Level Access Control

Use table-level RBAC to prevent unnecessary data collection:
- Grant developers access only to application logs
- Restrict security logs to security team
- Reduces temptation to log excessively if users can't query it

## Basic KQL Queries

Kusto Query Language (KQL) is the query language for Log Analytics. Essential queries for AVD:

### Find Errors in Last 24 Hours

```kql
WVDErrors
| where TimeGenerated > ago(24h)
| summarize ErrorCount = count() by ServiceError, bin(TimeGenerated, 1h)
| order by ErrorCount desc
```

**Explanation**:
- `WVDErrors`: Table containing AVD error events
- `where TimeGenerated > ago(24h)`: Filter to last 24 hours
- `summarize`: Aggregate rows (like SQL GROUP BY)
- `bin(TimeGenerated, 1h)`: Group timestamps into 1-hour buckets
- `order by ErrorCount desc`: Sort by most frequent errors

**Output**:
| TimeGenerated | ServiceError | ErrorCount |
|---------------|--------------|------------|
| 2025-12-14 10:00 | ConnectionFailedClientDisconnect | 23 |
| 2025-12-14 11:00 | AuthenticationFailed | 12 |

### Check Login Patterns

```kql
WVDConnections
| where TimeGenerated > ago(7d)
| where State == "Connected"
| summarize Connections = count() by UserName, bin(TimeGenerated, 1d)
| order by TimeGenerated desc, Connections desc
```

**Use Case**: Identify power users (high connection counts), detect unusual after-hours logins (potential compromise).

### Identify Performance Issues

```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where CounterValue > 80  // CPU > 80%
| summarize AvgCPU = avg(CounterValue), MaxCPU = max(CounterValue) by Computer
| order by MaxCPU desc
```

**Output**:
| Computer | AvgCPU | MaxCPU |
|----------|--------|--------|
| avd-pool-0 | 85.2 | 97.3 |
| avd-pool-1 | 72.1 | 81.5 |

**Action**: Add more session hosts or resize VMs.

### Join Multiple Tables

Find users with slow connections (high RTT) and correlate with session host:

```kql
WVDConnections
| where TimeGenerated > ago(24h)
| where State == "Connected"
| where UdpRoundTripTimeInMS > 150  // RTT > 150ms
| project TimeGenerated, UserName, CorrelationId, UdpRoundTripTimeInMS
| join kind=inner (
    WVDCheckpoints
    | where Name == "SessionHostSelection"
    | project CorrelationId, SessionHostName = tostring(Parameters.SessionHostName)
) on CorrelationId
| summarize AvgRTT = avg(UdpRoundTripTimeInMS), Connections = count() by UserName, SessionHostName
| order by AvgRTT desc
```

**Use Case**: Identify if specific session hosts have network issues or if user's ISP is slow.

### Time-Series Analysis

Visualize CPU usage over time (for dashboard):

```kql
Perf
| where TimeGenerated > ago(24h)
| where ObjectName == "Processor" and CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize AvgCPU = avg(CounterValue) by Computer, bin(TimeGenerated, 10m)
| render timechart
```

**Output**: Line chart with one line per session host showing CPU trend.

## Best Practices

- **Enable Diagnostics on All AVD Resources** - Missing diagnostics means blind spots. Use Azure Policy to enforce diagnostic settings on new resources.

- **Use Descriptive Workspace Names** - `log-avd-prod` is clear. Avoid generic names like `DefaultWorkspace123456`. Use naming conventions.

- **Start with 30-Day Retention, Extend Selectively** - Default 30 days is free. Only extend retention for compliance-required tables (e.g., WVDConnections for 90 days).

- **Monitor Ingestion Costs** - Set budget alert: Azure Portal → Cost Management → Budgets → Create budget for Log Analytics. Alert at 80% of monthly budget.

- **Learn KQL Incrementally** - Start with simple queries (filter, summarize). Use Microsoft's KQL tutorial: https://learn.microsoft.com/azure/data-explorer/kusto/query/tutorial

- **Pin Useful Queries** - Save frequently-used queries as "Functions" in workspace. Example: Create function `GetFailedConnections` for reuse in dashboards.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No data in WVDConnections table | Diagnostic settings not enabled on host pool | Enable diagnostics: Azure Portal → Host pool → Diagnostic settings → Add. Select all log categories. |
| Azure Monitor Agent not collecting data | DCR not associated with VM | Check associations: `az monitor data-collection rule association list --resource [vm-id]`. Create if missing. |
| Queries timeout or return incomplete data | Querying too much data (months of high-volume tables) | Reduce time range: `where TimeGenerated > ago(7d)`. Use `summarize` to aggregate before returning results. |
| High ingestion costs | Collecting verbose logs (e.g., IIS every request) | Use DCR filters to exclude low-value data. Sample performance counters (PT1M instead of PT10S). |
| Can't see workspace in list | Insufficient RBAC permissions | Require "Log Analytics Reader" role or higher. Check: Azure Portal → Workspace → Access control (IAM). |
| Custom logs not appearing | Incorrect Log-Type header or API authentication | Verify shared key: Workspace → Agents → Primary key. Check header: `Log-Type: MyAppLog` (no spaces, alphanumeric only). |

## Related Resources

- Log Analytics Overview: https://learn.microsoft.com/azure/azure-monitor/logs/log-analytics-overview
- KQL Quick Reference: https://learn.microsoft.com/azure/data-explorer/kusto/query/
- Data Collection Rules: https://learn.microsoft.com/azure/azure-monitor/essentials/data-collection-rule-overview
- Pricing Calculator: https://azure.microsoft.com/pricing/calculator/ (select Log Analytics)
- Workspace Design Best Practices: https://learn.microsoft.com/azure/azure-monitor/logs/workspace-design