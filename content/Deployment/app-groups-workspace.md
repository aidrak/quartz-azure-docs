---
title: App Groups & Workspace
description: 
published: true
date: 2025-12-14T04:52:28.347Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:21.959Z
---

# App Groups & Workspace

Configure application groups and workspace settings to control what users see in their Remote Desktop client.

## Understanding the Hierarchy

```
Workspace (ws-avd-prod)
├── Desktop App Group (hp-pooled-prod1-DAG)
│   └── Full Desktop access
├── Desktop App Group (hp-personal-prod1-DAG)
│   └── Full Desktop access
└── RemoteApp Group (optional)
    └── Individual published apps
```

## Desktop Application Groups (Created with Host Pool)

When you create a host pool, a Desktop Application Group (DAG) is automatically created:

- `hp-pooled-prod1-DAG` → Full desktop from pooled pool
- `hp-personal-prod1-DAG` → Full desktop from personal pool

### Verify DAG Configuration

**Portal:** Host Pool → Application groups

| Setting | Value |
|---------|-------|
| Application group type | Desktop |
| Workspace | ws-avd-prod |
| User assignments | AVD-Users-Pooled (or Personal) |

## Create RemoteApp Group (Optional)

For publishing individual applications instead of full desktops:

**Portal:** Host Pool → Application groups → Create

### Basics
- **Name:** `hp-pooled-prod1-RemoteApp`
- **Host pool:** hp-pooled-prod1
- **Application group type:** RemoteApp

### Applications
- **Add applications:** Select from session host
- Common apps: Calculator, Notepad, company LOB apps

### Workspace
- **Register:** Yes
- **Workspace:** ws-avd-prod

## Configure Workspace

**Portal:** Azure Virtual Desktop → Workspaces → ws-avd-prod

### General Settings
- **Friendly name:** AVD Production Workspace
- **Description:** Production virtual desktops and applications

### Application Groups
Verify all desired app groups are registered:
- hp-pooled-prod1-DAG ✓
- hp-personal-prod1-DAG ✓
- hp-pooled-prod1-RemoteApp (if created) ✓

## User Experience

After configuration, users see in Remote Desktop client:

| App Group Type | What User Sees |
|----------------|----------------|
| Desktop | "SessionDesktop" icon (or friendly name) |
| RemoteApp | Individual app icons |

### Customize Desktop Name

**Portal:** Application Group → Properties

- **Friendly name:** Change "SessionDesktop" to "Cloud Desktop" or similar

## Testing User Access

1. Open Remote Desktop client
2. Subscribe with user credentials
3. Verify workspace appears: "AVD Production Workspace"
4. Verify resources appear: Pooled Desktop, Personal Desktop
5. Connect and verify session works

## Verification

- [ ] Desktop app groups configured
- [ ] RemoteApp group created (if needed)
- [ ] All app groups registered to workspace
- [ ] User groups assigned to appropriate app groups
- [ ] Friendly names configured
- [ ] Test user can see and connect to resources

---

**Next:** [[fslogix-configuration|Step 10: FSLogix Configuration]]