---
title: Session Host Provisioning
description: 
published: true
date: 2025-12-14T04:52:43.971Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:38.085Z
---

# Session Host Provisioning

Deploy session host VMs and register them with the host pool. Session hosts run user sessions and must be properly configured for optimal performance.

## Add Session Hosts to Host Pool

**Portal:** Host Pool → Session hosts → Add

### Virtual Machines Tab

- **Resource group:** RG-Azure-VDI-01
- **Name prefix:** `avd-pool` (creates avd-pool-0, avd-pool-1, etc.)
- **Virtual machine location:** Central US
- **Availability options:** Availability zone (recommended)
- **Image:** Select from gallery → avd_gallery → Win11_Multi_25H2_Gen2 → 1.0.2 (latest)
- **VM size:** Standard_D2s_v4 (2 vCPU, 8GB RAM)
- **Number of VMs:** Start with 1-2 for testing

### Disks
- **OS disk type:** Premium SSD (for performance)
- **OS disk size:** 127 GiB (default)

### Network
- **Virtual network:** vnet-avd
- **Subnet:** snet-sessionhosts
- **Network security group:** nsg-snet-sessionhosts
- **Public IP:** No

### Domain to Join
- **Domain:** Microsoft Entra ID (Entra Join)
- **Intune enrollment:** Yes

> **Decision Point:**
> - **Entra Join (cloud-only):** Select "Microsoft Entra ID" - our standard
> - **Hybrid Join (on-prem AD):** Select "Active Directory" and provide domain credentials

### Virtual Machine Administrator Account
- **Username:** localadmin
- **Password:** (secure password, store in Key Vault)

### Custom Configuration (Optional)
- **Custom configuration script:** Can reference FSLogix GPO or Intune policies

## Post-Deployment Verification

### Check Session Host Status

**Portal:** Host Pool → Session hosts

| Status | Meaning | Action |
|--------|---------|--------|
| Available | Ready for connections | None |
| Unavailable | Not responding | Check VM, agent |
| Needs attention | Agent issue | Reinstall agent |
| Shutdown | VM deallocated | Start VM |

### Verify Agent Installation

Connect to session host and check:

```powershell
# Check AVD agent services
Get-Service RDAgentBootLoader, RdAgent

# Check agent version
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\RDInfraAgent" | Select-Object Version

# Check Entra Join status
dsregcmd /status
```

### Verify FSLogix

```powershell
# Check FSLogix service
Get-Service frxsvc, frxccds

# Check FSLogix configuration
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles"
```

## Scale Out

After testing, add more session hosts:

| User Type | Users per Host | Recommendation |
|-----------|----------------|----------------|
| Light (Office, web) | 8-10 | Standard_D2s_v4 |
| Medium (Light apps) | 4-6 | Standard_D4s_v4 |
| Heavy (Development) | 2-3 | Standard_D8s_v4 |
| GPU (Graphics) | 2-4 | NV-series |

## Verification

- [ ] Session hosts deployed
- [ ] Status shows "Available"
- [ ] Entra Join completed
- [ ] Intune enrollment completed
- [ ] FSLogix agent installed
- [ ] AVD agent connected

---

**Next:** [[app-groups-workspace|Step 9: App Groups & Workspace]]