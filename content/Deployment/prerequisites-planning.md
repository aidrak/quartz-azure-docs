---
title: Prerequisites Planning
description: 
published: true
date: 2025-12-14T04:53:28.580Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:33.100Z
---

# Prerequisites & Planning

Before deploying Azure Virtual Desktop for a client, ensure all licensing, subscriptions, and tenant prerequisites are in place. This checklist prevents mid-deployment blockers.

## Licensing Requirements

| License Type | What It Provides | Required For |
|--------------|------------------|--------------|
| **Microsoft 365 E3/E5** | Windows 11 Enterprise, Intune, Entra ID P1 | Most AVD deployments |
| **Windows 11 Enterprise E3/E5** | Windows entitlement only | Non-M365 customers |
| **Entra ID P1** | Conditional Access, dynamic groups | Security policies |
| **Entra ID P2** | Identity Protection, PIM | Advanced security |
| **Intune Plan 1** | Device management | Endpoint management |

> **Note:** M365 E3 includes Windows 11 Enterprise multi-session rights. Verify with `https://portal.office.com/AdminPortal/Home#/licenses`.

## Azure Subscription

**Portal:** Azure Portal → Subscriptions

1. Confirm active subscription with sufficient quota
2. Verify user has Owner or Contributor + User Access Administrator
3. Check regional availability for AVD (Central US used in this guide)
4. Register required resource providers:

```
Microsoft.DesktopVirtualization
Microsoft.Compute
Microsoft.Network
Microsoft.Storage
```

## Entra ID Tenant

- [ ] Tenant ID documented
- [ ] Custom domain configured (optional but recommended)
- [ ] Break-glass admin account created
- [ ] MFA enforced for admins

## Network Planning

| Decision | Options | Our Choice |
|----------|---------|------------|
| IP Address Space | Must not overlap with on-prem | 10.0.0.0/16 |
| Connectivity | Cloud-only, VPN, ExpressRoute | VPN (S2S) |
| DNS | Azure DNS, Custom DNS, Hybrid | Azure DNS |

> **Decision Point:**
> - **Cloud-only:** Continue to Step 2
> - **Hybrid (VPN/ExpressRoute):** See [[vpn-gateway|VPN Gateway Reference]] first

## User & Group Strategy

Plan your Entra ID group structure before deployment:

| Group | Type | Purpose |
|-------|------|---------|
| AVD-Users-Pooled | Assigned | Users for shared desktops |
| AVD-Users-Personal | Assigned | Users for dedicated desktops |
| AVD-Users-Admins | Assigned | IT administrators |
| AVD-Devices-All | Dynamic | All AVD session hosts |

## Checklist Before Proceeding

- [ ] Licensing confirmed and assigned
- [ ] Azure subscription active with quota
- [ ] Network design documented
- [ ] Group naming convention agreed
- [ ] Deployment timeline communicated

---

**Next:** [[resource-group-tagging|Step 2: Resource Group & Tagging]]