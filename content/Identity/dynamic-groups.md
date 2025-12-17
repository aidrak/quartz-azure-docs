---
title: Dynamic Groups
description: 
published: true
date: 2025-12-14T04:52:52.099Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:46.257Z
---

# Dynamic Groups

Dynamic groups automate membership management using rule-based queries against user or device attributes. For AVD deployments, dynamic groups enable zero-touch assignment of session hosts to security groups, automatic user categorization, and scalable policy enforcement as your environment grows.

## What Are Dynamic Groups

**Dynamic groups** automatically add or remove members based on attribute-based rules evaluated continuously by Entra ID. When a user or device matches the rule, they are added to the group. When they no longer match, they are removed.

### Dynamic vs Assigned Groups

| Feature | Assigned (Static) Groups | Dynamic Groups |
|---------|-------------------------|----------------|
| **Membership** | Manually added/removed | Rule-based, automatic |
| **Scaling** | Manual effort per member | Scales to thousands with zero effort |
| **Accuracy** | Can become stale | Always current (within processing delay) |
| **Licensing** | Free tier | Requires Entra ID P1 or P2 |
| **Use case** | Small, stable groups | Large, changing groups with consistent attributes |

**Example Scenario:**
- **Static group:** `avd-users-finance-assigned` - You manually add each finance user. If someone changes departments, you must remember to remove them.
- **Dynamic group:** `avd-users-finance-dynamic` - Rule: `user.department -eq "Finance"`. When HR updates the department attribute, the user is automatically added or removed.

### Licensing Requirements

Dynamic groups require **Entra ID P1** or **P2** licenses. Specifically:
- Entra ID Free: No dynamic groups
- Entra ID P1: Dynamic groups for users and devices
- Entra ID P2: Same as P1 (no additional dynamic group features)

The license can be assigned to the tenant (all users) or to individual users who manage the groups (admins).

## Membership Rules Syntax

Dynamic group rules use a simple expression language based on object properties.

### Basic Syntax

```
(property operator value)
```

**Operators:**
- `-eq` : Equals
- `-ne` : Not equals
- `-startsWith` : Starts with
- `-notStartsWith` : Does not start with
- `-contains` : Contains
- `-notContains` : Does not contain
- `-match` : Regex match
- `-in` : In a collection
- `-notIn` : Not in a collection

**Logical Operators:**
- `-and` : Both conditions must be true
- `-or` : Either condition must be true
- `-not` : Negates a condition

### Example Rules

**User Rules:**

```
user.department -eq "Finance"
```
All users in the Finance department.

```
user.jobTitle -startsWith "Manager"
```
All users whose job title starts with "Manager".

```
(user.city -eq "Seattle") -and (user.department -eq "Sales")
```
Users in Seattle who work in Sales.

```
user.mail -match ".*@contoso\\.com$"
```
Users with email addresses ending in @contoso.com (regex).

**Device Rules:**

```
device.displayName -startsWith "vm-pooled-"
```
All devices whose name starts with "vm-pooled-".

```
device.deviceOSType -eq "Windows"
```
All Windows devices.

```
(device.displayName -startsWith "vm-pooled-") -and (device.deviceOSType -eq "Windows")
```
Windows devices whose names start with "vm-pooled-".

### Common User Properties

| Property | Example Value | Use Case |
|----------|---------------|----------|
| `user.userPrincipalName` | `jdoe@contoso.com` | Email-based rules |
| `user.department` | `Finance`, `HR`, `Sales` | Department-based access |
| `user.jobTitle` | `Manager`, `Engineer` | Role-based groups |
| `user.city` | `Seattle`, `New York` | Location-based policies |
| `user.country` | `US`, `UK`, `Canada` | Regional compliance |
| `user.extensionAttribute1-15` | Custom values | Custom categorization |
| `user.companyName` | `Contoso` | Multi-tenant scenarios |
| `user.usageLocation` | `US`, `GB` | License assignment regions |

### Common Device Properties

| Property | Example Value | Use Case |
|----------|---------------|----------|
| `device.displayName` | `vm-pooled-001` | Naming convention grouping |
| `device.deviceOSType` | `Windows`, `Linux` | OS-specific policies |
| `device.deviceOSVersion` | `10.0.19045.0` | OS version targeting |
| `device.deviceTrustType` | `AzureAd`, `ServerAd` | Join type (Entra vs Hybrid) |
| `device.isCompliant` | `true`, `false` | Intune compliance state |
| `device.extensionAttribute1-15` | Custom values | Custom device categorization |
| `device.deviceId` | GUID | Unique device identifier |

## AVD-Specific Dynamic Group Patterns

Our environment uses dynamic groups to automate session host management and user assignment.

### Device Groups for Session Hosts

These groups exist in our RG-Azure-VDI-01 deployment:

#### 1. avd-devices-all

**Purpose:** Contains all AVD session hosts, regardless of host pool.

**Rule:**
```
(device.displayName -startsWith "vm-pooled-") -or (device.displayName -startsWith "vm-personal-")
```

**Use Case:**
- Intune policies that apply to all AVD VMs (security baselines, monitoring agents)
- Conditional Access policies requiring compliant AVD devices
- Reporting and inventory queries

#### 2. avd-devices-pooled

**Purpose:** Only session hosts in pooled host pools (multi-session).

**Rule:**
```
device.displayName -startsWith "vm-pooled-"
```

**Use Case:**
- Pooled-specific configurations (FSLogix profile settings for multi-session)
- Capacity monitoring (alert when pooled hosts exceed threshold)
- Conditional Access requiring shared device compliance

#### 3. avd-devices-personal

**Purpose:** Only session hosts in personal host pools (dedicated VMs).

**Rule:**
```
device.displayName -startsWith "vm-personal-"
```

**Use Case:**
- Personal desktop policies (allow local admin, persistent storage)
- Backup policies (personal desktops may need VM-level backups)
- License tracking (personal desktops consume VDA licenses differently)

#### 4. avd-sessionhosts-sso

**Purpose:** Devices configured for Entra ID-based SSO.

**Rule (using extension attribute):**
```
device.extensionAttribute1 -eq "SSO-Enabled"
```

**Use Case:**
- SSO-specific configurations (Kerberos Cloud Trust settings)
- Troubleshooting SSO issues (isolate SSO-enabled devices)
- Gradual SSO rollout (add devices to SSO group in waves)

**How extensionAttribute1 is set:**
During VM deployment, a script sets the custom attribute via Microsoft Graph:
```powershell
Connect-MgGraph -Scopes "Device.ReadWrite.All"
$device = Get-MgDevice -Filter "displayName eq 'vm-pooled-001'"
Update-MgDevice -DeviceId $device.Id -AdditionalProperties @{extensionAttribute1="SSO-Enabled"}
```

### User Groups by Department or Attribute

#### avd-users-admin

**Purpose:** IT administrators who manage AVD infrastructure.

**Rule:**
```
(user.department -eq "IT") -and (user.jobTitle -contains "Admin")
```

**Use Case:**
- Assign "Desktop Virtualization Contributor" RBAC role
- Conditional Access allowing admin access from specific locations
- RemoteApp assignments for management tools

#### avd-users-pooled

**Purpose:** Users assigned to pooled (multi-session) host pools.

**Rule:**
```
user.extensionAttribute2 -eq "Pooled"
```

**Use Case:**
- Assign "Desktop Virtualization User" role on pooled host pool
- Conditional Access enforcing MFA for pooled sessions
- RemoteApp assignments (Office apps published to pooled users)

**How extensionAttribute2 is set:**
HR system or onboarding script sets the attribute during user provisioning:
```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All"
Update-MgUser -UserId "jdoe@contoso.com" -ExtensionAttribute2 "Pooled"
```

#### avd-users-personal

**Purpose:** Users assigned to personal (dedicated) host pools.

**Rule:**
```
user.extensionAttribute2 -eq "Personal"
```

**Use Case:**
- Assign "Desktop Virtualization User" role on personal host pool
- Allow persistent local data (no profile roaming required)
- Higher resource quotas (personal desktops for power users)

### Naming Convention Grouping

Our AVD session hosts follow this naming convention:
- **Pooled:** `vm-pooled-{nnn}` → `vm-pooled-001`, `vm-pooled-002`, etc.
- **Personal:** `vm-personal-{nnn}` → `vm-personal-011`, `vm-personal-012`, etc.

Dynamic groups use the `-startsWith` operator to categorize devices:

```
(device.displayName -startsWith "vm-pooled-")
```
Captures all pooled session hosts.

```
(device.displayName -startsWith "vm-personal-")
```
Captures all personal session hosts.

This pattern scales infinitely: when a new session host joins Entra ID with the correct naming convention, it automatically appears in the appropriate groups within 24 hours (usually faster).

## Processing Time Considerations

Dynamic group membership is **not instant**. Entra ID processes rules in the background.

### Expected Processing Time

- **Small groups (< 1,000 members):** 5-15 minutes
- **Medium groups (1,000-10,000 members):** 30 minutes to 2 hours
- **Large groups (> 10,000 members):** Up to 24 hours
- **Very large groups (> 50,000 members):** Can take multiple days

### Factors Affecting Processing Speed

1. **Rule complexity:** Simple rules (`device.displayName -startsWith "avd-"`) process faster than complex multi-condition rules with regex.
2. **Tenant size:** Tenants with millions of objects have longer queue times.
3. **Concurrent changes:** If many groups are being updated simultaneously, processing slows.
4. **Attribute indexing:** Commonly used properties (displayName, department) are indexed and process faster than custom extension attributes.

### Monitoring Processing Status

**Portal:**
1. Go to **Entra ID > Groups > [Group Name]**
2. Expand **Dynamic membership rules**
3. Check **Rule processing status**

**Status Values:**
- `Succeeded` - Rule evaluated successfully, membership current
- `Processing` - Rule currently being evaluated
- `Failed` - Syntax error or invalid property
- `Paused` - Processing paused (usually due to tenant issues)

**PowerShell:**
```powershell
Connect-MgGraph -Scopes "Group.Read.All"
$group = Get-MgGroup -Filter "displayName eq 'avd-devices-all'"
$group.MembershipRuleProcessingState
```

### Best Practices for Processing Time

- **Don't rely on instant membership** - If you deploy a new session host and immediately assign it to a host pool based on group membership, expect delays. Use direct assignment for time-sensitive operations.
- **Schedule group creation during low-usage windows** - Creating large dynamic groups during business hours can slow down other Entra operations.
- **Test rules before deploying to production** - Use the "Validate rules" feature in the portal to test syntax before saving. A failed rule can delay processing for 24+ hours.
- **Monitor after rule changes** - If you edit a rule, monitor the processing status for at least an hour to ensure it completes successfully.

> **Note:** Dynamic group processing is a background job in Entra ID. Microsoft does not provide SLA guarantees on processing time, though 99% of small groups process within 15 minutes in practice.

## Best Practices

- **Use extension attributes for custom categorization** - `extensionAttribute1-15` are designed for this purpose. Don't overload `department` or `jobTitle` with values like "avd-pooled" when extension attributes exist.
- **Prefer simple rules when possible** - `device.displayName -startsWith "vm-pooled-"` is faster and more maintainable than complex regex patterns.
- **Document the business logic** - In the group description field, explain WHY the rule exists ("Auto-assigns pooled host pools for Intune compliance policies").
- **Avoid circular dependencies** - Don't create rules like "user is member of Group A" when Group A is also dynamic. Use attributes directly.
- **Test rules with "Validate rules" feature** - Entra portal has a built-in validator showing which objects would match your rule before you save it.
- **Set clear naming conventions early** - Consistent device and user naming makes dynamic group rules trivial. Inconsistent naming makes them impossible.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Dynamic group is empty | Rule syntax error or no objects match | Use "Validate rules" in portal, check attribute values on sample objects |
| Members not appearing after 24 hours | Processing stuck or paused | Check rule processing status, contact Microsoft support if paused |
| Duplicate memberships (static + dynamic) | Object added manually to dynamic group | Remove manual assignment; dynamic groups ignore static adds |
| Rule validation says "Invalid property" | Property doesn't exist on object type | Verify property name (user.department vs device.department) |
| Extension attribute rule not matching | Attribute not set or wrong format | Use Graph Explorer to verify attribute value: `GET /devices/{id}` |
| Group membership lags 24+ hours | Large tenant or complex rule | Simplify rule, or use multiple smaller groups with simpler rules |

> **Warning:** Changing a dynamic group's rule **removes all existing members** and re-evaluates from scratch. This can cause temporary service disruption if Conditional Access or RBAC depends on the group. Test rule changes in a cloned group first.

> **Warning:** Dynamic groups cannot have manually added members. If you add a user/device manually via "Add member", Entra ID will remove them on the next rule evaluation. Use assigned groups for manual membership.