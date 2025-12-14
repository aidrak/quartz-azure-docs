---
title: Step 08 - Application Groups & Workspace
description: Create desktop application groups, assign users, and configure workspace for end-user access
published: true
date: 2025-12-14T00:00:00.000Z
tags:
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 08: Application Groups & Workspace

Create desktop application groups for both pooled and personal host pools, assign Entra ID user groups via RBAC, and configure a unified workspace. This step makes AVD resources visible and accessible to end users through the Remote Desktop client.

## Example Scenario

Using naming conventions from [[00-naming-conventions]]:

| Resource | Name | Type | Assigned Users | Host Pool |
|----------|------|------|----------------|-----------|
| Application Group (Pooled) | `ag-pooled-prod` | Desktop | AVD-Pooled-Users (160) | hp-pooled-prod |
| Application Group (Personal) | `ag-personal-prod` | Desktop | AVD-Personal-Users (40) | hp-personal-prod |
| Workspace | `ws-prod` | Workspace | Both app groups | N/A |

**User Experience:**
- **Pooled Users (General + Finance):** See one desktop icon ("Pooled Production Desktop") in workspace
- **Personal Users (Executives + Creative):** See one desktop icon ("Personal Workstation") in workspace
- **IT Admins (both groups):** See BOTH desktop icons in workspace

## Prerequisites

- [ ] Host pools created: `hp-pooled-prod` and `hp-personal-prod` (from Step 07)
- [ ] Session hosts deployed and joined to host pools (from Step 07)
- [ ] Entra ID groups created: `AVD-Pooled-Users`, `AVD-Personal-Users` (from [[02-identity-setup]])
- [ ] Desktop Virtualization Contributor or Owner role on resource group

> **Note:** Application groups link users to host pool resources. Without app groups and workspace, users cannot discover or connect to AVD.

---

## Part 1: Create Desktop Application Group (Pooled)

Create desktop application group for the pooled multi-session host pool.

### Understanding Desktop vs RemoteApp

**Portal:** Azure Portal → Virtual Desktop → Application groups

**Application Group Types:**

| Type | User Experience | Use Case | App Groups per Host Pool |
|------|----------------|----------|-------------------------|
| **Desktop** | Full Windows desktop with Start menu, taskbar, all installed apps | General-purpose users, traditional desktop replacement | Max 1 per host pool |
| **RemoteApp** | Individual apps in seamless windows on user's local desktop | Specific app access, BYOD, contractors | Unlimited |

> **Decision:** Use **Desktop** type for both pooled and personal host pools. Users need full Windows experience with Office, web browsers, and LOB apps.

**See:** [[../AVD/application-groups|Application Groups]] for Desktop vs RemoteApp comparison.

### Create ag-pooled-prod Application Group

**Portal:** Azure Portal → Virtual Desktop → Application groups → + Create

1. **Basics:**
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Host pool:** `hp-pooled-prod`
   - **Application group type:** Desktop
   - **Application group name:** `ag-pooled-prod`
   - **Location:** East US (inherited from host pool)

2. **Applications:**
   - (Desktop application groups have no app selection step - all installed apps are available)

3. **Assignments:**
   - **Skip for now** (will assign users via RBAC in Part 3)

4. **Workspace:**
   - **Register application group:** No (will create workspace first, then register)

5. **Advanced:**
   - **Friendly name:** `Pooled Production Desktop`
   - **Description:** `Multi-session desktop for General and Finance users`

6. Click **Review + create**
7. Click **Create**

**Deployment time:** ~30 seconds

### Configure Friendly Name

Friendly name is what users see in Remote Desktop client. Set descriptive, user-friendly names.

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-pooled-prod → Properties

1. **Friendly name:** `Pooled Production Desktop`
2. Click **Save**

> **Best Practice:** Use friendly names that describe the resource purpose, not technical infrastructure names. Users see "Pooled Production Desktop", not "ag-pooled-prod".

**See:** [[../AVD/workspaces#friendly-names-for-user-experience|Workspace Friendly Names]] for naming best practices.

---

## Part 2: Create Desktop Application Group (Personal)

Create desktop application group for the personal single-session host pool.

### Create ag-personal-prod Application Group

**Portal:** Azure Portal → Virtual Desktop → Application groups → + Create

1. **Basics:**
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Host pool:** `hp-personal-prod`
   - **Application group type:** Desktop
   - **Application group name:** `ag-personal-prod`
   - **Location:** East US (inherited from host pool)

2. **Assignments:**
   - **Skip for now** (will assign users via RBAC in Part 3)

3. **Workspace:**
   - **Register application group:** No (will create workspace first)

4. **Advanced:**
   - **Friendly name:** `Personal Workstation`
   - **Description:** `Dedicated single-session desktop for Executives and Creative users`

5. Click **Review + create**
6. Click **Create**

**Deployment time:** ~30 seconds

### Configure Friendly Name

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-personal-prod → Properties

1. **Friendly name:** `Personal Workstation`
2. Click **Save**

---

## Part 3: Assign Users to Application Groups

Assign Entra ID security groups to application groups using Azure RBAC. Users must have "Desktop Virtualization User" role to access resources.

### Why Group-Based Assignment (Not Individual Users)

**Recommended Approach:** Assign Entra ID security groups to application groups

**Benefits:**
- **Scalability:** Add/remove users by managing group membership (delegated to HR/managers)
- **Self-Service:** HR adds new employee to AVD-Pooled-Users, user automatically gets AVD access
- **Auditing:** Group membership changes tracked in Entra ID audit logs
- **Consistency:** All users in same role get identical access

**Avoid:** Direct user assignment (only suitable for testing or <10 users)

**See:** [[../AVD/application-groups#user-assignment|Application Group User Assignment]] for detailed comparison.

### Assign AVD-Pooled-Users to ag-pooled-prod

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-pooled-prod → Access Control (IAM)

1. Click **+ Add** → **Add role assignment**
2. **Role:**
   - Search for: `Desktop Virtualization User`
   - Select the role
   - Click **Next**
3. **Members:**
   - **Assign access to:** User, group, or service principal
   - Click **+ Select members**
   - Search for: `AVD-Pooled-Users`
   - Select the group
   - Click **Select**
   - Click **Next**
4. **Conditions:** (none required)
5. Click **Review + assign**
6. Click **Review + assign** (confirmation)

**Verification:**

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-pooled-prod → Access Control (IAM) → Role assignments

- [ ] `AVD-Pooled-Users` shows as "Desktop Virtualization User"

### Assign AVD-Personal-Users to ag-personal-prod

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-personal-prod → Access Control (IAM)

1. Click **+ Add** → **Add role assignment**
2. **Role:** `Desktop Virtualization User`
3. **Members:** Select `AVD-Personal-Users` group
4. Click **Review + assign**

**Verification:**

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-personal-prod → Access Control (IAM) → Role assignments

- [ ] `AVD-Personal-Users` shows as "Desktop Virtualization User"

### RBAC Role Explanation

| Role | Permissions | Use Case |
|------|------------|----------|
| **Desktop Virtualization User** | Connect to published desktops/apps in application group | Standard end users |
| **Desktop Virtualization Power User** | Connect AND manage personal sessions (disconnect others, etc.) | Advanced users, rarely used |
| **Desktop Virtualization Contributor** | Manage AVD resources (host pools, app groups, etc.) | IT administrators |

> **Note:** We assigned "Desktop Virtualization User" - this grants connection rights only, no administrative permissions.

**See:** [[../AVD/application-groups#user-assignment|User Assignment]] for role details and PowerShell automation.

---

## Part 4: Create Workspace

Create a unified workspace that aggregates both application groups. Users subscribe to ONE workspace and see all available resources.

### Understanding Workspace Purpose

**What is a Workspace:**
- User-facing container that organizes application groups in Remote Desktop client
- Users connect to workspace URL (https://rdweb.wvd.microsoft.com) or use email-based auto-discovery
- Workspace displays all resources from registered application groups

**User Experience Flow:**
1. User opens Remote Desktop client
2. User enters email (user@contoso.com) - auto-discovery finds workspace
3. User authenticates with Entra ID (MFA if required)
4. Workspace displays available desktops/apps (based on app group assignments)
5. User clicks resource icon to launch session

**See:** [[../AVD/workspaces|Workspaces]] for detailed workspace architecture and multi-workspace scenarios.

### Create ws-prod Workspace

**Portal:** Azure Portal → Virtual Desktop → Workspaces → + Create

1. **Basics:**
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Workspace name:** `ws-prod`
   - **Friendly name:** `AVD Production Workspace`
   - **Location:** East US

2. **Application groups:**
   - **Register application groups:** Yes
   - Click **+ Register application groups**
   - Select:
     - `ag-pooled-prod`
     - `ag-personal-prod`
   - Click **Select**

3. **Advanced:**
   - **Public network access:** Enabled (default)
   - **Description:** `Production workspace for all AVD users`

4. **Tags:**
   - **Environment:** Production
   - **Purpose:** AVD-Workspace
   - **Owner:** IT-Operations

5. Click **Review + create**
6. Click **Create**

**Deployment time:** ~1 minute

### Verify Workspace Registration

**Portal:** Azure Portal → Virtual Desktop → Workspaces → ws-prod → Application groups

**Expected:**
- [ ] `ag-pooled-prod` (Pooled Production Desktop)
- [ ] `ag-personal-prod` (Personal Workstation)

Both application groups should show as registered.

### Configure Workspace Properties

**Portal:** Azure Portal → Virtual Desktop → Workspaces → ws-prod → Properties

Verify settings:
- **Friendly name:** `AVD Production Workspace` (what users see in client)
- **Description:** Brief description of workspace purpose
- **Public network access:** Enabled (unless using Private Link)

> **Note:** Friendly name appears in Remote Desktop client. Use organization-specific branding (e.g., "Contoso Virtual Desktops").

---

## Part 5: Test User Access (Pooled Desktop)

Verify end-to-end connectivity for pooled desktop users.

### Prerequisites for Testing

- [ ] Test user account in Entra ID (e.g., `testuser@contoso.com`)
- [ ] Test user is member of `AVD-Pooled-Users` group
- [ ] Test workstation with Remote Desktop client installed (Windows, macOS, web)

### Test with Remote Desktop Client (Windows)

**Client Download:** https://aka.ms/rdwindows

1. **Install Remote Desktop Client:**
   - Download and install latest Windows Desktop client
   - Launch Remote Desktop app

2. **Subscribe to Workspace:**
   - Click **Subscribe**
   - **Email or Workspace URL:** Enter `testuser@contoso.com`
   - Client auto-discovers workspace via Entra ID

3. **Authenticate:**
   - Enter Entra ID password for `testuser@contoso.com`
   - Complete MFA challenge (if Conditional Access enabled)

4. **View Workspace:**
   - Workspace name displays: **"AVD Production Workspace"**
   - Resource icons appear:
     - **"Pooled Production Desktop"** (available to test user)
     - **"Personal Workstation"** (NOT visible - test user not in AVD-Personal-Users)

5. **Launch Desktop:**
   - Double-click **"Pooled Production Desktop"** icon
   - Connection establishes to session host in `hp-pooled-prod`
   - User logs in with Entra ID credentials (SSO if already authenticated)
   - Full Windows desktop appears

6. **Verify Session:**
   - Check Start menu, taskbar, installed apps visible
   - Open Command Prompt: Run `hostname` to verify session host name (e.g., `vm-pooled-prod-001`)
   - Open File Explorer: Navigate to `%USERPROFILE%` to verify FSLogix profile loaded

**Expected Result:** User successfully connects to pooled desktop and sees Windows 11 multi-session environment.

### Test with Web Client

**Web Client URL:** https://client.wvd.microsoft.com

1. **Navigate to Web Client:**
   - Open browser (Edge, Chrome, or Safari)
   - Go to: https://client.wvd.microsoft.com

2. **Authenticate:**
   - Enter `testuser@contoso.com`
   - Complete Entra ID authentication + MFA

3. **View Workspace:**
   - Workspace displays: **"AVD Production Workspace"**
   - Desktop icon: **"Pooled Production Desktop"**

4. **Launch Desktop:**
   - Click **"Pooled Production Desktop"**
   - Session launches in browser window
   - Full desktop experience rendered via WebRTC

**Expected Result:** Web client provides same desktop experience as native client (slightly lower performance due to browser overhead).

> **Note:** Web client is excellent for BYOD users, ChromeOS, Linux, or unmanaged devices.

### Troubleshooting Connection Issues

| Issue | Symptom | Cause | Solution |
|-------|---------|-------|----------|
| **Workspace not found** | Auto-discovery fails | User email domain not matching Entra ID tenant | Manually enter feed URL: https://rdweb.wvd.microsoft.com/api/arm/feeddiscovery |
| **No resources available** | Workspace empty | User not assigned to any app groups | Verify user in AVD-Pooled-Users group, check RBAC role assignment |
| **Access Denied** | Connection refused | User not in app group or RBAC role missing | Verify "Desktop Virtualization User" role assigned to group |
| **Session host unavailable** | "No resources available" | All session hosts deallocated or unhealthy | Start session hosts, check host pool status |
| **Credential prompt loops** | Repeated authentication | SSO not working, Conditional Access policy blocking | Check Entra ID Conditional Access policies, verify session host domain join |

**See:** [[../AVD/workspaces#common-issues|Workspace Troubleshooting]] for detailed error resolution.

---

## Part 6: Test User Access (Personal Desktop)

Verify personal desktop access for users assigned to personal host pool.

### Prerequisites for Personal Desktop Testing

- [ ] Test user account in Entra ID (e.g., `execuser@contoso.com`)
- [ ] Test user is member of `AVD-Personal-Users` group
- [ ] Test user assigned to specific session host in personal host pool (from Step 07)

> **Important:** Personal host pools use automatic assignment. User's first connection triggers assignment to a specific session host. All future sessions connect to the SAME session host.

### Test with Remote Desktop Client (Windows)

1. **Subscribe to Workspace:**
   - Login as `execuser@contoso.com` in Remote Desktop client
   - Subscribe to workspace (same process as pooled)

2. **View Workspace:**
   - Workspace name: **"AVD Production Workspace"**
   - Resource icons:
     - **"Personal Workstation"** (visible to exec user)
     - **"Pooled Production Desktop"** (NOT visible - exec user not in AVD-Pooled-Users)

3. **Launch Personal Desktop (First Time):**
   - Double-click **"Personal Workstation"** icon
   - AVD broker assigns user to specific session host (e.g., `vm-personal-prod-011`)
   - Connection establishes
   - User logs in with Entra ID credentials
   - Full Windows desktop appears

4. **Verify Persistent Assignment:**
   - Log off from session
   - Reconnect to **"Personal Workstation"**
   - Connection returns to **SAME** session host (`vm-personal-prod-011`)
   - User's desktop state, local files, installed apps persist across sessions

**Expected Result:** User receives dedicated session host on first connection and always reconnects to same host.

### Verify User Assignment in Azure Portal

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-personal-prod → Session hosts

1. Click on session host assigned to user (e.g., `vm-personal-prod-011`)
2. **Assigned User:** Shows `execuser@contoso.com`

**Verification:**
- [ ] User's UPN appears under "Assigned User" column
- [ ] Assignment is permanent until manually changed

> **Note:** Personal host pools support 1:1 user-to-VM assignment. VM deallocates when user logs off (if using Autoscale) but remains assigned to same user.

**See:** [[../AVD/host-pools#personal-assignment-types|Personal Assignment Types]] for automatic vs direct assignment comparison.

---

## Part 7: Configure Client Settings (Optional)

Customize Remote Desktop client behavior via workspace properties and RDP settings.

### Workspace Display Settings

**Portal:** Azure Portal → Virtual Desktop → Workspaces → ws-prod → Properties

**Optional Configurations:**

| Setting | Options | Impact |
|---------|---------|--------|
| **Friendly name** | Custom display name | What users see as workspace name in client |
| **Description** | Brief description | Helps users understand workspace purpose |
| **Public network access** | Enabled, Disabled | Controls internet access (use Disabled with Private Link) |

### Application Group Display Settings

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-pooled-prod → Properties

**Optional Configurations:**

| Setting | Options | Impact |
|---------|---------|--------|
| **Friendly name** | Custom display name | Desktop icon label in Remote Desktop client |
| **Description** | Brief description | Shown in client details panel |

**Example Friendly Names:**

| Resource | Technical Name | Friendly Name | User Experience |
|----------|---------------|---------------|-----------------|
| Workspace | `ws-prod` | "Contoso Virtual Desktops" | Workspace title in client |
| App Group (Pooled) | `ag-pooled-prod` | "Office Desktop" | Desktop icon label |
| App Group (Personal) | `ag-personal-prod` | "Executive Workstation" | Desktop icon label |

> **Best Practice:** Use organization-specific branding in friendly names. Avoid technical resource names like "ag-pooled-prod" in user-facing interfaces.

### RDP Properties (Advanced)

**Portal:** Azure Portal → Virtual Desktop → Host pools → hp-pooled-prod → RDP Properties

**Common Customizations:**

```
drivestoredirect:s:;
audiomode:i:0;
videoplaybackmode:i:1;
redirectclipboard:i:1;
redirectprinters:i:0;
devicestoredirect:s:;
use multimon:i:1;
enablerdsaadauth:i:1;
```

**Key Settings Explained:**

| Property | Value | Description |
|----------|-------|-------------|
| `drivestoredirect:s:` | Empty (no redirection) | Disable local drive redirection for security |
| `audiomode:i:0` | Play on remote host | Audio plays on session host (redirect to client) |
| `redirectclipboard:i:1` | Enabled | Allow clipboard copy/paste between local and remote |
| `redirectprinters:i:0` | Disabled | Disable printer redirection (use cloud printers) |
| `use multimon:i:1` | Enabled | Support multi-monitor setups |
| `enablerdsaadauth:i:1` | Enabled | Enable Entra ID SSO (passwordless authentication) |

> **Warning:** Drive redirection (`drivestoredirect`) allows users to access local drives from session. Disable in high-security environments.

**See:** [[../AVD/rdp-properties|RDP Properties]] for complete property reference and security recommendations.

---

## Variant: RemoteApp Application Groups

For publishing individual applications instead of full desktops (not used in this deployment, but documented for future reference).

### When to Use RemoteApp

**Use Cases:**
- Contractors need access to 1-2 specific LOB apps (e.g., QuickBooks, Salesforce)
- BYOD users who need AVD apps alongside local apps
- Least-privilege security model (users only see published apps, not full OS)

**Example Scenario:**
- Publish Microsoft Excel and custom CRM app to finance contractors
- Users see individual app windows on their local desktop (seamless mode)
- No access to Start menu, File Explorer, or other Windows features

### Create RemoteApp Application Group

**Portal:** Azure Portal → Virtual Desktop → Application groups → + Create

1. **Basics:**
   - **Host pool:** `hp-pooled-prod` (can share host pool with Desktop app group)
   - **Application group type:** RemoteApp
   - **Application group name:** `ag-remoteapp-finance-prod`

2. **Applications:**
   - Click **+ Add application**
   - **Application source:** Start menu
   - **Application:** Excel (or browse for .exe path)
   - **Friendly name:** `Microsoft Excel`
   - **Icon:** Extracted from Excel.exe
   - Click **Save**
   - Repeat for additional apps (CRM, QuickBooks, etc.)

3. **Assignments:**
   - Assign Entra ID group: `AVD-Contractors` (group containing contractor accounts)

4. **Workspace:**
   - Register to: `ws-prod`

**User Experience:**
- Contractors subscribe to `ws-prod` workspace
- See individual app icons (Excel, CRM) instead of desktop icon
- Apps launch in seamless windows on local desktop
- No access to full Windows desktop or other apps

**See:** [[../AVD/application-groups#remoteapp-application-groups|RemoteApp Configuration]] for publishing applications and command-line arguments.

---

## Verification Checklist

Confirm all components configured correctly.

### Application Groups Created

**Portal:** Azure Portal → Virtual Desktop → Application groups

- [ ] `ag-pooled-prod` exists (Desktop type, linked to `hp-pooled-prod`)
- [ ] `ag-personal-prod` exists (Desktop type, linked to `hp-personal-prod`)
- [ ] Friendly names set: "Pooled Production Desktop", "Personal Workstation"

### User Assignments Configured

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-pooled-prod → Access Control (IAM)

- [ ] `AVD-Pooled-Users` assigned "Desktop Virtualization User" role
- [ ] Role assignment scope: Application group level

**Portal:** Azure Portal → Virtual Desktop → Application groups → ag-personal-prod → Access Control (IAM)

- [ ] `AVD-Personal-Users` assigned "Desktop Virtualization User" role

### Workspace Created and Registered

**Portal:** Azure Portal → Virtual Desktop → Workspaces → ws-prod

- [ ] Workspace exists with friendly name "AVD Production Workspace"
- [ ] Application groups registered:
  - `ag-pooled-prod`
  - `ag-personal-prod`
- [ ] Public network access: Enabled (or Disabled if using Private Link)

### End-to-End User Testing

**Test Pooled Desktop:**

- [ ] Pooled user can subscribe to workspace via Remote Desktop client
- [ ] Workspace displays "Pooled Production Desktop" resource
- [ ] User can launch desktop and access Windows 11 multi-session
- [ ] FSLogix profile loads correctly (check `%USERPROFILE%`)

**Test Personal Desktop:**

- [ ] Personal user can subscribe to workspace via Remote Desktop client
- [ ] Workspace displays "Personal Workstation" resource
- [ ] User assigned to specific session host on first connection
- [ ] Subsequent connections return to same session host

**Test Web Client:**

- [ ] Users can access https://client.wvd.microsoft.com
- [ ] Workspace and resources visible in browser
- [ ] Desktop launches successfully via WebRTC

---

## Troubleshooting

### Issue: User sees "No resources available" in workspace

**Symptom:** Workspace appears empty after authentication

**Cause:**
- User not assigned to any application groups
- RBAC role assignment missing or incorrect
- Application groups not registered to workspace

**Fix:**
1. Verify user is member of `AVD-Pooled-Users` or `AVD-Personal-Users` group
2. Check RBAC role assignments:
   - **Portal:** Application group → Access Control (IAM) → Role assignments
   - Verify "Desktop Virtualization User" role assigned to user's group
3. Verify app groups registered to workspace:
   - **Portal:** Workspace → Application groups
   - If missing, click **+ Add** and select app groups

### Issue: User sees workspace but gets "Access Denied" when launching desktop

**Symptom:** Desktop icon visible but connection refused

**Cause:**
- RBAC role assigned but user not in group
- Session host unavailable or drain mode enabled
- NSG blocking AVD traffic

**Fix:**
1. Verify user group membership:
   - **Portal:** Entra ID → Groups → AVD-Pooled-Users → Members
   - Confirm user appears in list
2. Check session host status:
   - **Portal:** Host pool → Session hosts
   - Verify session hosts are "Available" (not drain mode or deallocated)
3. Check NSG rules on session host subnet:
   - Verify outbound HTTPS (443) allowed to AVD service tags

### Issue: User connects but profile doesn't load (new profile created each session)

**Symptom:** User settings reset every login

**Cause:**
- FSLogix not configured on session hosts
- Storage permissions incorrect
- VHDLocations registry pointing to wrong path

**Fix:**
1. Verify FSLogix installed on session host:
   - RDP to session host as admin
   - Check registry: `HKLM\SOFTWARE\FSLogix\Profiles`
   - Verify `VHDLocations` value: `\\stavdprod01.file.core.windows.net\profiles-pooled`
2. Check storage RBAC permissions:
   - **Portal:** Storage account → File share → Access Control (IAM)
   - Verify user's group has "Storage File Data SMB Share Contributor"
3. Test storage connectivity from session host:
   - PowerShell: `net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled`
   - Should mount without credentials (Entra ID authentication)

**See:** [[../Storage/troubleshooting-fslogix-profiles|FSLogix Troubleshooting]] for profile-specific issues.

### Issue: Personal desktop user assigned to wrong session host

**Symptom:** User connects to unexpected VM or assignment changes each session

**Cause:**
- Automatic assignment enabled but assignments not persisting
- Multiple users sharing same personal VM (incorrect configuration)

**Fix:**
1. Verify host pool assignment type:
   - **Portal:** Host pool → Properties
   - **Preferred app group type:** Desktop
   - **Assignment type:** Automatic (or Direct if manual assignment)
2. Check user assignments:
   - **Portal:** Host pool → Session hosts
   - Each VM should show ONE assigned user (or none if unassigned)
3. Manually reassign user if needed:
   - **Portal:** Session host → Properties → Assigned user
   - Select user, click **Assign**

**See:** [[../AVD/host-pools#personal-assignment-types|Personal Assignment]] for assignment type details.

---

## Next Steps

**Application groups and workspace configured.** Users can now access AVD desktops via Remote Desktop client.

**Next:** Step 09: Session Host Configuration & Management (coming soon)

In Step 09, you will:
- Configure session host scaling (manual or autoscale)
- Set up session timeout policies
- Configure user profile settings
- Implement monitoring and alerts

---

## Related Reference Pages

- [[../AVD/application-groups|Application Groups]] - Desktop vs RemoteApp, publishing apps, assignment methods
- [[../AVD/workspaces|Workspaces]] - User discovery, multi-workspace scenarios, friendly names
- [[../AVD/host-pools|Host Pools]] - Host pool types, load balancing, personal assignment
- [[../AVD/rdp-properties|RDP Properties]] - Customizing connection behavior, device redirection
- [[../Identity/rbac-for-avd|RBAC for AVD]] - Role assignments, custom roles, security principals
