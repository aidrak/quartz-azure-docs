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

Create Azure Files Premium storage account, configure file shares, set RBAC permissions, and prepare NTFS permissions for FSLogix profile containers. This storage layer ensures users have persistent profiles across non-persistent session hosts.

## Example Scenario

Using naming conventions from [[00-naming-conventions]]:

| Resource | Name | Purpose | Tier |
|----------|------|---------|------|
| Storage Account | `stavdprod01` | FSLogix profile containers | Premium FileStorage |
| File Share (Pooled) | `profiles-pooled` | AVD-Pooled-Users profiles | 30GB quota per user |
| File Share (Personal) | `profiles-personal` | AVD-Personal-Users profiles | 30GB quota per user |
| Private Endpoint | `pe-storage-prod` | Secure storage access | Optional |

**Target Users:**
- **AVD-Pooled-Users:** 150 users (General + Finance) → `profiles-pooled`
- **AVD-Personal-Users:** 50 users (Creative + Executives) → `profiles-personal`

## Prerequisites

- [ ] Resource group created: `rg-avd-prod-01` (from [[03-networking-setup]])
- [ ] VNET and subnets configured: `vnet-avd-prod-01` with `snet-avd-prod-privateendpoints` (for private endpoint)
- [ ] Entra ID groups created: `AVD-Pooled-Users`, `AVD-Personal-Users`, `AVD-Admins`
- [ ] Contributor or Storage Account Contributor role on subscription

> **Note:** This step creates storage only. FSLogix agent installation on session hosts occurs in Step 06 (Host Pool Deployment).

---

## Part 1: Create Storage Account

Create Premium FileStorage account for low-latency profile access.

### Storage Tier Selection

**Portal:** Azure Portal → Storage accounts → + Create

**Why Premium vs Standard:**

| Tier | IOPS | Latency | Cost | Use Case |
|------|------|---------|------|----------|
| Standard (HDD) | Up to 10,000 (burst) | 10-20ms | Low | Development/testing only |
| Premium (SSD) | 80,000+ IOPS | 0.5-1ms | Medium | Production (recommended) |

> **Decision:** Use **Premium** for production. Standard tier causes slow logons (30+ seconds) and poor user experience.

**See:** [[../Storage/azure-files-overview|Azure Files Overview]] for detailed tier comparison.

### Create Premium Storage Account

**Portal:** Azure Portal → Storage accounts → + Create

1. **Basics:**
   - **Subscription:** Select your subscription
   - **Resource group:** `rg-avd-prod-01`
   - **Storage account name:** `stavdprod01`
   - **Region:** East US (same as session hosts)
   - **Performance:** Premium
   - **Premium account type:** File shares
   - **Redundancy:** Locally-redundant storage (LRS)

2. **Advanced:**
   - **Security:**
     - **Require secure transfer (HTTPS):** Enabled (default)
     - **Enable infrastructure encryption:** Disabled (default)
     - **Enable blob public access:** Disabled (default)
     - **Minimum TLS version:** Version 1.2 (default)
   - **Hierarchical namespace:** Disabled
   - **Access protocols:** SMB (default for file shares)

3. **Networking:**
   - **Network connectivity:**
     - **Public endpoint (all networks)** (will add private endpoint later)
   - **Network routing:** Microsoft network routing (default)

4. **Data protection:**
   - **Recovery:**
     - **Enable soft delete for blobs:** 7 days (optional)
     - **Enable soft delete for file shares:** 7 days (recommended)
   - **Tracking:**
     - **Enable versioning:** Disabled (not needed for FSLogix)

5. **Encryption:**
   - **Encryption type:** Microsoft-managed keys (default)

6. **Tags:**
   - **Environment:** Production
   - **Purpose:** AVD-FSLogix-Profiles
   - **Owner:** IT-Operations

7. Click **Review + create**
8. Click **Create**

**Deployment time:** 1-2 minutes

### Enable Entra ID Authentication

After storage account deploys, enable Entra ID authentication for identity-based access.

**Portal:** Azure Portal → Storage accounts → stavdprod01 → Settings → Configuration

1. Scroll to **Azure Files Authentication**
2. Under **Active Directory**, select **Microsoft Entra Domain Services**
3. Click **Save**
4. Wait for configuration update (30 seconds)

**Verification:**

```bash
# Via Azure CLI
az storage account show \
  --name stavdprod01 \
  --resource-group rg-avd-prod-01 \
  --query "azureFilesIdentityBasedAuthentication"
```

Expected output:
```json
{
  "directoryServiceOptions": "AADDS"
}
```

> **Important:** Entra ID authentication eliminates storage account keys (which provide unrestricted access and cannot be audited per-user).

**See:** [[../Storage/storage-permissions-for-fslogix|Storage Permissions]] for Entra ID authentication deep dive.

---

## Part 2: Create File Shares

Create separate file shares for pooled and personal user profiles.

### Create profiles-pooled Share

**Portal:** Azure Portal → Storage accounts → stavdprod01 → Data storage → File shares → + File share

1. **Basics:**
   - **Name:** `profiles-pooled`
   - **Access tier:** Not applicable (Premium uses provisioned model)
   - **Provisioned capacity:** 512 GiB (starting size)
   - **Protocol:** SMB

2. **Backup:**
   - **Enable backup:** Optional (configure via Azure Backup later)

3. Click **Review + create**
4. Click **Create**

**Quota Sizing Guidance:**

| Users | Profile Size per User | Total Storage | Provisioned Capacity |
|-------|---------------------|---------------|---------------------|
| 150 | 30 GB | 4,500 GB | 5,000 GiB (5 TiB) |

**Why 512 GiB Start:**
- Allows immediate testing with small user base
- Scale up by editing share (increase provisioned capacity)
- Premium charges for provisioned capacity (not used space)

> **Note:** Start with 512 GiB for testing. Increase to 5,000 GiB after pilot confirms all 150 users.

### Create profiles-personal Share

**Portal:** Azure Portal → Storage accounts → stavdprod01 → Data storage → File shares → + File share

1. **Basics:**
   - **Name:** `profiles-personal`
   - **Provisioned capacity:** 256 GiB (starting size)
   - **Protocol:** SMB

2. Click **Review + create**
3. Click **Create**

**Quota Sizing Guidance:**

| Users | Profile Size per User | Total Storage | Provisioned Capacity |
|-------|---------------------|---------------|---------------------|
| 50 | 30 GB | 1,500 GB | 2,000 GiB (2 TiB) |

**Why Separate Shares:**
- Different NTFS permissions per user group
- Separate quota management (personal users may need 50GB profiles for CAD/dev work)
- Easier to track usage per deployment type

### Configure SMB Settings

**Portal:** Azure Portal → Storage accounts → stavdprod01 → Settings → Configuration

1. Scroll to **File share settings**
2. **SMB security settings:**
   - **Minimum SMB version:** SMB 3.1.1 (highest security)
   - **SMB channel encryption:** AES-256-GCM (enabled by default with 3.1.1)
   - **SMB Multichannel:** Enabled (improves throughput on multi-NIC session hosts)

3. Click **Save**

> **Note:** SMB 3.1.1 provides AES-256 encryption in transit. Windows 10/11 session hosts support this by default.

**See:** [[../Storage/azure-files-overview#protocol-support-details|SMB Protocol Support]] for version comparison.

---

## Part 3: Configure RBAC Permissions

Assign Entra ID groups to storage account with "Storage File Data SMB Share Contributor" role.

### Assign RBAC to AVD-Pooled-Users

**Portal:** Azure Portal → Storage accounts → stavdprod01 → File shares → profiles-pooled → Access Control (IAM)

1. Click **+ Add** → **Add role assignment**
2. **Role:**
   - Search for: `Storage File Data SMB Share Contributor`
   - Select the role
   - Click **Next**
3. **Members:**
   - **Assign access to:** User, group, or service principal
   - Click **+ Select members**
   - Search for: `AVD-Pooled-Users`
   - Select the group
   - Click **Select**
   - Click **Next**
4. **Review + assign:**
   - Review assignments
   - Click **Review + assign**

**What This Role Grants:**
- Read, write, delete files in share
- Modify NTFS permissions on user-owned files
- Create new profile folders

**What This Role Denies:**
- Cannot modify NTFS permissions on other users' files
- Cannot take ownership of files
- Cannot bypass NTFS permissions

### Assign RBAC to AVD-Personal-Users

Repeat process for personal users:

**Portal:** Azure Portal → Storage accounts → stavdprod01 → File shares → profiles-personal → Access Control (IAM)

1. Click **+ Add** → **Add role assignment**
2. **Role:** `Storage File Data SMB Share Contributor`
3. **Members:** `AVD-Personal-Users`
4. Click **Review + assign**

### Assign Elevated Contributor to AVD-Admins

Administrators need elevated permissions for troubleshooting and managing profiles.

**Portal:** Azure Portal → Storage accounts → stavdprod01 → Access Control (IAM)

1. Click **+ Add** → **Add role assignment**
2. **Role:** `Storage File Data SMB Share Elevated Contributor`
3. **Members:** `AVD-Admins`
4. Scope: Storage account level (grants access to all shares)
5. Click **Review + assign**

**What Elevated Contributor Adds:**
- Modify NTFS permissions on any file (including user-owned files)
- Take ownership of files
- Bypass NTFS permissions for troubleshooting

> **Warning:** Only assign Elevated Contributor to administrators. Over-granting breaks NTFS security model.

**See:** [[../Storage/storage-permissions-for-fslogix#required-rbac-roles|RBAC Roles Reference]] for detailed permission comparison.

---

## Part 4: Configure NTFS Permissions

Set NTFS permissions on file share root to allow users to create profile folders while preventing access to other users' profiles.

> **Important:** NTFS configuration requires a domain-joined Windows machine with Elevated Contributor role.

### Prerequisites for NTFS Configuration

- [ ] Windows 11/10 workstation joined to Entra ID (same tenant as storage account)
- [ ] Logged in as user in `AVD-Admins` group (has Elevated Contributor role)
- [ ] Network connectivity to storage account (via private endpoint or public endpoint)

### Mount File Share as Admin

From domain-joined admin workstation:

**PowerShell (Run as Administrator):**

```powershell
# Mount profiles-pooled share
net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled

# Verify mount successful
Get-PSDrive Z
```

Expected output: No credential prompt (Entra ID authentication automatic)

### Configure Root Folder NTFS Permissions (PowerShell)

**PowerShell (Run as Administrator):**

```powershell
# Get current ACL
$acl = Get-Acl "Z:\"

# Remove inherited permissions (start clean)
$acl.SetAccessRuleProtection($true, $false)

# Add CREATOR OWNER (full control on subfolders/files only)
$creatorOwner = New-Object System.Security.Principal.SecurityIdentifier("S-1-3-0")
$rule1 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $creatorOwner,
  "FullControl",
  "ContainerInherit,ObjectInherit",
  "InheritOnly",
  "Allow"
)
$acl.AddAccessRule($rule1)

# Add AVD-Pooled-Users (Modify on this folder only - allows creating profile folder)
$group = New-Object System.Security.Principal.NTAccount("AVD-Pooled-Users")
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

# Verify permissions applied
Get-Acl "Z:\" | Format-List
```

### NTFS Permission Summary

**Root Folder (`\\stavdprod01.file.core.windows.net\profiles-pooled\`):**

| Principal | Permissions | Apply To | Purpose |
|-----------|------------|----------|---------|
| CREATOR OWNER | Full Control | Subfolders and Files Only | Users get full control over their own profile folders |
| AVD-Pooled-Users | Modify | This Folder Only | Users can create their own profile folders |
| AVD-Admins | Full Control | This Folder, Subfolders, and Files | Admins can manage all profiles |

**User Profile Folder (Auto-created by FSLogix):**

Example: `\\stavdprod01.file.core.windows.net\profiles-pooled\jdoe_jdoe@contoso.com\`

| Principal | Permissions | Apply To | Purpose |
|-----------|------------|----------|---------|
| jdoe@contoso.com | Full Control | This Folder, Subfolders, and Files | User owns their profile data |
| AVD-Admins | Full Control | This Folder, Subfolders, and Files | Admins can troubleshoot/recover |

### Repeat for profiles-personal Share

**PowerShell:**

```powershell
# Dismount profiles-pooled
net use Z: /delete

# Mount profiles-personal share
net use Z: \\stavdprod01.file.core.windows.net\profiles-personal

# Run same ACL configuration script (replace AVD-Pooled-Users with AVD-Personal-Users)
$acl = Get-Acl "Z:\"
$acl.SetAccessRuleProtection($true, $false)

# CREATOR OWNER (same as pooled)
$creatorOwner = New-Object System.Security.Principal.SecurityIdentifier("S-1-3-0")
$rule1 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $creatorOwner,
  "FullControl",
  "ContainerInherit,ObjectInherit",
  "InheritOnly",
  "Allow"
)
$acl.AddAccessRule($rule1)

# AVD-Personal-Users (Modify on this folder only)
$group = New-Object System.Security.Principal.NTAccount("AVD-Personal-Users")
$rule2 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $group,
  "Modify",
  "None",
  "None",
  "Allow"
)
$acl.AddAccessRule($rule2)

# AVD-Admins (Full Control)
$adminGroup = New-Object System.Security.Principal.NTAccount("AVD-Admins")
$rule3 = New-Object System.Security.AccessControl.FileSystemAccessRule(
  $adminGroup,
  "FullControl",
  "ContainerInherit,ObjectInherit",
  "None",
  "Allow"
)
$acl.AddAccessRule($rule3)

Set-Acl "Z:\" $acl
Get-Acl "Z:\" | Format-List
```

**See:** [[../Storage/storage-permissions-for-fslogix#ntfs-permissions-on-the-share|NTFS Permissions Reference]] for GUI method and troubleshooting.

---

## Part 5: Create Private Endpoint (Optional)

Private endpoints secure storage access over Azure backbone network (recommended for production).

> **Decision Point:**
> - **With Private Endpoint:** Traffic stays on Azure network, lower latency, more secure
> - **Without Private Endpoint:** Traffic uses public internet, simpler setup, suitable for testing

**If skipping private endpoint:** Proceed to Part 6 (Verification).

### Create Private Endpoint

**Portal:** Azure Portal → Storage accounts → stavdprod01 → Security + networking → Networking

1. Click **Private endpoint connections** tab
2. Click **+ Private endpoint**
3. **Basics:**
   - **Subscription:** Select subscription
   - **Resource group:** `rg-avd-prod-01`
   - **Name:** `pe-storage-prod`
   - **Region:** East US (same as storage account)
4. **Resource:**
   - **Connection method:** Connect to an Azure resource in my directory
   - **Subscription:** (pre-filled)
   - **Resource type:** Microsoft.Storage/storageAccounts
   - **Resource:** stavdprod01
   - **Target sub-resource:** file (for Azure Files)
5. **Virtual Network:**
   - **Virtual network:** `vnet-avd-prod-01`
   - **Subnet:** `snet-avd-prod-privateendpoints`
   - **Network policy for private endpoints:** Disabled (default)
   - **Private IP configuration:** Dynamically allocate IP address
6. **DNS:**
   - **Integrate with private DNS zone:** Yes (recommended)
   - **Subscription:** Select subscription
   - **Resource group:** `rg-avd-prod-01`
   - **Private DNS Zone:** Create new → `privatelink.file.core.windows.net` (auto-created)
7. **Tags:**
   - **Purpose:** AVD-Storage-PrivateAccess
8. Click **Review + create**
9. Click **Create**

**Deployment time:** 2-3 minutes

### Update Storage Account Network Rules

**Portal:** Azure Portal → Storage accounts → stavdprod01 → Security + networking → Networking

1. Under **Firewalls and virtual networks:**
   - **Public network access:** Disabled (force all traffic through private endpoint)
   - **Resource instances:** None (no exceptions)
2. Click **Save**

> **Warning:** Disabling public access breaks connectivity from on-premises or internet unless VPN/ExpressRoute configured.

### Verify Private Endpoint DNS Resolution

From session host or domain-joined workstation:

**PowerShell:**

```powershell
# Test DNS resolution
Resolve-DnsName stavdprod01.file.core.windows.net

# Expected output: Private IP from snet-avd-prod-privateendpoints (10.0.2.x)
# Incorrect output: Public IP (52.x.x.x) - indicates DNS zone not working
```

**Troubleshooting DNS:**
- Verify private DNS zone `privatelink.file.core.windows.net` exists
- Check VNET links in DNS zone (should link to `vnet-avd-prod-01`)
- Restart session host DNS cache: `ipconfig /flushdns`

---

## Part 6: Verification

Confirm storage account, file shares, permissions, and connectivity working correctly.

### Verify Storage Account Configuration

**Portal:** Azure Portal → Storage accounts → stavdprod01

- [ ] **Performance tier:** Premium
- [ ] **Redundancy:** LRS
- [ ] **Entra ID authentication:** Enabled (Configuration blade shows "AADDS")
- [ ] **SMB version:** 3.1.1 minimum
- [ ] **Public access:** Disabled (if using private endpoint)

### Verify File Shares Created

**Portal:** Azure Portal → Storage accounts → stavdprod01 → File shares

- [ ] `profiles-pooled` exists (512 GiB provisioned)
- [ ] `profiles-personal` exists (256 GiB provisioned)
- [ ] Both shares show SMB protocol

### Verify RBAC Assignments

**Portal:** Azure Portal → Storage accounts → stavdprod01 → Access Control (IAM) → Role assignments

- [ ] `AVD-Pooled-Users` has "Storage File Data SMB Share Contributor" on `profiles-pooled`
- [ ] `AVD-Personal-Users` has "Storage File Data SMB Share Contributor" on `profiles-personal`
- [ ] `AVD-Admins` has "Storage File Data SMB Share Elevated Contributor" at storage account level

### Test SMB Connectivity

From domain-joined session host (after deployment in Step 06) or admin workstation:

**PowerShell:**

```powershell
# Test mount without credentials (should succeed with Entra ID auth)
net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled

# Expected: "The command completed successfully."
# If prompts for credentials: Entra ID auth not working (check prerequisites)

# Test write access (as user in AVD-Pooled-Users)
New-Item "Z:\testuser_testuser@contoso.com" -ItemType Directory

# Expected: Folder created successfully
# If "Access Denied": RBAC role not assigned or NTFS permissions incorrect

# Clean up test
Remove-Item "Z:\testuser_testuser@contoso.com" -Force
net use Z: /delete
```

### Verify NTFS Permissions

**PowerShell (as admin in AVD-Admins):**

```powershell
# Mount share
net use Z: \\stavdprod01.file.core.windows.net\profiles-pooled

# Check root folder permissions
Get-Acl "Z:\" | Format-List

# Expected output includes:
# - CREATOR OWNER: FullControl (InheritOnly)
# - AVD-Pooled-Users: Modify (This folder only)
# - AVD-Admins: FullControl (This folder, subfolders, files)
```

### Verify Private Endpoint (If Configured)

**PowerShell:**

```powershell
# Check DNS resolution
Resolve-DnsName stavdprod01.file.core.windows.net

# Expected: Private IP (10.0.2.x) from privateendpoints subnet
# If public IP (52.x.x.x): DNS not configured correctly
```

**Portal:** Azure Portal → Storage accounts → stavdprod01 → Networking → Private endpoint connections

- [ ] `pe-storage-prod` shows **Connection state: Approved**
- [ ] **Private IP address:** Assigned from `snet-avd-prod-privateendpoints`

---

## Troubleshooting

### Issue: Cannot mount share - "Network path not found"

**Symptom:** `net use` fails with error 53

**Cause:**
- Private endpoint DNS not resolving correctly
- Firewall/NSG blocking SMB (port 445)
- Public access disabled but no private endpoint configured

**Fix:**
1. Test DNS: `Resolve-DnsName stavdprod01.file.core.windows.net`
2. If returns public IP but private endpoint configured: Fix DNS zone VNET links
3. If DNS correct: Check NSG on `snet-avd-prod-sessionhosts` allows outbound 445
4. Verify storage account Networking blade allows access from VNET

### Issue: Prompted for credentials when mounting share

**Symptom:** `net use` prompts for username/password

**Cause:**
- Entra ID authentication not enabled on storage account
- Workstation not joined to Entra ID
- User not in RBAC role

**Fix:**
1. Verify storage account Configuration → Azure Files Authentication = "Microsoft Entra Domain Services"
2. Verify workstation: `dsregcmd /status` shows "AzureAdJoined: YES"
3. Verify RBAC: Check IAM blade for user's group assignment
4. Test with storage account key as temporary workaround (insecure, for testing only)

### Issue: Access Denied when creating profile folder

**Symptom:** FSLogix logs show "Access Denied" during profile creation

**Cause:**
- NTFS root folder missing Modify permission for user group
- User not member of RBAC role

**Fix:**
1. Check NTFS permissions on root folder: `Get-Acl "Z:\" | Format-List`
2. Verify AVD-Pooled-Users has Modify on "This Folder Only"
3. Verify user is member of AVD-Pooled-Users group
4. Re-apply NTFS permissions using PowerShell script in Part 4

### Issue: User can access other users' profile folders

**Symptom:** User sees other users' profile folders in file share

**Cause:**
- CREATOR OWNER permission not configured correctly
- NTFS permissions too permissive

**Fix:**
1. Verify CREATOR OWNER rule exists with "InheritOnly" flag
2. Check individual user folders have correct ownership
3. Re-apply NTFS permissions on root folder
4. Delete test folders and recreate to inherit correct permissions

**See:** [[../Storage/troubleshooting-fslogix-profiles|FSLogix Troubleshooting]] for FSLogix-specific issues.

---

## Next Steps

**Storage and permissions configured.** FSLogix agent configuration occurs during session host deployment.

**Next:** [[06-host-pool-deployment|Step 06: Host Pool & Session Host Deployment]]

In Step 06, you will:
- Install FSLogix agent on session hosts
- Configure VHDLocations registry pointing to `\\stavdprod01.file.core.windows.net\profiles-pooled`
- Test user logon and profile container creation

---

## Related Reference Pages

- [[../Storage/azure-files-overview|Azure Files Overview]] - Storage tier comparison, performance characteristics
- [[../Storage/fslogix-profile-containers|FSLogix Profile Containers]] - How FSLogix works, container sizing, concurrent access
- [[../Storage/storage-permissions-for-fslogix|Storage Permissions]] - Entra ID authentication, RBAC roles, NTFS configuration
- [[../Storage/fslogix-cloud-cache|FSLogix Cloud Cache]] - High availability and multi-region scenarios
- [[../Storage/troubleshooting-fslogix-profiles|Troubleshooting FSLogix]] - Common issues and solutions
