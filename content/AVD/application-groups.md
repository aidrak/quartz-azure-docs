---
title: Application Groups
description: 
published: true
date: 2025-12-14T04:52:18.189Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:04.300Z
---

# Application Groups

Application groups define what users can access within Azure Virtual Desktop. They act as the bridge between host pools (the compute resources) and workspaces (the user-facing interface). Understanding application group types, assignment methods, and configuration options is critical for delivering the right experience to different user personas.

## What are Application Groups

An application group is a logical collection of applications or desktops published from a host pool. When you assign users or groups to an application group, they gain access to the resources it contains. Application groups are then registered to workspaces, which users see in their Remote Desktop client.

**Key Concepts:**

- **One-to-Many Relationship:** A single host pool can have multiple application groups (e.g., one for Desktop, one for RemoteApp)
- **User Assignment:** Users or Entra ID security groups are assigned to application groups via RBAC
- **Workspace Registration:** Application groups must be registered to a workspace to be visible to users
- **Multiple Access:** A user can be assigned to multiple application groups across different host pools

**Example from Our Environment:**

- **Host Pool:** hp-pooled-prod1
- **Application Group:** hp-pooled-prod1-DAG (Desktop type)
- **Workspace:** ws-avd-prod
- **Assigned Users:** VDI-Pooled-Users (Entra ID security group with 50 members)

When members of VDI-Pooled-Users open the Remote Desktop client and connect to ws-avd-prod, they see "AVD Production Workspace" with a full Windows desktop icon.

## Desktop vs RemoteApp

AVD supports two types of application groups, each serving different use cases.

### Desktop Application Groups

**What it Provides:**
A full Windows desktop experience delivered to the user's device. Users interact with a complete Windows session, including Start menu, taskbar, File Explorer, and all installed applications.

**Characteristics:**
- **Automatically Created:** When you create a host pool, a default desktop application group is created (named {HostPoolName}-DAG)
- **One Per Host Pool:** A host pool can only have ONE desktop application group
- **Full OS Access:** Users see the entire Windows environment (subject to GPO/policy restrictions)
- **Use Case:** General-purpose computing, users who need access to multiple applications, traditional desktop replacement

**Example Scenario:**
Office workers who need Windows desktop, Office 365, web browsers, and occasional access to LOB apps. Instead of publishing individual apps, you give them a full desktop.

**Our Environment:**
- **App Group:** hp-pooled-prod1-DAG
- **Type:** Desktop
- **Host Pool:** hp-pooled-prod1 (Pooled, DepthFirst, 10 max sessions)
- **Users:** VDI-Pooled-Users (50 office workers)
- **Experience:** Users log in and see a complete Windows 10/11 desktop with Office, Edge, Teams, and custom LOB apps pre-installed

### RemoteApp Application Groups

**What it Provides:**
Individual applications published as if they were running locally on the user's device. Apps appear in their own windows, seamlessly integrated with the local taskbar and Alt+Tab switcher.

**Characteristics:**
- **Multiple Per Host Pool:** You can create multiple RemoteApp application groups on a single host pool
- **Selective Publishing:** Choose which .exe files or Start menu apps to publish
- **Mixed Desktop + RemoteApp:** Users can access RemoteApps from one host pool and desktops from another simultaneously
- **Use Case:** Publishing specific LOB apps to users who don't need a full desktop, BYOD scenarios, mixed app delivery

**Example Scenario:**
Contractors who only need access to QuickBooks and a custom CRM app. Instead of giving them a full desktop, you publish just those two applications. They run in seamless windows on the user's laptop alongside their local apps.

**Configuration Example:**

```bash
# Create RemoteApp application group
az desktopvirtualization applicationgroup create \
  --name hp-pooled-prod1-RAG \
  --resource-group RG-Azure-VDI-01 \
  --location eastus \
  --application-group-type RemoteApp \
  --host-pool-arm-path "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/hostPools/hp-pooled-prod1"

# Publish QuickBooks from Start menu
az desktopvirtualization application create \
  --name QuickBooks \
  --resource-group RG-Azure-VDI-01 \
  --application-group-name hp-pooled-prod1-RAG \
  --command-line-setting DoNotAllow \
  --file-path "C:\Program Files\QuickBooks\QuickBooks.exe" \
  --friendly-name "QuickBooks Desktop" \
  --icon-index 0 \
  --icon-path "C:\Program Files\QuickBooks\QuickBooks.exe"
```

**Our Environment:**
We currently do NOT use RemoteApp groups in production. All users receive full desktops (hp-pooled-prod1-DAG or hp-personal-prod1-DAG). However, we plan to create a RemoteApp group for seasonal contractors who only need access to time-tracking software.

## Comparison: Desktop vs RemoteApp

| Factor | Desktop | RemoteApp |
|--------|---------|-----------|
| **User Experience** | Full Windows desktop | Individual apps in seamless windows |
| **Application Access** | All installed apps available | Only explicitly published apps |
| **Start Menu** | Full Start menu visible | No Start menu (apps launch directly) |
| **Taskbar** | AVD session taskbar | Apps appear in local device taskbar |
| **Use Case** | General-purpose workers | Specific app access, BYOD, contractors |
| **Configuration Effort** | Low (auto-created) | Medium (must publish each app) |
| **Security** | Broader access (entire OS) | Least privilege (only published apps) |
| **App Groups per Host Pool** | Max 1 | Unlimited |

**Decision Guide:**
- **Desktop:** Users need Windows desktop, multiple apps, or prefer traditional workspace
- **RemoteApp:** Users need 1-5 specific apps, BYOD scenario, or enhanced security (least privilege)
- **Both:** Power users get desktop, contractors/partners get RemoteApp access to specific LOB apps

## User Assignment

Users must be assigned to an application group to access its resources. AVD uses Azure RBAC (Role-Based Access Control) for assignments.

### Direct User Assignment

Assign individual Entra ID user accounts to the application group.

**When to Use:**
- Small environments (<20 users)
- Testing or proof-of-concept
- Temporary access for specific individuals

**Portal:**
Azure Portal → Virtual Desktops → Application Groups → hp-pooled-prod1-DAG → Assignments → Add → Select Users → Choose individuals

**CLI:**

```bash
az role assignment create \
  --assignee user@contoso.com \
  --role "Desktop Virtualization User" \
  --scope "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/applicationGroups/hp-pooled-prod1-DAG"
```

### Group-Based Assignment (Recommended)

Assign Entra ID security groups to the application group.

**When to Use:**
- Production environments (>20 users)
- Delegated management (HR or managers add users to groups)
- Consistent access patterns (all accountants get the same apps)

**Best Practice:**
Create dedicated Entra ID security groups for each application group:
- VDI-Pooled-Users → hp-pooled-prod1-DAG
- VDI-Developers → hp-personal-prod1-DAG
- VDI-Contractors → hp-pooled-prod1-RAG (RemoteApp)

**Portal:**
Azure Portal → Virtual Desktops → Application Groups → hp-pooled-prod1-DAG → Assignments → Add → Select Groups → Choose VDI-Pooled-Users

**CLI:**

```bash
# Get group object ID
GROUP_ID=$(az ad group show --group VDI-Pooled-Users --query id -o tsv)

# Assign group to app group
az role assignment create \
  --assignee $GROUP_ID \
  --role "Desktop Virtualization User" \
  --scope "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/applicationGroups/hp-pooled-prod1-DAG"
```

**Our Environment:**
- **VDI-Pooled-Users** (50 members) → hp-pooled-prod1-DAG (Desktop)
- **VDI-Developers** (10 members) → hp-personal-prod1-DAG (Desktop)

When a new employee joins the accounting team, HR adds them to VDI-Pooled-Users, and they automatically get access to the pooled desktop. No manual Azure portal changes needed.

## Multiple Application Group Access

A single user can be assigned to multiple application groups, even across different host pools. This enables flexible app delivery.

**Example Scenario:**

User: john@contoso.com
- **Assignment 1:** VDI-Pooled-Users → hp-pooled-prod1-DAG (Desktop for general work)
- **Assignment 2:** Developers → hp-personal-prod1-DAG (Personal desktop for development)

When John opens the Remote Desktop client, he sees TWO resources in the workspace:
1. "Pooled Production Desktop" (from hp-pooled-prod1)
2. "Developer Workstation" (from hp-personal-prod1)

He can launch both simultaneously and switch between them.

**Use Cases:**
- Power users who need both a shared desktop for daily work and a personal desktop for specialized tasks
- Mixed delivery: Desktop from one host pool, RemoteApps from another
- Testing: IT staff assigned to production AND validation host pools

## Configuration Options

| Setting | Options | Impact |
|---------|---------|--------|
| **Application Group Type** | Desktop, RemoteApp | Determines if full desktop or individual apps are published |
| **Host Pool** | (Select existing host pool) | Source of compute resources for this app group |
| **Friendly Name** | Custom display name | What users see in Remote Desktop client |
| **Workspace** | (Select workspace to register) | Makes app group visible to users |
| **User Assignments** | Users or Entra ID groups | Who can access the resources |

## How to Configure

### Portal: Create Application Group

**Path:** Azure Portal → Virtual Desktops → Application Groups → Create

1. **Basics:**
   - Subscription: (your subscription)
   - Resource Group: RG-Azure-VDI-01
   - Name: hp-pooled-prod1-DAG
   - Location: East US
   - Application Group Type: Desktop
   - Host Pool: hp-pooled-prod1

2. **Applications:**
   - (Desktop type has no app selection, RemoteApp would show app publishing options)

3. **Assignments:**
   - Add users or groups: VDI-Pooled-Users
   - Role: Desktop Virtualization User

4. **Workspace:**
   - Register application group: Yes
   - Register to workspace: ws-avd-prod

5. **Review + Create**

### CLI: Create Desktop Application Group

```bash
# Create desktop app group
az desktopvirtualization applicationgroup create \
  --name hp-pooled-prod1-DAG \
  --resource-group RG-Azure-VDI-01 \
  --location eastus \
  --application-group-type Desktop \
  --friendly-name "Pooled Production Desktop" \
  --host-pool-arm-path "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/hostPools/hp-pooled-prod1"

# Register to workspace
az desktopvirtualization workspace update \
  --name ws-avd-prod \
  --resource-group RG-Azure-VDI-01 \
  --application-group-references "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/applicationGroups/hp-pooled-prod1-DAG"
```

### CLI: Publish RemoteApp

```bash
# Publish app from Start menu
az desktopvirtualization application create \
  --name Excel \
  --resource-group RG-Azure-VDI-01 \
  --application-group-name hp-pooled-prod1-RAG \
  --command-line-setting DoNotAllow \
  --file-path "C:\Program Files\Microsoft Office\root\Office16\EXCEL.EXE" \
  --friendly-name "Microsoft Excel" \
  --icon-index 0 \
  --icon-path "C:\Program Files\Microsoft Office\root\Office16\EXCEL.EXE" \
  --show-in-portal true
```

## Best Practices

- **Group-Based Assignment** - Always use Entra ID security groups for user assignment, never assign individual users directly; this simplifies management and enables self-service via group membership
- **Friendly Names** - Set descriptive friendly names for application groups and published apps; users see these names in the Remote Desktop client, not technical resource names
- **RemoteApp for Least Privilege** - For users who only need 1-3 specific apps, use RemoteApp instead of Desktop to reduce attack surface and improve security posture
- **Separate App Groups by Persona** - Create different application groups for different user types (e.g., Accounting-Apps, HR-Apps, Engineering-Apps) even if they share the same host pool
- **Default Desktop App Group** - Don't delete the auto-created desktop app group unless you're exclusively using RemoteApp; it's needed for full desktop access
- **Test Before Production** - Create a validation app group assigned to IT staff to test published apps before rolling out to all users

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Users don't see any resources in Remote Desktop client** | User not assigned to any application group | Assign user or Entra ID group to app group via RBAC, verify role is "Desktop Virtualization User" |
| **Application group not visible in workspace** | App group not registered to workspace | Azure Portal → Workspace → Application Groups → Add → Select app group |
| **RemoteApp launches but immediately closes** | Incorrect file path or missing .exe on session hosts | Verify app is installed on all session hosts, check file path in app group settings |
| **Users get "Access Denied" when launching app** | User assigned to app group but lacks permission to run the app | Check NTFS permissions, AppLocker, or GPO restrictions on session host |
| **Cannot create second desktop app group** | Only one desktop app group allowed per host pool | This is by design; use RemoteApp for additional app groups or create another host pool |
| **Published app icon missing or blank** | Icon path incorrect or icon-index wrong | Specify full path to .exe or .ico file, set icon-index to 0 unless app has multiple icons |

---

**Next:** Proceed to "Workspaces" to understand how users discover and connect to published resources.