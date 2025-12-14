---
title: RBAC for AVD
description: Azure Role-Based Access Control for Azure Virtual Desktop management
published: true
date: 2025-12-14T04:52:58.536Z
tags: 
editor: markdown
dateCreated: 2025-12-14T04:44:04.778Z
---

# RBAC for AVD

## What It Is
Azure Role-Based Access Control (RBAC) determines who can manage Azure Virtual Desktop resources and what actions they can perform. RBAC operates at the Azure resource management layer (control plane), not the user connection layer (data plane).

RBAC uses role assignments that combine three elements:
- **Security principal** - Who (user, group, service principal)
- **Role definition** - What actions (read, write, delete)
- **Scope** - Where (subscription, resource group, individual resource)

For AVD, this means separating administrative access (who can create/modify host pools) from user access (who can connect to AVD sessions).

## AVD Built-in Roles

| Role | Scope | Permissions | Use Case |
|------|-------|-------------|----------|
| **Desktop Virtualization Contributor** | Full AVD management | Create/modify/delete host pools, app groups, workspaces, session hosts | AVD administrators who need complete control |
| **Desktop Virtualization Reader** | Read-only access | View AVD resources, read configuration, no modifications | Auditors, support staff viewing configurations |
| **Desktop Virtualization User** | User access (data plane) | Connect to published applications and desktops in assigned app groups | End users who need to connect to AVD |
| **Desktop Virtualization Host Pool Contributor** | Host pool management | Manage session hosts, scaling plans, user sessions | Operations team managing capacity |
| **Desktop Virtualization Workspace Contributor** | Workspace management | Manage workspaces, publish app groups | Team managing user workspace experience |
| **Desktop Virtualization Application Group Contributor** | Application group management | Manage app groups, published apps/desktops | Application team managing published resources |
| **Desktop Virtualization Session Host Operator** | Session host operations | Drain mode, send messages, log off users | Help desk performing session management |

> **Note:** The "Desktop Virtualization User" role is special - it's the only role that grants data plane access (ability to connect). All others are control plane (Azure management).

## How to Assign Roles

### Assign Role at Resource Group Level
**Portal:** Azure Portal → Resource Groups → [RG-AVD-Prod] → Access Control (IAM) → Add role assignment

1. Click **+ Add** → **Add role assignment**
2. **Role tab:**
   - Search for role (e.g., "Desktop Virtualization Host Pool Contributor")
   - Select role from list
   - Click **Next**
3. **Members tab:**
   - Assign access to: **User, group, or service principal**
   - Click **+ Select members**
   - Search for and select Entra ID group (e.g., "SG-AVD-Admins")
   - Click **Next**
4. **Conditions tab** (optional):
   - Leave default unless implementing ABAC
   - Click **Next**
5. **Review + assign:**
   - Review settings
   - Click **Review + assign**

### Assign AVD User Access to App Group
**Portal:** Azure Portal → Azure Virtual Desktop → Application groups → [AppGroup] → Access Control (IAM)

1. Click **+ Add** → **Add role assignment**
2. Select **Desktop Virtualization User**
3. Click **Next**
4. **Members tab:**
   - Assign access to: **User, group, or service principal**
   - Select Entra ID group containing AVD users (e.g., "SG-AVD-Finance-Users")
   - Click **Next**
5. Click **Review + assign**

> **Important:** Users must be assigned "Desktop Virtualization User" role at the **application group** level, not the host pool or workspace. This is what grants connection access.

### Common RBAC Scope Patterns

**Subscription level:**
```
Subscription (AVD-Production)
└── Desktop Virtualization Contributor → SG-AVD-GlobalAdmins
```
Use for: Platform team managing all AVD resources

**Resource group level:**
```
Resource Group (RG-AVD-Finance)
├── Desktop Virtualization Contributor → SG-AVD-Finance-Admins
└── Desktop Virtualization Reader → SG-AVD-Finance-Support
```
Use for: Department-specific admin teams

**Application group level:**
```
Application Group (Finance-Desktop-AppGroup)
└── Desktop Virtualization User → SG-Finance-AVD-Users
```
Use for: End user connection access (always at this scope)

## Best Practices

- **Use least privilege principle** - Grant the minimum permissions needed; use Host Pool Contributor instead of full Contributor if admins only manage capacity

- **Assign to groups, not individuals** - Always assign RBAC roles to Entra ID groups, never directly to user accounts (except emergency break-glass scenarios)

- **Use Desktop Virtualization User for end users** - This is the ONLY role that should be assigned to end users; all other roles are for administrators

- **Separate admin roles by function** - Create different groups for different admin responsibilities:
  - `SG-AVD-Admins` → Desktop Virtualization Contributor (full control)
  - `SG-AVD-HelpDesk` → Desktop Virtualization Session Host Operator (user session management)
  - `SG-AVD-Auditors` → Desktop Virtualization Reader (view-only access)

- **Apply roles at appropriate scope** - Use resource group scope for most admin roles, application group scope for Desktop Virtualization User role

- **Document role assignments** - Maintain documentation of which groups have which roles at which scopes

- **Use dynamic groups for Desktop Virtualization User** - Automatically assign user access based on attributes (department, job title, etc.)

- **Review assignments regularly** - Audit RBAC assignments quarterly to remove stale assignments

- **Don't use classic subscription administrator roles** - Use Azure RBAC roles exclusively, not legacy Co-Administrator roles

- **Combine with Conditional Access** - Layer RBAC (who can manage) with Conditional Access (under what conditions they can connect)

## Common Role Assignment Scenarios

### Scenario: New AVD Deployment Team
**Requirement:** Three admins need full control of AVD in RG-AVD-Pilot

**Solution:**
1. Create Entra ID group: `SG-AVD-Pilot-Admins`
2. Add three admin users to group
3. Assign Desktop Virtualization Contributor role to group at RG-AVD-Pilot scope
4. Assign Contributor role at RG-AVD-Pilot scope (for underlying VMs, storage, networking)

### Scenario: Department-Specific Access
**Requirement:** Finance users need access to Finance-Desktop app group, HR users need access to HR-Desktop app group

**Solution:**
1. Create groups: `SG-Finance-AVD-Users`, `SG-HR-AVD-Users`
2. Assign Desktop Virtualization User to `SG-Finance-AVD-Users` at Finance-Desktop-AppGroup scope
3. Assign Desktop Virtualization User to `SG-HR-AVD-Users` at HR-Desktop-AppGroup scope

### Scenario: Help Desk Session Management
**Requirement:** Help desk needs to drain session hosts and log off users, but not modify host pool configuration

**Solution:**
1. Create group: `SG-AVD-HelpDesk`
2. Assign Desktop Virtualization Session Host Operator role at host pool scope
3. Do NOT assign Contributor or Host Pool Contributor roles

### Scenario: Auditor Read-Only Access
**Requirement:** Compliance team needs to view all AVD configurations but make no changes

**Solution:**
1. Create group: `SG-AVD-Auditors`
2. Assign Desktop Virtualization Reader role at subscription or resource group scope
3. Assign Reader role (standard Azure role) for underlying resources

## Common Issues

### Issue: Admin can't modify host pool
**Symptom:** User receives "Permission denied" or "Forbidden" errors when trying to modify host pool settings in Azure Portal

**Cause:**
- Missing Desktop Virtualization Host Pool Contributor or Desktop Virtualization Contributor role
- Role assigned at wrong scope (e.g., individual resource instead of resource group)
- Role assignment propagation delay

**Fix:**
1. Verify role assignment: Azure Portal → Resource Group → Access Control (IAM) → Check access
2. Search for user's UPN, review assigned roles
3. If missing, assign Desktop Virtualization Host Pool Contributor at resource group scope
4. Wait 5-10 minutes for role propagation
5. Have user sign out of Azure Portal and sign back in

### Issue: User can't connect to AVD session
**Symptom:** User sees "You don't have access" or application group appears empty in Remote Desktop client

**Cause:**
- Missing Desktop Virtualization User role on application group
- Role assigned at wrong scope (workspace or host pool instead of app group)
- User not member of assigned Entra ID group

**Fix:**
1. Navigate to: Azure Portal → Azure Virtual Desktop → Application groups → [AppGroup] → Access Control (IAM)
2. Check access → Search for user → Verify Desktop Virtualization User role present
3. If missing: Add role assignment → Desktop Virtualization User → Select user's group
4. Verify user is member of group: Entra Admin Center → Groups → [Group] → Members
5. User should refresh Remote Desktop client (sign out/in) after 5-10 minutes

### Issue: Service principal can't manage AVD resources via automation
**Symptom:** PowerShell/Azure CLI scripts fail with authorization errors

**Cause:**
- Service principal missing appropriate role assignment
- Service principal using wrong subscription context
- Managed identity not configured for Azure Automation runbook

**Fix:**
1. Identify service principal: Entra Admin Center → Applications → App registrations → [App] → Object ID
2. Assign role: Azure Portal → Resource Group → Access Control (IAM) → Add role assignment
3. Select Desktop Virtualization Contributor role
4. Assign access to: User, group, or service principal
5. Search for service principal by name or app ID
6. In script, ensure correct subscription context: `Set-AzContext -Subscription "AVD-Production"`

### Issue: User has role but can't see resources in Portal
**Symptom:** User assigned Desktop Virtualization Contributor but Azure Portal shows "No resources found"

**Cause:**
- User filtering by wrong subscription or resource group
- Role assignment delay (can take up to 30 minutes)
- Azure Portal cache issue

**Fix:**
1. Verify user is viewing correct subscription: Azure Portal → Directory + Subscription filter
2. Check all subscriptions checkbox if resources span multiple subscriptions
3. Wait 30 minutes after role assignment
4. Clear browser cache or try incognito mode
5. Verify assignment: Resource Group → Access Control (IAM) → Role assignments

### Issue: Over-privileged access (security concern)
**Symptom:** Users have more access than needed, security audit findings

**Cause:**
- Assigning broad roles (Contributor) instead of specific AVD roles
- Assigning roles at subscription level when resource group scope sufficient
- Direct user assignments instead of groups

**Fix:**
1. Review assignments: Azure Portal → Subscriptions → Access Control (IAM) → Role assignments → Download CSV
2. Identify over-privileged assignments (Contributor instead of Desktop Virtualization Contributor)
3. Create appropriate Entra ID groups for each function
4. Reassign using least-privilege AVD roles at narrowest scope
5. Remove broad assignments after validation
6. Document standard role assignment matrix

## Related Resources
- [[entra-id-fundamentals]] - Understanding Azure identity foundation
- [[dynamic-groups]] - Automating group membership for RBAC
- [[conditional-access]] - Layering access policies with RBAC
