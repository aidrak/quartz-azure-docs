---
title: Azure Files Entra Kerberos Authentication for FSLogix
description: Complete guide to configuring Entra Kerberos on Azure Files for FSLogix profile containers
published: true
date: 2025-12-15T00:00:00.000Z
tags: [Storage, FSLogix, Entra, Kerberos, authentication, Azure Files]
---

# Azure Files Entra Kerberos Authentication for FSLogix

Complete guide to enabling Microsoft Entra Kerberos authentication on Azure Files for FSLogix profile containers. Entra Kerberos eliminates the need for password-based authentication and works with cloud-only or hybrid identities.

## Prerequisites and Requirements

### Identity Architecture

**For hybrid identities** (recommended for FSLogix):
- On-premises Active Directory Domain Services (AD DS) with domain
- Microsoft Entra Connect or Entra Connect cloud sync syncing users and groups to Entra ID
- User accounts created in AD DS, synced to Entra ID

**For cloud-only identities** (limited support):
- Users created directly in Entra ID (not synced from AD DS)
- Simplified configuration but less mature support for FSLogix

### Storage Account Constraints

- Storage account can authenticate with **only ONE identity source**
- If AD DS or Entra Domain Services already enabled, must disable first before enabling Entra Kerberos
- Cannot mix authentication methods on same storage account

### Client Operating System Requirements

**Supported versions:**
- Windows 11 Enterprise/Pro (single or multi-session)
- Windows 10 Enterprise/Pro version 2004+ with KB5007253 (November 2021 or later)
- Windows Server 2025 with latest cumulative updates
- Windows Server 2022 with KB5007254 (November 2021 or later)

**Device join requirements:**
- Must be **Entra joined** OR **Entra hybrid joined**
- Cannot be AD DS-only or Entra Domain Services-only joined

### Required Services on Clients

These services must be running on all session hosts:
- **WinHTTP Web Proxy Auto-Discovery Service** (`WinHttpAutoProxySvc`)
- **IP Helper Service** (`iphlpsvc`)

### Network and Encryption

- **Port 445 (SMB)**: Must be open outbound from clients to storage account
- **Kerberos encryption**: Always AES-256 (non-negotiable)
- **Client support**: Must support AES256_HMAC_SHA1 encryption type

---

## Part 1: Enable Entra Kerberos on Storage Account

### Option A: Azure Portal (Recommended)

**Navigation**: Azure Portal → Storage Accounts → [Storage Account] → Data storage → File shares

1. Navigate to your storage account
2. Select **Data storage** → **File shares**
3. Click the configuration status link next to **Identity-based access** (shows "Not configured" if not set)
4. Under **Microsoft Entra Kerberos** section, select **Set up**
5. Check the **Microsoft Entra Kerberos** checkbox
6. **(Optional)** If you have on-premises AD DS and will configure ACLs through File Explorer:
   - Enter **Domain Name** (DNS root of AD DS domain)
   - Enter **Domain GUID** (ObjectGUID from AD DS)
   - Skip if using `icacls` command-line method instead
7. Click **Save**

**Retrieve Domain Info** (if needed):

Run on an AD-joined machine:

```powershell
$domainInformation = Get-ADDomain
$domainName = $domainInformation.DnsRoot
$domainGuid = $domainInformation.ObjectGUID.ToString()

Write-Host "Domain Name: $domainName"
Write-Host "Domain GUID: $domainGuid"
```

### Option B: Azure PowerShell

**Basic enablement**:

```powershell
Set-AzStorageAccount -ResourceGroupName <resourceGroupName> `
  -StorageAccountName <storageAccountName> `
  -EnableAzureActiveDirectoryKerberosForFile $true
```

**With domain configuration** (recommended):

```powershell
# Get domain info first (from AD-joined machine)
$domainInformation = Get-ADDomain
$domainGuid = $domainInformation.ObjectGUID.ToString()
$domainName = $domainInformation.DnsRoot

# Enable Entra Kerberos with domain parameters
Set-AzStorageAccount -ResourceGroupName <resourceGroupName> `
  -StorageAccountName <storageAccountName> `
  -EnableAzureActiveDirectoryKerberosForFile $true `
  -ActiveDirectoryDomainName $domainName `
  -ActiveDirectoryDomainGuid $domainGuid
```

### Verification

```powershell
$storageAccount = Get-AzStorageAccount -ResourceGroupName <resourceGroupName> -StorageAccountName <storageAccountName>
$storageAccount.AzureFilesIdentityBasedAuth.DirectoryServiceOptions

# Should output: AADKERB
```

---

## Part 2: Grant Admin Consent to Service Principal

**Critical step**: After enabling Entra Kerberos, the system creates a service principal in Entra ID that must receive explicit admin consent.

**Navigation**: Microsoft Entra ID → App registrations → All Applications

1. Open Microsoft Entra ID in Azure Portal
2. Go to **Manage** → **App registrations**
3. Select **All Applications** tab
4. Search for: **[Storage Account] `<storage-account-name>.file.core.windows.net`**
5. Click to select the app
6. Go to **Manage** → **API permissions**
7. Click **Grant admin consent for [Directory Name]**
8. Confirm by clicking **Yes**

**Verification**: Status column shows green checkmark with "Granted for [Directory Name]"

---

## Part 3: Disable MFA on Storage Account Service Principal

**Why**: If MFA is required via conditional access policy, users see error: "System error 1327: Account restrictions are preventing this user from signing in."

**Navigation**: Microsoft Entra ID → Security → Conditional Access

1. Go to **Security** → **Conditional Access**
2. For each conditional access policy requiring MFA:
   - Edit the policy
   - Go to **Exclude** → **Cloud apps or actions**
   - Add the storage account: **[Storage Account] `<storage-account-name>.file.core.windows.net`**
   - Save the policy

**Alternative**: Create a new conditional access policy specifically excluding the storage account service principal from MFA.

**Reference**: [Exclude service principals from Conditional Access policies](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-policy-all-users-mfa#user-exclusions)

---

## Part 4: Configure SMB Security Settings

### Understanding SMB Protocol Settings

Azure Files allows granular control of SMB protocol security at the storage account level. All file shares inherit these settings.

**Available profiles:**
- **Maximum Compatibility** (default) - Supports SMB 2.1, 3.0, 3.1.1 with multiple encryption types
- **Maximum Security** - SMB 3.1.1 only, AES-256-GCM only, Kerberos only
- **Custom** - Granular control per setting

### Recommended Configuration for FSLogix

**Navigation**: Storage Account → Data storage → File shares → File share settings

1. Navigate to storage account → **Data storage** → **File shares**
2. Under **File share settings**, click the value next to **Security**
3. Select **Custom** profile
4. Configure:
   - **SMB protocol versions**: Select **SMB3.0** and **SMB3.1.1** (for compatibility)
   - **SMB channel encryption**: Select **AES-128-GCM** and **AES-256-GCM**
   - **Authentication methods**: Select **Kerberos** only (required for Entra Kerberos)
   - **Kerberos ticket encryption**: Select **AES-256** (enforced for Entra Kerberos)
5. Click **Save**

### PowerShell Configuration

**View current settings**:

```powershell
$resourceGroupName = "<resource-group>"
$storageAccountName = "<storage-account>"

$storageAccount = Get-AzStorageAccount -ResourceGroupName $resourceGroupName -StorageAccountName $storageAccountName
Get-AzStorageFileServiceProperty -StorageAccount $storageAccount
```

**Set recommended configuration**:

```powershell
Update-AzStorageFileServiceProperty `
    -ResourceGroupName $resourceGroupName `
    -StorageAccountName $storageAccountName `
    -SmbAuthenticationMethod "Kerberos" `
    -SmbChannelEncryption "AES-128-GCM","AES-256-GCM" `
    -SmbKerberosTicketEncryption "AES-256" `
    -SmbProtocolVersion "SMB3.0","SMB3.1.1"
```

---

## Part 5: Assign Share-Level RBAC Permissions

### Understanding the Two-Tier Permission Model

Azure Files uses two permission layers:

1. **Share-level (Azure RBAC)** - High-level gatekeeper
2. **File/directory-level (Windows NTFS)** - Granular per-file access

**Most restrictive permission wins**: Users need both share and NTFS permissions.

### Available Roles

| Role | Use For | Permissions |
|------|---------|-------------|
| **Storage File Data SMB Share Contributor** | End users accessing shares | Read, write, delete files/folders |
| **Storage File Data SMB Share Elevated Contributor** | Admins configuring NTFS ACLs | Read, write, delete, modify ACLs |
| **Storage File Data SMB Share Reader** | Read-only access | Read-only |

**For FSLogix deployments**:
- **End users**: Assign **Storage File Data SMB Share Contributor**
- **Admins**: Assign **Storage File Data SMB Share Elevated Contributor**

### Option A: Assign to Specific Groups

**Navigation**: Storage Account → Data storage → File shares → [File Share] → Access Control (IAM)

1. Go to file share → **Access Control (IAM)**
2. Click **+ Add** → **Add role assignment**
3. On **Role** tab:
   - Search: **Storage File Data SMB Share Contributor**
   - Select it
   - Click **Next**
4. On **Members** tab:
   - Select **User, group, or service principal**
   - Click **+ Select members**
   - Search for user group (e.g., "AVD Users")
   - **Important**: Group must be hybrid identity synced from AD DS
   - Click **Select**
   - Click **Next**
5. On **Review + assign** tab:
   - Review and click **Review + assign**
6. Repeat for admins with **Storage File Data SMB Share Elevated Contributor**

**Wait time**: Allow up to 3 hours for permissions to propagate.

### Option B: Default Share-Level Permissions (All Authenticated Users)

**Navigation**: Storage Account → Data storage → File shares → Active Directory

1. Go to storage account → **Data storage** → **File shares**
2. Click configuration status next to **Identity-based access**
3. Under **Step 2: Set share-level permissions**, select **Enable permissions for all authenticated users and groups**
4. From dropdown, select: **Storage File Data SMB Share Contributor**
5. Click **Save**

**Effect**: All authenticated users automatically get the selected role.

### PowerShell Assignment

```powershell
$resourceGroupName = "<resource-group>"
$storageAccountName = "<storage-account>"
$fileShareName = "<file-share>"
$userPrincipalName = "user@contoso.com"
$roleName = "Storage File Data SMB Share Contributor"

$scope = "/subscriptions/<subscription-id>/resourceGroups/$resourceGroupName/providers/Microsoft.Storage/storageAccounts/$storageAccountName/fileServices/default/fileshares/$fileShareName"

New-AzRoleAssignment -SignInName $userPrincipalName -RoleDefinitionName $roleName -Scope $scope
```

---

## Part 6: Configure NTFS Permissions

### Recommended FSLogix NTFS Permissions

**Root of FSLogix share** (`\\<storage-account>.file.core.windows.net\<file-share>\`):

| Principal | Permission | Applies To | Purpose |
|-----------|-----------|------------|---------|
| **CREATOR OWNER** | Modify | Subfolders and files only | User owns their profile |
| **Domain Admins** | Full Control | This folder, subfolders, files | Admin access |
| **Domain Users** | Modify | This folder only | Ability to create profile folders |

### Mount the File Share

**Prerequisites**:
- User has `Storage File Data SMB Share Elevated Contributor` RBAC role
- Machine is domain-joined or Entra joined
- Network access to domain controllers (for hybrid identities)

**Mount command**:

```cmd
net use Z: \\<storageAccountName>.file.core.windows.net\<fileShareName>
```

**Example**:
```cmd
net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled
```

**Verify**:
```cmd
dir Z:
```

### Option A: Configure with icacls (Recommended)

**Most precise and repeatable method.**

```powershell
# Define your values
$sharePath = "Z:\"
$domainAdmins = "CONTOSO\Domain Admins"
$domainUsers = "CONTOSO\Domain Users"

# Remove inherited permissions
icacls $sharePath /inheritance:r

# CREATOR OWNER - modify on subfolders/files only
icacls $sharePath /grant:r "CREATOR OWNER:(OI)(CI)(IO)(M)"

# Domain Admins - full control everywhere
icacls $sharePath /grant:r "${domainAdmins}:(OI)(CI)(F)"

# Domain Users - modify on this folder only
icacls $sharePath /grant:r "${domainUsers}:(M)"
```

**Inheritance flags**:
- **(OI)** - Object Inherit (applies to files)
- **(CI)** - Container Inherit (applies to subdirectories)
- **(IO)** - Inherit Only (child objects only, not this folder)
- **(M)** - Modify permission
- **(F)** - Full Control permission

**Verify**:

```cmd
icacls Z:
```

**Expected output**:
```
Z:\ CREATOR OWNER:(OI)(CI)(IO)(M)
    CONTOSO\Domain Admins:(OI)(CI)(F)
    CONTOSO\Domain Users:(M)
```

### Option B: Windows File Explorer (Hybrid Identities Only)

1. Mount share (see above)
2. Open File Explorer → mapped drive
3. Right-click root folder → **Properties**
4. Select **Security** tab → **Advanced**
5. Click **Disable inheritance** → **Remove all inherited permissions**
6. **Add CREATOR OWNER**:
   - Click **Add** → **Select a principal**
   - Enter: **CREATOR OWNER** → **OK**
   - **Applies to**: **Subfolders and files only**
   - **Permissions**: Check **Modify**
   - Click **OK**
7. **Add Domain Admins**:
   - Click **Add** → **Select a principal**
   - Enter: **CONTOSO\Domain Admins** → **OK**
   - **Applies to**: **This folder, subfolders and files**
   - **Permissions**: Check **Full control**
   - Click **OK**
8. **Add Domain Users**:
   - Click **Add** → **Select a principal**
   - Enter: **CONTOSO\Domain Users** → **OK**
   - **Applies to**: **This folder only**
   - **Permissions**: Check **Modify**
   - Click **OK**
9. Click **Apply** → **OK**

### Option C: Azure Portal Manage Access (Limited)

**Navigation**: https://aka.ms/portal/fileperms

This special URL provides a manage access interface but with limitations. Primarily for viewing/basic edits. Use icacls for production.

1. Navigate to: https://aka.ms/portal/fileperms
2. Select storage account and file share
3. Click **Manage access** from top menu
4. Click **+ Add permission**
5. Select user/group and configure permissions
6. Click **Save**

---

## Part 7: Configure Client-Side Kerberos Ticket Retrieval

### Registry Setting: CloudKerberosTicketRetrievalEnabled

Clients must retrieve cloud-based Kerberos Ticket Granting Tickets (TGT) during logon.

**Registry path**: `HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters`
**Key**: `CloudKerberosTicketRetrievalEnabled`
**Value**: `1` (DWORD)

### Option A: Intune Settings Catalog (Recommended for AVD)

**Best method for AVD session hosts.**

1. Open **Microsoft Intune Admin Center**
2. Go to **Devices** → **Configuration profiles** → **+ Create profile**
3. Select:
   - **Platform**: Windows 10 and later
   - **Profile type**: Settings catalog
4. Click **Create**
5. Search for: **CloudKerberosTicketRetrievalEnabled**
6. Path: **Authentication** → **Kerberos** → **Cloud Kerberos Ticket Retrieval Enabled**
7. Set value to: **1** (Enabled)
8. Click **Next**
9. **Assignments**: Select `avd-devices-pooled` device group
10. Click **Review + create** → **Create**

**Important**: Use **Settings Catalog**, NOT OMA-URI. OMA-URI does not work on AVD multi-session.

### Option B: Group Policy (On-Premises)

1. Open **Group Policy Management Console**
2. Edit GPO for AVD session hosts
3. Navigate to: **Computer Configuration** → **Administrative Templates** → **System** → **Kerberos**
4. Open: **Allow retrieving the Azure AD Kerberos Ticket Granting Ticket during logon**
5. Set to: **Enabled**
6. Apply policy
7. Run `gpupdate /force` on hosts (or wait for refresh)

### Option C: Registry (Manual/Testing)

```powershell
# Run in elevated PowerShell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" `
  /v CloudKerberosTicketRetrievalEnabled /t REG_DWORD /d 1
```

**Requires reboot** or policy refresh to take effect.

### Verify Configuration

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" `
  -Name CloudKerberosTicketRetrievalEnabled

# Should return: 1
```

---

## Part 8: Verification and Testing

### Step 1: Verify Storage Account Configuration

```powershell
$resourceGroupName = "<resource-group>"
$storageAccountName = "<storage-account>"

$storageAccount = Get-AzStorageAccount -ResourceGroupName $resourceGroupName -StorageAccountName $storageAccountName
$storageAccount.AzureFilesIdentityBasedAuth.DirectoryServiceOptions

# Expected output: AADKERB
```

### Step 2: Verify Service Principal and Admin Consent

1. **Entra ID** → **App registrations** → **All Applications**
2. Search for: **[Storage Account] `<storageAccountName>.file.core.windows.net`**
3. Select app → **API permissions**
4. Verify **Status** shows green checkmark: **Granted for [Directory Name]**

### Step 3: Verify Client Configuration

**Run on session host or test client:**

```powershell
# Check registry key
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" `
  -Name CloudKerberosTicketRetrievalEnabled

# Should return: 1

# Check required services are running
Get-Service -Name WinHttpAutoProxySvc, iphlpsvc | Select-Object Name, Status

# Both should show Status: Running
```

### Step 4: Run Azure Storage Diagnostics

```powershell
# Connect to Azure
Connect-AzAccount

$ResourceGroupName = "<resource-group>"
$StorageAccountName = "<storage-account>"

# Run diagnostics
Debug-AzStorageAccountAuth `
    -StorageAccountName $StorageAccountName `
    -ResourceGroupName $ResourceGroupName `
    -Verbose
```

**Checks performed**:
- Port 445 connectivity
- Entra ID connectivity
- Storage account service principal exists
- CloudKerberosTicketRetrievalEnabled registry key set
- Kerberos realm mappings
- Admin consent granted
- WinHTTP service running
- IP Helper service running
- Machine is Entra or hybrid joined

### Step 5: Test Kerberos Ticket Retrieval

```cmd
# Purge existing Kerberos tickets
klist purge

# Sign out and sign back in to the client machine (or reboot)

# Check for Kerberos ticket
klist
```

**Expected in output**:
```
Client: user@CONTOSO.COM
Server: cifs/<storageaccountname>.file.core.windows.net @ KERBEROS.MICROSOFTONLINE.COM
KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
```

**If you see RC4** instead of AES-256, check encryption type configuration.

### Step 6: Test File Share Mount

```cmd
net use Z: \\<storageAccountName>.file.core.windows.net\<fileShareName>
```

**Success indicators**:
- Drive maps without credential prompt
- `dir Z:` lists files/folders
- User can create test folder: `mkdir Z:\TestFolder`

**Common errors**:
- **Error 1326**: Username/password incorrect → Check RBAC, admin consent, MFA
- **Error 1327**: Account restrictions → Check MFA is disabled on service principal
- **Error 53**: Network path not found → Check port 445, DNS resolution

### Step 7: Test FSLogix Profile Creation

```powershell
# Configure FSLogix registry on session host
$fslogixPath = "HKLM:\SOFTWARE\FSLogix\Profiles"
$vhdLocations = "\\stavdprod01.file.core.windows.net\profiles-pooled"

New-Item -Path $fslogixPath -Force
Set-ItemProperty -Path $fslogixPath -Name "Enabled" -Value 1
Set-ItemProperty -Path $fslogixPath -Name "VHDLocations" -Value $vhdLocations
```

**Sign in as test user** to AVD session

**Verify profile was created**:

```cmd
dir \\stavdprod01.file.core.windows.net\profiles-pooled
```

**Expected**: Folder named `<username>_<SID>` containing `Profile_<username>.vhdx`

**Check FSLogix logs**:

```powershell
Get-Content "C:\ProgramData\FSLogix\Logs\Profile\Profile-*.log" -Tail 50
```

---

## Troubleshooting Common Issues

### Error 1326: Username or Password Incorrect

**Cause**: RBAC role not assigned, admin consent not granted, or MFA issue.

**Solutions**:

**A. Verify RBAC assignment**:
```powershell
$scope = "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>/fileServices/default/fileshares/<share>"
Get-AzRoleAssignment -Scope $scope
```

**B. Verify admin consent** in Entra ID → App registrations → API permissions

**C. Verify MFA is disabled** on service principal via Conditional Access

**D. For private endpoints**, update service principal identifier URIs to include private FQDN

### Error 1327: Account Restrictions Preventing Sign-In

**Cause**: MFA required by conditional access policy.

**Solution**: Exclude storage account service principal from MFA policies.

### Unsupported Encryption Types

**Cause**: Registry `SupportedEncryptionTypes` doesn't include AES.

**Solution A** (recommended):
```powershell
# Delete registry key to restore Windows defaults
reg delete "HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters\SupportedEncryptionTypes" /f
# Reboot
```

**Solution B**: Enable AES via Group Policy:
- **Local Computer Policy** → **Computer Configuration** → **Windows Settings** → **Security Settings** → **Local Policies** → **Security Options**
- **Network Security: Configure encryption types allowed for Kerberos**
- Enable: **AES256_HMAC_SHA1**

### Cannot Contact Domain Controller

**Cause**: Client using traditional auth instead of Entra Kerberos.

**Solutions**:

**A. Verify CloudKerberosTicketRetrievalEnabled**:
```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" -Name CloudKerberosTicketRetrievalEnabled
# Should return: 1
```

**B. Purge tickets and sign in again**:
```cmd
klist purge
# Sign out and back in
```

**C. Verify device join status**:
```cmd
dsregcmd /status
# Check: AzureAdJoined: Yes OR (DomainJoined: Yes AND AzureAdJoined: Yes)
```

**D. Restart services**:
```powershell
Get-Service -Name WinHttpAutoProxySvc, iphlpsvc | Restart-Service
```

### Profile Container Not Created by FSLogix

**Cause**: Insufficient RBAC or NTFS permissions, or FSLogix misconfigured.

**Solutions**:

**A. Verify RBAC assignment** (user/group has Contributor role)

**B. Verify NTFS permissions**:
```cmd
net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled
icacls Z:
# Domain Users should have (M) modify on root
```

**C. Verify FSLogix registry**:
```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\FSLogix\Profiles"
# Required: Enabled = 1, VHDLocations = share path
```

**D. Check FSLogix logs**:
```powershell
Get-Content "C:\ProgramData\FSLogix\Logs\Profile\Profile-*.log" -Tail 100
```

### Service Principal Password Expired

**Symptom**: File share access fails after months of working.

**Cause**: Manual preview enrollment with 6-month password expiration.

**Solution**: Disable and re-enable Entra Kerberos to use GA version.

```powershell
# Disable
Set-AzStorageAccount -ResourceGroupName <rg> -StorageAccountName <account> -EnableAzureActiveDirectoryKerberosForFile $false

# Delete old service principal (use Entra ID portal)

# Re-enable (follows new GA process with auto-managed passwords)
Set-AzStorageAccount -ResourceGroupName <rg> -StorageAccountName <account> -EnableAzureActiveDirectoryKerberosForFile $true
```

---

## Summary Checklist

- [ ] **Storage Account**: Entra Kerberos enabled
- [ ] **Service Principal**: Admin consent granted
- [ ] **Conditional Access**: MFA disabled on storage service principal
- [ ] **SMB Settings**: Kerberos auth only, AES-256 encryption configured
- [ ] **RBAC**: Users have Contributor role, admins have Elevated Contributor
- [ ] **NTFS**: CREATOR OWNER, Domain Admins, Domain Users configured correctly
- [ ] **Clients**: CloudKerberosTicketRetrievalEnabled registry set (via Intune)
- [ ] **Clients**: WinHTTP and IP Helper services running
- [ ] **Verification**: `Debug-AzStorageAccountAuth` passes all checks
- [ ] **Verification**: Kerberos ticket shows AES-256 encryption
- [ ] **Verification**: File share mounts without credential prompt
- [ ] **Verification**: FSLogix profile container created on logon

---

## Related References

- [[../Deployment/05-storage-fslogix|Step 05: Storage & FSLogix Setup]] - Deployment playbook
- [[../Deployment/07-fslogix-configuration|Step 07: FSLogix Configuration via Intune]] - FSLogix Intune policies
- [[../Storage/fslogix-profile-containers|FSLogix Profile Containers]] - Architecture and best practices
- [[../Storage/storage-permissions-for-fslogix|Storage Permissions for FSLogix]] - Detailed permission configuration
- [[../Intune/device-configuration-profiles|Device Configuration Profiles]] - Intune policy management

---

## Microsoft Documentation

- [Azure Files Entra Kerberos Authentication | Microsoft Learn](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-hybrid-identities-enable)
- [Configure File and Directory Level Permissions | Microsoft Learn](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-identity-configure-file-level-permissions)
- [Assign Share-Level Permissions | Microsoft Learn](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-identity-assign-share-level-permissions)
- [Store FSLogix Profile Containers with Entra ID | Microsoft Learn](https://learn.microsoft.com/en-us/fslogix/how-to-configure-profile-container-entra-id-hybrid)
- [Troubleshoot Azure Files SMB Authentication | Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-storage/files/security/files-troubleshoot-smb-authentication)
