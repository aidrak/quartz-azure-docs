---
title: Step 09 - Session Host Provisioning
description: Deploy session hosts to pooled host pool
published: true
date: 2025-12-14T00:00:00.000Z
tags: [Quick-Deploy, session-hosts, vms, deployment]
---

# Step 09: Session Host Provisioning

Deploy VMs to the pooled host pool using the Golden Image from Azure Compute Gallery.

## Prerequisites
- [ ] **Host Pool** created: `hp-pooled-prod`
- [ ] **Registration Token** generated (valid)
- [ ] **Golden Image** ready: `gal-avd-prod-01` / `win11-multisession-25h2`
- [ ] **Network** ready: `vnet-avd-prod-01` / `snet-avd-prod-sessionhosts`
- [ ] **Entra ID Group**: `avd-devices-pooled`

> **VM Sizing:** See [[AVD/session-host-sizing|Session Host Sizing]] for SKU recommendations.

---

## Deploy Session Hosts

**Navigation:** Azure Portal -> **Azure Virtual Desktop** -> **Host pools** -> `hp-pooled-prod` -> **Session hosts** -> **+ Add**.

| Setting | Value |
| :--- | :--- |
| **Resource group** | `rg-avd-prod-01` |
| **Name prefix** | `vm-pool-` |
| **Location** | East US |
| **Security type** | Trusted launch virtual machines |
| **Number of VMs** | 10 |
| **Image** | Shared Images > `gal-avd-prod-01` > `win11-multisession-25h2` |
| **Size** | Standard_D4s_v5 |
| **Network** | `vnet-avd-prod-01` |
| **Subnet** | `snet-avd-prod-sessionhosts` |
| **Domain to join** | Microsoft Entra ID |
| **Intune enrollment** | Yes |
| **Identity** | System assigned (if using custom script extension) |
| **Local Admin** | `entra-admin` (save credentials) |
| **Register desktop app group** | Yes |
| **Workspace** | `ws-prod` |
| **Registration Token** | Verify populated (generate new if needed) |

**Review + Create** when validation passes.

---

## Part 2: Configure Custom Script Extension (Optional)

> **Decision Point:**
> - **Need post-deployment configuration?** Configure custom script below
> - **Golden image handles everything?** Skip to Post-Deployment Checklist

Custom scripts automate post-deployment tasks (software installation, Windows Defender configuration, registry settings, etc.). Scripts run during VM creation, before AVD Agent installation.

**During Session Host Deployment:**

In the form above, scroll down and find **Custom configuration** section:

| Setting | Value |
| :--- | :--- |
| **Custom configuration script URL** | `https://raw.githubusercontent.com/aidrak/azure-scripts/main/Deploy-AVDHost.ps1` |

**Notes:**
- Scripts are hosted in public GitHub repository: [github.com/aidrak/azure-scripts](https://github.com/aidrak/azure-scripts)
- Script runs as SYSTEM (elevated) before AVD Agent installation
- No firewall or permission configuration needed
- For detailed setup and troubleshooting, see [[../AVD/custom-script-extensions|Custom Script Extensions Reference]]

---

## Part 3: Post-Deployment Checklist

1. **Verify Status:** Azure Portal -> `hp-pooled-prod` -> **Session hosts**. Status should show **Available**.
2. **Verify Custom Script:** If script configured, check logs on one VM:
   - RDP to `vm-pool-001`
   - View: `C:\AVDDeployment.log` (or your script's log file)
   - Verify all commands executed successfully
   - Check: `C:\WindowsAzure\Logs\Plugins\Microsoft.Compute.CustomScriptExtension\` for extension logs
3. **Entra ID Check:** Go to **Entra ID** > **Devices**. VMs should show as **Entra joined** and **Intune-managed**.
4. **Configure FSLogix:** See [[../Deployment/07-fslogix-configuration|Step 07]] for profile container setup (if not already done).
5. **Monitoring:** See [[../Deployment/11-go-live-monitoring|Step 11]] for Insights configuration.

---

## Critical Issues

**Session Host Stuck in "Needs Assistance"**
- **Cause:** Expired registration token or network connectivity blocked
- **Fix:** Generate new token in Host Pool > **Session hosts** > **Register a registration token**. Re-run deployment or manually install agent on VM

**VM Not Joining Entra ID**
- **Cause:** Network blocking Entra endpoints (port 443 outbound)
- **Fix:** On VM: `dsregcmd /status`. Verify outbound HTTPS allowed to Microsoft Entra endpoints

---

## Next Steps

> **Decision Point:**
> - **Standard path:** Proceed to [[Deployment/08-app-groups-workspace|Step 08 - App Groups & Workspace]]
> - **Customization needed:** See [[Images/golden-image-process|Golden Image Process]] for VM customization

---

## Related References

- [[../AVD/custom-script-extensions|Custom Script Extensions]] - Automated post-deployment configuration
- [[../AVD/session-host-sizing|Session Host Sizing]] - SKU selection and user density
- [[../Images/golden-image-process|Golden Image Process]] - Windows OS configuration
- [[../Security/windows-security-baseline|Windows Security Baseline]] - Post-deployment hardening
- [[../Deployment/07-fslogix-configuration|Step 07 - FSLogix Configuration]]
- [[../Deployment/11-go-live-monitoring|Step 11 - Monitoring & Insights]]
