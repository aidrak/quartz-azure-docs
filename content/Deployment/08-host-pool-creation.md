---
title: Step 08 - Host Pool Creation
description: Create AVD host pools and generate registration tokens for session host VM deployment
published: true
date: 2025-12-14
tags: [Quick-Deploy, AVD, host-pool]
---

# Step 08: Host Pool Creation

Create host pools (containers for session hosts) and generate registration tokens. This step does NOT deploy VMs - VMs deploy in Step 09.

## Prerequisites

- [ ] [[05-storage-fslogix|Step 05]] completed
- [ ] [[06-application-deployment|Step 06]] (Application Deployment prerequisites) completed
- [ ] [[07-fslogix-configuration|Step 07]] (FSLogix Intune policies) completed
- [ ] Resource group: `rg-avd-prod-01` created
- [ ] VNET ready: `vnet-avd-prod-01` with `snet-avd-prod-sessionhosts`
- [ ] Contributor or Desktop Virtualization Contributor role

---

## Part 1: Create Pooled Host Pool

**Portal:** Azure Portal → Virtual Desktop → Host pools → + Create

| Setting | Value | Notes |
|---------|-------|-------|
| Resource group | `rg-avd-prod-01` | |
| Host pool name | `hp-pooled-prod` | Per [[00-naming-conventions]] |
| Location | East US | |
| Validation environment | No | |
| Host pool type | Pooled | Multi-session |
| Load balancing algorithm | Breadth-first | Distribute evenly across hosts |
| Max session limit | 20 | Per D4s_v5 VM (150 users ÷ 20 = 8 hosts min) |
| Start VM on Connect | Enabled | Auto-start deallocated hosts |
| Friendly name | `Pooled Production Desktop` | User-visible name |
| Description | `Multi-session pooled desktops` | |
| Tags | Environment: Production | |

Steps:
1. **Basics:** Fill in Project details + Host pool type settings above
2. **Virtual machines:** Select "No" (deploy VMs separately in Step 09)
3. **Workspace:** Select "No" (assign in Step 10)
4. **Advanced:** Set Friendly name, Description, Enable Start VM on Connect
5. **Tags:** Add tags above
6. Click **Review + create** → **Create**


---

## Part 2: Generate Registration Token

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-pooled-prod → Properties

1. Scroll to **Registration key** section
2. Click **Generate new key**
3. **Expiration:** 24 hours from now (limits security risk)
4. Click **Generate**
5. **Copy and save token securely** - needed in Step 09

> **Token expires in 24 hours.** Expiration does NOT affect already-joined session hosts - only new VMs joining the pool need valid tokens.

---

## Part 3: Create Personal Host Pool (Optional)

> **Decision Point:**
> - **Need personal desktops for power users?** Create host pool below
> - **Pooled only?** Skip to Troubleshooting

**Portal:** Azure Portal → Virtual Desktop → Host pools → + Create

| Setting | Value | Notes |
|---------|-------|-------|
| Resource group | `rg-avd-prod-01` | |
| Host pool name | `hp-personal-prod` | Per [[00-naming-conventions]] |
| Location | East US | |
| Host pool type | Personal | 1:1 VM to user |
| Assignment type | Automatic | First available VM to user |
| Start VM on Connect | Enabled | |
| Friendly name | `Personal Production Desktop` | |
| Tags | Environment: Production | |

Steps: Same as Part 1 (Basics → Virtual machines: No → Workspace: No → Advanced → Tags → Review + create)

**Generate registration token:** Same as Part 2 (Host pools → hp-personal-prod → Properties → Generate new key → Save)

---

## Troubleshooting

### Issue: Registration Token Expired

**Symptom:** Session hosts fail to join with "Registration token invalid" error

**Cause:** Token expiration date passed (24 hours)

**Fix:**
1. **Portal:** Host pools → hp-pooled-prod → Properties → Generate new key
2. Copy new token
3. Re-run Step 09 with new token

### Issue: "Start VM on Connect" Not Working

**Symptom:** Deallocated VMs don't start when users connect

**Cause:** Missing "Desktop Virtualization Power On Contributor" role on Azure Virtual Desktop service principal

**Fix:**
1. **Portal:** Resource groups → rg-avd-prod-01 → Access Control (IAM)
2. Search for "Azure Virtual Desktop" service principal
3. Add role: "Desktop Virtualization Power On Contributor"

**See:** [[../AVD/host-pools-deep-dive|Host Pools Deep Dive]] for load balancing algorithms, sizing, validation environments, and multi-region setup.

---

## Next Steps

**Host pools created and tokens ready.** Session hosts deploy in Step 09.

**Next:** [[09-session-hosts|Step 09: Session Hosts]]

---

## Related Reference Pages

- [[../AVD/host-pools-deep-dive|Host Pools Deep Dive]] - Load balancing, max sessions, variants
- [[../AVD/scaling-plans|Scaling Plans]] - Cost optimization with auto-scale
- [[../Operations/capacity-planning|Capacity Planning]] - VM sizing and user density
