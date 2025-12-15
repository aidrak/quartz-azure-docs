---
title: Step 10 - Application Groups & Workspace
description: Create application groups and workspace for user access
published: true
date: 2025-12-14T00:00:00.000Z
tags:
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 10: Application Groups & Workspace

Create desktop application groups for pooled and personal host pools, assign users via RBAC, and configure workspace. This makes AVD resources accessible in the Remote Desktop client.

## Prerequisites

- [ ] Host pools created: `hp-pooled-prod`, `hp-personal-prod` (Step 08)
- [ ] Session hosts deployed (Step 09)
- [ ] Entra ID groups created: `avd-users-pooled`, `avd-users-personal`
- [ ] Desktop Virtualization Contributor role on resource group

---

## Part 1: Create Application Groups

**Portal:** Azure Portal → Virtual Desktop → Application groups → + Create

| Field | Pooled Value | Personal Value |
|-------|---|---|
| **Resource group** | `rg-avd-prod-01` | `rg-avd-prod-01` |
| **Host pool** | `hp-pooled-prod` | `hp-personal-prod` |
| **Type** | Desktop | Desktop |
| **Name** | `ag-pooled-prod` | `ag-personal-prod` |
| **Friendly name** | Pooled Production Desktop | Personal Workstation |

Steps:
1. Click **+ Create**
2. Fill in Basics tab (values from table above)
3. Skip Applications & Assignments tabs
4. **Workspace:** Select "No" (will register after workspace creation)
5. **Advanced:** Set friendly name from table
6. Click **Review + create** → **Create** (~30 seconds)

**Portal:** Azure Portal → Virtual Desktop → Application groups → [resource] → Properties
- Verify **Friendly name** matches table above

---

## Part 2: Assign Users to Application Groups

**Portal:** Azure Portal → Virtual Desktop → Application groups → [resource] → Access Control (IAM)

| Application Group | Entra ID Group | Role |
|---|---|---|
| `ag-pooled-prod` | `avd-users-pooled` | Desktop Virtualization User |
| `ag-personal-prod` | `avd-users-personal` | Desktop Virtualization User |

Steps for each app group:
1. Click **+ Add** → **Add role assignment**
2. **Role:** Search "Desktop Virtualization User"
3. **Members:** Click **+ Select members**, search group name from table
4. Click **Review + assign**

**Reference:** [[../AVD/application-groups|Application Groups]] for user assignment strategies and RBAC role details.

---

## Part 3: Create Workspace

**Portal:** Azure Portal → Virtual Desktop → Workspaces → + Create

| Field | Value |
|---|---|
| **Resource group** | `rg-avd-prod-01` |
| **Name** | `ws-prod` |
| **Friendly name** | AVD Production Workspace |
| **Location** | East US |
| **Register app groups** | Yes - select both `ag-pooled-prod` and `ag-personal-prod` |
| **Public network access** | Enabled |

Steps:
1. Click **+ Create**
2. Fill Basics tab with values from table
3. **Application groups:** Click **+ Register** and select both app groups
4. **Advanced:** Keep defaults, add optional description
5. Click **Review + create** → **Create** (~1 minute)

**Verify:** Portal → Workspaces → ws-prod → Application groups
- Both app groups appear as registered

---

## Part 4: Test Access

**Prerequisites:**
- Test user in `avd-users-pooled` or `avd-users-personal` group
- Remote Desktop client installed or web browser

**Quick Test (Windows client):**
1. Download Remote Desktop client: https://aka.ms/rdwindows
2. Open client, click **Subscribe**, enter test user email
3. Authenticate with Entra ID
4. Should see workspace and available desktop icons
5. Click desktop to launch session

**Web Test:**
- Navigate to https://client.wvd.microsoft.com
- Authenticate and verify workspace appears

**Reference:** [[../AVD/workspaces|Workspaces]] for detailed user access flow and troubleshooting.

---

## Critical Issues

**No resources available in workspace**
- Verify user in correct Entra ID group
- Check RBAC role assigned: Portal → App group → Access Control (IAM)
- Confirm app groups registered to workspace

**Access Denied on connection**
- Verify group membership includes user
- Check session hosts are available (not deallocated)
- Review FSLogix from Step 09

**Reference:** [[../AVD/workspaces|Workspaces]] for complete troubleshooting guide.

---

## Next Steps

Application groups and workspace are configured. Users can now access AVD desktops.

**Next:** Step 09 - Session Host Configuration

---

## Related References

- [[../AVD/application-groups|Application Groups]] - App group types, user assignment
- [[../AVD/workspaces|Workspaces]] - User access, discovery, troubleshooting
- [[../AVD/host-pools|Host Pools]] - Pool types, assignment, load balancing
- [[../Identity/rbac-for-avd|RBAC for AVD]] - Role details, assignment methods
