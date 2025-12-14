---
title: Storage Permissions for FSLogix
description: 
published: true
date: 2025-12-14T04:53:50.199Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:34.185Z
---

# Storage Permissions for FSLogix

Proper permission configuration is the most critical aspect of FSLogix deployments. Incorrect permissions cause profile access failures, data corruption, and security vulnerabilities. Azure Files with Entra ID authentication provides enterprise-grade access control using RBAC roles and NTFS permissions, eliminating the security risks of storage account keys.

## Entra ID-Based Authentication for Azure Files

Entra ID (formerly Azure AD) integration enables Azure Files to authenticate users using their organization identities, enforcing permissions via RBAC and NTFS ACLs.

### How Entra ID Authentication Works

Traditional file shares use username/password (SMB) or network security (NFS). Azure Files with Entra ID uses Azure Active Directory tokens for authentication:

1. **User logs into AVD session host** using Entra ID credentials
2. **Session host requests Kerberos ticket** from Entra ID for storage account
3. **Entra ID validates user** and issues ticket with group memberships
4. **Session host presents ticket** when accessing Azure Files share
5. **Azure Files validates ticket** and checks RBAC roles
6. **If RBAC allows, NTFS permissions evaluated** on requested file/folder
7. **Access granted** if both RBAC and NTFS permit operation

**Key Benefits:**
- No shared credentials (no storage account keys exposed)
- Per-user audit trails (track who accessed what files)
- Group-based access control (use existing Entra ID groups)
- Conditional Access policies (MFA, location restrictions)
- Automatic credential rotation (Entra ID managed)

### Prerequisites

Before enabling Entra ID authentication:

**Required:**
- Azure Files Premium or Standard (any tier)
- Storage account in same Azure AD tenant as users
- Session hosts domain-joined (Entra ID joined or Hybrid joined)
- Line-of-sight from session hosts to storage account (private endpoint or service endpoint recommended)

**Not Required:**
- On-premises Active Directory (cloud-only Entra ID works)
- Storage account keys (disabled after Entra ID enabled)
- Azure AD DS (Premium-tier managed domain service)

**Our Environment:**
- Storage Account: fslogix121025 (PremiumV2_LRS, FileStorage)
- Resource Group: RG-Azure-VDI-01
- Private Endpoint: pe-fslogix-files (connected to vnet-avd/snet-privateendpoints)
- DNS: privatelink.file.core.windows.net zone configured
- Session Hosts: Entra ID joined to contoso.onmicrosoft.com tenant

## Enabling Entra ID Authentication

### Step-by-Step Setup

**Portal: Azure Portal → Storage accounts → fslogix121025 → Settings → Configuration**

1. Navigate to storage account configuration blade
2. Locate "Azure Active Directory Domain Services (Azure AD DS)" setting
3. Change from "Disabled" to "Enabled"
4. Click "Save" and wait for deployment (1-2 minutes)

**Alternative: Azure CLI**

```bash
az storage account update \
  --name fslogix121025 \
  --resource-group RG-Azure-VDI-01 \
  --enable-files-aadds true
```

**Alternative: PowerShell**

```powershell
$storageAccount = Get-AzStorageAccount -ResourceGroupName "RG-Azure-VDI-01" -Name "fslogix121025"
$storageAccount.AzureFilesIdentityBasedAuth.DirectoryServiceOptions = "AADDS"
Set-AzStorageAccount -ResourceGroupName "RG-Azure-VDI-01" -Name "fslogix121025" -EnableAzureActiveDirectoryDomainServicesForFile $true
```

### Verification

**Check Configuration:**

```bash
az storage account show \
  --name fslogix121025 \
  --resource-group RG-Azure-VDI-01 \
  --query "azureFilesIdentityBasedAuthentication"
```

Expected output:
```json
{
  "directoryServiceOptions": "AADDS"
}
```

**Test from Session Host:**

```powershell
# Authenticate using current user's Entra ID identity
net use Z: \\fslogix121025.file.core.windows.net\profiles
```

If successful, share mounts without prompting for credentials. If it prompts for username/password, Entra ID authentication is not working (check prerequisites).

## vs Storage Account Keys (Legacy)

Storage account keys are the traditional authentication method for Azure Storage. Understanding why they are obsolete is important for security compliance.

### Storage Account Keys Overview

Every storage account has two access keys (primary and secondary):
- 512-bit randomly generated keys
- Provide unrestricted access to all data in storage account
- No per-user auditing (all access logs show "key authentication")
- Must be manually rotated and distributed to all clients

### Why Avoid Storage Account Keys

**Security Issues:**
- Any user with key can access all data (no least-privilege)
- Keys often hardcoded in scripts, saved in RDP profiles, shared via email
- Leaked keys grant attacker full access to all files
- No way to revoke access for single user (must rotate key, breaking all clients)

**Operational Issues:**
- Key rotation requires updating all session hosts, scripts, automation
- No audit trail of which user accessed which file
- Cannot apply Conditional Access (MFA, location restrictions)
- Violates compliance frameworks (SOC 2, HIPAA, ISO 27001)

**Microsoft's Position:**
- Deprecated for file share access (use Entra ID instead)
- Disabling keys enforced by Azure Policy in secure environments
- Required for legacy tools only (Azure Storage Explorer with key auth)

### Migration from Keys to Entra ID

If your environment currently uses storage account keys:

**Step 1: Enable Entra ID Authentication (see above)**

**Step 2: Assign RBAC Roles to Users/Groups**

```bash
az role assignment create \
  --role "Storage File Data SMB Share Contributor" \
  --assignee-object-id <group-object-id> \
  --scope "/subscriptions/<sub>/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Storage/storageAccounts/fslogix121025"
```

**Step 3: Update Session Host Registry (Remove Keys)**

Old configuration (using keys):
```
VHDLocations: \\fslogix121025.file.core.windows.net\profiles
# Credentials stored in Windows Credential Manager (insecure)
```

New configuration (using Entra ID):
```
VHDLocations: \\fslogix121025.file.core.windows.net\profiles
# No credentials needed (Entra ID automatic)
```

**Step 4: Test User Logons**

Verify FSLogix mounts profiles using Entra ID auth:
- Check FSLogix logs (C:\ProgramData\FSLogix\Logs\Profile\*.log)
- Look for "Kerberos authentication succeeded" messages
- No "Access denied" or "Credentials required" errors

**Step 5: Disable Storage Account Keys (Optional)**

```bash
az storage account update \
  --name fslogix121025 \
  --resource-group RG-Azure-VDI-01 \
  --allow-shared-key-access false
```

> **Warning:** Disabling shared key access breaks Azure Storage Explorer (unless using Entra ID mode), AzCopy (without --login flag), and legacy scripts. Test thoroughly before enforcing in production.

## Required RBAC Roles

RBAC roles control share-level permissions. Think of RBAC as "who can access the share" and NTFS as "what they can do with files inside."

### Storage File Data SMB Share Contributor

**Permission Level:** Read, write, delete files. Modify NTFS permissions on files owned by user.

**Use Case:** Standard users accessing their own profile containers.

**Permissions Granted:**
- Read files and directories
- Write new files and modify existing files
- Delete files and directories owned by user
- Modify NTFS ACLs on user-owned files
- Execute files

**Permissions Denied:**
- Modify NTFS ACLs on files owned by other users
- Take ownership of files
- Bypass NTFS permissions

**Assign to:**
- AVD-Users-Pooled (Entra ID group for pooled host pool users)
- AVD-Users-Personal (Entra ID group for personal host pool users)

**Assignment Command:**

```bash
# Get group object ID
GROUP_ID=$(az ad group show --group "AVD-Users-Pooled" --query id -o tsv)

# Assign role at storage account scope
az role assignment create \
  --role "Storage File Data SMB Share Contributor" \
  --assignee-object-id $GROUP_ID \
  --scope "/subscriptions/<subscription-id>/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Storage/storageAccounts/fslogix121025"
```

**Scope Options:**
- Storage account level (access to all shares in account)
- Individual share level (access to specific share only, more granular)

For share-level scope:
```bash
az role assignment create \
  --role "Storage File Data SMB Share Contributor" \
  --assignee-object-id $GROUP_ID \
  --scope "/subscriptions/<subscription-id>/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Storage/storageAccounts/fslogix121025/fileServices/default/fileshares/profiles"
```

### Storage File Data SMB Share Elevated Contributor

**Permission Level:** Full control over all files, including taking ownership and modifying NTFS permissions.

**Use Case:** Administrators managing FSLogix profiles, troubleshooting permission issues, performing bulk operations.

**Permissions Granted:**
- All permissions from SMB Share Contributor
- Modify NTFS ACLs on any file (even files owned by other users)
- Take ownership of any file
- Bypass NTFS permissions (like "Full Control" in Windows)

**Assign to:**
- AVD-Admins (Entra ID group for AVD administrators)
- Service accounts (backup software, monitoring tools)
- Break-glass admin accounts

**Assignment Command:**

```bash
GROUP_ID=$(az ad group show --group "AVD-Admins" --query id -o tsv)

az role assignment create \
  --role "Storage File Data SMB Share Elevated Contributor" \
  --assignee-object-id $GROUP_ID \
  --scope "/subscriptions/<subscription-id>/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Storage/storageAccounts/fslogix121025"
```

**Security Consideration:**
- Grant this role sparingly (high privilege)
- Use just-in-time (JIT) access for temporary elevated access
- Audit assignment changes monthly

### Other RBAC Roles (Informational)

**Storage File Data SMB Share Reader:**
- Read-only access to files
- Cannot modify or delete
- Use case: Reporting, auditing, read-only file shares (not for FSLogix)

**Storage Blob Data Contributor:**
- For blob storage (not file shares)
- Does not grant access to Azure Files
- Common mistake: assigning blob role to file share users (no effect)

## NTFS Permissions on the Share

RBAC controls who can access the share. NTFS permissions control what users can do with specific files and folders inside the share. Both layers must permit the action.

### Recommended NTFS Structure

**Root Folder (\\fslogix121025.file.core.windows.net\profiles):**

| Principal | Permissions | Apply To | Explanation |
|-----------|------------|----------|-------------|
| CREATOR OWNER | Full Control | Subfolders and Files Only | Users get full control over their own profile folders |
| AVD-Users-Pooled | Modify | This Folder Only | Users can create their own profile folders |
| AVD-Admins | Full Control | This Folder, Subfolders, and Files | Admins can manage all profiles |

**User Folder (\\fslogix121025.file.core.windows.net\profiles\jdoe_jdoe@contoso.com):**

| Principal | Permissions | Apply To | Explanation |
|-----------|------------|----------|-------------|
| jdoe@contoso.com | Full Control | This Folder, Subfolders, and Files | User owns their profile data |
| AVD-Admins | Full Control | This Folder, Subfolders, and Files | Admins can troubleshoot/recover |

### Setting NTFS Permissions (Step-by-Step)

NTFS permissions must be configured from a domain-joined Windows machine with Elevated Contributor RBAC role.

**Portal: No Portal Support (Must Use Windows File Explorer or PowerShell)**

**Step 1: Mount Share from Domain-Joined Admin Workstation**

```powershell
# Authenticate as user with Elevated Contributor role
net use Z: \\fslogix121025.file.core.windows.net\profiles
```

**Step 2: Configure Root Folder Permissions (PowerShell Method)**

```powershell
# Get current ACL
$acl = Get-Acl "Z:\"

# Remove inherited permissions (start clean)
$acl.SetAccessRuleProtection($true, $false)

# Add CREATOR OWNER (full control on subfolders/files only)
$creatorOwner = New-Object System.Security.Principal.SecurityIdentifier("S-1-3-0") # CREATOR OWNER SID
$rule1 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $creatorOwner,
  "FullControl",
  "ContainerInherit,ObjectInherit",
  "InheritOnly",
  "Allow"
)
$acl.AddAccessRule($rule1)

# Add AVD-Users-Pooled (Modify on this folder only)
$group = New-Object System.Security.Principal.NTAccount("AVD-Users-Pooled")
$rule2 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $group,
  "Modify",
  "None",
  "None",
  "Allow"
)
$acl.AddAccessRule($rule2)

# Add AVD-Admins (Full Control on everything)
$adminGroup = New-Object System.Security.Principal.NTAccount("AVD-Admins")
$rule3 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $adminGroup,
  "FullControl",
  "ContainerInherit,ObjectInherit",
  "None",
  "Allow"
)
$acl.AddAccessRule($rule3)

# Apply ACL
Set-Acl "Z:\" $acl
```

**Step 3: Verify Permissions**

```powershell
Get-Acl "Z:\" | Format-List
```

Expected output shows three access rules matching configuration above.

**Step 4: Test User Access**

Log into session host as standard user (member of AVD-Users-Pooled):
```powershell
# Should succeed (create user's profile folder)
New-Item "\\fslogix121025.file.core.windows.net\profiles\testuser_testuser@contoso.com" -ItemType Directory

# Should succeed (write inside user's folder)
New-Item "\\fslogix121025.file.core.windows.net\profiles\testuser_testuser@contoso.com\test.txt" -ItemType File

# Should fail (cannot access another user's folder)
Get-ChildItem "\\fslogix121025.file.core.windows.net\profiles\jdoe_jdoe@contoso.com\"
# Expected: Access Denied
```

### GUI Method (File Explorer)

**Step 1:** Map drive to share from domain-joined admin workstation

**Step 2:** Right-click root folder → Properties → Security tab

**Step 3:** Click "Advanced" → "Disable inheritance" → "Convert inherited permissions"

**Step 4:** Remove all entries except Administrators (or AVD-Admins)

**Step 5:** Add entry for CREATOR OWNER:
- Principal: CREATOR OWNER
- Type: Allow
- Applies to: Subfolders and files only
- Permissions: Full control

**Step 6:** Add entry for AVD-Users-Pooled:
- Principal: AVD-Users-Pooled
- Type: Allow
- Applies to: This folder only
- Permissions: Modify

**Step 7:** Add entry for AVD-Admins:
- Principal: AVD-Admins
- Type: Allow
- Applies to: This folder, subfolders, and files
- Permissions: Full control

**Step 8:** Click OK, OK, OK (apply changes)

## Our Setup: AVD-Users-Pooled and AVD-Users-Personal

In the RG-Azure-VDI-01 resource group, we use two Entra ID groups for access control:

### AVD-Users-Pooled

**Purpose:** Users assigned to pooled (multi-session) host pools

**Members:** Standard business users (accounting, sales, HR, general office workers)

**RBAC Assignment:**
```bash
az role assignment create \
  --role "Storage File Data SMB Share Contributor" \
  --assignee-object-id $(az ad group show --group "AVD-Users-Pooled" --query id -o tsv) \
  --scope "/subscriptions/<sub-id>/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Storage/storageAccounts/fslogix121025/fileServices/default/fileshares/profiles"
```

**NTFS Permissions:** Modify on root folder (can create profile subfolder), Full Control on own subfolder via CREATOR OWNER

**Host Pool:** hp-pooled (Windows 11 multi-session VMs)

### AVD-Users-Personal

**Purpose:** Users assigned to personal (single-session) host pools

**Members:** Power users (developers, executives, users requiring dedicated VMs)

**RBAC Assignment:**
```bash
az role assignment create \
  --role "Storage File Data SMB Share Contributor" \
  --assignee-object-id $(az ad group show --group "AVD-Users-Personal" --query id -o tsv) \
  --scope "/subscriptions/<sub-id>/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Storage/storageAccounts/fslogix121025/fileServices/default/fileshares/profiles"
```

**NTFS Permissions:** Same as pooled users (share same profiles storage)

**Host Pool:** hp-personal (dedicated Windows 11 VMs per user)

**Why Separate Groups:**
- Different licensing models (multi-session vs single-session)
- Different Conditional Access policies (personal users may have stricter MFA)
- Different host pool assignments in AVD
- Easier to track usage and costs by user type

**Same Storage Account:**
Both groups use fslogix121025 storage account because:
- Profiles are isolated by user folder (no cross-contamination)
- Simplifies management (one set of permissions to maintain)
- Cost efficiency (single Premium v2 storage account scales to 1000+ users)
- User mobility: User can move between pooled and personal without profile issues

## Best Practices

**Always Use Entra ID Authentication** - Never use storage account keys for FSLogix. Entra ID provides per-user auditing, supports MFA, and eliminates shared credential risks. Microsoft deprecates keys for file share access.

**Assign RBAC at Share Level** - Grant permissions to specific file shares (profiles share only) rather than entire storage account. Prevents accidental access to other shares (backups, scripts, etc.) in same account.

**Use Groups, Not Individual Users** - Assign RBAC to Entra ID groups (AVD-Users-Pooled), not individual user accounts. Group-based management scales better and simplifies access reviews during employee changes.

**Minimal Elevated Contributor Grants** - Only assign Elevated Contributor role to administrators who actively manage profiles. Most users need only Contributor role. Over-granting Elevated breaks NTFS security model.

**Test NTFS Permissions Before Production** - Create test user, log into AVD, verify FSLogix creates profile folder successfully. Common error: Root folder missing "Modify" for user group (users can't create profile folder).

**Document Permission Changes** - Maintain runbook documenting RBAC role assignments and NTFS permission structure. Include screenshots of ACL configuration. Reference during troubleshooting and audits.

**Monitor Access Logs** - Enable Storage Analytics logging and review access patterns monthly. Alert on failed authentication attempts (potential compromised accounts or misconfiguration).

**Periodic Access Reviews** - Quarterly review of RBAC assignments and Entra ID group memberships. Remove users who left organization or changed roles. Complies with SOC 2, ISO 27001 requirements.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Access Denied when mounting share | User lacks RBAC role | Assign Storage File Data SMB Share Contributor to user's group |
| FSLogix cannot create profile folder | NTFS root folder missing Modify for users | Add Modify permission for AVD-Users group on root folder |
| User can access other users' profiles | NTFS subfolder permissions too permissive | Verify CREATOR OWNER configured correctly, check subfolder ACLs |
| Kerberos authentication failed | Session host not domain-joined | Join session host to Entra ID or hybrid AD domain |
| Share accessible with keys but not Entra ID | Entra ID auth not enabled on storage account | Run: az storage account update --enable-files-aadds true |
| Cannot modify NTFS permissions | User lacks Elevated Contributor role | Assign Elevated Contributor role to admin account |
| Permissions work from one session host but not another | Session host clock skew (Kerberos sensitivity) | Sync time on session hosts: w32tm /resync /force |
| User prompted for credentials when accessing share | Private endpoint DNS resolution failing | Verify privatelink.file.core.windows.net DNS zone configured |

> **Warning:** Never assign "Owner" or "Contributor" RBAC roles at storage account scope to grant file access. These are management plane roles (create/delete storage account), not data plane roles (read/write files). Use "Storage File Data SMB Share Contributor" instead.

## Next Steps

- **Page 4:** Implement FSLogix Cloud Cache for high availability and multi-region deployments
- **Page 5:** Troubleshoot profile permission issues using FSLogix logs and Event Viewer