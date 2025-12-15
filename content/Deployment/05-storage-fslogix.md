---
title: Step 05 - Storage & FSLogix Setup
description: Create Azure Files storage account and configure FSLogix profile containers
published: true
date: 2025-12-14T00:00:00.000Z
tags:
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 05: Storage & FSLogix Setup

Create Premium Azure Files storage account and configure file shares with RBAC and NTFS permissions for FSLogix profile containers. See [[../Storage/fslogix-profile-containers|FSLogix Profile Containers]] for architecture and sizing details.

## Prerequisites

- [ ] Resource group created: `rg-avd-prod-01` (from [[03-networking-setup]])
- [ ] VNET and subnets configured: `vnet-avd-prod-01` with `snet-avd-prod-privateendpoints`
- [ ] Entra ID groups created: `avd-users-pooled`, `avd-users-personal`, `avd-users-admin`
- [ ] Storage Account Contributor role on subscription

---

## Part 1: Create Storage Account

**Portal:** Azure Portal → Storage accounts → + Create

| Setting | Value |
|---------|-------|
| **Subscription** | Your subscription |
| **Resource group** | `rg-avd-prod-01` |
| **Storage account name** | `stavdprod01` |
| **Region** | East US |
| **Performance** | Premium |
| **Premium account type** | File shares |
| **Redundancy** | Locally-redundant storage (LRS) |

1. Navigate to Azure Portal → Storage accounts → + Create
2. **Basics tab:** Fill in table above
3. **Advanced tab:**
   - Hierarchical namespace: Disabled
   - Access protocols: SMB
4. **Networking tab:** Public endpoint (all networks)
5. **Tags:** Environment=Production, Purpose=AVD-FSLogix
6. Click **Review + create** → **Create** (1-2 minutes)

---

## Part 2: Configure Entra Kerberos Authentication

FSLogix requires Entra Kerberos authentication on Azure Files. This replaces password-based access and works seamlessly with both hybrid and cloud-only identities.

> **Important**: Entra Kerberos setup involves multiple steps: enabling Kerberos, granting admin consent, disabling MFA on the service principal, configuring SMB security, assigning RBAC permissions, and setting NTFS ACLs. See [[../Storage/azure-files-entra-kerberos|Azure Files Entra Kerberos Configuration]] for complete step-by-step instructions.

**Quick Setup Path:**

1. **Enable Entra Kerberos:**
   - **Portal:** stavdprod01 → Data storage → File shares → Identity-based access → Set up
   - Check **Microsoft Entra Kerberos** → **Save**

2. **Grant admin consent to service principal:**
   - **Portal:** Entra ID → App registrations → All Applications
   - Search: **stavdprod01.file.core.windows.net**
   - API permissions → **Grant admin consent for [Directory]**

3. **Disable MFA on storage service principal:**
   - **Portal:** Entra ID → Security → Conditional Access
   - For each MFA policy: Add **stavdprod01.file.core.windows.net** to Exclude

4. **Configure SMB security settings:**
   - **Portal:** stavdprod01 → Data storage → File shares → File share settings → Security
   - Select **Custom** profile
   - Authentication: **Kerberos only**
   - Channel Encryption: **AES-128-GCM, AES-256-GCM**
   - Kerberos Ticket Encryption: **AES-256**

5. **Assign RBAC share-level permissions:**
   - **Portal:** Storage account → Data storage → File shares → [File Share] → Access Control (IAM)
   - Add role: **Storage File Data SMB Share Contributor** to user group
   - Add role: **Storage File Data SMB Share Elevated Contributor** to admin group

**For detailed instructions, prerequisites, troubleshooting, and verification steps**, see [[../Storage/azure-files-entra-kerberos|Azure Files Entra Kerberos Configuration]].

---

## Part 3: Create File Share

**Portal:** Azure Portal → stavdprod01 → Data storage → File shares → + File share

Create one file share for pooled session hosts:

| Name | Capacity | Purpose |
|------|----------|---------|
| `profiles-pooled` | 512 GiB | avd-users-pooled profiles |

1. **Name:** `profiles-pooled`
2. **Provisioned capacity:** 512 GiB
3. **Protocol:** SMB
4. Scale capacity after pilot (edit share to increase)

**Configure SMB settings:**

**Portal:** Azure Portal → stavdprod01 → Settings → Configuration

1. Scroll to **File share settings**
2. Set: Minimum SMB version=**3.1.1**, SMB Multichannel=**Enabled**
3. Click **Save**

---

## Part 4: Configure RBAC Permissions

RBAC role assignment was configured in Part 2 (Entra Kerberos). Verify assignments are in place:

**Verification:**

**Portal:** Azure Portal → stavdprod01 → Access Control (IAM) → Role assignments

Confirm:
| Group | Role | Scope |
|-------|------|-------|
| `avd-users-pooled` | Storage File Data SMB Share Contributor | profiles-pooled share |
| `avd-users-admin` | Storage File Data SMB Share Elevated Contributor | Storage account |

If not assigned, see [[../Storage/azure-files-entra-kerberos#part-5-assign-share-level-rbac-permissions|Entra Kerberos: Assign Share-Level RBAC Permissions]] for detailed steps.

See [[../Storage/storage-permissions-for-fslogix|Storage Permissions]] for additional role details.

---

## Part 5: Configure NTFS Permissions

Configure directory and file-level permissions on the FSLogix file share. From domain-joined admin workstation (in `avd-users-admin` group):

**PowerShell (Run as Administrator):**

```powershell
# Mount profiles-pooled
net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled

# Configure NTFS permissions
$acl = Get-Acl "Z:\"
$acl.SetAccessRuleProtection($true, $false)

# CREATOR OWNER - full control on user folders
$creatorOwner = New-Object System.Security.Principal.SecurityIdentifier("S-1-3-0")
$rule1 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $creatorOwner,"FullControl","ContainerInherit,ObjectInherit","InheritOnly","Allow")
$acl.AddAccessRule($rule1)

# avd-users-pooled - modify on root (allows folder creation)
$group = New-Object System.Security.Principal.NTAccount("avd-users-pooled")
$rule2 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $group,"Modify","None","None","Allow")
$acl.AddAccessRule($rule2)

# avd-users-admin - full control
$adminGroup = New-Object System.Security.Principal.NTAccount("avd-users-admin")
$rule3 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $adminGroup,"FullControl","ContainerInherit,ObjectInherit","None","Allow")
$acl.AddAccessRule($rule3)

Set-Acl "Z:\" $acl
```

See [[../Storage/storage-permissions-for-fslogix|Storage Permissions]] for GUI method and additional details.

---

## Part 6: Create Private Endpoint (Optional)

> **Decision Point:** Use private endpoint for production (traffic on Azure network), or skip for testing (public endpoint).

**Portal:** Azure Portal → stavdprod01 → Security + networking → Networking

1. Click **Private endpoint connections** → **+ Private endpoint**
2. Fill in:
   - Name: `pe-storage-prod`
   - Resource group: `rg-avd-prod-01`
   - Virtual network: `vnet-avd-prod-01`
   - Subnet: `snet-avd-prod-privateendpoints`
   - Target sub-resource: `file`
   - Integrate with private DNS: Yes
3. Click **Review + create** → **Create**

Then disable public access:
1. Return to stavdprod01 → Networking
2. Set **Public network access:** Disabled
3. Click **Save**

---

## Troubleshooting

For issues with Entra Kerberos authentication, NTFS permissions, RBAC roles, or FSLogix profile creation, see:

- [[../Storage/azure-files-entra-kerberos#troubleshooting-common-issues|Entra Kerberos Troubleshooting]] - Comprehensive troubleshooting guide
  - Error 1326: Username/password incorrect
  - Error 1327: Account restrictions
  - Encryption type errors
  - Domain controller connectivity
  - Profile container creation failures
  - Service principal password expiration

### Quick Diagnostics

**Test storage configuration:**

```powershell
# Verify Entra Kerberos is enabled
$storageAccount = Get-AzStorageAccount -ResourceGroupName <rg> -StorageAccountName <account>
$storageAccount.AzureFilesIdentityBasedAuth.DirectoryServiceOptions
# Should return: AADKERB

# Run Azure diagnostics
Debug-AzStorageAccountAuth -StorageAccountName <account> -ResourceGroupName <rg> -Verbose
```

**Test mount:**

```cmd
# Mount the share
net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled

# Check for errors (Error 1326/1327 = Entra Kerberos issue, see troubleshooting guide)
dir Z:
```

---

## Next Steps

Storage account, Entra Kerberos authentication, and file share permissions are now configured.

**Next:** [[06-intune-configuration|Step 06: Intune Configuration]]

In Step 06, you will configure:
- Win32 applications for deployment to all AVD devices
- FSLogix profile container policies via Intune (before VMs boot)
- Default device configuration profiles

Then in Step 07, you will configure FSLogix registry settings via Intune policies before deploying session hosts.

---

## Related References

- [[../Storage/azure-files-entra-kerberos|Azure Files Entra Kerberos Configuration]] - Complete Entra Kerberos setup guide with troubleshooting
- [[../Storage/fslogix-profile-containers|FSLogix Profile Containers]] - Architecture, sizing, concurrent access
- [[../Storage/storage-permissions-for-fslogix|Storage Permissions]] - Detailed RBAC and NTFS permission guidance
- [[../Deployment/06-intune-configuration|Step 06: Intune Configuration]] - Win32 apps and FSLogix policies
- [[../Deployment/07-fslogix-configuration|Step 07: FSLogix Configuration via Intune]] - Registry policy deployment
