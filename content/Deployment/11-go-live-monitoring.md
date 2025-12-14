---
title: Step 11 - Go-Live & Monitoring Setup
description: Configure Log Analytics, AVD Insights, alerting, and operational runbooks for production AVD deployment
published: true
date: 2025-12-14
tags: [Quick-Deploy, monitoring, operations, go-live]
---

# Step 11: Go-Live & Monitoring Setup

Configure comprehensive monitoring, alerting, and operational procedures before production go-live. This step sets up Log Analytics workspace, enables AVD Insights dashboards, creates critical alert rules, and provides runbooks for common operational tasks. Completing this step ensures you can proactively detect issues, respond to incidents, and maintain SLA commitments.

## Example Scenario

Using naming conventions from [[00-naming-conventions]]:

| Resource | Name | Purpose |
|----------|------|---------|
| Log Analytics Workspace | `law-avd-prod-eus-01` | Centralized logging for AVD telemetry |
| AVD Insights Workbook | `workbook-avd-insights` | Pre-built AVD monitoring dashboard |
| Action Group | `ag-avd-ops-team` | Email/SMS notifications for alerts |
| Scaling Plan | `sp-pooled-prod` | Auto-start/stop pooled hosts during off-peak |

**Monitoring Coverage:**
- Connection success/failure tracking
- Session host performance (CPU, memory, disk)
- User experience metrics (RTT, input delay)
- FSLogix profile load status
- Capacity utilization and trending

**Alert Scenarios:**
- Session host unavailable (>10 minutes)
- Connection failure rate >10%
- Low disk space (<10% free)
- High CPU sustained >80%
- FSLogix profile load failures

## Prerequisites

- [ ] Host pools deployed: `hp-pooled-prod`, `hp-personal-prod` (from [[06-host-pool-creation]])
- [ ] Session hosts running and joined to host pools (from [[07-session-hosts]])
- [ ] Application groups and workspace configured (from [[08-app-groups-workspace]])
- [ ] Users able to connect and access desktops (tested in Step 08)
- [ ] Contributor or Monitoring Contributor role on resource group

---

## Part 1: Create Log Analytics Workspace

Deploy centralized logging repository for all AVD telemetry.

### Portal: Create Log Analytics Workspace

**Portal:** Azure Portal → Log Analytics workspaces → + Create

1. **Basics:**
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Name:** `law-avd-prod-eus-01`
   - **Region:** East US

2. **Pricing tier:**
   - **Pricing tier:** Pay-as-you-go (default)

   > **Note:** Start with Pay-as-you-go. If ingestion exceeds 100 GB/day consistently, switch to Commitment Tier for cost savings.

3. **Tags:**
   - **Environment:** Production
   - **Purpose:** AVD-Monitoring
   - **Owner:** IT-Operations

4. Click **Review + create**
5. Click **Create**

**Deployment time:** ~2 minutes

### Configure Data Retention

**Portal:** Azure Portal → Log Analytics workspaces → law-avd-prod-eus-01 → Usage and estimated costs

1. Click **Data Retention**
2. Set retention slider: **90 days**

   > **Why 90 days?** Supports quarterly trend analysis and compliance. Default 30 days may be insufficient for capacity planning.

3. Click **OK**

**Cost Impact:** Retention beyond 30 days incurs additional charges (~$0.10/GB/month). Typical AVD deployment: 5-10 GB/month = $5-10/month for extended retention.

### Verify Workspace

**Portal:** Azure Portal → Log Analytics workspaces → law-avd-prod-eus-01 → Overview

- [ ] **Status:** Active
- [ ] **Data retention:** 90 days
- [ ] **Location:** East US
- [ ] **Resource group:** rg-avd-prod-eastus-01

**Copy Workspace ID for later:**

**Portal:** law-avd-prod-eus-01 → Agents → Workspace ID

```
Workspace ID: ________________________________________
Primary Key: ________________________________________
(save securely - needed for agent configuration)
```

---

## Part 2: Enable Diagnostic Settings on AVD Resources

Configure all AVD resources to send logs and metrics to Log Analytics.

### Enable Diagnostics on Host Pool (Pooled)

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-pooled-prod → Diagnostic settings

1. Click **+ Add diagnostic setting**
2. **Diagnostic setting name:** `SendToLogAnalytics`
3. **Logs - Categories:**
   - [x] Connection
   - [x] Error
   - [x] Checkpoint
   - [x] Management
   - [x] AgentHealthStatus

   > **Note:** Enable ALL log categories for complete visibility. Each category serves specific troubleshooting needs.

4. **Destination details:**
   - [x] Send to Log Analytics workspace
   - **Subscription:** Select your subscription
   - **Log Analytics workspace:** law-avd-prod-eus-01

5. Click **Save**

### Enable Diagnostics on Host Pool (Personal)

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-personal-prod → Diagnostic settings

Repeat the same configuration:
- **Name:** `SendToLogAnalytics`
- **Logs:** All categories (Connection, Error, Checkpoint, Management, AgentHealthStatus)
- **Destination:** law-avd-prod-eus-01

### Enable Diagnostics on Application Groups

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-pooled-prod → Diagnostic settings

1. Click **+ Add diagnostic setting**
2. **Diagnostic setting name:** `SendToLogAnalytics`
3. **Logs - Categories:**
   - [x] Checkpoint
   - [x] Error
   - [x] Management

4. **Destination:** law-avd-prod-eus-01
5. Click **Save**

**Repeat for ag-personal-prod:**

**Portal:** Application groups → ag-personal-prod → Diagnostic settings

Same configuration as ag-pooled-prod.

### Enable Diagnostics on Workspace

**Portal:** Azure Portal → Virtual Desktop → Workspaces → ws-prod → Diagnostic settings

1. Click **+ Add diagnostic setting**
2. **Diagnostic setting name:** `SendToLogAnalytics`
3. **Logs - Categories:**
   - [x] Checkpoint
   - [x] Error
   - [x] Management
   - [x] Feed

4. **Destination:** law-avd-prod-eus-01
5. Click **Save**

### Verification: Check Data Flow

Wait 10-15 minutes for data ingestion, then verify:

**Portal:** Log Analytics workspaces → law-avd-prod-eus-01 → Logs

Run query:

```kql
WVDConnections
| where TimeGenerated > ago(1h)
| summarize count() by _ResourceId
| order by count_ desc
```

**Expected Result:** Shows connection events from hp-pooled-prod and hp-personal-prod.

If no results:
- Wait another 10 minutes (initial ingestion delay)
- Verify diagnostic settings saved correctly
- Check that users have connected to AVD (no connections = no logs)

---

## Part 3: Deploy Azure Monitor Agent to Session Hosts

Install Azure Monitor Agent on all session host VMs to collect performance counters and Windows Event Logs.

### Install Azure Monitor Agent Extension

Use Azure Portal to deploy agent to all session hosts at once.

**Portal:** Azure Portal → Virtual machines → Select multiple VMs

1. Filter by tag: **HostPool = hp-pooled-prod** (or use resource group filter)
2. Select all pooled session hosts:
   - vm-pooled-prod-001
   - vm-pooled-prod-002
   - ...
   - vm-pooled-prod-010

3. Click **Extensions + applications**
4. Click **+ Add**
5. Search for: `AzureMonitorWindowsAgent`
6. Select **AzureMonitorWindowsAgent** by Microsoft.Azure.Monitor
7. Click **Next**
8. **Enable automatic upgrade:** Yes
9. Click **Review + create**
10. Click **Create**

**Deployment time:** ~5 minutes per VM (runs in parallel)

**Repeat for personal session hosts:**

Select all personal session hosts (vm-personal-prod-011 through vm-personal-prod-050) and install AzureMonitorWindowsAgent.

### Create Data Collection Rule

Data Collection Rules (DCRs) define what performance counters and logs to collect.

**Portal:** Azure Portal → Monitor → Data collection rules → + Create

1. **Basics:**
   - **Rule name:** `dcr-avd-prod-sessionhosts`
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Region:** East US
   - **Platform Type:** Windows

2. **Resources:**
   - Click **+ Add resources**
   - **Scope:** Resource group = rg-avd-prod-eastus-01
   - Filter: **Virtual machines**
   - Select all session host VMs (pooled + personal)
   - Click **Apply**

3. **Collect and deliver:**

   **Data source 1: Performance Counters**

   - Click **+ Add data source**
   - **Data source type:** Performance Counters
   - **Sampling frequency:** 60 seconds
   - **Counters:**
     - [x] Processor(_Total)\% Processor Time
     - [x] Memory\Available MBytes
     - [x] LogicalDisk(C:)\% Free Space
     - [x] LogicalDisk(C:)\Avg. Disk sec/Read
     - [x] LogicalDisk(C:)\Avg. Disk sec/Write
     - [x] Network Interface(*)\Bytes Total/sec
     - [x] User Input Delay per Session(*)\Max Input Delay

   - **Destination:**
     - **Destination type:** Azure Monitor Logs
     - **Subscription:** Select your subscription
     - **Account or namespace:** law-avd-prod-eus-01

   - Click **Add data source**

   **Data source 2: Windows Event Logs**

   - Click **+ Add data source**
   - **Data source type:** Windows Event Logs
   - **Application logs:**
     - [x] Critical
     - [x] Error
     - [x] Warning
   - **System logs:**
     - [x] Critical
     - [x] Error
     - [x] Warning

   - **Destination:** law-avd-prod-eus-01
   - Click **Add data source**

4. **Tags:**
   - **Environment:** Production
   - **Purpose:** AVD-Monitoring

5. Click **Review + create**
6. Click **Create**

**Deployment time:** ~5 minutes

### Verify Agent Installation and Data Collection

**Portal:** Virtual machines → vm-pooled-prod-001 → Extensions + applications

- [ ] **AzureMonitorWindowsAgent** shows Status = "Provisioning succeeded"
- [ ] **Version:** Latest (e.g., 1.24.0)

**Verify data collection:**

**Portal:** Log Analytics workspaces → law-avd-prod-eus-01 → Logs

Run query:

```kql
Perf
| where TimeGenerated > ago(30m)
| where Computer startswith "vm-pooled-prod-" or Computer startswith "vm-personal-prod-"
| summarize count() by Computer, ObjectName, CounterName
| order by Computer asc
```

**Expected Result:** Shows performance counters from all session hosts.

---

## Part 4: Enable AVD Insights

Deploy pre-built AVD Insights workbook for comprehensive monitoring dashboards.

### Configure AVD Insights

**Portal:** Azure Portal → Azure Virtual Desktop → Insights

1. **Get started with Azure Virtual Desktop Insights**
2. Click **Open configuration workbook**
3. **Select subscription:** Your subscription
4. **Select host pool:** hp-pooled-prod (repeat for hp-personal-prod)
5. **Select workspace:** law-avd-prod-eus-01
6. Click **Configure**

**Configuration process:**
- Enables diagnostic settings on AVD resources (already done in Part 2)
- Deploys AVD-specific Data Collection Rule (DCR)
- Associates DCR with session host VMs

**Deployment time:** ~5 minutes

### Open AVD Insights Workbook

**Portal:** Azure Virtual Desktop → Insights

1. **Time range:** Last 24 hours
2. **Host pool:** All (or select hp-pooled-prod)

**Workbook Sections:**

**Overview Tab:**
- Connection success rate (target: >95%)
- Active users
- Session host availability
- Top errors in last 24 hours

**Connection Diagnostics Tab:**
- Failed connection attempts by user
- Connection stages (feed, auth, gateway, broker, session host)
- Error code distribution

**Connection Performance Tab:**
- Round-trip time (RTT) by user location
- Input delay per session host
- Bandwidth utilization

**Host Diagnostics Tab:**
- CPU utilization per session host
- Memory available
- Disk space free
- Active sessions per host

**User Reports Tab:**
- Session duration by user
- Connection frequency
- Geographic distribution

> **Best Practice:** Review AVD Insights daily for 5-10 minutes. Establish baseline metrics during first 2 weeks of production.

### Verify AVD Insights Data

**Portal:** Azure Virtual Desktop → Insights → Overview

**Expected:**
- Connection success rate shows percentage (may show "No data" if no connections yet)
- Session hosts listed with health status
- Active user count (0 if none connected)

**If "No data available":**
- Wait 15-30 minutes for initial data ingestion
- Verify diagnostic settings enabled (Part 2)
- Verify Azure Monitor Agent installed (Part 3)
- Check that users have connected to AVD

---

## Part 5: Create Action Groups

Define notification and automation actions for alert rules.

### Create Action Group for Operations Team

**Portal:** Azure Portal → Monitor → Alerts → Action groups → + Create

1. **Basics:**
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Action group name:** `ag-avd-ops-team`
   - **Display name:** `AVD Ops Team`
   - **Region:** Global

2. **Notifications:**

   Click **+ Add notification**

   **Notification 1: Email**
   - **Notification type:** Email/SMS message/Push/Voice
   - **Email:** ops@contoso.com (replace with your ops email)
   - **Name:** Email-OpsTeam

   Click **OK**

   **Notification 2: SMS (Optional - for critical alerts only)**
   - **Notification type:** Email/SMS message/Push/Voice
   - [x] SMS
   - **Country code:** +1
   - **Phone number:** 555-123-4567 (replace with on-call number)
   - **Name:** SMS-OnCall

   Click **OK**

3. **Actions:**

   Click **+ Add action**

   **Action 1: Webhook to Teams/Slack (Optional)**
   - **Action type:** Webhook
   - **Name:** Webhook-Teams
   - **URI:** https://your-teams-webhook-url
   - **Enable common alert schema:** Yes

   Click **OK**

4. **Tags:**
   - **Environment:** Production
   - **Purpose:** AVD-Alerting

5. Click **Review + create**
6. Click **Create**

### Create Action Group for Security Team (Optional)

For security-related alerts (unauthorized changes, NSG modifications):

**Portal:** Monitor → Alerts → Action groups → + Create

1. **Action group name:** `ag-avd-security-team`
2. **Notifications:**
   - Email: security@contoso.com
3. Click **Review + create**

---

## Part 6: Configure Alert Rules

Create critical alert rules for proactive monitoring.

### Alert 1: Session Host Unavailable

**Condition:** Session host not reporting to AVD control plane for >10 minutes

**Portal:** Monitor → Alerts → + Create → Alert rule

1. **Scope:**
   - Click **Select scope**
   - **Resource type:** Log Analytics workspaces
   - Select: law-avd-prod-eus-01
   - Click **Done**

2. **Condition:**
   - Click **Add condition**
   - **Signal type:** Log
   - Search for: `Custom log search`
   - **Query:**

   ```kql
   WVDAgentHealthStatus
   | where TimeGenerated > ago(10m)
   | where Status != "Available"
   | summarize UnavailableHosts = dcount(SessionHostName) by SessionHostName, Status
   | where UnavailableHosts > 0
   ```

   - **Threshold:** Static
   - **Operator:** Greater than
   - **Threshold value:** 0
   - **Evaluation frequency:** 5 minutes
   - **Lookback period:** 10 minutes

   - Click **Done**

3. **Actions:**
   - Click **Add action groups**
   - Select: ag-avd-ops-team
   - Click **Select**

4. **Alert rule details:**
   - **Alert rule name:** `AVD-SessionHost-Unavailable`
   - **Description:** Session host not responding or offline for more than 10 minutes
   - **Severity:** Warning (Sev 2)
   - **Enable alert rule:** Yes
   - **Resource group:** rg-avd-prod-eastus-01

5. Click **Create alert rule**

### Alert 2: High Connection Failure Rate

**Condition:** >10 failed connections in 5 minutes

**Portal:** Monitor → Alerts → + Create → Alert rule

1. **Scope:** law-avd-prod-eus-01

2. **Condition:**

   **Query:**

   ```kql
   WVDConnections
   | where TimeGenerated > ago(5m)
   | where State == "Failed"
   | summarize FailureCount = count()
   ```

   - **Threshold:** Greater than 10
   - **Evaluation frequency:** 5 minutes
   - **Lookback period:** 5 minutes

3. **Actions:** ag-avd-ops-team

4. **Alert rule details:**
   - **Name:** `AVD-HighConnectionFailureRate`
   - **Description:** More than 10 failed AVD connections in 5 minutes
   - **Severity:** Warning (Sev 2)

5. Click **Create alert rule**

### Alert 3: Low Disk Space

**Condition:** C: drive <10% free space on session host

**Portal:** Monitor → Alerts → + Create → Alert rule

1. **Scope:**
   - **Resource type:** Virtual machines
   - Select all session hosts (vm-pooled-prod-*, vm-personal-prod-*)

2. **Condition:**
   - **Signal:** Percentage CPU (Metric)
   - Change to: **OS Disk Free Space Percentage** (if available)

   > **Note:** If metric not available, create log-based alert instead.

   **Alternative: Log-based alert**

   **Query:**

   ```kql
   Perf
   | where TimeGenerated > ago(15m)
   | where ObjectName == "LogicalDisk" and CounterName == "% Free Space"
   | where InstanceName == "C:"
   | where CounterValue < 10
   | summarize AvgFreeSpace = avg(CounterValue) by Computer
   ```

   - **Threshold:** Greater than 0 (means at least one host <10%)
   - **Evaluation frequency:** 15 minutes
   - **Lookback period:** 15 minutes

3. **Actions:** ag-avd-ops-team

4. **Alert rule details:**
   - **Name:** `AVD-LowDiskSpace`
   - **Description:** Session host C: drive has less than 10% free space
   - **Severity:** Warning (Sev 2)

5. Click **Create alert rule**

### Alert 4: High CPU Sustained

**Condition:** CPU >80% for 10 minutes

**Portal:** Monitor → Alerts → + Create → Alert rule

1. **Scope:** All session host VMs

2. **Condition:**
   - **Signal:** Percentage CPU (Metric)
   - **Threshold:** Static
   - **Operator:** Greater than
   - **Threshold value:** 80
   - **Aggregation type:** Average
   - **Evaluation frequency:** 5 minutes
   - **Lookback period:** 10 minutes

3. **Actions:** ag-avd-ops-team

4. **Alert rule details:**
   - **Name:** `AVD-HighCPU`
   - **Description:** Session host CPU usage sustained above 80% for 10 minutes
   - **Severity:** Warning (Sev 2)

5. Click **Create alert rule**

### Alert 5: FSLogix Profile Load Failure

**Condition:** FSLogix profile fails to attach

**Portal:** Monitor → Alerts → + Create → Alert rule

1. **Scope:** law-avd-prod-eus-01

2. **Condition:**

   **Query:**

   ```kql
   Event
   | where TimeGenerated > ago(5m)
   | where Source == "FSLogix-Profile" and EventLevelName == "Error"
   | where RenderedDescription contains "Failed to attach VHD" or RenderedDescription contains "Profile load failed"
   | summarize FailureCount = count() by Computer
   ```

   - **Threshold:** Greater than 0
   - **Evaluation frequency:** 5 minutes
   - **Lookback period:** 5 minutes

3. **Actions:** ag-avd-ops-team

4. **Alert rule details:**
   - **Name:** `AVD-FSLogixProfileFailure`
   - **Description:** FSLogix profile failed to load on session host
   - **Severity:** Error (Sev 1)

5. Click **Create alert rule**

### Verify Alert Rules

**Portal:** Monitor → Alerts → Alert rules

- [ ] AVD-SessionHost-Unavailable (Sev 2, Enabled)
- [ ] AVD-HighConnectionFailureRate (Sev 2, Enabled)
- [ ] AVD-LowDiskSpace (Sev 2, Enabled)
- [ ] AVD-HighCPU (Sev 2, Enabled)
- [ ] AVD-FSLogixProfileFailure (Sev 1, Enabled)

**Test an alert (optional):**

**Portal:** Alert rule → Test

1. Select: AVD-HighCPU
2. Click **Test**
3. Verify email received at ops@contoso.com

---

## Part 7: Configure Scaling Plan (Pooled Host Pool Only)

Create scaling plan to auto-start/stop pooled session hosts based on schedule, reducing costs during off-peak hours.

> **Note:** Scaling plans apply ONLY to pooled host pools. Personal host pools use "Start VM on Connect" instead.

### Portal: Create Scaling Plan

**Portal:** Azure Portal → Virtual Desktop → Scaling plans → + Create

1. **Basics:**
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Name:** `sp-pooled-prod`
   - **Location:** East US
   - **Friendly name:** `Pooled Host Pool Business Hours`
   - **Time zone:** (UTC-05:00) Eastern Time (US & Canada)

2. **Schedules:**

   **Schedule 1: Weekdays (Monday-Friday)**

   Click **+ Add schedule**

   **Schedule name:** Weekday-Schedule

   **Days:** Monday, Tuesday, Wednesday, Thursday, Friday

   **Ramp-up:**
   - **Start time:** 07:00
   - **Load balancing algorithm:** Breadth-first
   - **Minimum % hosts:** 60%
   - **Capacity threshold:** 75%

   **Peak hours:**
   - **Start time:** 09:00
   - **Load balancing algorithm:** Depth-first
   - **Minimum % hosts:** 80%

   **Ramp-down:**
   - **Start time:** 17:00
   - **Load balancing algorithm:** Depth-first
   - **Minimum % hosts:** 20%
   - **Capacity threshold:** 50%
   - **Stop hosts when:** Users log off (wait for sessions to end)

   **Off-peak:**
   - **Start time:** 19:00
   - **Load balancing algorithm:** Depth-first
   - **Minimum % hosts:** 20%

   Click **Add**

   **Schedule 2: Weekends (Saturday-Sunday)**

   Click **+ Add schedule**

   **Schedule name:** Weekend-Schedule

   **Days:** Saturday, Sunday

   **Ramp-up:**
   - **Start time:** 09:00
   - **Minimum % hosts:** 20%
   - **Capacity threshold:** 80%

   **Peak hours:**
   - **Start time:** 11:00
   - **Minimum % hosts:** 20%

   **Ramp-down:**
   - **Start time:** 17:00
   - **Minimum % hosts:** 10%

   **Off-peak:**
   - **Start time:** 19:00
   - **Minimum % hosts:** 10%

   Click **Add**

3. **Host pool assignments:**
   - Click **+ Add host pool**
   - Select: hp-pooled-prod
   - **Enable autoscale:** Yes
   - Click **Add**

4. **Tags:**
   - **Environment:** Production
   - **Purpose:** AVD-Scaling

5. Click **Review + create**
6. Click **Create**

**Deployment time:** ~1 minute

### Verify Scaling Plan

**Portal:** Virtual Desktop → Scaling plans → sp-pooled-prod

- [ ] **Status:** Enabled
- [ ] **Host pools assigned:** hp-pooled-prod
- [ ] **Schedules:** Weekday-Schedule, Weekend-Schedule

**Monitor scaling actions:**

Wait until next scheduled phase transition (e.g., 7:00 AM, 9:00 AM, 5:00 PM) and check:

**Portal:** Virtual Desktop → Host pools → hp-pooled-prod → Session hosts

**Expected behavior:**
- **7:00 AM:** 60% of hosts start (e.g., 6 out of 10 VMs)
- **9:00 AM:** 80% of hosts running (8 out of 10 VMs)
- **5:00 PM:** Hosts with zero sessions begin deallocating
- **7:00 PM:** Only 20% remain running (2 VMs)

**Cost Savings Estimate:**

Host Pool: 10 session hosts, Standard_D4s_v5 ($0.192/hour)

**Without Scaling Plan:**
- 10 VMs × 24 hours × 30 days × $0.192/hour = $1,382.40/month

**With Scaling Plan:**
- Peak (9 AM-5 PM, 8 hours): 8 VMs = 64 VM-hours/day
- Ramp-Up/Down (7-9 AM, 5-7 PM, 4 hours): 6 VMs avg = 24 VM-hours/day
- Off-Peak (7 PM-7 AM, 12 hours): 2 VMs = 24 VM-hours/day
- Total: 112 VM-hours/day × 30 days × $0.192 = $645.12/month

**Savings:** $737.28/month (53% reduction)

**See:** [[../AVD/scaling-plans|Scaling Plans]] for advanced configuration and troubleshooting.

---

## Part 8: User Acceptance Testing Checklist

Conduct user acceptance testing before full production rollout.

### UAT Test Plan

**Test Group:** 10-15 pilot users representing different departments and use cases

**Test Duration:** 1-2 weeks

**Test Scenarios:**

#### Scenario 1: Initial Connection

**Pooled Users:**
- [ ] User opens Remote Desktop client
- [ ] User enters email (auto-discovery finds workspace)
- [ ] User authenticates with Entra ID credentials
- [ ] MFA challenge completes successfully
- [ ] Workspace displays "Pooled Production Desktop" icon
- [ ] User double-clicks desktop icon
- [ ] Connection establishes within 10 seconds
- [ ] Full Windows desktop appears
- [ ] Start menu, taskbar, and applications load

**Personal Users:**
- [ ] Same connection flow as pooled
- [ ] Workspace displays "Personal Workstation" icon
- [ ] User connects to dedicated session host
- [ ] Verify VM name matches user assignment (Portal → Host pools → Session hosts)

#### Scenario 2: Application Functionality

- [ ] Microsoft Office apps launch (Word, Excel, Outlook)
- [ ] Email (Outlook) accesses mailbox
- [ ] OneDrive/SharePoint files accessible
- [ ] Web browser (Edge/Chrome) works
- [ ] Line-of-business applications launch
- [ ] Printing works (redirect to local printer or cloud printer)
- [ ] Copy/paste between local and remote desktops works

#### Scenario 3: Profile Persistence

**Day 1:**
- [ ] User creates file on desktop: `Test-Profile.txt`
- [ ] User changes wallpaper
- [ ] User pins application to taskbar
- [ ] User logs off

**Day 2:**
- [ ] User reconnects to AVD
- [ ] `Test-Profile.txt` still on desktop
- [ ] Wallpaper unchanged
- [ ] Pinned application persists

**This confirms FSLogix profile roaming is working.**

#### Scenario 4: Performance

- [ ] Typing in Word feels responsive (<50ms input delay)
- [ ] Mouse movement smooth (no lag)
- [ ] Video playback works (YouTube, Teams calls)
- [ ] Screen refresh feels native (no visible compression artifacts)

**Check AVD Insights → Connection Performance:**
- [ ] Round-trip time (RTT) <150ms for 95% of connections
- [ ] Input delay <100ms average

#### Scenario 5: Multi-Session (Pooled Only)

- [ ] User logs off pooled desktop
- [ ] User reconnects
- [ ] User may connect to different session host (breadth-first load balancing)
- [ ] Profile still loads correctly
- [ ] No data loss

#### Scenario 6: Disconnection Handling

- [ ] User closes Remote Desktop client without logging off
- [ ] Session enters "Disconnected" state
- [ ] User reconnects within 5 minutes
- [ ] Session resumes exactly where left off (same applications open)

#### Scenario 7: Different Clients

Test connection from:
- [ ] Windows desktop client (aka.ms/rdwindows)
- [ ] macOS client (aka.ms/rdmac)
- [ ] Web client (client.wvd.microsoft.com)
- [ ] Mobile (iOS/Android)

### UAT Issue Tracking

Document all issues found during UAT:

| Issue ID | Description | Severity | Status | Resolution |
|----------|-------------|----------|--------|------------|
| UAT-001 | Outlook slow to open | Low | Closed | Increased VM RAM |
| UAT-002 | Printer redirection not working | Medium | Open | Investigating GPO |
| UAT-003 | Profile load >30 seconds | High | Closed | Moved to Premium storage |

**Acceptance Criteria:**
- [ ] Zero severity High issues
- [ ] <3 severity Medium issues
- [ ] >90% user satisfaction (survey)
- [ ] All test scenarios pass

---

## Part 9: Operations Runbook

Document common operational tasks for IT team.

### Runbook 1: Add New User to Pooled Desktop

**When:** New employee needs AVD access

**Steps:**

1. **Create user in Entra ID** (if not already created)

   **Portal:** Entra Admin Center → Users → + New user

2. **Assign M365 E3 license**

   **Portal:** Entra Admin Center → Users → [User] → Licenses → + Assignments

3. **Add user to Entra ID group**

   **Portal:** Entra Admin Center → Groups → AVD-Pooled-Users → Members → + Add members

4. **Verify RBAC assignment**

   **Portal:** Virtual Desktop → Application groups → ag-pooled-prod → Access control (IAM)

   Confirm: AVD-Pooled-Users has "Desktop Virtualization User" role

5. **Test user access**

   Have user connect via Remote Desktop client and verify:
   - Workspace appears
   - Desktop launches
   - Profile loads

**Time required:** 10 minutes

### Runbook 2: Add New Session Host to Pooled Host Pool

**When:** Need more capacity (users reporting "No resources available")

**Steps:**

1. **Generate new registration token**

   **Portal:** Virtual Desktop → Host pools → hp-pooled-prod → Properties

   Click **Generate new key** → Set expiration 24 hours → Copy token

2. **Deploy new VM from Azure Compute Gallery**

   **Portal:** Virtual machines → + Create

   - **Name:** vm-pooled-prod-011 (next in sequence)
   - **Image:** gal-avd-prod-eus-01 → win11-multisession-23h2 (latest version)
   - **Size:** Standard_D4s_v5
   - **Networking:** vnet-avd-prod-eastus-01 / snet-avd-prod-sessionhosts
   - **Join type:** Entra ID Join
   - **Enroll in Intune:** Yes

   **Extensions:**
   - AVDAgent (use registration token from step 1)
   - AzureMonitorWindowsAgent

3. **Verify session host joined host pool**

   **Portal:** Virtual Desktop → Host pools → hp-pooled-prod → Session hosts

   Wait 5-10 minutes, then confirm:
   - vm-pooled-prod-011 appears in list
   - Status = "Available"
   - Drain mode = Disabled

4. **Test connection**

   Have test user connect and verify:
   - New session can be assigned to vm-pooled-prod-011
   - User can access desktop normally

**Time required:** 30 minutes (mostly VM provisioning)

**See:** [[07-session-hosts|Session Host Deployment]] for detailed steps.

### Runbook 3: Troubleshoot User Connection Failure

**Symptom:** User reports "Can't connect to workspace" or "No resources available"

**Troubleshooting Steps:**

1. **Verify user account status**

   **Portal:** Entra Admin Center → Users → [Search user]

   Check:
   - [ ] Account enabled (not blocked)
   - [ ] License assigned (M365 E3)
   - [ ] MFA registered

2. **Verify group membership**

   **Portal:** Entra Admin Center → Users → [User] → Groups

   Check:
   - [ ] User is member of AVD-Pooled-Users or AVD-Personal-Users

3. **Verify RBAC permissions**

   **Portal:** Virtual Desktop → Application groups → ag-pooled-prod → Access control (IAM)

   Check:
   - [ ] AVD-Pooled-Users has "Desktop Virtualization User" role

4. **Check application group registration**

   **Portal:** Virtual Desktop → Workspaces → ws-prod → Application groups

   Check:
   - [ ] ag-pooled-prod registered
   - [ ] ag-personal-prod registered (if applicable)

5. **Check session host availability**

   **Portal:** Virtual Desktop → Host pools → hp-pooled-prod → Session hosts

   Check:
   - [ ] At least one session host shows Status = "Available"
   - [ ] No hosts in drain mode
   - [ ] Hosts not at max session limit

6. **Check diagnostic logs**

   **Portal:** Log Analytics workspaces → law-avd-prod-eus-01 → Logs

   Query:

   ```kql
   WVDConnections
   | where TimeGenerated > ago(1h)
   | where UserName contains "[username]"
   | project TimeGenerated, State, ServiceError, CorrelationId
   | order by TimeGenerated desc
   ```

   Look for:
   - State = "Failed"
   - ServiceError (e.g., "NoCandidateSessionHostFound", "AuthenticationFailed")

7. **Common fixes:**

   - **No resources available:** Add session hosts or start deallocated VMs
   - **Authentication failed:** Reset user password, re-register MFA
   - **Workspace not found:** Verify user email domain matches Entra ID tenant

**Time required:** 15-30 minutes

### Runbook 4: Update Golden Image

**When:** Monthly patching or application updates needed

**Steps:**

1. **Create new VM from current image version**

   **Portal:** Virtual machines → + Create

   - **Name:** vm-image-builder
   - **Image:** Latest version from gal-avd-prod-eus-01
   - Do NOT join to host pool

2. **Install updates and applications**

   - RDP to vm-image-builder as local admin
   - Install Windows updates
   - Update Microsoft 365 Apps
   - Install/update LOB applications
   - Test all applications launch

3. **Generalize VM with Sysprep**

   Run on vm-image-builder:

   ```cmd
   C:\Windows\System32\Sysprep\sysprep.exe /oobe /generalize /shutdown
   ```

4. **Create new image version**

   **Portal:** Virtual machines → vm-image-builder → Capture

   - **Share image to Azure Compute Gallery:** Yes
   - **Target gallery:** gal-avd-prod-eus-01
   - **Image definition:** win11-multisession-23h2
   - **Version number:** Increment (e.g., 1.0.1 → 1.0.2)
   - Click **Review + create**

5. **Test new image**

   Deploy one session host with new image version:
   - Verify it joins host pool
   - Test user can connect
   - Verify all applications work

6. **Roll out to production**

   **Option A: Drain and replace (zero downtime)**

   - Set old session hosts to drain mode
   - Wait for users to log off
   - Delete old VMs
   - Deploy new VMs with updated image

   **Option B: Maintenance window (faster)**

   - Schedule maintenance window (e.g., Sunday 2 AM - 6 AM)
   - Delete all session hosts
   - Deploy new session hosts with updated image

**Time required:** 4-6 hours (including testing)

**See:** [[../Images/golden-image-management|Golden Image Management]] for best practices.

### Runbook 5: Review AVD Insights and Capacity Planning

**When:** Weekly (every Monday morning)

**Steps:**

1. **Open AVD Insights**

   **Portal:** Azure Virtual Desktop → Insights

   **Time range:** Last 7 days

2. **Review Overview tab**

   Check:
   - [ ] Connection success rate >95%
   - [ ] Active users trending (compare to previous week)
   - [ ] Session host availability (all hosts healthy)
   - [ ] Top errors (any new error patterns)

3. **Review Connection Performance tab**

   Check:
   - [ ] Average RTT <100ms
   - [ ] 95th percentile RTT <150ms
   - [ ] Input delay <100ms
   - [ ] No users with consistently poor performance

4. **Review Host Diagnostics tab**

   Check CPU/Memory utilization:

   **Action if CPU consistently >70%:**
   - Add more session hosts, OR
   - Resize VMs to larger SKU, OR
   - Reduce max sessions per host

   **Action if Memory <20% free:**
   - Add more session hosts, OR
   - Resize VMs to larger SKU

5. **Review User Reports tab**

   Check:
   - Top users by session hours (identify power users)
   - Connection patterns (peak hours)
   - Geographic distribution (plan regional deployments)

6. **Capacity forecast**

   Query active sessions over time:

   ```kql
   WVDCheckpoints
   | where TimeGenerated > ago(30d)
   | where Name == "SessionHostChange"
   | summarize ActiveSessions = dcount(CorrelationId) by bin(TimeGenerated, 1h), HostPoolName
   | summarize AvgSessions = avg(ActiveSessions), MaxSessions = max(ActiveSessions) by HostPoolName
   ```

   **Action if MaxSessions approaching capacity:**
   - Plan to add session hosts in next sprint

7. **Document findings**

   Create weekly report:
   - Connection success rate: 98.2% (target: >95%)
   - Peak concurrent users: 142 (capacity: 200)
   - Average RTT: 45ms
   - Session host CPU: 55% average
   - Action items: None (metrics healthy)

**Time required:** 15-30 minutes

---

## Part 10: Go-Live Checklist

Complete final verification before production rollout.

### Pre-Go-Live Verification

#### Monitoring & Alerting

- [ ] Log Analytics workspace deployed and receiving data
- [ ] AVD diagnostic settings enabled (host pools, app groups, workspace)
- [ ] Azure Monitor Agent installed on all session hosts
- [ ] Data Collection Rule (DCR) configured and collecting performance counters
- [ ] AVD Insights workbook displays data
- [ ] Action groups created (ag-avd-ops-team)
- [ ] Alert rules created and enabled:
  - [ ] Session host unavailable
  - [ ] High connection failure rate
  - [ ] Low disk space
  - [ ] High CPU
  - [ ] FSLogix profile failure
- [ ] Test alert sent and received

#### Scaling & Cost Optimization

- [ ] Scaling plan created for pooled host pool
- [ ] Scaling schedules configured (weekday/weekend)
- [ ] Autoscale enabled and verified (VMs start/stop as scheduled)
- [ ] Personal host pools have "Start VM on Connect" enabled

#### User Acceptance Testing

- [ ] UAT completed with pilot users
- [ ] All test scenarios passed
- [ ] No high-severity issues outstanding
- [ ] User feedback collected and addressed
- [ ] UAT sign-off obtained from business stakeholders

#### Operations Readiness

- [ ] Operations runbooks documented
- [ ] IT team trained on:
  - [ ] AVD Insights dashboard review
  - [ ] Adding new users
  - [ ] Adding session hosts
  - [ ] Troubleshooting connections
  - [ ] Golden image updates
- [ ] On-call schedule established
- [ ] Escalation paths defined
- [ ] Support ticket process documented

#### Documentation

- [ ] Network diagram created
- [ ] Resource naming documented
- [ ] User communication prepared (launch email)
- [ ] Quick start guide for users created
- [ ] FAQ document created

### Go-Live Plan

**Timeline:**

**Week -1 (Pre-Go-Live):**
- Complete UAT with pilot users
- Fix all high/medium severity issues
- Conduct IT team training
- Send user communication (go-live date announcement)

**Day 0 (Go-Live Day):**
- Morning: Final verification of all monitoring
- 8:00 AM: Enable access for first 25% of users
- Monitor AVD Insights for issues
- 12:00 PM: Enable access for next 25% of users (if no issues)
- 3:00 PM: Enable access for remaining 50% of users

**Day 1-7 (Post-Go-Live):**
- Daily review of AVD Insights (morning standup)
- Daily check of alert history (any fires overnight)
- Collect user feedback via survey
- Address issues in order of severity

**Week 2:**
- Weekly capacity review
- Adjust scaling plan based on actual usage patterns
- Fine-tune alert thresholds (reduce false positives)
- Celebrate success with team

---

## Troubleshooting

### Issue: No Data in Log Analytics

**Symptom:** Queries in Log Analytics return no results

**Cause:**
- Diagnostic settings not enabled
- Data ingestion delay (10-15 minutes)
- Query time range too narrow

**Fix:**

1. Verify diagnostic settings:

   **Portal:** Virtual Desktop → Host pools → hp-pooled-prod → Diagnostic settings

   Check: "SendToLogAnalytics" exists with all log categories enabled

2. Wait 15 minutes and retry query

3. Expand query time range:

   ```kql
   WVDConnections
   | where TimeGenerated > ago(24h)  // Change from ago(1h) to ago(24h)
   | summarize count()
   ```

4. If still no data, check that users have connected to AVD (no connections = no logs)

### Issue: AVD Insights Shows "No Data Available"

**Symptom:** AVD Insights workbook displays "No data available for this time range"

**Cause:**
- Azure Monitor Agent not installed on session hosts
- Data Collection Rule not associated with VMs
- No user connections in selected time range

**Fix:**

1. Verify Azure Monitor Agent installed:

   **Portal:** Virtual machines → vm-pooled-prod-001 → Extensions

   Check: AzureMonitorWindowsAgent shows "Provisioning succeeded"

2. Verify DCR association:

   **Portal:** Monitor → Data collection rules → dcr-avd-prod-sessionhosts → Resources

   Check: All session host VMs listed

3. Check for user connections:

   **Portal:** Virtual Desktop → Host pools → hp-pooled-prod → Session hosts

   Verify: At least one session shows "Active sessions" > 0

4. Expand time range in AVD Insights to "Last 7 days"

### Issue: Alert Not Firing

**Symptom:** Expected alert condition met but no email/notification received

**Cause:**
- Alert rule disabled
- Action group not configured
- Condition not actually met (check query results)
- Email blocked by spam filter

**Fix:**

1. Verify alert enabled:

   **Portal:** Monitor → Alerts → Alert rules → [Alert name]

   Check: "Enabled" toggle is On

2. Verify action group attached:

   **Portal:** Alert rule → Actions tab

   Check: ag-avd-ops-team listed

3. Test alert:

   **Portal:** Alert rule → Test

   Verify: Email received within 5 minutes

4. Check spam/junk folder for alert emails

5. Verify email address correct in action group

### Issue: Scaling Plan Not Starting VMs

**Symptom:** VMs remain deallocated during scheduled ramp-up time

**Cause:**
- Scaling plan not assigned to host pool
- Time zone mismatch
- "Enable autoscale" set to No
- Insufficient RBAC permissions

**Fix:**

1. Verify scaling plan assignment:

   **Portal:** Virtual Desktop → Scaling plans → sp-pooled-prod → Host pool assignments

   Check: hp-pooled-prod listed with "Enable autoscale" = Yes

2. Verify time zone:

   **Portal:** Scaling plan → Schedules

   Check: Time zone matches business location (e.g., Eastern Time)

3. Verify RBAC permissions:

   **Portal:** Resource groups → rg-avd-prod-eastus-01 → Access control (IAM)

   Check: "Azure Virtual Desktop" service principal has "Desktop Virtualization Power On Contributor" role

4. Manually start one VM and verify scaling plan can stop it during next phase transition

---

## Next Steps

**Production monitoring and operations now configured.** AVD deployment is ready for full user rollout.

**Ongoing Operational Tasks:**

1. **Daily:**
   - Review AVD Insights Overview (5 minutes)
   - Check fired alerts and resolve issues

2. **Weekly:**
   - Review capacity trends (session count, CPU, memory)
   - Plan session host additions if approaching capacity
   - Review scaling plan performance (cost savings)

3. **Monthly:**
   - Update golden images (Windows updates, app updates)
   - Review alert rules (tune thresholds, reduce false positives)
   - Generate management report (uptime, user count, cost)

4. **Quarterly:**
   - Disaster recovery test (failover to secondary region)
   - Review security posture (Conditional Access policies, MFA adoption)
   - User satisfaction survey

**Congratulations on completing the Azure Virtual Desktop deployment!**

---

## Related Reference Pages

- [[../Security/log-analytics-workspace|Log Analytics Workspace]] - Detailed workspace architecture, KQL queries, cost optimization
- [[../Security/avd-insights-azure-monitor-workbook|AVD Insights]] - Workbook sections, metrics, troubleshooting
- [[../Security/azure-monitor-alerting|Azure Monitor Alerting]] - Alert types, action groups, advanced scenarios
- [[../AVD/scaling-plans|Scaling Plans]] - Schedule configuration, load balancing changes, cost calculations
- [[../Operations/capacity-planning|Capacity Planning]] - VM sizing, user density, growth planning
- [[../Operations/disaster-recovery|Disaster Recovery]] - Multi-region deployments, failover procedures
