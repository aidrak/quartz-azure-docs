---
title: Identity Configuration
description: 
published: true
date: 2025-12-14T04:52:44.475Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:26.796Z
---

# Identity Configuration

Configure Entra ID groups for AVD user and device management. These groups are used for RBAC, app group assignments, Conditional Access, and Intune policies.

## User Groups (Assigned Membership)

Create groups for AVD user access:

**Portal:** Entra Admin Center → Groups → New group

### AVD-Users-Admins

- **Group type:** Security
- **Group name:** AVD-Users-Admins
- **Membership type:** Assigned
- **Members:** IT administrators who need full AVD access

### AVD-Users-Pooled

- **Group type:** Security
- **Group name:** AVD-Users-Pooled
- **Membership type:** Assigned
- **Members:** Users who will access pooled desktops

### AVD-Users-Personal

- **Group type:** Security
- **Group name:** AVD-Users-Personal
- **Membership type:** Assigned
- **Members:** Users who will have dedicated desktops

## Device Groups (Dynamic Membership)

Create dynamic groups for session host management:

**Portal:** Entra Admin Center → Groups → New group

### AVD-Devices-All

- **Group type:** Security
- **Membership type:** Dynamic Device
- **Dynamic query:**
```
(device.displayName -startsWith "avd-")
```

### AVD-Devices-Pooled

- **Group type:** Security
- **Membership type:** Dynamic Device
- **Dynamic query:**
```
(device.displayName -startsWith "avd-pool")
```

### AVD-Devices-Personal

- **Group type:** Security
- **Membership type:** Dynamic Device
- **Dynamic query:**
```
(device.displayName -startsWith "avd-personal")
```

### AVD-SessionHosts-SSO

- **Group type:** Security
- **Membership type:** Dynamic Device
- **Dynamic query:**
```
(device.displayName -startsWith "avd-") and (device.trustType -eq "AzureAD")
```

> **Warning:** Dynamic group membership can take up to 24 hours to process initially. Plan accordingly.

## Verify Dynamic Group Rules

**Portal:** Group → Dynamic membership rules → Validate Rules

Test with a sample device name to confirm the rule works before deployment.

## RBAC Role Assignments

Assign AVD-specific roles:

**Portal:** Azure Portal → Resource Group → Access control (IAM)

| Role | Group | Scope |
|------|-------|-------|
| Desktop Virtualization User | AVD-Users-Pooled | hp-pooled-prod1-DAG |
| Desktop Virtualization User | AVD-Users-Personal | hp-personal-prod1-DAG |
| Desktop Virtualization Contributor | AVD-Users-Admins | RG-Azure-VDI-01 |

> **Note:** Desktop Virtualization User role grants access to the desktop. Assign at the application group level. See [[RBAC for AVD]] for details.

## Verification

- [ ] User groups created with members
- [ ] Device groups created with dynamic rules
- [ ] Dynamic rules validated
- [ ] RBAC roles assigned

---

**Next:** [[storage-for-profiles|Step 5: Storage for Profiles]]