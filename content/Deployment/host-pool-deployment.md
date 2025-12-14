---
title: Host Pool Deployment
description: 
published: true
date: 2025-12-14T04:52:42.935Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:25.217Z
---

# Host Pool Deployment

Create AVD host pools to organize session hosts. Deploy both pooled (shared) and personal (dedicated) host pools based on user requirements.

## Create Pooled Host Pool

**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → Create

### Basics
- **Resource group:** RG-Azure-VDI-01
- **Host pool name:** `hp-pooled-prod1`
- **Location:** Central US
- **Validation environment:** No (Yes for testing updates first)
- **Preferred app group type:** Desktop
- **Host pool type:** Pooled

### Load Balancing
- **Load balancing algorithm:** Depth-first
- **Max session limit:** 10

> **Note:** Depth-first fills one host before moving to the next (cost-effective). Breadth-first spreads users across all hosts (better performance per user).

### Session Timeouts
- **Disconnect timeout:** 60 minutes
- **Idle timeout:** 30 minutes
- **Reconnection:** Users can reconnect

### Virtual Machines (Skip for now)
- We'll add VMs separately in the next step

### Workspace
- **Register desktop app group:** Yes
- **Workspace:** Create new → `ws-avd-prod`
- **Friendly name:** AVD Production Workspace

## Create Personal Host Pool

**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → Create

### Basics
- **Resource group:** RG-Azure-VDI-01
- **Host pool name:** `hp-personal-prod1`
- **Location:** Central US
- **Host pool type:** Personal

### Personal Desktop Settings
- **Assignment type:** Automatic (or Direct for manual assignment)

### Workspace
- **Register desktop app group:** Yes
- **Workspace:** Select existing → `ws-avd-prod`

## Configure Host Pool RDP Properties

**Portal:** Host Pool → RDP Properties

Recommended settings:

| Property | Value | Reason |
|----------|-------|--------|
| Multi-monitor | Yes | User experience |
| Smart card | Yes | If using smart cards |
| Clipboard | Bidirectional | User productivity |
| Drive redirection | Disabled | Security |
| Printer redirection | Yes | User experience |
| Audio | Play on client | Performance |
| Timezone | Client timezone | User experience |

## Assign Users to App Group

**Portal:** Host Pool → Application groups → Select DAG → Assignments

- Add `AVD-Users-Pooled` to `hp-pooled-prod1-DAG`
- Add `AVD-Users-Personal` to `hp-personal-prod1-DAG`

> **Note:** Users assigned to the Desktop Application Group (DAG) can see the desktop in their Remote Desktop client. See [[application-groups]] for RemoteApp configuration.

## Verification

- [ ] Pooled host pool created
- [ ] Personal host pool created
- [ ] Both registered to workspace
- [ ] RDP properties configured
- [ ] User groups assigned to app groups

---

**Next:** [[session-host-provisioning|Step 8: Session Host Provisioning]]