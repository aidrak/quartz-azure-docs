---
title: Windows Update Policies for AVD
description: 
published: true
date: 2025-12-14T04:53:16.861Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:44.615Z
---

# Windows Update Policies for AVD

Managing Windows Updates for AVD session hosts is critical for security, stability, and compliance. Microsoft Intune provides Windows Update for Business (WUfB) policies that control when and how updates are deployed. However, the update strategy differs significantly between pooled (non-persistent) and personal (persistent) AVD environments. This page covers update rings, feature update policies, driver update policies, and best practices for both pooled and personal AVD deployments.

## Windows Update for Business (WUfB)

Windows Update for Business (WUfB) is Microsoft's cloud-based solution for managing Windows Updates without requiring on-premises infrastructure (WSUS servers). WUfB integrates with Intune to provide:

- **Update Rings**: Control deployment timing for quality updates (monthly security patches) and feature updates (semi-annual OS versions)
- **Feature Update Policies**: Pin devices to specific Windows versions (e.g., Windows 11 22H2)
- **Driver Update Policies**: Control automatic driver updates from Windows Update
- **Expedited Updates**: Deploy critical zero-day patches immediately (bypass deferral periods)

WUfB relies on Windows Update cloud service for update content delivery. Devices connect to `*.windowsupdate.com` to download updates, eliminating the need for on-premises WSUS servers.

**Key Concepts:**

**Quality Updates:**
- Monthly security patches (Patch Tuesday, second Tuesday of each month)
- Cumulative (each update includes all previous updates)
- Critical for security (address CVEs, zero-day vulnerabilities)
- Can be deferred 0-30 days

**Feature Updates:**
- Semi-annual OS version upgrades (e.g., Windows 10 21H2 → 22H2)
- Introduces new features, UI changes, and capabilities
- Can be deferred 0-365 days
- Require larger downloads (2-4 GB) and longer installation time

**Driver Updates:**
- Hardware drivers distributed via Windows Update
- Can be automatically installed or blocked
- Controlled separately from quality/feature updates

**Update Channels:**
- **General Availability Channel**: Default for all Windows 10/11 devices (receives updates when generally available)
- **Windows Insider Program**: Pre-release builds for testing (not recommended for production AVD)

## Update Rings

Update rings control when quality updates and feature updates are deployed to devices. Intune supports creating multiple rings with different deferral periods, allowing staged rollouts (pilot → production).

### Update Ring Settings

**Quality Update Deferral:**
- **Range**: 0-30 days
- **Default**: 0 days (install updates immediately on Patch Tuesday)
- **Recommended**: 7 days for pilot ring, 14-21 days for production ring

**Feature Update Deferral:**
- **Range**: 0-365 days
- **Default**: 0 days (upgrade to new feature update when available)
- **Recommended**: 60 days for pilot ring, 120-180 days for production ring (wait for stability)

**Maintenance Windows:**
- **Active Hours**: Time range when updates should not install (e.g., 8 AM - 5 PM)
- **Maintenance Window**: Scheduled time for update installation (e.g., 2 AM - 6 AM)

**Restart Behavior:**
- **Auto-Restart**: Automatically restart after update installation (default)
- **Auto-Restart with Active User Suppression**: Delay restart if user is active
- **Auto-Restart Outside Active Hours**: Only restart outside active hours
- **Require User Approval**: Prompt user before restart

**Deadline Settings:**
- **Quality Update Deadline**: Maximum days before forced installation (default: 7 days)
- **Feature Update Deadline**: Maximum days before forced installation (default: 7 days)

**Portal:** Intune Admin Center → Devices → Windows → Update Rings for Windows 10 and Later → Create

### Pilot Ring Configuration

**Purpose:** Validate updates before broad deployment

**Assigned To:** avd-devices-pilot (2-5 test session hosts)

**Settings:**
- Quality Update Deferral: 0 days (install immediately on Patch Tuesday)
- Feature Update Deferral: 60 days (wait 2 months for initial stability)
- Active Hours: 8 AM - 5 PM (block updates during business hours)
- Restart Behavior: Auto-restart outside active hours
- Quality Update Deadline: 3 days (force install after 3 days)

**Workflow:**
1. Patch Tuesday: Quality updates released by Microsoft
2. Day 0: Pilot ring devices download and install updates
3. Day 0-3: Monitor pilot devices for issues (check event logs, user reports)
4. Day 3: Deadline reached, pilot devices auto-restart if not already restarted
5. Day 7: If no issues, deploy to production ring

**Best Practice:** Monitor pilot ring for 7 days before promoting to production. Check event logs for errors (Event Viewer → Windows Logs → System, filter by source "WindowsUpdateClient").

### Production Ring Configuration

**Purpose:** Deploy updates to all production session hosts after validation

**Assigned To:** avd-devices-personal (personal AVD hosts only; see note below for pooled AVD)

**Settings:**
- Quality Update Deferral: 14 days (wait 2 weeks after Patch Tuesday)
- Feature Update Deferral: 180 days (wait 6 months for stability)
- Active Hours: 8 AM - 5 PM (block updates during business hours)
- Restart Behavior: Auto-restart outside active hours
- Quality Update Deadline: 7 days (force install 7 days after deferral ends, i.e., day 21)

**Workflow:**
1. Patch Tuesday: Quality updates released by Microsoft
2. Day 0-14: Deferral period (updates not offered to production ring)
3. Day 14: Production ring devices download and install updates
4. Day 14-21: Users can defer restart (active hours respected)
5. Day 21: Deadline reached, production devices auto-restart if not already restarted

**Best Practice:** Set deferral to 14-21 days for production. This allows time for Microsoft to identify and fix any issues discovered after Patch Tuesday.

### Update Ring Assignment

**Recommended Approach:**

**Pilot Ring:** Assign to avd-devices-pilot (dynamic group with 2-5 test session hosts)
- Example Query: `(device.displayName -startsWith "hp-") -and (device.displayName -contains "test")`

**Production Ring:** Assign to avd-devices-personal (personal AVD hosts only)
- Example Query: `(device.displayName -contains "personal") -and -not (device.displayName -contains "test")`

**Pooled AVD:** Do not assign update rings (see "Pooled AVD Update Strategy" below)

## Feature Update Policies

Feature update policies allow you to pin devices to a specific Windows version, preventing automatic feature upgrades. This is useful for maintaining consistency and avoiding unexpected UI changes.

### Feature Update Policy Settings

**Target Version:**
- **Windows 10 21H2**: Build 19044 (long-term support)
- **Windows 10 22H2**: Build 19045 (latest Windows 10 feature update, final version)
- **Windows 11 22H2**: Build 22621 (long-term support)
- **Windows 11 25H2**: Build 22631 (latest Windows 11 feature update)

**Rollout Options:**
- **Make update available as soon as possible**: Deploy immediately to all assigned devices
- **Make update available on a specific date**: Schedule deployment (e.g., deploy Windows 11 25H2 on January 1, 2024)

**Portal:** Intune Admin Center → Devices → Windows → Feature Updates for Windows 10 and Later → Create

### Example Feature Update Policy

**Policy Name:** AVD - Pin to Windows 11 22H2

**Assigned To:** avd-devices-personal

**Settings:**
- Feature Update to Deploy: Windows 11, version 22H2
- Rollout Options: Make update available as soon as possible

**Purpose:** Prevent automatic upgrade to Windows 11 25H2, maintaining consistency until 25H2 is validated

**Best Practice:** Pin devices to a specific Windows version for 6-12 months, then update policy to next version after validation. This provides stability while ensuring devices remain supported.

### Feature Update End of Support

Microsoft publishes end-of-support dates for each Windows version. Devices running unsupported versions do not receive security updates.

**Windows 10 Versions:**
- 21H2: End of Support June 11, 2024 (Home/Pro), June 8, 2027 (Enterprise/IoT)
- 22H2: End of Support October 14, 2025 (final Windows 10 version)

**Windows 11 Versions:**
- 22H2: End of Support October 8, 2024 (Home/Pro), October 14, 2025 (Enterprise)
- 25H2: End of Support November 11, 2025 (Home/Pro), November 10, 2026 (Enterprise)

**Recommendation:** Pin devices to Windows 10 22H2 (if remaining on Windows 10) or Windows 11 22H2/25H2 (if migrating to Windows 11). Plan feature update deployments 3-6 months before end of support.

## Driver Update Policies

Driver update policies control automatic driver installation from Windows Update. Drivers can introduce compatibility issues, so careful management is recommended.

### Driver Update Settings

**Approval Policy:**
- **Automatic**: Install all drivers from Windows Update (default)
- **Manual**: Block automatic driver installation; IT admin approves drivers manually

**Deferral Period:**
- **Range**: 0-30 days
- **Default**: 0 days (install drivers immediately when available)
- **Recommended**: 14 days (wait for driver stability)

**Portal:** Intune Admin Center → Devices → Windows → Driver Updates for Windows 10 and Later → Create

### Recommended Driver Policy

**Policy Name:** AVD - Driver Updates - Deferred

**Assigned To:** avd-devices-all

**Settings:**
- Approval Policy: Automatic
- Deferral Period: 14 days

**Purpose:** Allow automatic driver updates but defer for 2 weeks to avoid issues from newly released drivers

**Best Practice:** Defer driver updates by 14 days. If a driver causes issues, Microsoft typically releases a fix within 7-14 days. Manual approval is overly restrictive for most environments.

## Expedited Updates for Zero-Days

Expedited updates allow immediate deployment of critical security patches, bypassing normal deferral periods. This is used for zero-day vulnerabilities actively exploited in the wild.

### When to Use Expedited Updates

**Criteria:**
- Microsoft releases out-of-band security update (not part of Patch Tuesday)
- CVE is actively exploited (zero-day)
- High severity (CVSS 9.0+)

**Examples:**
- PrintNightmare (CVE-2021-34527): Print Spooler RCE vulnerability
- BlueKeep (CVE-2019-0708): RDP RCE vulnerability
- Log4Shell (CVE-2021-44228): Log4j RCE vulnerability (affects Java apps, not Windows directly)

### Deploying Expedited Updates

**Portal:** Intune Admin Center → Devices → Windows → Expedite Quality Updates → Create

**Settings:**
- Quality Update to Expedite: Select specific KB number (e.g., KB5012345)
- Deployment Schedule: Deploy as soon as possible
- Restart Behavior: Force restart after installation (user cannot defer)

**Workflow:**
1. Microsoft releases out-of-band security update (e.g., KB5012345)
2. IT admin creates expedited update policy
3. Devices download and install update within 1-2 hours
4. Devices auto-restart immediately after installation (no deferral)

**Best Practice:** Only use expedited updates for critical zero-day patches. Normal Patch Tuesday updates should use standard update rings.

## Pooled AVD Update Strategy

For pooled (non-persistent) AVD environments, updates should be applied to the golden image, not individual session hosts. This ensures consistency and avoids update-related issues during user sessions.

### Why Not Use Update Rings for Pooled AVD

**Problems with Updating Individual Session Hosts:**
1. **User Session Disruption**: Updates require restart, disconnecting active users
2. **Inconsistent Experience**: Some session hosts have updates, others don't (users experience different OS versions)
3. **Wasted Resources**: Updates downloaded/installed repeatedly to each session host
4. **Complexity**: Managing updates across 10-100 session hosts is operationally complex

**Solution:** Update golden image, redeploy session hosts

### Golden Image Update Process

**Monthly Update Workflow:**

**Step 1: Update Golden Image VM**
1. Start golden image VM (stopped when not in use to save costs)
2. Install latest Windows Updates:
   ```powershell
   Install-Module PSWindowsUpdate -Force
   Get-WindowsUpdate -Install -AcceptAll -AutoReboot
   ```
3. Install application updates (Microsoft 365 Apps, Adobe Reader, etc.)
4. Verify updates:
   ```powershell
   Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10
   ```
5. Sysprep golden image:
   ```powershell
   C:\Windows\System32\Sysprep\sysprep.exe /oobe /generalize /shutdown
   ```
6. Capture new image version (Azure Compute Gallery or Managed Image)

**Step 2: Redeploy Session Hosts from Updated Image**
1. Update host pool configuration to use new image version
2. Enable drain mode on old session hosts:
   ```bash
   az desktopvirtualization sessionhost update \
     --resource-group RG-Azure-VDI-01 \
     --host-pool-name hp-pooled-prod1 \
     --name sessionhost01 \
     --allow-new-session false
   ```
3. Wait for existing user sessions to end (or forcibly sign users out during maintenance window)
4. Deploy new session hosts from updated image
5. Delete old session hosts

**Step 3: Verification**
1. Connect to new session host
2. Verify OS version:
   ```powershell
   Get-ComputerInfo | Select-Object WindowsVersion, OsHardwareAbstractionLayer
   ```
3. Verify updates installed:
   ```powershell
   Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 5
   ```

**Frequency:** Monthly (after Patch Tuesday + 7-14 days for validation)

**Best Practice:** Maintain 2 versions of golden image (current month and previous month). If new image has issues, quickly roll back to previous version.

### Exception: Emergency Patches

For critical zero-day vulnerabilities, update individual session hosts via Intune expedited updates while preparing updated golden image:

**Emergency Workflow:**
1. Microsoft releases out-of-band security update for critical CVE
2. Deploy expedited update to all pooled session hosts via Intune (immediate installation)
3. Schedule maintenance window to restart session hosts (drain mode + forcibly sign out users)
4. Update golden image with security patch
5. Redeploy session hosts from updated golden image (within 7 days)

**Best Practice:** Only use this approach for actively exploited zero-days. Normal Patch Tuesday updates should follow golden image workflow.

## Personal AVD Update Strategy

For personal (persistent) AVD environments, use Intune update rings to deploy updates directly to session hosts. Since users have dedicated session hosts, updates can be scheduled outside business hours without impacting other users.

### Recommended Update Rings

**Pilot Ring:**
- **Assigned To**: avd-users-pilot (2-5 test users)
- **Quality Deferral**: 0 days
- **Feature Deferral**: 60 days
- **Restart Behavior**: Auto-restart outside active hours
- **Deadline**: 3 days

**Production Ring:**
- **Assigned To**: avd-devices-personal
- **Quality Deferral**: 14 days
- **Feature Deferral**: 180 days
- **Restart Behavior**: Auto-restart outside active hours
- **Deadline**: 7 days

### User Communication

Since updates require restart, communicate update schedules to users:

**Email Template:**
```
Subject: AVD Session Host Maintenance - Updates and Restart

Your AVD session host will install Windows security updates and restart outside business hours (8 PM - 6 AM) over the next 7 days.

To avoid losing unsaved work:
1. Save your work daily before leaving
2. Sign out of AVD session when done for the day
3. Do not leave applications running overnight

If your session host has not restarted after 7 days, it will be forcibly restarted to ensure security updates are applied.

Questions? Contact IT Support.
```

**Best Practice:** Send email notification 3 days before deadline. Include instructions to save work and sign out.

## Monitoring Update Compliance

### Update Compliance Reports

**Portal:** Intune Admin Center → Reports → Windows Updates

**Available Reports:**

**1. Windows Update Summary:**
- Total devices
- Devices up-to-date
- Devices with pending updates
- Devices with failed updates

**2. Windows Feature Update Report:**
- Devices by Windows version (e.g., Windows 11 22H2, Windows 11 25H2)
- Devices awaiting feature update
- Devices with failed feature updates

**3. Windows Quality Update Report:**
- Devices with latest quality update
- Devices with pending quality updates
- Devices with failed quality updates

**Best Practice:** Review update compliance weekly. Investigate devices with failed updates or devices >30 days out of date.

### Update Compliance Workbook (Log Analytics)

For advanced monitoring, enable Update Compliance workbook in Azure Monitor:

**Setup:**
1. Create Log Analytics workspace (if not already exists)
2. Enable Windows Update data collection:
   - Intune Admin Center → Tenant Administration → Diagnostics Settings → Add Diagnostic Setting
   - Send logs to Log Analytics workspace
3. Access Update Compliance workbook:
   - Azure Portal → Monitor → Workbooks → Update Compliance

**Workbook Features:**
- Update deployment status by device
- Update installation errors and error codes
- Time to install updates (average, 90th percentile)
- Devices not checking for updates (offline or non-compliant)

**Best Practice:** Use Log Analytics for large environments (100+ devices). For small environments, Intune Admin Center reports are sufficient.

### Common Update Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Update installation fails with error 0x80070002** | Insufficient disk space or corrupted Windows Update components | 1. Free up disk space (delete temp files, clean up old updates: `DISM /Online /Cleanup-Image /StartComponentCleanup`)<br>2. Reset Windows Update components: `net stop wuauserv && net start wuauserv` |
| **Update installation fails with error 0x8007000E** | Insufficient memory (RAM) during update installation | 1. Increase VM size (e.g., Standard_D2s_v3 → Standard_D4s_v3)<br>2. Close applications before update installation<br>3. Restart VM and retry update |
| **Device shows "Pending Restart" for >7 days** | User deferring restart beyond deadline | 1. Check restart deadline settings (Update Ring → Deadline)<br>2. Force restart via Intune: Devices → [Device] → Restart<br>3. Communicate to user: "Save work, session host will restart in 1 hour" |
| **Feature update fails with error 0xC1900101** | Driver or application compatibility issue | 1. Identify incompatible driver/app: Check SetupDiag log (`C:\$Windows.~BT\Sources\Panther\setupdiag.log`)<br>2. Update or uninstall incompatible driver/app<br>3. Retry feature update |
| **Device not receiving updates** | Windows Update service disabled or device offline | 1. Verify Windows Update service is running: `Get-Service wuauserv`<br>2. Start service: `Start-Service wuauserv`<br>3. Force update check: `Start-WUScan` (requires PSWindowsUpdate module) |

## Best Practices

### Update Golden Image Monthly for Pooled AVD

**Why:** Ensures all session hosts have latest security patches, eliminates update-related session disruptions, and provides consistent user experience.

**Schedule:** First Monday after Patch Tuesday + 7 days (allows time for Microsoft to fix any issues discovered after Patch Tuesday).

**Example Schedule:**
- Patch Tuesday: Second Tuesday of month (e.g., June 11)
- Validation Period: June 11-18 (7 days)
- Golden Image Update: June 18 (first Monday after validation period)
- Session Host Redeployment: June 18-25 (phased rollout)

### Use Update Rings for Personal AVD Only

**Why:** Personal AVD hosts are persistent (dedicated to specific users), so updates can be scheduled outside business hours without impacting other users. Pooled AVD hosts should be updated via golden image redeployment.

### Defer Quality Updates by 14 Days for Production

**Why:** Allows time for Microsoft to identify and fix issues discovered after Patch Tuesday. Reduces risk of update-related outages.

**Exception:** Use expedited updates for critical zero-day vulnerabilities (bypass deferral).

### Pin Feature Updates for 6-12 Months

**Why:** Feature updates introduce UI changes, new features, and potential compatibility issues. Pinning to a specific version provides stability while ensuring devices remain supported.

**Recommendation:** Pin to Windows 11 22H2 for 6-12 months, then update to Windows 11 25H2 after validation.

### Set Active Hours to Match Business Hours

**Why:** Prevents updates from installing during user sessions (8 AM - 5 PM). Updates install overnight (outside active hours), minimizing user disruption.

**Configuration:** Active Hours = 8 AM - 5 PM (or 7 AM - 7 PM for extended business hours)

### Communicate Update Schedules to Users

**Why:** Informs users when session hosts will restart, allowing them to save work and sign out before automatic restart.

**Frequency:** Send email notification 3 days before deadline.

### Monitor Update Compliance Weekly

**Why:** Ensures devices are receiving updates, identifies devices with failed updates, and catches devices that are out of date.

**Weekly Checklist:**
- Review update compliance reports (Reports → Windows Updates)
- Investigate devices with failed updates
- Remediate issues (free up disk space, reset Windows Update service)

## Summary

Windows Update management for AVD differs significantly between pooled and personal environments. Pooled AVD should use golden image updates with monthly redeployment, while personal AVD should use Intune update rings with pilot and production phases. Feature update policies pin devices to specific Windows versions for stability, and expedited updates provide emergency patching for zero-day vulnerabilities.

**Key Takeaways:**
- **Pooled AVD**: Update golden image monthly, redeploy session hosts (do not use update rings for individual hosts)
- **Personal AVD**: Use update rings with pilot (0-day deferral) and production (14-day deferral)
- **Feature Updates**: Pin to specific Windows version for 6-12 months (e.g., Windows 11 22H2)
- **Expedited Updates**: Deploy critical zero-day patches immediately (bypass deferral)
- **Active Hours**: Set to 8 AM - 5 PM to avoid update installation during user sessions
- **Monitoring**: Review update compliance weekly (Reports → Windows Updates)

## Next Steps

You have completed Chapter 7: Intune & Device Management. Proceed to the next chapter for additional AVD management topics:

- **Monitoring and Optimization** - Azure Monitor, Log Analytics, performance tuning
- **Security and Governance** - Conditional Access, MFA, RBAC, Azure Policy
- **Disaster Recovery** - Backup strategies, failover planning, business continuity