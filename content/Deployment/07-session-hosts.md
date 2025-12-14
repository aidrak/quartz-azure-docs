---
title: Step 07 - Session Host Provisioning
description: Deploy and configure session hosts for AVD pooled host pool
published: true
date: 2025-12-14T00:00:00.000Z
tags: [Quick-Deploy, session-hosts, vms, deployment]
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 07: Session Host Provisioning

Deploy virtual machines to the pooled host pool using custom golden images from Azure Compute Gallery. This step creates the compute infrastructure where users will connect to run their desktop sessions.

## Example Scenario

Following the 200-user deployment from [[00-naming-conventions]]:

| Component | Configuration | Details |
|-----------|--------------|---------|
| **Host Pool** | `hp-pooled-prod` | Created in Step 06 |
| **VM Count** | 10 session hosts | Supports 150 users (15 users per VM) |
| **VM SKU** | Standard_D4s_v5 | 4 vCPU, 16GB RAM |
| **Image Source** | Azure Compute Gallery | `gal-avd-prod-01/win11-multisession-23h2/1.0.0` |
| **Naming Pattern** | `vm-pooled-prod-001` through `vm-pooled-prod-010` | Sequential numbering |
| **Join Type** | Entra ID Join | Cloud-only deployment |
| **Network** | `vnet-avd-prod-eastus-01` | Subnet: `snet-avd-prod-sessionhosts` |
| **Intune Enrollment** | Yes | Automatic enrollment on Entra Join |

## Prerequisites

- [ ] Completed [[06-host-pool|Step 06: Host Pool Creation]]
- [ ] Host pool created: `hp-pooled-prod`
- [ ] Registration token generated (valid for 27 days)
- [ ] Azure Compute Gallery image ready: `win11-multisession-23h2` version 1.0.0
- [ ] Virtual network configured: `vnet-avd-prod-eastus-01` with `snet-avd-prod-sessionhosts`
- [ ] Storage account ready: `stavdprodeus01` with file shares configured
- [ ] Entra ID device group created: `AVD-Devices-Pooled`

**Duration:** ~30 minutes (deployment) + 15 minutes (post-deployment verification)

---

## Part 1: VM Sizing Guidance

Choose the right VM size based on workload characteristics and user density.

### Sizing for Multi-Session (Pooled)

**Recommended starting point: Standard_D4s_v5**

| VM SKU | vCPUs | RAM | Max Users per VM | Workload Type | Monthly Cost/VM |
|--------|-------|-----|------------------|---------------|----------------|
| Standard_D2s_v5 | 2 | 8 GB | 4-6 | Light (web, email only) | ~$70 |
| **Standard_D4s_v5** | **4** | **16 GB** | **10-15** | **General productivity** | **~$140** |
| Standard_D8s_v5 | 8 | 32 GB | 20-25 | Power users (Office + web apps) | ~$280 |
| Standard_D16s_v5 | 16 | 64 GB | 30-40 | Heavy multitasking | ~$560 |

**This deployment uses Standard_D4s_v5:**
- 150 pooled users / 15 users per VM = **10 VMs required**
- General productivity workload (Office 365, web apps, Teams)
- 4 vCPU provides headroom for peak usage

### Workload Sizing Calculator

**Formula:**
```
Required VMs = (Total Users) / (Users per VM)
Users per VM = (VM RAM in GB - 2 GB OS overhead) / (RAM per user)
```

**Example for General Productivity:**
- RAM per user: ~1 GB (Office 365 + browser)
- Standard_D4s_v5: (16 GB - 2 GB) / 1 GB = **14 users per VM**
- 150 users / 14 = **11 VMs** (rounded up for safety → deploy 10 and monitor)

**Example for Power Users:**
- RAM per user: ~2 GB (Office + CAD/Adobe)
- Standard_D8s_v5: (32 GB - 2 GB) / 2 GB = **15 users per VM**
- 50 power users / 15 = **4 VMs**

> **Important:** Start conservative (fewer users per VM). Monitor CPU/RAM utilization for 2 weeks, then adjust density or VM size.

**See:** [[../AVD/session-host-sizing|Session Host Sizing Deep Dive]] for detailed workload profiling and performance optimization.

---

## Part 2: Add Session Hosts to Host Pool

Deploy VMs directly from the host pool blade using Azure Compute Gallery image.

### Navigate to Host Pool

**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → `hp-pooled-prod`

### Start Session Host Wizard

1. Click **Session hosts** (left navigation)
2. Click **+ Add**
3. **Add virtual machines to a host pool** wizard opens

### Basics Tab

1. **Subscription:** Select your Azure subscription
2. **Resource group:** `rg-avd-prod-eastus-01`
3. **Host pool name:** `hp-pooled-prod` (pre-filled)
4. **Location:** East US (pre-filled from host pool region)
5. **Validation environment:** No (production deployment)

6. **Virtual machine details:**
   - **Name prefix:** `vm-pooled-prod-`
   - **Virtual machine type:** Azure virtual machine (default)
   - **Virtual machine location:** East US
   - **Availability options:** No infrastructure redundancy required
   - **Security type:** Trusted launch virtual machines (recommended)
   - **Number of VMs:** `10`

7. **Image:**
   - **Image type:** Gallery (not Marketplace)
   - **Gallery:** Click **See all images**
   - Navigate to: **My items** → **Shared Images**
   - Select: `gal-avd-prod-01` → `win11-multisession-23h2` → Version `1.0.0`
   - Click **Select**

8. **Virtual machine size:**
   - Click **Change size**
   - Filter by: **D-Series v5**
   - Select: `Standard_D4s_v5` (4 vCPU, 16 GB RAM)
   - Click **Select**

9. **Number of disks:** `10` (one OS disk per VM, automatically configured)

Click **Next: Virtual Machines**

### Virtual Machines Tab

1. **Network and security:**
   - **Virtual network:** `vnet-avd-prod-eastus-01`
   - **Subnet:** `snet-avd-prod-sessionhosts` (10.0.1.0/24)
   - **Network security group:** Basic (NSG already attached to subnet)
   - **Public inbound ports:** None (AVD uses reverse connect)

2. **Domain to join:**
   - **Select which directory you would like to join:** Microsoft Entra ID
   - **Enroll VM with Intune:** Yes (requires Entra ID P1 or higher)

3. **Virtual Machine Administrator account:**
   - **Username:** `localadmin`
   - **Password:** Create strong password (20+ chars, save to password manager)
   - **Confirm password:** Re-enter password

   > **Note:** This local admin account is for emergency access only. Do NOT use for daily operations.

4. **Custom configuration:**
   - Leave blank (no custom scripts needed)

Click **Next: Workspace**

### Workspace Tab

1. **Register desktop app group:**
   - **Register desktop app group:** Yes (default)
   - **To this workspace:** Select existing
   - **Workspace:** `ws-prod` (created in Step 06)

   > **Note:** This automatically creates a Desktop Application Group and assigns it to the workspace.

Click **Next: Advanced**

### Advanced Tab

1. **Boot diagnostics:**
   - **Enable boot diagnostics:** Enabled with managed storage account (recommended)

2. **Extensions:**
   - Extensions will be automatically installed:
     - AVD Agent
     - AVD Agent Bootloader
     - Azure Monitor Agent (if configured in host pool diagnostics)

3. **Host pool registration token:**
   - **Use registration token:** Automatically populated (from Step 06)
   - **Token expiration:** Shows expiration date (27 days from creation)

   > **Warning:** If token expired, return to host pool → Registration token → Generate new token.

Click **Next: Tags**

### Tags Tab

Add organizational tags:

| Name | Value |
|------|-------|
| `Environment` | `Production` |
| `Workload` | `AVD-Pooled` |
| `Department` | `IT` |
| `HostPool` | `hp-pooled-prod` |

Click **Next: Review + create**

### Review and Create

1. Verify configuration:
   - **VM count:** 10
   - **VM size:** Standard_D4s_v5
   - **Image:** win11-multisession-23h2 version 1.0.0
   - **Join type:** Microsoft Entra ID
   - **Intune enrollment:** Yes
   - **Subnet:** snet-avd-prod-sessionhosts

2. Review estimated cost:
   - **Per VM:** ~$140/month (compute) + ~$10/month (disk)
   - **Total for 10 VMs:** ~$1,500/month

3. Click **Create**

**Deployment time:** 25-30 minutes for 10 VMs

---

## Part 3: Monitor Deployment Progress

Track VM provisioning and AVD agent registration in real-time.

### Watch Deployment Status

**Portal:** Azure Portal → Notifications (bell icon) → Deployment in progress

1. Click on notification: "Deployment 'Microsoft.HostPool-...' is in progress"
2. Monitor deployment blade showing:
   - Virtual machines: Creating...
   - Network interfaces: Creating...
   - OS disks: Creating...
   - Extensions: Installing...

**Expected timeline:**
- **0-5 minutes:** VM infrastructure created (VMs, NICs, disks)
- **5-15 minutes:** VMs boot and join Entra ID
- **15-25 minutes:** AVD agent and extensions install
- **25-30 minutes:** Deployment completes

### Check AVD Agent Registration

After deployment completes, verify session hosts registered with host pool:

**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → `hp-pooled-prod` → Session hosts

**Expected state progression:**
1. **Unavailable** (0-10 minutes after deployment): Agent installing
2. **Needs Assistance** (10-15 minutes): Agent registering with control plane
3. **Available** (15+ minutes): Registration complete, ready for user sessions

**If stuck in "Needs Assistance" after 30 minutes:**
- See Part 6 Troubleshooting: Session hosts not registering

### Verify Entra ID Join

**Portal:** Entra Admin Center → Devices → All devices

1. Filter by: Device name starts with `vm-pooled-prod-`
2. Verify all 10 VMs appear with:
   - **Join Type:** Microsoft Entra joined
   - **Enabled:** Yes
   - **MDM:** Microsoft Intune (if enrollment succeeded)

**If VMs not showing:**
- Wait 10 minutes (Entra sync delay)
- Check VM deployment logs for join errors

### Verify Intune Enrollment

**Portal:** Intune Admin Center → Devices → Windows → Windows devices

1. Filter by: Device name contains `vm-pooled-prod-`
2. Verify all 10 VMs show:
   - **Enrollment status:** Enrolled
   - **Compliance status:** Not evaluated (normal - policies not assigned yet)
   - **Managed by:** Intune

**If enrollment failed:**
- Verify Entra ID P1/P2 licenses assigned
- Check Intune automatic enrollment enabled: Entra Admin Center → Mobility (MDM and MAM) → Microsoft Intune → MDM user scope = All

---

## Part 4: Post-Deployment Configuration

Configure FSLogix, time zone, and monitoring after VMs deploy successfully.

### Configure FSLogix via Intune (Recommended)

Deploy FSLogix settings to all session hosts using Intune configuration profile.

**Portal:** Intune Admin Center → Devices → Configuration profiles → + Create profile

1. **Platform:** Windows 10 and later
2. **Profile type:** Templates → Administrative Templates
3. **Name:** `AVD - FSLogix Profile Container Settings`

4. **Configuration settings:**
   - Search: `FSLogix`
   - Configure:
     ```
     Computer Configuration → Administrative Templates → FSLogix → Profile Containers

     Enabled: Yes
     VHD Locations: \\stavdprodeus01.file.core.windows.net\profiles-pooled
     Size in MBs: 30000 (30 GB)
     Is Dynamic: True
     Volume Type: VHDX

     Advanced Settings:
     Delete Local Profile When FSLogix Profile Should Apply: Enabled
     Prevent Login with Failure: Enabled
     Prevent Login with Temp Profile: Enabled
     ```

5. **Assignments:**
   - **Included groups:** `AVD-Devices-Pooled` (dynamic device group created in Step 02)

6. Click **Create**

**Profile application time:** 15-30 minutes after VM enrollment

**Verification:**

Connect to a session host via RDP as local admin:

```powershell
# Check FSLogix registry settings
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles"

# Expected output:
# Enabled: 1
# VHDLocations: \\stavdprodeus01.file.core.windows.net\profiles-pooled
# SizeInMBs: 30000
# IsDynamic: 1
# VolumeType: VHDX
```

**Alternative: Manual FSLogix Configuration (Not Recommended)**

If Intune not available, configure FSLogix manually on each session host:

```powershell
# Run on session host as local admin
New-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "Enabled" -Value 1 -PropertyType DWord -Force
New-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "VHDLocations" -Value "\\stavdprodeus01.file.core.windows.net\profiles-pooled" -PropertyType MultiString -Force
New-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "SizeInMBs" -Value 30000 -PropertyType DWord -Force
New-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "IsDynamic" -Value 1 -PropertyType DWord -Force
New-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles" -Name "VolumeType" -Value "VHDX" -PropertyType String -Force
```

**See:** [[../Images/fslogix-profile-containers|FSLogix Profile Containers Reference]] for advanced configuration options.

### Configure Time Zone (Optional)

Set session host time zone to match user locations.

**Intune method (recommended):**

Create custom configuration profile:

```xml
# OMA-URI Setting
OMA-URI: ./Device/Vendor/MSFT/Policy/Config/TimeLanguageSettings/ConfigureTimeZone
Data type: String
Value: Eastern Standard Time
```

**PowerShell method (manual):**

```powershell
# Run on each session host
Set-TimeZone -Id "Eastern Standard Time"
```

**Common time zone IDs:**
- `Eastern Standard Time` (US East Coast)
- `Central Standard Time` (US Central)
- `Pacific Standard Time` (US West Coast)
- `UTC` (Coordinated Universal Time)

### Enable Azure Monitor Agent (Optional)

Collect performance metrics and logs for monitoring.

**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → `hp-pooled-prod` → Insights

1. Click **Enable** on Azure Monitor for AVD
2. Select existing Log Analytics workspace: `law-avd-prod-eus-01`
3. Click **Enable**

Agents automatically install on all session hosts in host pool.

**Verification after 15 minutes:**

**Portal:** Log Analytics workspace → Logs

```kusto
Perf
| where Computer startswith "vm-pooled-prod-"
| where TimeGenerated > ago(1h)
| summarize count() by Computer
```

Expected: 10 session hosts reporting performance data.

---

## Part 5: Verification Checklist

Confirm all session hosts deployed correctly and ready for user sessions.

### VM Deployment

**Portal:** Azure Portal → Resource groups → `rg-avd-prod-eastus-01` → Type: Virtual machine

- [ ] 10 VMs created: `vm-pooled-prod-001` through `vm-pooled-prod-010`
- [ ] All VMs status: **Running**
- [ ] All VMs size: **Standard_D4s_v5**
- [ ] All VMs joined to: **Microsoft Entra**

**Check VM properties:**
1. Click on `vm-pooled-prod-001`
2. Verify:
   - **Image:** win11-multisession-23h2 1.0.0
   - **Virtual network/subnet:** vnet-avd-prod-eastus-01/snet-avd-prod-sessionhosts
   - **Public IP address:** None (expected)
   - **Boot diagnostics:** Enabled

### AVD Agent Registration

**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → `hp-pooled-prod` → Session hosts

- [ ] 10 session hosts listed
- [ ] All session hosts show **Status: Available**
- [ ] **Agent version:** Latest (e.g., 1.0.xxxx.xxxx)
- [ ] **Assigned User:** None (pooled = no specific assignment)
- [ ] **Sessions:** 0 (no users connected yet)

**Refresh view if status not "Available":**
- Click **Refresh** button
- Wait 5 minutes between refreshes
- If still "Needs Assistance" after 30 minutes → see Troubleshooting

### Entra ID Join Status

**Portal:** Entra Admin Center → Devices → All devices

- [ ] 10 devices visible: vm-pooled-prod-001 through vm-pooled-prod-010
- [ ] All show **Join type:** Microsoft Entra joined
- [ ] All show **Enabled:** Yes
- [ ] All show **MDM:** Microsoft Intune

**Dynamic group membership verification:**

**Portal:** Entra Admin Center → Groups → `AVD-Devices-Pooled`

- [ ] **Membership type:** Dynamic Device
- [ ] **Total members:** 10 (may take 15 minutes to populate)
- [ ] Click **Members** → Verify all vm-pooled-prod-* VMs listed

### Intune Enrollment

**Portal:** Intune Admin Center → Devices → Windows → Windows devices

- [ ] 10 devices enrolled: vm-pooled-prod-001 through vm-pooled-prod-010
- [ ] All show **Enrollment:** Microsoft Entra joined
- [ ] All show **Managed by:** Intune
- [ ] All show **Compliance:** Not evaluated (normal - no policies assigned yet)

**Check configuration profile assignment:**

1. Click on `vm-pooled-prod-001`
2. Navigate to **Device configuration**
3. Verify `AVD - FSLogix Profile Container Settings` shows:
   - **Status:** Succeeded or Pending (if recent deployment)

### FSLogix Configuration

RDP to a session host as local admin and verify FSLogix:

```powershell
# Connect via Azure Portal → Virtual machines → vm-pooled-prod-001 → Connect → RDP
# Login as: localadmin

# Check FSLogix service running
Get-Service frxsvc
# Expected: Status = Running, StartType = Automatic

# Check registry configuration
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles" | Select-Object Enabled, VHDLocations, SizeInMBs

# Expected output:
# Enabled: 1
# VHDLocations: \\stavdprodeus01.file.core.windows.net\profiles-pooled
# SizeInMBs: 30000
```

### Network Connectivity

From session host, test connectivity to required endpoints:

```powershell
# Test AVD control plane
Test-NetConnection -ComputerName rdbroker.wvd.microsoft.com -Port 443
# Expected: TcpTestSucceeded = True

# Test storage account (profile container)
Test-NetConnection -ComputerName stavdprodeus01.file.core.windows.net -Port 445
# Expected: TcpTestSucceeded = True

# Test Entra authentication
Test-NetConnection -ComputerName login.microsoftonline.com -Port 443
# Expected: TcpTestSucceeded = True
```

If any test fails → see Troubleshooting: Network connectivity issues.

---

## Part 6: Troubleshooting

### Issue: Session hosts stuck in "Needs Assistance" state

**Symptom:** VMs deployed but show "Needs Assistance" in host pool session hosts list for 30+ minutes

**Cause:**
- Registration token expired
- AVD agent installation failed
- Network connectivity blocked to AVD control plane

**Fix:**

1. **Check registration token validity:**

   **Portal:** Host pools → `hp-pooled-prod` → Registration token

   - If **Expired:** Generate new token → Copy token
   - RDP to session host → Run as admin:

   ```powershell
   # Uninstall AVD agent
   Get-WmiObject -Class Win32_Product | Where-Object {$_.Name -like "*Remote Desktop*"} | ForEach-Object {$_.Uninstall()}

   # Download and reinstall agent with new token
   $agentUrl = "https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RWrmXv"
   $agentInstaller = "C:\Temp\AVDAgent.msi"
   Invoke-WebRequest -Uri $agentUrl -OutFile $agentInstaller

   Start-Process msiexec.exe -ArgumentList "/i $agentInstaller /quiet REGISTRATIONTOKEN=<YOUR-NEW-TOKEN>" -Wait

   # Restart VM
   Restart-Computer -Force
   ```

2. **Verify AVD agent running:**

   ```powershell
   Get-Service RDAgentBootLoader
   # Expected: Status = Running

   Get-Service RDAgent
   # Expected: Status = Running
   ```

3. **Check network connectivity:**

   ```powershell
   # Test AVD control plane endpoints
   Test-NetConnection -ComputerName rdbroker.wvd.microsoft.com -Port 443
   Test-NetConnection -ComputerName rdgateway.wvd.microsoft.com -Port 443

   # If fails: Check NSG on snet-avd-prod-sessionhosts allows outbound 443 to WindowsVirtualDesktop service tag
   ```

4. **Check agent logs:**

   ```
   C:\Program Files\Microsoft RDInfra\AgentInstall.txt
   C:\Program Files\Microsoft RDInfra\GenevaInstall.txt
   ```

   Look for errors related to:
   - Token validation
   - Network connectivity
   - Certificate trust issues

### Issue: VMs not joining Entra ID

**Symptom:** VMs deployed but not visible in Entra Admin Center → Devices after 15 minutes

**Cause:**
- Entra ID Join extension failed
- Insufficient permissions
- Network blocking Entra endpoints

**Fix:**

1. **Check VM extensions:**

   **Portal:** Virtual machines → vm-pooled-prod-001 → Extensions + applications

   - Verify `AADLoginForWindows` extension shows **Provisioning succeeded**
   - If failed: Click extension → View error details

2. **Verify network connectivity to Entra:**

   RDP to VM as local admin:

   ```powershell
   Test-NetConnection -ComputerName login.microsoftonline.com -Port 443
   # Expected: TcpTestSucceeded = True
   ```

3. **Check join status on VM:**

   ```powershell
   dsregcmd /status

   # Expected output includes:
   # AzureAdJoined: YES
   # DomainJoined: NO
   ```

   If `AzureAdJoined: NO`:

   ```powershell
   # Manually join to Entra ID (requires local admin)
   dsregcmd /join

   # Restart VM
   Restart-Computer -Force
   ```

### Issue: Intune enrollment failed

**Symptom:** VMs joined to Entra but not showing in Intune Admin Center

**Cause:**
- Automatic MDM enrollment not enabled
- Licensing issue (requires Entra ID P1/P2 + Intune license)
- Enrollment blocked by policy

**Fix:**

1. **Verify automatic enrollment enabled:**

   **Portal:** Entra Admin Center → Mobility (MDM and MAM) → Microsoft Intune

   - **MDM user scope:** All (or select group containing device accounts)
   - If set to None: Change to All → Save

2. **Check licensing:**

   Verify VMs' device account has required licenses (rare issue):

   Device accounts auto-created during Entra Join typically don't need licenses, but verify:

   **Portal:** Entra Admin Center → Devices → vm-pooled-prod-001 → Licenses

   - Should show: Inherited from group or no license required

3. **Manually trigger enrollment:**

   RDP to VM as local admin:

   ```powershell
   # Open Settings app
   Start-Process ms-settings:workplace

   # Navigate to: Access work or school → Connect
   # Sign in with Entra admin account
   # This forces manual enrollment
   ```

### Issue: FSLogix profile containers not created

**Symptom:** User logs in but no profile folder created in `\\stavdprodeus01.file.core.windows.net\profiles-pooled`

**Cause:**
- FSLogix not configured on session host
- Storage permissions incorrect
- Network blocking SMB (port 445)

**Fix:**

1. **Verify FSLogix enabled:**

   On session host:

   ```powershell
   Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles" | Select-Object Enabled, VHDLocations

   # Expected:
   # Enabled: 1
   # VHDLocations: \\stavdprodeus01.file.core.windows.net\profiles-pooled
   ```

   If missing: Reapply Intune configuration profile or configure manually (see Part 4).

2. **Test storage connectivity:**

   On session host as logged-in user (not local admin):

   ```powershell
   # Mount file share
   net use Z: \\stavdprodeus01.file.core.windows.net\profiles-pooled

   # Expected: "The command completed successfully."
   # If prompts for credentials: Entra auth issue
   # If "Access Denied": RBAC role not assigned
   ```

3. **Check FSLogix logs:**

   On session host:

   ```
   C:\ProgramData\FSLogix\Logs\Profile\Profile-<timestamp>.log
   ```

   Look for errors:
   - "Access Denied" → RBAC or NTFS permissions issue
   - "Network path not found" → DNS or connectivity issue
   - "Invalid VHDLocations" → Registry configuration wrong

**See:** [[../Storage/troubleshooting-fslogix-profiles|FSLogix Troubleshooting Reference]] for detailed FSLogix debugging.

### Issue: Network connectivity blocked

**Symptom:** Session hosts can't reach AVD control plane, storage, or Entra endpoints

**Cause:** NSG rules blocking outbound traffic

**Fix:**

1. **Verify NSG rules:**

   **Portal:** Network security groups → `nsg-avd-prod-sessionhosts` → Outbound security rules

   Confirm these rules exist with **Action: Allow**:
   - Priority 100: Destination `WindowsVirtualDesktop`, Port 443
   - Priority 110: Destination `AzureCloud`, Port 443
   - Priority 120: Destination `Internet`, Port 80,443

2. **Check effective security rules on VM NIC:**

   **Portal:** Virtual machines → vm-pooled-prod-001 → Networking → Network interface → Effective security rules

   - Review outbound rules
   - Verify no higher-priority deny rule blocking traffic

3. **Test connectivity from session host:**

   ```powershell
   # AVD control plane
   Test-NetConnection -ComputerName rdbroker.wvd.microsoft.com -Port 443

   # Storage account
   Test-NetConnection -ComputerName stavdprodeus01.file.core.windows.net -Port 445

   # Entra ID
   Test-NetConnection -ComputerName login.microsoftonline.com -Port 443

   # All should return: TcpTestSucceeded = True
   ```

---

## Next Steps

**Session hosts deployed and registered.** Users can now connect to pooled desktops.

**Immediate next step:** [[08-application-deployment|Step 08: Application Deployment (Intune)]]

In Step 08, you will:
- Deploy Microsoft 365 Apps via Intune
- Install Adobe Acrobat Reader
- Configure application assignments per department
- Test application availability in AVD sessions

**Alternative path:** If not using Intune, see:
- [[../Images/golden-image-process|Golden Image Process]] for baking apps into image

---

## Cost Estimate

**Monthly costs for 10 Standard_D4s_v5 session hosts:**

| Resource | Specification | Monthly Cost (USD) |
|----------|--------------|-------------------|
| Compute (10 VMs) | Standard_D4s_v5, 730 hours/month | ~$1,400 |
| OS Disks (10 VMs) | Premium SSD 128 GB | ~$100 |
| Networking | Data transfer (outbound) ~100 GB | ~$10 |
| Storage (profiles) | Premium FileStorage 512 GiB provisioned | ~$80 |
| **Total** | | **~$1,590/month** |

**Cost per user:** $1,590 / 150 users = **~$10.60/user/month** (infrastructure only)

**Cost optimization tips:**
- Use Azure Hybrid Benefit (if Windows Server licenses available) → save ~30%
- Implement scaling plans to shut down VMs during off-hours → save ~50% on compute
- Right-size VMs after monitoring usage for 2 weeks

**See:** [[../Operations/cost-optimization|Cost Optimization Reference]] for detailed cost-saving strategies.

---

## Related Reference Pages

- [[../AVD/session-host-sizing|Session Host Sizing]] - VM SKU selection, user density calculations, performance tuning
- [[../Images/azure-compute-gallery|Azure Compute Gallery]] - Image versioning, replication, sharing
- [[../Images/fslogix-profile-containers|FSLogix Profile Containers]] - Container configuration, sizing, troubleshooting
- [[../Operations/scaling-plans|Scaling Plans]] - Automated start/stop schedules for cost savings
- [[../Security/monitoring-and-diagnostics|Monitoring & Diagnostics]] - Azure Monitor setup, performance baselines
