---
title: Storage for Profiles
description: 
published: true
date: 2025-12-14T04:52:45.496Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:39.793Z
---

# Storage for Profiles

Deploy Azure Files storage for FSLogix profile containers. This is critical for pooled desktops where user profiles must persist across sessions.

## Create Storage Account

**Portal:** Azure Portal → Storage accounts → Create

### Basics
- **Resource group:** RG-Azure-VDI-01
- **Storage account name:** `fslogix121025` (must be globally unique)
- **Region:** Central US (same as VNet)
- **Performance:** Premium
- **Premium account type:** File shares

### Advanced
- **Require secure transfer:** Yes
- **Enable large file shares:** Enabled (for >5TB shares)

### Networking
- **Network access:** Disable public access and use private access

> **Note:** Premium FileStorage provides the lowest latency for profile operations. Size the share based on user count × expected profile size.

## Create File Share

**Portal:** Storage Account → File shares → + File share

- **Name:** `profiles`
- **Provisioned capacity:** 100 GiB (adjust based on user count)
- **Protocol:** SMB

> **Note:** Premium tier is billed on provisioned capacity, not usage. Size appropriately.

## Create Private Endpoint

**Portal:** Storage Account → Networking → Private endpoint connections → + Private endpoint

### Basics
- **Name:** `pe-fslogix`
- **Region:** Central US

### Resource
- **Target sub-resource:** file

### Virtual Network
- **Virtual network:** vnet-avd
- **Subnet:** snet-privateendpoints

### DNS
- **Integrate with private DNS zone:** Yes
- **Private DNS Zone:** privatelink.file.core.windows.net

## Configure Entra ID Authentication

**Portal:** Storage Account → File shares → Active Directory → Set up

1. Select **Microsoft Entra Domain Services** or **Microsoft Entra Kerberos** (for Entra-joined devices)
2. Follow the wizard to enable Entra authentication

> **Note:** For Entra-joined session hosts, use Entra Kerberos. See [[Storage Permissions for FSLogix|Storage Permissions]] for detailed setup.

## Assign RBAC Permissions

**Portal:** Storage Account → Access Control (IAM) → Add role assignment

| Role | Assignee | Purpose |
|------|----------|---------|
| Storage File Data SMB Share Contributor | AVD-Users-Pooled | Read/write profiles |
| Storage File Data SMB Share Contributor | AVD-Users-Personal | Read/write profiles |
| Storage File Data SMB Share Elevated Contributor | AVD-Users-Admins | Full control |

## Configure NTFS Permissions

Connect to the share from a domain-joined machine and set NTFS permissions:

```powershell
# Map the drive using storage account key (one-time setup)
$storageKey = (Get-AzStorageAccountKey -ResourceGroupName "RG-Azure-VDI-01" -Name "fslogix121025")[0].Value
net use Z: \\fslogix121025.file.core.windows.net\profiles /user:Azure\fslogix121025 $storageKey

# Set root folder permissions
icacls Z:\ /grant "AVD-Users-Pooled:(M)"
icacls Z:\ /grant "CREATOR OWNER:(OI)(CI)(IO)(F)"
```

## Verification

- [ ] Storage account created (Premium FileStorage)
- [ ] File share created
- [ ] Private endpoint configured
- [ ] Private DNS zone linked
- [ ] Entra authentication enabled
- [ ] RBAC roles assigned
- [ ] NTFS permissions configured

---

**Next:** [[image-management|Step 6: Image Management]]