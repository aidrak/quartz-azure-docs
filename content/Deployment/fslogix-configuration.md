---
title: FSLogix Configuration
description: 
published: true
date: 2025-12-14T04:52:34.677Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:23.647Z
---

# FSLogix Configuration

Configure FSLogix profile containers on session hosts to provide persistent user profiles for pooled desktops.

## Why FSLogix?

| Without FSLogix | With FSLogix |
|-----------------|--------------|
| New profile each session | Persistent profile |
| Settings lost on logoff | Settings preserved |
| Slow logins (profile copy) | Fast logins (VHD mount) |
| No Outlook search index | Full Outlook experience |

## Configuration Methods

| Method | When to Use |
|--------|-------------|
| **Intune** | Entra-joined devices (recommended) |
| GPO | Hybrid-joined devices |
| Registry (manual) | Testing only |

## Configure via Intune (Recommended)

**Portal:** Intune Admin Center → Devices → Configuration profiles → Create

### Profile Settings
- **Platform:** Windows 10 and later
- **Profile type:** Settings catalog
- **Name:** FSLogix Profile Configuration

### Settings to Configure

Search for "FSLogix" and add:

| Setting | Value | Purpose |
|---------|-------|---------|
| **Enabled** | 1 | Enable profile containers |
| **VHDLocations** | `\\fslogix121025.file.core.windows.net\profiles` | Profile storage path |
| **DeleteLocalProfileWhenVHDShouldApply** | 1 | Clean up local profiles |
| **FlipFlopProfileDirectoryName** | 1 | Username first in folder name |
| **SizeInMBs** | 30000 | 30GB max profile size |
| **VolumeType** | VHDX | Modern format |
| **IsDynamic** | 1 | Dynamic disk (grows as needed) |

### Office Container (Optional)

For better Outlook/Teams performance, enable Office Container:

| Setting | Value |
|---------|-------|
| **ODFC Enabled** | 1 |
| **ODFC VHDLocations** | `\\fslogix121025.file.core.windows.net\profiles` |
| **ODFC IncludeOfficeActivation** | 1 |

### Assignment
- **Assign to:** AVD-Devices-All (dynamic group)

## Configure via GPO (Hybrid Join)

If using Hybrid Join, configure via Group Policy:

**Path:** Computer Configuration → Administrative Templates → FSLogix

Create GPO linked to session hosts OU with same settings as above.

## Verify FSLogix Operation

### On Session Host

```powershell
# Check FSLogix service
Get-Service frxsvc

# Check registry configuration
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles"

# Check current profile location
Get-ItemProperty "HKLM:\SOFTWARE\FSLogix\Profiles" | Select VHDLocations
```

### On Storage Account

Check Azure Files share for profile folders:

```
\\fslogix121025.file.core.windows.net\profiles\
├── user1_S-1-5-21-xxx\
│   └── Profile_user1.VHDX
├── user2_S-1-5-21-xxx\
│   └── Profile_user2.VHDX
```

### During User Session

```powershell
# Verify profile is mounted
Get-Volume | Where-Object { $_.FileSystemLabel -like "*Profile*" }

# Check FSLogix status
& "C:\Program Files\FSLogix\Apps\frx.exe" list-redirects
```

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Profile not loading | Network connectivity | Check private endpoint DNS |
| Slow login | Storage IOPS | Increase share size (more IOPS) |
| "Profile in use" | Previous session didn't close | Wait or manually detach VHD |
| Profile too large | User data in profile | Enable Cloud Cache or increase limit |

> **Note:** For detailed troubleshooting, see [[Troubleshooting FSLogix Profiles|Troubleshooting Profiles]].

## Verification

- [ ] FSLogix settings deployed via Intune
- [ ] Session hosts received policy
- [ ] Test user login creates profile VHD
- [ ] Profile persists across sessions
- [ ] Outlook search index works

---

**Next:** [[monitoring-setup|Step 11: Monitoring Setup]]