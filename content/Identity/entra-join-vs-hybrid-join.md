---
title: Entra Join vs Hybrid Join
description: 
published: true
date: 2025-12-14T04:52:55.418Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:49.566Z
---

# Entra Join vs Hybrid Join

Choosing between Entra Join (cloud-only) and Hybrid Entra Join (on-premises + cloud) is one of the most critical architectural decisions for AVD deployments. This choice impacts authentication flows, resource access, management complexity, and security posture. Our environment uses Entra Join as the modern, cloud-first approach.

## Cloud-Only (Entra Join)

**Entra Join** registers devices directly in Entra ID without any on-premises Active Directory dependency. The device identity exists only in the cloud.

### How It Works

1. During Windows OOBE or Settings > Accounts, the device joins Entra ID
2. Device object is created in Entra ID with `trustType: AzureAd`
3. Users sign in with Entra ID credentials (user@contoso.com)
4. Device receives Intune MDM policies (if licensed)
5. Primary Refresh Token (PRT) enables SSO to cloud apps

### Requirements

**Technical:**
- Windows 10/11 (Pro, Enterprise, or Education)
- Internet connectivity to Entra ID endpoints
- Entra ID P1/P2 for Conditional Access and dynamic groups (recommended)
- Intune license for device management (optional but recommended)

**Network:**
- Outbound HTTPS (443) to:
  - `login.microsoftonline.com`
  - `*.windows.net`
  - `enterpriseregistration.windows.net`
  - `*.manage.microsoft.com` (if using Intune)

**No Requirements For:**
- On-premises Active Directory
- Domain controllers in Azure or accessible via VPN
- Entra Connect sync infrastructure
- Line of sight to on-prem resources

### Authentication Flow (Entra Join)

```
1. User connects to AVD (web client or desktop client)
2. Entra ID authenticates user → issues token
3. User connects to session host (Entra Joined VM)
4. Session host validates token with Entra ID
5. User logged in with Entra ID profile (no on-prem AD lookup)
```

SSO works seamlessly because both the user and device trust only Entra ID.

### What You Can Access

**✓ Works Natively:**
- Microsoft 365 apps (Outlook, Teams, OneDrive, SharePoint)
- Azure resources (Azure Files with Entra ID auth, SQL Database with Entra auth)
- SaaS apps federated with Entra ID (Salesforce, Workday, etc.)
- Web apps using Entra ID authentication
- Azure File Shares with Entra Kerberos (GA since 2023)

**✗ Requires Additional Configuration:**
- On-premises file shares (SMB) - need Kerberos Cloud Trust or VPN
- On-premises applications - need Entra Application Proxy or VPN
- Legacy apps requiring NTLM/Kerberos to on-prem DCs - need Hybrid Join or Kerberos Cloud Trust

## Hybrid Entra Join

**Hybrid Entra Join** maintains device identity in both on-premises Active Directory and Entra ID. The device is domain-joined first, then synchronized to Entra ID via Entra Connect.

### How It Works

1. Device joins on-premises AD domain (traditional domain join)
2. Entra Connect syncs device object to Entra ID
3. Device registers with Entra ID via Service Connection Point (SCP) or Group Policy
4. Device object exists in both directories with `trustType: ServerAd`
5. Users sign in with AD credentials (synced to Entra ID)
6. Device can authenticate to both on-prem resources and cloud resources

### Requirements

**Technical:**
- All Entra Join requirements PLUS:
- On-premises Active Directory Domain Services
- Entra Connect sync server
- Domain controllers accessible from session hosts
- Federation (ADFS) or Seamless SSO configured

**Network:**
- All Entra Join network requirements PLUS:
- Line of sight to on-prem domain controllers (LDAP 389, Kerberos 88, DNS 53)
- VPN or ExpressRoute if session hosts are in Azure
- SMB 445 to on-prem file servers (if accessing file shares)

### Authentication Flow (Hybrid Join)

```
1. User connects to AVD (web client or desktop client)
2. Entra ID authenticates user → issues token
3. User connects to session host (Hybrid Joined VM)
4. Session host validates token with Entra ID
5. User logged in with Entra ID profile
6. When accessing on-prem resource:
   a. Session host contacts on-prem DC
   b. DC issues Kerberos ticket
   c. User accesses file share/app with on-prem ticket
```

The dual identity enables access to both cloud and on-prem resources without re-authentication.

### What You Can Access

**✓ Works Natively:**
- Everything Entra Join supports
- On-premises file shares (SMB) without additional config
- On-premises applications using Kerberos/NTLM
- Group Policy from on-prem AD (device settings)
- Legacy apps requiring AD-based authentication

**✗ Limitations:**
- Requires always-on connectivity to on-prem DCs (VPN or ExpressRoute)
- Increased attack surface (session hosts can be used to pivot to on-prem network)
- Dependency on Entra Connect sync (if sync breaks, new devices can't hybrid join)

## Decision Criteria

Use this table to determine the right approach for your deployment:

| Factor | Entra Join | Hybrid Join |
|--------|------------|-------------|
| **On-prem file shares (SMB)** | Use Azure Files with Entra Kerberos OR Kerberos Cloud Trust | Native support |
| **On-prem applications** | Use Entra App Proxy or VPN | Native support |
| **Existing AD dependency** | Requires migration or coexistence | Leverages existing AD |
| **Group Policy requirement** | Use Intune MDM policies | Can use AD Group Policy |
| **Network requirements** | Internet only | Internet + DC connectivity |
| **Disaster recovery** | Entra ID HA (99.99% SLA) | Depends on on-prem DC availability |
| **Security posture** | Cloud-only attack surface | Cloud + on-prem attack surface |
| **Complexity** | Low (no on-prem dependencies) | Medium (sync, federation, network) |
| **Future-proof** | Microsoft's strategic direction | Supported but not recommended for new deployments |

### Recommendation Matrix

**Choose Entra Join if:**
- ✓ New AVD deployment with no legacy on-prem constraints
- ✓ Cloud-first organization or startups
- ✓ Users primarily access Microsoft 365 and SaaS apps
- ✓ File storage is Azure Files or OneDrive
- ✓ Willing to migrate on-prem apps to cloud (rehost/refactor)

**Choose Hybrid Join if:**
- ✓ Large on-prem file share estate that cannot migrate to Azure Files
- ✓ Mission-critical on-prem apps requiring Kerberos to on-prem DCs
- ✓ Group Policy is deeply embedded in your management strategy
- ✓ Migration timeline to cloud-only is multi-year
- ✓ You have ExpressRoute or reliable VPN to on-prem DCs

## Our Environment: Why Entra Join

The RG-Azure-VDI-01 resource group uses **Entra Join** for session hosts in host pools `hp-pooled-prod1` and `hp-personal-prod1`. This decision was made because:

1. **No on-prem file share dependency** - User profiles use Azure Files with FSLogix, authenticated via Entra ID Kerberos (no AD domain required).

2. **Cloud-first application strategy** - All LOB apps are migrated to Azure or available as SaaS (Office 365, Dynamics 365, etc.).

3. **Reduced complexity** - No Entra Connect sync, no domain controllers in Azure, no VPN dependency for authentication.

4. **Modern management** - Intune MDM replaces Group Policy, enabling cloud-based configuration and compliance.

5. **Security** - Session hosts have no line of sight to on-prem network, eliminating lateral movement risk.

6. **Scalability** - Entra ID autoscale and Intune policies apply instantly to new session hosts without waiting for AD replication.

**Device Groups (all Entra Joined):**
- `avd-devices-all`: All session hosts (dynamic membership)
- `avd-devices-pooled`: Pooled host pool VMs
- `avd-devices-personal`: Personal host pool VMs
- `avd-sessionhosts-sso`: Devices enabled for Entra SSO

These groups are dynamic, populated by rules like:
```
(device.displayName -startsWith "avd-pool") -or (device.displayName -startsWith "avd-pers")
```

## Migration from Hybrid to Entra Join

Many organizations start with Hybrid Join due to legacy constraints but want to migrate to Entra Join as they modernize. This is a one-way transition.

### Migration Steps (High-Level)

1. **Audit dependencies:**
   - Inventory on-prem file shares (migrate to Azure Files)
   - Identify apps using on-prem AD auth (rehost or use App Proxy)
   - Review Group Policies (convert to Intune MDM policies)

2. **Prepare Azure Files for FSLogix:**
   - Enable Entra Kerberos on Azure Files storage account
   - Test profile container access with Entra ID-only auth

3. **Deploy pilot Entra Join host pool:**
   - Create new host pool with Entra Join VMs
   - Assign pilot users
   - Validate all apps and resources work

4. **Migrate users in waves:**
   - Move user groups to Entra Join host pool
   - Monitor for access issues
   - Decommission Hybrid Join session hosts

5. **Decommission on-prem dependencies:**
   - Turn off Entra Connect device sync (keep user sync if needed)
   - Remove domain controllers from Azure (if no other dependencies)
   - Close VPN/ExpressRoute if no longer needed

### Common Migration Blockers

| Blocker | Workaround |
|---------|------------|
| Legacy app requires on-prem AD auth | Use Entra Application Proxy or keep small Hybrid Join pool for that app only |
| File shares too large to migrate | Use Azure File Sync to hybrid-sync on-prem shares, access via cloud endpoint |
| Group Policy still required | Most policies have Intune equivalents; convert to Settings Catalog or OMA-URI |
| Compliance requires on-prem AD | Some regulations misinterpret "cloud-only" as less secure; educate auditors on Entra security |

> **Note:** Entra Kerberos for Azure Files (GA since April 2023) eliminates the primary reason organizations chose Hybrid Join. You can now authenticate FSLogix profiles to Azure Files using only Entra ID credentials, no domain controllers required.

## Best Practices

- **Default to Entra Join for new deployments** - Hybrid Join should be the exception, not the default, in 2024+. Only choose Hybrid if you have a documented technical requirement that cannot be met with cloud-native alternatives.
- **Use Kerberos Cloud Trust if you must access on-prem** - If you have Entra Join session hosts but occasional need for on-prem file shares, enable Kerberos Cloud Trust instead of moving to Hybrid Join.
- **Test both join types in pilot** - Deploy a small Entra Join host pool and a small Hybrid Join host pool with the same apps and data. Let pilot users compare the experience before committing.
- **Document the decision** - Whichever join type you choose, document WHY in your design document. This prevents future engineers from second-guessing the architecture.
- **Plan for migration timeline** - If starting with Hybrid Join, document the migration path to Entra Join and set a target date (e.g., "Migrate to Entra Join within 18 months as on-prem file shares move to Azure Files").

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Entra Join fails during VM deployment | Network blocked Entra endpoints | Verify outbound 443 to `login.microsoftonline.com`, `enterpriseregistration.windows.net` |
| Hybrid Join shows "Device is not hybrid joined" | Entra Connect not syncing devices | Enable device sync in Entra Connect, verify SCP in on-prem AD |
| Can't access on-prem file share from Entra Join VM | No Kerberos ticket to on-prem DC | Enable Kerberos Cloud Trust or migrate share to Azure Files |
| User has two profiles (one on-prem, one cloud) | UPN mismatch between AD and Entra ID | Sync on-prem AD UPN to match Entra ID UPN |
| Group Policy not applying to Entra Join VMs | Entra Join devices don't receive AD GPOs | Convert Group Policies to Intune MDM policies |

> **Warning:** You cannot convert a Hybrid Joined device to Entra Joined in-place. You must unjoin the device from on-prem AD, remove from Entra ID, and re-join as Entra Joined. This requires redeploying session hosts and migrating user profiles.