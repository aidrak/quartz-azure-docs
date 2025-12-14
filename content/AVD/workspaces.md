---
title: Workspaces
description: 
published: true
date: 2025-12-14T04:52:26.219Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:16.029Z
---

# Workspaces

A workspace is the user-facing container that organizes and presents application groups in the Remote Desktop client. It's the top-level resource that users connect to when accessing Azure Virtual Desktop. While host pools and application groups define the technical infrastructure, workspaces define the end-user experience.

## What is a Workspace

A workspace is a logical grouping of one or more application groups that appear together in the user's Remote Desktop client. When a user subscribes to a workspace, they see all the desktops and applications from the registered application groups in a single, unified interface.

**From the User's Perspective:**

1. User opens Remote Desktop client (Windows, macOS, web, iOS, Android)
2. User subscribes to a workspace URL or authenticates with Entra ID
3. Workspace displays all available resources (desktops and apps) from assigned application groups
4. User clicks a resource icon to launch it

**From the Administrator's Perspective:**

A workspace is a configuration object that:
- **Registers Application Groups:** Links one or more app groups to make them visible
- **Sets Friendly Name:** Defines what users see as the workspace name
- **Provides Access URL:** Users subscribe via https://rdweb.wvd.microsoft.com or client auto-discovery
- **Simplifies User Experience:** One workspace can contain multiple host pools and app groups

**Example from Our Environment:**

- **Workspace:** ws-avd-prod
- **Friendly Name:** "AVD Production Workspace"
- **Registered Application Groups:**
  - hp-pooled-prod1-DAG (Desktop from pooled host pool)
  - hp-personal-prod1-DAG (Desktop from personal host pool)
- **User Experience:** Users see "AVD Production Workspace" with two desktop icons:
  - "Pooled Production Desktop" (shared resources, DepthFirst load balancing)
  - "Developer Workstation" (personal dedicated VM)

When a user opens the Remote Desktop client, they authenticate once and see both desktops in the workspace. Clicking either icon launches the respective session.

## End-User Perspective

Understanding what users see and how they interact with workspaces is critical for support and training.

### Remote Desktop Client Experience

**Step 1: Subscribe to Workspace**

- **Automatic Discovery (Recommended):** User opens Remote Desktop client, enters email (user@contoso.com), client auto-discovers workspace via Entra ID
- **Manual Subscription:** User adds workspace by URL: https://rdweb.wvd.microsoft.com/api/arm/feeddiscovery

**Step 2: Authenticate**

User enters Entra ID credentials (username/password, MFA if required). Conditional Access policies apply here.

**Step 3: View Resources**

Workspace displays with friendly name at the top. Below are resource icons:
- **Desktop Icons:** Full desktop resources (from Desktop app groups)
- **App Icons:** Individual RemoteApp applications (from RemoteApp app groups)

Each icon shows:
- **Friendly Name:** Set in application group or published app configuration
- **Resource Type:** Desktop or RemoteApp
- **Icon:** Extracted from .exe or custom icon path

**Step 4: Launch Resource**

User clicks icon, AVD connection broker:
1. Checks user assignment to application group (RBAC)
2. Selects session host from host pool (based on load balancing)
3. Initiates connection via AVD Gateway
4. User sees desktop or app window

### Web Client Experience

Users can also access AVD via browser at https://client.wvd.microsoft.com

**Pros:**
- No client installation required
- Works on ChromeOS, Linux, tablets
- Same workspace and resource view as native client

**Cons:**
- Requires Edge, Chrome, or Safari (WebRTC support)
- Performance slightly lower than native client
- Limited peripheral support (USB redirection, etc.)

**Our Environment:**
We recommend the native Windows/macOS client for daily use, but provide the web client URL for BYOD users and contractors on non-managed devices.

## Multiple Application Groups in One Workspace

A workspace can host multiple application groups from different host pools, enabling flexible resource delivery.

### Single Workspace, Multiple Host Pools

**Example Configuration:**

Workspace: ws-avd-prod
- **App Group 1:** hp-pooled-prod1-DAG (Desktop, Pooled, DepthFirst)
  - Assigned to: VDI-Pooled-Users (50 office workers)
- **App Group 2:** hp-personal-prod1-DAG (Desktop, Personal, Automatic assignment)
  - Assigned to: VDI-Developers (10 developers)

**User Experience:**

- **Office Worker (member of VDI-Pooled-Users):** Sees ONE desktop icon ("Pooled Production Desktop")
- **Developer (member of VDI-Developers):** Sees ONE desktop icon ("Developer Workstation")
- **IT Admin (member of both groups):** Sees TWO desktop icons (both resources in one workspace)

### Mixed Desktop + RemoteApp

**Example Configuration:**

Workspace: ws-avd-contractors
- **App Group 1:** hp-pooled-prod1-DAG (Desktop)
  - Assigned to: VDI-FullAccess
- **App Group 2:** hp-pooled-prod1-RAG (RemoteApp: QuickBooks, Timesheet)
  - Assigned to: VDI-Contractors

**User Experience:**

- **Full Access User:** Sees full desktop icon + QuickBooks + Timesheet icons (can use either)
- **Contractor:** Sees only QuickBooks + Timesheet icons (no full desktop access)

This approach enables least-privilege access: contractors only see the apps they need, while full-time employees get the full desktop.

### Multi-Region Scenario

**Example Configuration:**

Workspace: ws-avd-global
- **App Group 1:** hp-eastus-pooled-DAG (Desktop, East US host pool)
  - Assigned to: US-Users
- **App Group 2:** hp-westeurope-pooled-DAG (Desktop, West Europe host pool)
  - Assigned to: EU-Users

**User Experience:**

- **US User:** Sees "East US Desktop" icon (connects to nearby East US session hosts)
- **EU User:** Sees "West Europe Desktop" icon (connects to nearby West Europe session hosts)

Both user groups see a single workspace, but the resources point to geographically optimized host pools.

## Friendly Names for User Experience

The friendly name is the first thing users see when they connect. It should be clear, descriptive, and match organizational branding.

**Default vs Friendly Name:**

| Resource | Default Name | Friendly Name | What User Sees |
|----------|--------------|---------------|----------------|
| **Workspace** | ws-avd-prod | "AVD Production Workspace" | Workspace title in client |
| **App Group** | hp-pooled-prod1-DAG | "Pooled Production Desktop" | Desktop icon label |
| **RemoteApp** | EXCEL.EXE | "Microsoft Excel" | App icon label |

**Best Practices:**

- **Workspace Friendly Name:** Use organization name or purpose (e.g., "Contoso Virtual Desktops", "Engineering Workspace")
- **App Group Friendly Name:** Describe the resource (e.g., "General Desktop", "Developer Workstation", "Accounting Apps")
- **RemoteApp Friendly Name:** Use familiar app names (e.g., "Microsoft Word", "QuickBooks Desktop"), not technical .exe names

**Setting Friendly Name:**

```bash
# Workspace friendly name
az desktopvirtualization workspace update \
  --name ws-avd-prod \
  --resource-group RG-Azure-VDI-01 \
  --friendly-name "AVD Production Workspace"

# Application group friendly name
az desktopvirtualization applicationgroup update \
  --name hp-pooled-prod1-DAG \
  --resource-group RG-Azure-VDI-01 \
  --friendly-name "Pooled Production Desktop"

# RemoteApp friendly name
az desktopvirtualization application update \
  --name Excel \
  --resource-group RG-Azure-VDI-01 \
  --application-group-name hp-pooled-prod1-RAG \
  --friendly-name "Microsoft Excel"
```

**Our Environment:**

- **Workspace:** ws-avd-prod → "AVD Production Workspace"
- **App Group 1:** hp-pooled-prod1-DAG → "Pooled Production Desktop"
- **App Group 2:** hp-personal-prod1-DAG → "Developer Workstation"

Users see a clean, professional interface with recognizable names.

## Configuration Options

| Setting | Options | Impact |
|---------|---------|--------|
| **Workspace Name** | Technical identifier (e.g., ws-avd-prod) | Used in Azure Portal and CLI, not visible to users |
| **Friendly Name** | Display name (e.g., "AVD Production Workspace") | What users see in Remote Desktop client |
| **Application Groups** | One or more app groups to register | Determines which resources appear in workspace |
| **Location** | Azure region | Metadata location (does not affect session host location) |
| **Public Network Access** | Enabled, Disabled | Controls if workspace is accessible from internet or private networks only |

## How to Configure

### Portal: Create Workspace

**Path:** Azure Portal → Virtual Desktops → Workspaces → Create

1. **Basics:**
   - Subscription: (your subscription)
   - Resource Group: RG-Azure-VDI-01
   - Workspace Name: ws-avd-prod
   - Friendly Name: AVD Production Workspace
   - Location: East US

2. **Application Groups:**
   - Register application groups: Yes
   - Select application groups:
     - hp-pooled-prod1-DAG
     - hp-personal-prod1-DAG

3. **Advanced:**
   - Public network access: Enabled

4. **Review + Create**

### CLI: Create Workspace

```bash
# Create workspace
az desktopvirtualization workspace create \
  --name ws-avd-prod \
  --resource-group RG-Azure-VDI-01 \
  --location eastus \
  --friendly-name "AVD Production Workspace" \
  --public-network-access Enabled

# Register application groups
az desktopvirtualization workspace update \
  --name ws-avd-prod \
  --resource-group RG-Azure-VDI-01 \
  --application-group-references \
    "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/applicationGroups/hp-pooled-prod1-DAG" \
    "/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/applicationGroups/hp-personal-prod1-DAG"
```

### Add Application Group to Existing Workspace

```bash
# Get current app groups
CURRENT_GROUPS=$(az desktopvirtualization workspace show \
  --name ws-avd-prod \
  --resource-group RG-Azure-VDI-01 \
  --query applicationGroupReferences -o tsv | tr '\n' ' ')

# Add new app group
NEW_GROUP="/subscriptions/{sub-id}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.DesktopVirtualization/applicationGroups/hp-pooled-prod1-RAG"

az desktopvirtualization workspace update \
  --name ws-avd-prod \
  --resource-group RG-Azure-VDI-01 \
  --application-group-references $CURRENT_GROUPS $NEW_GROUP
```

## Best Practices

- **Single Production Workspace** - For most organizations, one workspace per environment (prod, test, dev) is sufficient; multiple workspaces add complexity without benefit for end users
- **Descriptive Friendly Names** - Use names that match your organization's branding and are immediately recognizable to users (e.g., "Contoso Virtual Desktops" instead of "ws-avd-prod")
- **Group Related Resources** - Put all production app groups in the production workspace, all validation/test resources in a separate test workspace
- **Public vs Private Access** - If using Azure Private Link or ExpressRoute, set public network access to Disabled to enforce private connectivity
- **Workspace Naming Convention** - Use pattern: ws-{purpose}-{environment} (e.g., ws-avd-prod, ws-avd-test, ws-contractors-prod)
- **User Communication** - Provide users with clear instructions: "Subscribe to AVD Production Workspace using your email address in the Remote Desktop client"

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Workspace empty (no resources shown)** | No application groups registered to workspace | Azure Portal → Workspace → Application Groups → Add → Select app groups |
| **User sees workspace but gets "No resources available"** | User not assigned to any app groups in the workspace | Assign user or group to app groups via RBAC (Desktop Virtualization User role) |
| **Cannot subscribe to workspace in client** | Conditional Access policy blocking, or client not updated | Check Entra ID Conditional Access policies, ensure client is latest version, try web client |
| **Workspace shows old/deleted resources** | Client cache not refreshed | Remote Desktop client → Settings → Unsubscribe → Re-subscribe, or clear client cache |
| **Multiple workspaces confuse users** | Too many workspaces created | Consolidate into single workspace per environment, use app group assignment to control visibility |
| **Friendly name not updating in client** | Client cached old metadata | Unsubscribe and re-subscribe to workspace, or restart client and wait 5 minutes for cache refresh |

## Workspace Metadata and Discovery

**How Auto-Discovery Works:**

1. User enters email (user@contoso.com) in Remote Desktop client
2. Client queries https://rdweb.wvd.microsoft.com/api/arm/feeddiscovery with Entra ID token
3. AVD control plane returns list of workspaces user has access to (based on app group RBAC)
4. Client subscribes to workspaces and fetches resource list

**Manual Subscription URL:**

If auto-discovery fails, users can manually subscribe:
- **Feed URL:** https://rdweb.wvd.microsoft.com/api/arm/feeddiscovery
- **Classic AVD (deprecated):** https://rdweb.wvd.microsoft.com/api/feeddiscovery (do not use)

**Workspace Metadata Location:**

The workspace object is stored in a specific Azure region (e.g., East US), but this does NOT affect where session hosts run. Session hosts are in the host pool's region, which can be different.

Example:
- **Workspace Location:** East US (metadata storage)
- **Host Pool 1 Location:** East US (session hosts physically in East US)
- **Host Pool 2 Location:** West Europe (session hosts physically in West Europe)

Users in Europe connecting to Host Pool 2 will have low latency because session hosts are nearby, even though workspace metadata is in East US.

## Multi-Workspace Scenarios

While a single workspace is recommended for most organizations, there are valid reasons to create multiple workspaces.

### Scenario 1: Production vs Validation

**Workspace 1:** ws-avd-prod ("AVD Production")
- hp-pooled-prod1-DAG
- hp-personal-prod1-DAG

**Workspace 2:** ws-avd-validation ("AVD Validation - Early Access")
- hp-pooled-validation-DAG
- hp-personal-validation-DAG

**Reasoning:** Separate IT/pilot users testing new images/updates from production users. Users subscribe to both workspaces if they're in the pilot group.

### Scenario 2: Regional Isolation

**Workspace 1:** ws-avd-us ("Contoso Virtual Desktops - US")
- hp-eastus-pooled-DAG
- hp-westus-pooled-DAG

**Workspace 2:** ws-avd-eu ("Contoso Virtual Desktops - EU")
- hp-westeurope-pooled-DAG
- hp-northeurope-pooled-DAG

**Reasoning:** Data residency requirements prevent EU users from accessing US resources. Separate workspaces ensure strict regional boundaries.

### Scenario 3: Customer Isolation (MSP)

**Workspace 1:** ws-avd-customera ("Customer A Virtual Desktop")
**Workspace 2:** ws-avd-customerb ("Customer B Virtual Desktop")

**Reasoning:** Managed service provider hosting multiple customers in one Azure subscription. Separate workspaces prevent cross-customer resource visibility.

---

**Next:** Proceed to "Scaling Plans" to learn how to automatically start/stop session hosts based on demand and save costs.