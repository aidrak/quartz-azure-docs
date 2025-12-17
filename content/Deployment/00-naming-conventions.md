---
title: Step 00 - Naming Conventions & Planning
description: Define consistent naming patterns for all Azure resources in AVD deployment
published: true
date: 2025-12-14
tags: [Quick-Deploy, naming, planning]
---

# Step 00: Naming Conventions & Planning

Define consistent naming patterns for all Azure resources in your AVD deployment. This ensures predictability across the 12-step deployment process and simplifies management at scale.

## Example Scenario

200-user deployment: Pooled + Personal host pools, Cloud-Only identity, Azure Files storage

| Resource | Name | Qty |
|----------|------|-----|
| Resource Group | `rg-avd-prod-01` | 1 |
| Pooled Host Pool | `hp-pooled-prod` | 1 |
| Pooled VMs | `vm-pooled-001` through `-010` | 10 |
| Personal Host Pool | `hp-personal-prod` | 1 |
| Personal VMs | `vm-personal-011` through `-050` | 40 |
| Storage Account | `stavdprod01` | 1 |

---

## Naming Patterns (Quick Reference)

### Core Infrastructure

| Resource | Pattern | Example |
|----------|---------|---------|
| Resource Group | `rg-avd-{env}-{num}` | `rg-avd-prod-01` |
| VNET | `vnet-avd-{env}-{num}` | `vnet-avd-prod-01` |
| Subnet (hosts) | `snet-avd-{env}-sessionhosts` | `snet-avd-prod-sessionhosts` |
| Subnet (endpoints) | `snet-avd-{env}-privateendpoints` | `snet-avd-prod-privateendpoints` |
| NSG | `nsg-avd-{env}-{purpose}` | `nsg-avd-prod-sessionhosts` |

**VNET Design:**
- Session Hosts Subnet: 10.0.1.0/24 (pooled + personal together)
- Private Endpoints Subnet: 10.0.2.0/24 (storage, Key Vault)

### AVD Resources

| Resource | Pattern | Example | Notes |
|----------|---------|---------|-------|
| Host Pool (pooled) | `hp-{type}-{env}` | `hp-pooled-prod` | Breadth-first load balancing |
| Host Pool (personal) | `hp-{type}-{env}` | `hp-personal-prod` | 1:1 VMs to users |
| Session Hosts (pooled) | `vm-pooled-{nnn}` | `vm-pooled-001` to `-010` | 13-char; 001-010 reserved for pooled |
| Session Hosts (personal) | `vm-personal-{nnn}` | `vm-personal-011` to `-050` | 15-char limit; 011-050 reserved for personal |
| Application Groups | `ag-{type}-{env}` | `ag-pooled-prod` | One per host pool type |
| Workspace | `ws-{env}` | `ws-prod` | Aggregates all AppGroups |

### Storage & Profiles

| Resource | Pattern | Example | Notes |
|----------|---------|---------|-------|
| Storage Account | `st{purpose}{env}{num}` | `stavdprod01` | No hyphens, max 24 chars |
| File Share (pooled) | `profiles-pooled` | `profiles-pooled` | 512GB, FSLogix profiles for pooled hosts |
| Private Endpoint | `pe-{resource}-{env}` | `pe-storage-prod` | Secure access to storage |

### Images & Gallery

| Resource | Pattern | Example | Notes |
|----------|---------|---------|-------|
| Azure Compute Gallery | `gal-avd-{env}-{num}` | `gal-avd-prod-01` | Stores image definitions |
| Image Definition (multi) | `win{ver}-multisession` | `win11-multisession-25h2` | For pooled hosts |
| Image Definition (single) | `win{ver}-singlesession` | `win11-singlesession-25h2` | For personal hosts |

### Identity (Entra ID)

| Resource | Pattern | Example | Notes |
|----------|---------|---------|-------|
| User Group (pooled) | `avd-users-{type}` | `avd-users-pooled` | 120 users |
| User Group (personal) | `avd-users-{type}` | `avd-users-personal` | 40 users |
| Admin Group | `avd-users-admin` | `avd-users-admin` | RBAC on RG |
| Managed Identity | `id-avd-automation-{env}` | `id-avd-automation-prod` | Automation/Image Builder identity |
| Device Group (pooled) | `avd-devices-{type}` | `avd-devices-pooled` | Dynamic: `(device.displayName -startsWith "vm-pooled-")` |
| Device Group (personal) | `avd-devices-{type}` | `avd-devices-personal` | Dynamic: `(device.displayName -startsWith "vm-personal-")` |
| Device Group (all) | `avd-devices-all` | `avd-devices-all` | Dynamic: `(device.displayName -startsWith "vm-pooled-") -or (device.displayName -startsWith "vm-personal-")` |

### Monitoring & Security

| Resource | Pattern | Example |
|----------|---------|---------|
| Log Analytics | `law-avd-{env}-{num}` | `law-avd-prod-01` |
| Key Vault | `kv-avd-{env}` | `kv-avd-prod` |

---

## Complete Resource Deployment Checklist

**All resources for 200-user deployment (~32 resources):**

### Quick Checklist

| # | Resource Type | Qty | Pattern | Example |
|---|---------------|-----|---------|---------|
| 1 | Subscription | 1 | `sub-{org}-{env}` | `sub-aidrak-prod-avd` |
| 2 | Resource Group | 1 | `rg-avd-{env}-{num}` | `rg-avd-prod-01` |
| 3 | VNET + Subnets | 1+2 | `vnet-avd-{env}-{num}` | `vnet-avd-prod-01` |
| 4 | NSGs | 2 | `nsg-avd-{env}-{purpose}` | `nsg-avd-prod-sessionhosts` |
| 5 | Host Pool (pooled) | 1 | `hp-pooled-{env}` | `hp-pooled-prod` |
| 6 | Session Hosts (pooled) | 10 | `vm-pooled-{nnn}` | `vm-pooled-001` to `-010` |
| 7 | Host Pool (personal) | 1 | `hp-personal-{env}` | `hp-personal-prod` |
| 8 | Session Hosts (personal) | 40 | `vm-personal-{nnn}` | `vm-personal-011` to `-050` |
| 9 | App Groups | 2 | `ag-{type}-{env}` | `ag-pooled-prod`, `ag-personal-prod` |
| 10 | Workspace | 1 | `ws-{env}` | `ws-prod` |
| 11 | Storage Account | 1 | `st{purpose}{env}{num}` | `stavdprod01` |
| 12 | File Share (pooled) | 1 | `profiles-pooled` | `profiles-pooled` |
| 13 | Azure Compute Gallery | 1 | `gal-avd-{env}-{num}` | `gal-avd-prod-01` |
| 14 | Image Definitions | 2 | `win{ver}-{type}` | `win11-multisession-25h2` |
| 15 | User Groups (Entra) | 3 | `avd-users-{type}` | `avd-users-pooled`, `avd-users-admin` |
| 16 | Device Groups (Entra) | 3 | `avd-devices-{type}` | `avd-devices-pooled`, `avd-devices-all` |
| 17 | Log Analytics | 1 | `law-avd-{env}-{num}` | `law-avd-prod-01` |
| 18 | Key Vault | 1 | `kv-avd-{env}` | `kv-avd-prod` |

**Total: ~31 resources** (pooled-only FSLogix)

---

## Azure Portal Setup

Before starting deployment, configure your Azure Portal for efficient resource management:

**Configure Portal Settings:**

1. **Portal:** Azure Portal (portal.azure.com)
2. Click the Settings icon (gear ⚙️) in the top-right corner
3. Select **Appearance + startup views** from the left menu
4. Configure the following settings:
   - **Menu behavior:** Docked
   - **Service menu behavior:** Expanded
   - **Theme:** Dark
   - **Startup page:** Home
5. Click **Apply** at the bottom

---

## Pre-Deployment Validation

Confirm before proceeding to Step 01:

- [ ] **Environment:** Production (or dev/test)
- [ ] **Naming patterns:** Understand patterns for all resource types
- [ ] **Save reference table:** Above table bookmarked for steps 1-11
- [ ] **Resource count:** ~31 resources needed for 200 users (pooled-only FSLogix)
- [ ] **Storage design:** Single FSLogix file share for pooled hosts; personal hosts do not use FSLogix
- [ ] **Network design:** Pooled/personal hosts share subnet, NSGs filter by VM role
- [ ] **AVD design:** 1 pooled host pool (120 users) + 1 personal host pool (40 users)
- [ ] **Identity design:** 3 user groups + 2 device groups for Intune assignment

---

## Key Design Decisions

**Resource Grouping:**
- All resources in single RG: `rg-avd-prod-01`
- Both host pool types in same VNET (cost savings, simplified management)

**Naming Philosophy:**
- Consistency: Same patterns across all resources
- Readability: Names describe purpose, not department
- Predictability: Can predict any resource name without looking it up
- Brevity: Stay within Azure limits (24 chars for storage)

**VM Numbering Strategy:**
- Pooled: `vm-pooled-001` to `vm-pooled-010` (10 VMs)
- Personal: `vm-personal-011` to `vm-personal-050` (40 VMs)
- Prevents accidental reuse, tracks capacity at a glance

---

## Next Steps

**1. Create Entra ID groups** (before deployment):
   - `avd-users-pooled` (120 users)
   - `avd-users-personal` (40 users)
   - `avd-users-admin` (5 admins)
   - `avd-devices-pooled` (dynamic: VM names start with "vm-pooled-")
   - `avd-devices-personal` (dynamic: VM names start with "vm-personal-")
   - `avd-devices-all` (dynamic: VM names start with "vm-pooled-" or "vm-personal-")
**2. Save naming reference** from table above

**3. Proceed to Step 01:**

[[01-prerequisites-licensing|Step 01: Prerequisites & Licensing]]

---

## Related Reference Pages

For detailed architecture and design decisions:
- [[../Networking/vnet-design|VNet Design Deep Dive]] - CIDR planning, subnets, NSGs
- [[../Storage/fslogix-profile-containers|FSLogix Profile Containers]] - Profile container architecture
- [[../Images/azure-compute-gallery|Azure Compute Gallery]] - Image definition versioning
- [[../Identity/dynamic-groups|Dynamic Group Configuration]] - Automatic device group membership
- [[../Operations/naming-strategy|Naming Strategy Rationale]] - Philosophy and best practices
