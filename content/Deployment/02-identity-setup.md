---
title: Step 02 - Identity Setup
description: Configure Entra ID users, groups, and security policies for AVD
published: true
date: 2025-12-14T00:00:00.000Z
tags:
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 02: Identity Setup

Configure Entra ID users, groups, Conditional Access policies, and SSO for Azure Virtual Desktop.

## Prerequisites

- [ ] Entra ID P1 or P2 license (required for dynamic groups and Conditional Access)
- [ ] Global Administrator or User Administrator role
- [ ] Naming conventions reviewed from [[../Deployment/00-naming-conventions]]

---

## Create User and Device Groups

**Portal:** Entra Admin Center → Groups → New group

| Group Name             | Type     | Membership     | Rule / Details                                                            |
| ---------------------- | -------- | -------------- | ------------------------------------------------------------------------- |
| `avd-users-pooled`     | Security | Dynamic User   | `(user.department -eq "General") -or (user.department -eq "Finance")`     |
| `avd-users-personal`   | Security | Dynamic User   | `(user.department -eq "Creative") -or (user.department -eq "Executives")` |
| `avd-users-admin`      | Security | Assigned       | Manual - add your admin accounts                                          |
| `avd-devices-pooled`   | Security | Dynamic Device | `device.displayName -startsWith "avd-pool"`                               |
| `avd-devices-personal` | Security | Dynamic Device | `device.displayName -startsWith "avd-pers"`                               |
| `avd-devices-all`      | Security | Dynamic Device | `(device.displayName -startsWith "avd-")`                                 |

**Steps for each group:**
1. Click **+ New group**
2. Fill **Group type**, **Group name**, **Membership type**
3. For dynamic groups: Click **Add dynamic query**, paste rule, click **Validate rules** (optional)
4. Click **Create**

> **Note:** Dynamic groups process in 5-15 minutes. Device groups populate automatically when session hosts join Entra ID during deployment. See [[../Identity/dynamic-groups]] for advanced rules.

---

## Configure Conditional Access

**Portal:** Entra Admin Center → Protection → Conditional Access → Policies → + New policy

Create two mandatory policies:

| Policy | Users | Target Apps | Grant Control | Mode |
|--------|-------|-------------|-------------------|------|
| `AVD - Require MFA` | avd-users-pooled, avd-users-personal, avd-users-admin (exclude break-glass) | Azure Virtual Desktop, Microsoft Remote Desktop | Require MFA | Report-only (test 3-7 days first) |
| `AVD - Compliant Device` | avd-users-pooled (exclude break-glass) | Azure Virtual Desktop, Microsoft Remote Desktop | Require device compliance + MFA | Report-only |

**Portal Path for both:** Entra Admin Center → Protection → Conditional Access

**Key Settings:**
- **Assignments → Conditions → Client apps:** Browser, Mobile apps and desktop clients
- **Exclude break-glass admin account from all policies** (emergency recovery)

Optional: Create named location `Corporate Office Networks` and location-based blocking policy for additional security.

**Test policies with What If tool:** Entra Admin Center → Protection → Conditional Access → What If (select user, Azure Virtual Desktop app, verify correct policies apply).

See [[../Identity/conditional-access]] for advanced scenarios and policy best practices.

---

## Configure Resource Group IAM

Azure role-based access control (RBAC) determines who can manage AVD infrastructure and perform administrative tasks. Set up roles at the resource group level (RG-Azure-VDI-01) to control permissions for your AVD deployment.

**Portal:** Azure Portal → RG-Azure-VDI-01 → Access control (IAM)

### Role Assignments

| Role | Group/User | Scope | Purpose |
|------|-----------|-------|---------|
| **Owner** | Break-glass admin account | Resource Group | Emergency-only root access; use just-in-time (JIT) access |
| **Contributor** | avd-users-admin group | Resource Group | Manage AVD resources: host pools, session hosts, app groups |
| **Virtual Machine Administrator Login** | avd-users-admin group | Resource Group | RDP/SSH login to session hosts for troubleshooting |
| **Virtual Machine User Login** | avd-users-pooled, avd-users-personal | Resource Group | Standard user RDP access to session hosts (optional if using application assignment) |

**Steps to Assign Roles:**

1. Go to **Access control (IAM)** tab
2. Click **+ Add** → **Add role assignment**
3. Select **Role** (e.g., "Contributor")
4. Under **Assign access to**, select **User, group, or service principal**
5. Click **+ Select members**
6. Search for and select your group (e.g., "avd-users-admin")
7. Click **Review + assign**

> **Best Practice:** Use groups for role assignment, not individual users. This simplifies offboarding—remove user from group and all role assignments are revoked automatically. For storage permissions, see [[../Deployment/05-storage-fslogix#required-rbac-roles|Step 05: Storage FSLogix - RBAC Roles]].

---

## Enable Single Sign-On (SSO)

Single Sign-On configuration has multiple components that span this step and subsequent deployment steps. See [[../Identity/sso-integration]] for comprehensive SSO setup including:

- Enabling Microsoft Entra authentication for RDP (tenant-wide)
- Hiding consent prompts with trusted device groups
- Creating Kerberos server objects (if applicable)
- Reviewing Conditional Access policies
- Configuring host pool SSO in Step 04

**Key point:** Some SSO prerequisites must be completed in this step (tenant-wide enablement) before proceeding to Step 04 (Host Pool Creation).

---

## Verification Checklist

- [ ] All 6 groups created: 3 user groups (Pooled, Personal, Admins) + 3 device groups (Pooled, Personal, All)
- [ ] Dynamic groups show **Rule processing status: Succeeded** (wait 5-15 minutes if processing)
- [ ] Conditional Access policies created in Report-only mode (monitor 3-7 days before enabling)
- [ ] Break-glass admin excluded from all policies
- [ ] Policies tested with What If tool: Entra Admin Center → Protection → Conditional Access → What If (verify MFA policy applies to test user)
- [ ] Resource Group IAM roles assigned: Owner (break-glass), Contributor (avd-users-admin), VM Admin/User logins configured
- [ ] Role assignments verified in RG-Azure-VDI-01 → Access control (IAM) → View all role assignments

---

## Troubleshooting

### Dynamic Groups Empty After 30 Minutes

**Symptom:** Groups show 0 members, rule shows `Failed`

**Fix:**
1. Verify user attributes: Entra Admin Center → Users → Select test user → Department field populated with exact case match (e.g., "General")
2. Check rule syntax: Edit group → Dynamic membership rules → Validate Rules
3. Confirm Entra ID P1 license: Entra Admin Center → Billing → Licenses

### Conditional Access Policy Not Enforcing

**Symptom:** Users bypass MFA when connecting to AVD

**Fix:**
1. Verify policy is "On" not "Report-only" (switch after 3-7 days monitoring)
2. Check target apps include both: Azure Virtual Desktop + Microsoft Remote Desktop
3. Verify user in correct group: Entra Admin Center → Groups → avd-users-pooled or avd-users-personal
4. Test with What If tool to validate policies apply

See [[../Identity/dynamic-groups]] and [[../Identity/conditional-access]] for detailed troubleshooting.

---

## Next Steps

**Next:** [[03-networking-setup|Step 03: Networking Setup]]

## Related References

- [[../Identity/dynamic-groups]] - Advanced rules, troubleshooting
- [[../Identity/conditional-access]] - Policy design and best practices
- [[../Identity/sso-integration]] - SSO for cloud and hybrid environments
- [[../Identity/entra-id-fundamentals]] - Licensing and identity overview
