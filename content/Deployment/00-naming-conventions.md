---
title: Step 00 - Naming Conventions & Planning
description: Define consistent naming patterns for all Azure resources in AVD deployment
published: true
date: 2025-12-14
tags: [Quick-Deploy, naming, planning]
---

# Step 00: Naming Conventions & Planning

**Portal Path:** N/A - Planning step (no Portal actions required)

**Prerequisites:**
- None (this is the foundation step)

**Duration:** ~30 minutes (planning only)

---

## Example Scenario

We're deploying Azure Virtual Desktop for a medium enterprise with ~200 users across four departments:
- **General** (120 users) - Pooled multi-session host pool
- **Finance** (40 users) - Pooled multi-session host pool
- **Creative** (30 users) - Personal single-session host pool
- **Executives** (10 users) - Personal single-session host pool

**Environment:** Production
**Storage:** Azure Files with FSLogix profile containers
**Management:** Intune with Entra ID Join
**Deployment Timeline:** Q1 2025

---

## Naming Convention Philosophy

All resource names follow these principles:

1. **Consistency**: Same resource type always uses the same pattern
2. **Readability**: Names are human-readable, not randomized
3. **Scoping**: Names include environment and purpose
4. **Brevity**: Stay within Azure limits (24 chars for storage accounts) while remaining clear
5. **Predictability**: Can predict resource name without looking it up

---

## Core Infrastructure Resources

### Subscription Naming
| Pattern | Example | Notes |
|---------|---------|-------|
| `sub-{org}-{env}-{purpose}` | `sub-aidrak-prod-avd` | Rarely change; kept simple |

**No Portal configuration needed.** Document in deployment runbook.

---

### Resource Groups

**Pattern:** `rg-avd-{env}-{num}`

| Environment | Example | Purpose |
|-------------|---------|---------|
| prod | `rg-avd-prod-01` | Production pooled resources |
| nonprod/dev | `rg-avd-dev-01` | Development/testing |

**Pooled + Personal Example:**
- `rg-avd-prod-01` contains pooled host pools, personal host pools, and shared infrastructure
- All resources deploy to same RG to simplify management and RBAC

---

### Virtual Networks (VNETs)

**Pattern:** `vnet-avd-{env}-{num}`

| Resource | Example | CIDR | Purpose |
|----------|---------|------|---------|
| Production VNET | `vnet-avd-prod-01` | 10.0.0.0/16 | All session hosts + infrastructure |

**Subnets within VNET:**

| Subnet Purpose | Pattern | Example | CIDR | Hosts |
|----------------|---------|---------|------|-------|
| Session hosts (pooled) | `snet-avd-{env}-sessionhosts` | `snet-avd-prod-sessionhosts` | 10.0.1.0/24 | 200+ VMs |
| Session hosts (personal) | `snet-avd-{env}-sessionhosts` | `snet-avd-prod-sessionhosts` | (same) | (same) |
| Private endpoints | `snet-avd-{env}-privateendpoints` | `snet-avd-prod-privateendpoints` | 10.0.2.0/24 | Storage, Key Vault |
| Gateway (VPN/ER) | `snet-avd-{env}-gateway` | `snet-avd-prod-gateway` | 10.0.3.0/24 | VPN/ER endpoint |

> **Note:** Pooled and personal session hosts use the SAME subnet. NSGs filter traffic by VM role.

---

### Network Security Groups (NSGs)

**Pattern:** `nsg-avd-{env}-{purpose}`

| NSG Purpose | Example | Attached To |
|-------------|---------|-------------|
| Session host NSG | `nsg-avd-prod-sessionhosts` | `snet-avd-prod-sessionhosts` |
| Private endpoint NSG | `nsg-avd-prod-privateendpoints` | `snet-avd-prod-privateendpoints` |

**Key Inbound Rules (example `nsg-avd-prod-sessionhosts`):**
- Allow RDP (3389) from admin IP range (e.g., on-prem jump box)
- Allow WinRM (5985/5986) from Intune service
- Allow Azure service tags (Storage, EventHub, etc.)

**Key Outbound Rules:**
- Allow HTTPS (443) to Azure services (AVD, Storage, Intune, Microsoft 365)
- Allow NTP (123) for time sync
- Deny all other (default-deny, whitelist approach)

---

## AVD-Specific Resources

### Host Pools

**Pattern:** `hp-{type}-{env}`

Naming keeps focus on TYPE (pooled vs personal) and ENVIRONMENT. Host pool names do NOT include department names.

| Host Pool Type | Example | Load Balancing | Max Sessions | VM Count |
|----------------|---------|-----------------|--------------|----------|
| Pooled (General/Finance) | `hp-pooled-prod` | Breadth-first | 4/VM | 10-15 |
| Personal (Executives/Creative) | `hp-personal-prod` | N/A (1:1) | 1/VM | 40 |

> **Why no department in name?** Users are assigned via Application Groups + User Groups. A single host pool can serve multiple departments via different AppGroups.

---

### Session Host Virtual Machines

**Pattern:** `vm-{type}-{env}-{nnn}`

| VM Type | Environment | Number | Example | Count |
|---------|-------------|--------|---------|-------|
| Pooled | prod | 001-010 | `vm-pooled-prod-001` through `vm-pooled-prod-010` | 10 |
| Personal | prod | 011-050 | `vm-personal-prod-011` through `vm-personal-prod-050` | 40 |

**Numbering Strategy:**
- Pooled VMs: 001-010 (reserved for pooled)
- Personal VMs: 011-050 (reserved for personal)
- Keeps VM count clear at a glance; prevents accidental reuse

**VM Sizing & Image:**
- Pooled: Standard_D4s_v5 (4 vCPU, 16GB RAM), Windows 11 multi-session
- Personal: Standard_D4s_v5 (can scale higher if needed), Windows 11 single-session

---

### Application Groups

**Pattern:** `ag-{type}-{env}`

| AppGroup Type | Example | Assigned Users | Host Pool |
|---------------|---------|-----------------|-----------|
| Pooled Desktop | `ag-pooled-prod` | AVD-Pooled-Users | hp-pooled-prod |
| Personal Desktop | `ag-personal-prod` | AVD-Personal-Users | hp-personal-prod |

> **Note:** If later adding RemoteApp groups, would use: `ag-remoteapp-{app}-{env}` (e.g., `ag-remoteapp-office-prod`).

---

### Workspaces

**Pattern:** `ws-{env}`

| Workspace | Example | Assigned AppGroups | Usage |
|-----------|---------|-------------------|-------|
| Production | `ws-prod` | All (pooled + personal) | Users see all available resources |

> **Single Workspace Approach:** One workspace aggregates all AppGroups. Users see one "workspace" with pooled AND personal desktops depending on their group membership.


---

## Identity & Access Control

### Entra ID User Groups

**Pattern:** `AVD-{Type}-Users`

| Group Name | Group Type | Membership | Used For |
|------------|-----------|-----------|----------|
| AVD-Pooled-Users | Assigned | General + Finance users (120) | Assigned to `ag-pooled-prod` |
| AVD-Personal-Users | Assigned | Executives + Creative users (40) | Assigned to `ag-personal-prod` |
| AVD-Admins | Assigned | IT admins (3-5) | RBAC on RG + host pool |

> **Consider:** Later can convert to Dynamic Groups:
> - `AVD-Pooled-Users` → Dynamic based on department attribute
> - Enables self-service user assignment via HR system

---

### Entra ID Device Groups

**Pattern:** `AVD-Devices-{Type}`

| Group Name | Group Type | Membership | Used For |
|------------|-----------|-----------|----------|
| AVD-Devices-Pooled | Dynamic (device property) | Pooled VMs (10) | Intune config profile assignment |
| AVD-Devices-Personal | Dynamic (device property) | Personal VMs (40) | Intune config profile assignment |

**Dynamic Rule Example (for Pooled):**
```
(device.displayName -startsWith "vm-pooled-prod-")
```

Benefits:
- Auto-adds new VMs to correct group
- Auto-removes decommissioned VMs
- No manual group management

---

### Conditional Access Policies

**Pattern:** `CA-AVD-{type}-{condition}`

| Policy | Example | Applies To | Condition |
|--------|---------|-----------|-----------|
| AVD MFA | `CA-AVD-MFA-Required` | AVD-Pooled-Users, AVD-Personal-Users | Require MFA from outside corp network |
| Block High Risk | `CA-AVD-Block-HighRisk` | All users | Block if impossible travel detected |

> **Reference:** See [[../../Reference/Identity/conditional-access|Conditional Access Deep Dive]] for full policy design.

---

## Storage & Profiles

### Storage Accounts

**Pattern:** `st{purpose}{env}{num}`

Azure storage account names: lowercase only, 24 chars max, no hyphens.

| Storage Account | Example | Purpose | Tier |
|-----------------|---------|---------|------|
| AVD Profiles | `stavdprod01` | FSLogix profile containers + MSIX app attach | Premium (SMB 3.1.1) |
| Backup (future) | `stbackupprod01` | VM snapshots, backups | Standard |

**Breakdown of `stavdprod01`:**
- `st` = storage account
- `avd` = purpose
- `prod` = environment
- `01` = instance number (if multiple needed)

---

### File Shares

**Pattern:** `profiles-{type}`

| File Share | Example | Size | Mount Point (VMs) |
|-----------|---------|------|-------------------|
| Pooled profiles | `profiles-pooled` | 30GB (max recommended) | `\\stavdprodeus01.file.core.windows.net\profiles-pooled` |
| Personal profiles | `profiles-personal` | 50GB | `\\stavdprodeus01.file.core.windows.net\profiles-personal` |

> **Why separate shares?** Different NTFS permissions and quota sizing per department.

**SMB Settings (same for both):**
- Minimum: SMB 3.1.1
- Encryption: AES-256-GCM
- Large file shares enabled
- Access tier: Hot (for frequent access)

---

## Images & Gallery

### Azure Compute Gallery

**Pattern:** `gal-avd-{env}-{num}`

| Gallery | Example | Shared With | Purpose |
|---------|---------|------------|---------|
| Production | `gal-avd-prod-01` | Current subscription (shared to multiple subnets if needed) | Custom Windows 11 images |

---

### Image Definitions

**Pattern:** `win{version}-{type}-{purpose}`

| Image | Example | Windows Version | Session Type | Used By |
|-------|---------|-----------------|--------------|---------|
| Multi-session | `win11-multisession-23h2` | Windows 11 Enterprise multi-session | Pooled | hp-pooled-prod |
| Single-session | `win11-singlesession-23h2` | Windows 11 Enterprise single-session | Personal | hp-personal-prod |

**Image Versioning (within each definition):**
- v1.0: Base OS + Windows updates
- v1.1: + FSLogix agent
- v2.0: Major update (new Windows version, framework, etc.)

---

## Monitoring & Management

### Log Analytics Workspace

**Pattern:** `law-avd-{env}-{num}`

| Workspace | Example | Linked To | Retention |
|-----------|---------|-----------|-----------|
| Production | `law-avd-prod-01` | All session hosts (via Intune) | 30 days |

**Connected Data Sources:**
- Windows Event Logs (System, Application)
- Syslog (Linux, if any)
- Performance counters
- Azure Diagnostics (storage account, NSG flow logs)

---

### Azure Monitor Workbooks

**Pattern:** `workbook-avd-{topic}`

| Workbook | Example | Data Source | Audience |
|----------|---------|------------|----------|
| AVD Insights | `workbook-avd-insights` | Log Analytics | Operations team |
| User Connections | `workbook-avd-connections` | Log Analytics | Help desk |

> Created via Azure Portal → Monitor → Workbooks. These are dashboards for viewing health/performance.

---

### Automation Accounts (optional, future)

**Pattern:** `aa-avd-{env}`

If later automating start/stop or scaling:
| Automation Account | Example | Purpose |
|-------------------|---------|---------|
| Production | `aa-avd-prod` | Runbooks for host pool scaling, VM start/stop |

---

### Key Vault

**Pattern:** `kv-avd-{env}`

| Key Vault | Example | Stores |
|-----------|---------|--------|
| Production | `kv-avd-prod` | Storage account keys, DSC credentials, certificates |

---

## Resource Deployment Mapping Table

**Complete resource list for medium enterprise AVD deployment (200 users):**

### Quick Checklist

| # | Resource Type | Quantity | Naming Pattern | Examples |
|---|---------------|----------|-----------------|----------|
| **Core Infrastructure** |
| 1 | Subscription | 1 | `sub-{org}-{env}-{purpose}` | `sub-aidrak-prod-avd` |
| 2 | Resource Group | 1 | `rg-avd-{env}-{num}` | `rg-avd-prod-01` |
| 3 | Virtual Network | 1 | `vnet-avd-{env}-{num}` | `vnet-avd-prod-01` |
| 4 | Subnet (session hosts) | 1 | `snet-avd-{env}-{purpose}` | `snet-avd-prod-sessionhosts` |
| 5 | Subnet (private endpoints) | 1 | `snet-avd-{env}-{purpose}` | `snet-avd-prod-privateendpoints` |
| 6 | NSG (session hosts) | 1 | `nsg-avd-{env}-{purpose}` | `nsg-avd-prod-sessionhosts` |
| 7 | NSG (private endpoints) | 1 | `nsg-avd-{env}-{purpose}` | `nsg-avd-prod-privateendpoints` |
| 8 | Azure Firewall | 0-1 | `afw-avd-{env}` | `afw-avd-prod` (optional) |
| **AVD Resources** |
| 9 | Host Pool (pooled) | 1 | `hp-{type}-{env}` | `hp-pooled-prod` |
| 10 | Host Pool (personal) | 1 | `hp-{type}-{env}` | `hp-personal-prod` |
| 11 | Session Hosts (pooled) | 10 | `vm-{type}-{env}-{nnn}` | `vm-pooled-prod-001` to `-010` |
| 12 | Session Hosts (personal) | 40 | `vm-{type}-{env}-{nnn}` | `vm-personal-prod-011` to `-050` |
| 13 | Application Group (pooled) | 1 | `ag-{type}-{env}` | `ag-pooled-prod` |
| 14 | Application Group (personal) | 1 | `ag-{type}-{env}` | `ag-personal-prod` |
| 15 | Workspace | 1 | `ws-{env}` | `ws-prod` |
| **Storage & Profiles** |
| 16 | Storage Account (profiles) | 1 | `st{purpose}{env}{num}` | `stavdprod01` |
| 17 | File Share (pooled profiles) | 1 | `profiles-{type}` | `profiles-pooled` |
| 18 | File Share (personal profiles) | 1 | `profiles-{type}` | `profiles-personal` |
| 19 | Private Endpoint (storage) | 1 | `pe-{resource}-{env}` | `pe-storage-prod` |
| **Images & Gallery** |
| 20 | Azure Compute Gallery | 1 | `gal-avd-{env}-{num}` | `gal-avd-prod-01` |
| 21 | Image Definition (multi-session) | 1 | `win{ver}-{type}-{purpose}` | `win11-multisession-23h2` |
| 22 | Image Definition (single-session) | 1 | `win{ver}-{type}-{purpose}` | `win11-singlesession-23h2` |
| **Identity (Entra ID)** |
| 23 | User Group (pooled) | 1 | `AVD-{Type}-Users` | `AVD-Pooled-Users` (120 users) |
| 24 | User Group (personal) | 1 | `AVD-{Type}-Users` | `AVD-Personal-Users` (40 users) |
| 25 | Admin Group | 1 | `AVD-Admins` | `AVD-Admins` (5 admins) |
| 26 | Device Group (pooled) | 1 | `AVD-Devices-{Type}` | `AVD-Devices-Pooled` (10 VMs) |
| 27 | Device Group (personal) | 1 | `AVD-Devices-{Type}` | `AVD-Devices-Personal` (40 VMs) |
| 28 | Conditional Access Policy | 2-3 | `CA-AVD-{type}-{condition}` | `CA-AVD-MFA-Required` |
| **Monitoring** |
| 29 | Log Analytics Workspace | 1 | `law-avd-{env}-{num}` | `law-avd-prod-01` |
| 30 | Azure Monitor Workbook (Insights) | 1 | `workbook-avd-{topic}` | `workbook-avd-insights` |
| 31 | Azure Monitor Workbook (Connections) | 1 | `workbook-avd-{topic}` | `workbook-avd-connections` |
| **Security & Keys** |
| 32 | Key Vault | 1 | `kv-avd-{env}` | `kv-avd-prod` |
| | | | | |
| **TOTAL** | **~32 primary resources** |

---

## Complete Example Naming Reference

### All Resource Names in Deployment Order

**Save this table for Step 1 (Prerequisites) onwards - every step references these names.**

| Step | Resource Type | Name | Notes |
|------|---------------|------|-------|
| 0 | Environment | Production | Planning only |
| 1-2 | Resource Group | `rg-avd-prod-01` | Contains all resources |
| 3 | Subscription | `sub-aidrak-prod-avd` | Document in runbook |
| 4 | VNET | `vnet-avd-prod-01` | 10.0.0.0/16 |
| 4 | Subnet (hosts) | `snet-avd-prod-sessionhosts` | 10.0.1.0/24 |
| 4 | Subnet (endpoints) | `snet-avd-prod-privateendpoints` | 10.0.2.0/24 |
| 4 | NSG (hosts) | `nsg-avd-prod-sessionhosts` | Attached to session host subnet |
| 4 | NSG (endpoints) | `nsg-avd-prod-privateendpoints` | Attached to private endpoint subnet |
| 5 | Storage Account | `stavdprod01` | FSLogix storage |
| 5 | File Share (pooled) | `profiles-pooled` | 30GB, in storage account |
| 5 | File Share (personal) | `profiles-personal` | 50GB, in storage account |
| 5 | Private Endpoint | `pe-storage-prod` | Optional: secure storage access |
| 6 | Host Pool (pooled) | `hp-pooled-prod` | Pooled, breadth-first |
| 6-7 | Session Hosts (pooled) | `vm-pooled-prod-001` to `-010` | 10 VMs for ~120 users |
| 8 | Host Pool (personal) | `hp-personal-prod` | Personal, 1:1 assignment |
| 8-7 | Session Hosts (personal) | `vm-personal-prod-011` to `-050` | 40 VMs for ~40 users |
| 9 | App Group (pooled) | `ag-pooled-prod` | Desktop AppGroup |
| 9 | App Group (personal) | `ag-personal-prod` | Desktop AppGroup |
| 8 | Workspace | `ws-prod` | Contains both AppGroups |
| 9 | User Group (pooled) | `AVD-Pooled-Users` | Assigned to `ag-pooled-prod` |
| 9 | User Group (personal) | `AVD-Personal-Users` | Assigned to `ag-personal-prod` |
| 9 | Device Group (pooled) | `AVD-Devices-Pooled` | Intune assignment target |
| 9 | Device Group (personal) | `AVD-Devices-Personal` | Intune assignment target |
| 3 | Azure Compute Gallery | `gal-avd-prod-01` | Image storage |
| 3 | Image Def (multi-session) | `win11-multisession-23h2` | For pooled hosts |
| 3 | Image Def (single-session) | `win11-singlesession-23h2` | For personal hosts |
| 11 | Log Analytics | `law-avd-prod-01` | Monitoring |
| 11 | Key Vault | `kv-avd-prod` | Secure storage of secrets |
| 11 | Admin Group | `AVD-Admins` | RBAC on resource group |

---

## Validation Checklist

Before proceeding to Step 1, confirm:

- [ ] **Environment confirmed:** Production
- [ ] **Naming patterns understood:** Can predict name for any resource type
- [ ] **Example names saved:** Reference table above bookmarked/copied
- [ ] **Resource count confirmed:** ~32 primary resources for 200 users
- [ ] **Storage separation clear:** Pooled and personal profiles separate, but in same storage account
- [ ] **Network layout clear:** Session hosts and endpoints share VNET, separate subnets
- [ ] **AVD design confirmed:** 1 pooled + 1 personal host pool, single workspace
- [ ] **Entra ID groups planned:** 3 user groups (pooled, personal, admins) + 2 device groups
- [ ] **Image strategy clear:** 2 image definitions (multi-session, single-session)

---

## Common Naming Mistakes to Avoid

| Mistake | Why It's Bad | Correct Approach |
|---------|-------------|------------------|
| Using department names in resource names | Causes confusion if department reassigns users | Use generic: pooled, personal. Assign via groups. |
| Random VM numbers (001, 045, 999) | Hard to track growth, predict next VM | Use sequential: 001, 002, 003... |
| Different naming per resource type | Inconsistent, error-prone | Follow patterns: always `{purpose}-{env}-{num}` |
| Abbreviating beyond recognition | `stprd01` vs `stavdprodeus01` unclear | Be explicit within Azure limits |
| Including region in names | Adds unnecessary length and complexity | Skip region - it's implicit from deployment location |
| Descriptive names like "FinalBuild" | Means nothing later, not scalable | Use versioning: v1.0, v1.1, v2.0 |

---

## Next Steps

After confirming this naming convention:

1. Document resource names in your deployment runbook
2. Create Entra ID groups (Step 1 onwards will reference them)
3. Proceed to [[01-prerequisites-licensing|Step 1: Prerequisites & Licensing]]

---

## Variant: Personal Host Pool Deployments

The naming conventions above support BOTH pooled and personal deployments simultaneously. If deploying ONLY personal (no pooled):

**Adjust as follows:**
- Still create `hp-personal-prod` host pool
- Session hosts: `vm-personal-prod-001` onwards (not -011)
- App group: `ag-personal-prod`
- User group: `AVD-Personal-Users`
- File share: `profiles-personal` (or single `profiles-shared` if only personal)
- Skip pooled-specific resources entirely

> **Note:** Most Azure Cost optimization recommends pooled for shared departments (cost per user lower). Personal best for power-users requiring persistent state.

---

## Related Reference

For deep technical details on Azure resources:
- [[../../Reference/Networking/vnet-design|VNet Design Deep Dive]]
- [[../../Reference/Storage/fslogix-profile-containers|FSLogix Profile Containers]]
- [[../../Reference/Images/azure-compute-gallery|Azure Compute Gallery]]
- [[../../Reference/Identity/dynamic-groups|Dynamic Group Configuration]]
