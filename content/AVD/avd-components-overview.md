---
title: AVD Components Overview
description: 
published: true
date: 2025-12-14T04:52:19.926Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:05.968Z
---

# AVD Components Overview

Azure Virtual Desktop (AVD) is a Microsoft-managed desktop and application virtualization service that runs on Azure. Understanding the architecture is critical for designing, deploying, and troubleshooting AVD environments. This page breaks down the core components, the division of responsibilities between Microsoft and customers, and how user connections flow through the system.

## What is AVD Architecture

AVD architecture is divided into two fundamental planes:

**Control Plane (Microsoft-Managed):**
The control plane handles orchestration, authentication, connection brokering, and web access. Microsoft manages and operates these components globally, providing high availability and automatic updates. You never deploy or maintain control plane infrastructure—it's a fully managed service. The control plane includes:

- **Web Access:** The browser-based portal where users can launch their desktops and apps
- **Gateway:** Handles reverse-connect technology to establish secure connections to session hosts
- **Connection Broker:** Determines which session host a user connects to based on host pool configuration and load balancing
- **Diagnostics:** Telemetry and health monitoring for the entire AVD environment
- **Management APIs:** Used by Azure Portal, PowerShell, CLI, and ARM templates to configure AVD

**Data Plane (Customer-Managed):**
The data plane consists of the actual session hosts (VMs) and supporting infrastructure that you deploy and manage in your Azure subscription. You control the configuration, patching, monitoring, and scaling of these resources:

- **Session Host VMs:** Windows 10/11 Multi-Session or Windows Server VMs that host user sessions
- **Virtual Networks:** Network connectivity, subnets, NSGs, and routing
- **Storage:** Profile storage (FSLogix), file shares, and data disks
- **Active Directory/Entra ID:** Identity services for authentication and authorization
- **Supporting Services:** Key Vault for secrets, Log Analytics for monitoring, Azure Files for profiles

This division of responsibility means Microsoft ensures the brokering and connection infrastructure is always available, while you focus on the compute and network resources that run user workloads.

## Connection Flow

Understanding how users connect to their desktops is essential for troubleshooting connectivity issues:

1. **User Authentication:** User opens Remote Desktop client (Windows, macOS, iOS, Android, web) and authenticates with Microsoft Entra ID credentials
2. **Resource Discovery:** Control plane queries workspace assignments and returns available desktops/apps to the user
3. **Connection Request:** User selects a desktop, client sends connection request to AVD Gateway
4. **Brokering Decision:** Connection Broker evaluates host pool configuration (load balancing algorithm, session limits, drain mode) and selects the appropriate session host
5. **Reverse Connect:** Session host maintains outbound HTTPS connection to AVD Gateway (no inbound ports required), Gateway brokers the connection
6. **Session Establishment:** RDP traffic flows through the Gateway to the session host, user sees their desktop
7. **Authentication to VM:** User credentials are passed to the session host for Windows logon (Entra ID, Hybrid Entra ID, or AD DS)

**Key Point:** Session hosts initiate outbound connections to the Gateway, not inbound. This eliminates the need for public IPs or inbound RDP ports on session hosts, significantly improving security.

## Required Azure Resources

Every AVD deployment requires these resources in your Azure subscription:

| Resource | Purpose | Example from Our Environment |
|----------|---------|------------------------------|
| **Resource Group** | Logical container for AVD resources | RG-Azure-VDI-01 |
| **Virtual Network** | Network connectivity for session hosts | vnet-avd-prod (10.10.0.0/16) |
| **Subnet** | Segment for session host NICs | snet-sessionhosts (10.10.1.0/24) |
| **Workspace** | User-facing collection of app groups | ws-avd-prod ("AVD Production Workspace") |
| **Host Pool** | Collection of identical session hosts | hp-pooled-prod1 (DepthFirst, 10 max sessions) |
| **Application Group** | Published desktops or apps | hp-pooled-prod1-DAG (Desktop type) |
| **Session Host VMs** | Windows VMs hosting user sessions | avd-pool-0 (Standard_D2s_v4) |
| **Storage Account** | FSLogix profile containers | saavdprofiles (Azure Files Premium) |
| **Identity Service** | User authentication | Microsoft Entra ID (Entra Joined VMs) |

Optional but recommended resources:

- **Network Security Group (NSG):** Control traffic to/from session hosts
- **Azure Bastion:** Secure administrative access without public IPs
- **Log Analytics Workspace:** Centralized logging and monitoring
- **Key Vault:** Secure storage for certificates and secrets
- **Scaling Plan:** Autoscale session hosts based on schedule or demand

## Licensing Requirements

AVD licensing can be confusing, but it follows these rules:

**Windows Client Licensing (Windows 10/11 Multi-Session):**

You need one of these per user:
- Microsoft 365 E3/E5/F3/Business Premium
- Windows Enterprise E3/E5
- Windows VDA (Virtual Desktop Access) per user

**Windows Server Licensing (Windows Server 2019/2022):**

You need:
- RDS CAL (Remote Desktop Services Client Access License) per user or device
- Windows Server license for the VMs (bring your own license or pay-as-you-go in Azure)

**External Users (B2B scenarios):**

- RDS Subscriber Access License (SAL) per user per month

**Our Environment:**
Since our session hosts are Entra Joined and running Windows 10/11 Multi-Session, users must have Microsoft 365 E3 or higher. This is the most common scenario for modern AVD deployments.

**Cost Optimization:**
If you have existing on-premises RDS CALs with Software Assurance, you may be able to use them for AVD Server-based deployments, but Windows Client multi-session is only available with M365/E3 licensing.

## Microsoft-Managed vs Customer-Managed

This table clarifies what Microsoft handles versus what you manage:

| Component | Managed By | Your Responsibility |
|-----------|------------|---------------------|
| **AVD Control Plane** | Microsoft | None—service availability guaranteed by SLA |
| **Gateway & Broker** | Microsoft | None—globally distributed, auto-updated |
| **Web Access Portal** | Microsoft | None—accessible at rdweb.wvd.microsoft.com |
| **Session Host VMs** | Customer | Deploy, patch, monitor, scale, secure |
| **Virtual Network** | Customer | Design subnets, NSGs, routing, connectivity |
| **Storage (Profiles)** | Customer | Provision Azure Files, configure FSLogix |
| **Identity (Entra/AD)** | Customer | User accounts, groups, authentication method |
| **Monitoring & Logs** | Shared | Microsoft provides diagnostics, you configure Log Analytics |
| **Updates & Patching** | Customer | Windows Updates, application patching on session hosts |
| **Disaster Recovery** | Customer | Backup profiles, session host images, IaC templates |

**Key Insight:** Microsoft ensures the control plane is always available (99.9% SLA), but session host availability and performance are your responsibility. This is why proper monitoring, scaling, and image management are critical.

## Best Practices

- **Separate Resource Groups** - Use different resource groups for AVD control plane objects (host pools, app groups, workspaces) versus infrastructure (VMs, networks, storage) to simplify RBAC and cost tracking
- **Dedicated Subnets** - Place session hosts in their own subnet with NSG rules limiting outbound traffic to only required endpoints (Entra ID, AVD control plane, KMS activation, Windows Update)
- **Image Management** - Use Azure Compute Gallery (formerly Shared Image Gallery) to version and replicate golden images across regions for consistency and disaster recovery
- **Profile Storage Redundancy** - Use Zone-Redundant Storage (ZRS) or Geo-Redundant Storage (GRS) for FSLogix profile containers to prevent data loss
- **Monitoring Baselines** - Enable Azure Monitor for AVD and establish performance baselines (CPU, memory, disk, logon time) before users report issues
- **Hybrid Identity** - For organizations with on-premises AD DS, use Hybrid Entra ID Join for session hosts to leverage existing GPOs and on-prem file shares

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Users see no resources in Remote Desktop client** | User not assigned to any application group | Assign user or group to app group in Azure Portal → AVD → Application Groups → Assignments |
| **Connection fails after broker selects session host** | Session host cannot reach AVD Gateway (NSG/firewall blocking outbound) | Verify session host can reach *.wvd.microsoft.com on port 443, check NSG outbound rules |
| **"No resources available" error** | Host pool has no active session hosts (all stopped or drained) | Start at least one session host, verify Registration Token is valid |
| **Profile fails to load (temporary profile)** | FSLogix cannot access Azure Files share (network, permissions, or SMB ports blocked) | Check storage account firewall allows VNET, verify session host can reach port 445, confirm RBAC roles |
| **Authentication loop in web client** | Conditional Access policy blocking session host authentication | Review Entra ID Conditional Access policies, ensure session hosts are excluded or compliant |
| **High latency or poor performance** | Session host undersized or over-subscribed | Check Azure Monitor metrics (CPU >80%, RAM >90%), reduce max sessions per host or scale up VM size |

---

**Next:** Proceed to "Host Pools Deep Dive" to understand how to configure and optimize host pools for pooled vs personal scenarios.