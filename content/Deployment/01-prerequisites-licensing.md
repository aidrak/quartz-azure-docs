---
title: Step 01 - Prerequisites & Licensing
description: Verify subscriptions, licenses, and make key architectural decisions
published: true
date: 2025-12-14
tags:
editor: markdown
---

# Step 01: Prerequisites & Licensing

Verify Azure subscription access, licensing entitlements, and make critical architectural decisions before deployment. This step prevents mid-deployment blockers and ensures compliance.

## Subscription Verification

**Portal:** Azure Portal → Subscriptions

1. Confirm active Azure subscription
2. Verify you have **Owner** or **Contributor + User Access Administrator** role
3. Document subscription details:
   - Subscription ID: `________________________________`
   - Subscription Name: `________________________________`
   - Region for deployment: **Central US** (recommended)

### Register Resource Providers

**Portal:** Subscription → Resource providers

Verify these providers are registered (Status = "Registered"):

- [ ] Microsoft.DesktopVirtualization
- [ ] Microsoft.Compute
- [ ] Microsoft.Network
- [ ] Microsoft.Storage
- [ ] Microsoft.Insights

If not registered, select each and click "Register" (takes 2-5 minutes).

## Licensing Check

Every user accessing AVD requires specific licenses. Verify licensing before user onboarding.

### Base AVD Licensing (Per User)

**Portal:** Microsoft 365 Admin Center → Billing → Licenses

Required: **ONE** of the following per user:

| License | Includes | Best For |
|---------|----------|----------|
| **Microsoft 365 E3** | Windows 11 Enterprise, Entra ID P1, Intune, Office 365 | Standard deployment (recommended) |
| **Microsoft 365 E5** | E3 + Entra ID P2, Defender for Endpoint, advanced compliance | High security environments |
| **Microsoft 365 F3** | Entra ID P1, web-based Office, limited Intune | Frontline workers |
| **Windows 11 Enterprise E3/E5** | Windows entitlement only (no Office) | Non-M365 customers |

> **Note:** This deployment assumes **M365 E3** licensing for 200 users. Verify assigned licenses at `https://portal.office.com/AdminPortal/Home#/licenses`

### Entra ID Licensing (Identity & Access)

**Portal:** Entra Admin Center → Billing → Licenses

| Feature | Entra ID Free | Entra ID P1 | Entra ID P2 |
|---------|---------------|-------------|-------------|
| User/Group Management | Yes | Yes | Yes |
| SSO | Yes | Yes | Yes |
| **Conditional Access** | No | **Yes** | **Yes** |
| **Dynamic Groups** | No | **Yes** | **Yes** |
| **MFA** | Limited | **Yes** | **Yes** |
| Identity Protection | No | No | **Yes** |
| Privileged Identity Management | No | No | **Yes** |

**Minimum for production AVD: Entra ID P1** (included in M365 E3)

- Conditional Access enforces security policies (e.g., MFA, location restrictions)
- Dynamic groups automate device management
- MFA protects against credential compromise

**Recommended: Entra ID P2** (included in M365 E5)

- Identity Protection blocks risky sign-ins automatically
- PIM provides just-in-time admin access

### Intune Licensing (Device Management)

**Portal:** Intune Admin Center → Tenant administration → Licenses

| License | Intune Included | AVD Management |
|---------|----------------|----------------|
| M365 E3/E5 | **Yes** (Plan 1) | Full MDM for session hosts |
| M365 F3 | Limited | Basic policies only |
| Intune Plan 1 | Standalone | Full MDM if no M365 |

**This deployment uses Intune for:**
- Security baselines on session hosts
- Application deployment (Office 365, Adobe)
- Compliance policies for device health
- Configuration profiles (antivirus, firewall)

Verify Intune licenses assigned: Entra Admin Center → Users → [User] → Licenses

## Decision Points

Make these architectural decisions before proceeding:

### Decision 1: Cloud-Only vs Hybrid Identity

| Factor | Cloud-Only (Entra Join) | Hybrid (Hybrid Entra Join) |
|--------|-------------------------|----------------------------|
| **On-prem AD dependency** | None | Required |
| **On-prem file shares** | Use Azure Files | Native access |
| **On-prem applications** | Use App Proxy or VPN | Native access |
| **Network requirements** | Internet only | VPN/ExpressRoute to DCs |
| **Management** | Intune only | Intune + Group Policy |
| **Complexity** | Low | Medium |
| **Our choice** | **Entra Join (cloud-only)** | - |

> **Decision Point:**
> - **Cloud-only:** Continue to Step 2
> - **Hybrid:** See [[../Identity/entra-join-vs-hybrid-join|Entra Join vs Hybrid Join]] before proceeding

**This deployment uses: Cloud-Only (Entra Join)**

Rationale:
- No on-prem file share dependencies (using Azure Files)
- All apps cloud-native or SaaS (Office 365, Adobe Creative Cloud)
- Reduced complexity (no Entra Connect, no DCs in Azure)
- Modern management via Intune

### Decision 2: SSO Configuration

**Single Sign-On (SSO)** eliminates double-authentication prompts when users connect to AVD.

| Component | Requirement |
|-----------|-------------|
| Session host join type | Entra Joined or Hybrid Entra Joined |
| Client version | Remote Desktop client 1.2.3317+ |
| Entra ID license | P1 or higher (for Conditional Access integration) |

**Prerequisites for SSO:**

If using **Cloud-Only (Entra Join)**:
- [ ] Session hosts will be Entra Joined (configured in Step 7)
- [ ] Users have Entra ID credentials (user@contoso.com)
- [ ] No additional SSO setup required

If using **Hybrid (Hybrid Entra Join)**:
- [ ] On-premises AD exists
- [ ] Entra Connect configured with:
  - [ ] Password Hash Sync (PHS) OR
  - [ ] Pass-Through Authentication (PTA) OR
  - [ ] AD FS federation
- [ ] Seamless SSO enabled (for PHS/PTA)
- [ ] Session hosts will be Hybrid Entra Joined

**This deployment:**
- SSO enabled via Entra Join (no additional setup required)
- Users authenticate with Entra ID credentials
- See [[../Identity/sso-integration|SSO Integration]] for configuration details

### Decision 3: Deployment Scenario

| Scenario | Users | Host Pools | Our Environment |
|----------|-------|-----------|-----------------|
| Small Business | <50 | 1 pooled | No |
| **Medium Enterprise** | **50-500** | **Pooled + Personal** | **Yes (200 users)** |
| Large Enterprise | 500+ | Multiple pools per region | No |

**This deployment:**
- 200 users total
- Pooled host pool for task workers (150 users)
- Personal host pool for power users (50 users)

## Entra Connect Setup (Hybrid Only)

> **Skip this section if using Cloud-Only (Entra Join)**

If you selected Hybrid identity, configure Entra Connect for directory synchronization:

**Portal:** Entra Admin Center → Hybrid identity → Entra Connect

### Entra Connect Prerequisites

- [ ] On-premises AD domain functional level 2012 R2 or higher
- [ ] Global Administrator account in Entra ID
- [ ] Enterprise Administrator account in on-prem AD
- [ ] Windows Server 2016+ for Entra Connect installation
- [ ] Outbound HTTPS (443) to Azure endpoints

### Installation Steps

1. Download Entra Connect from Entra Admin Center → Hybrid identity
2. Run installer on domain-joined server (not domain controller)
3. Choose authentication method:
   - **Password Hash Sync (PHS)**: Simplest, best for most deployments
   - **Pass-Through Authentication (PTA)**: Keeps passwords on-prem only
   - **Federation (AD FS)**: Complex, use only if required
4. Enable **Seamless SSO** for password-based auth
5. Select OUs to sync (include user and device OUs)
6. Configure filtering if not syncing entire directory

**Verification:**

**Portal:** Entra Admin Center → Users

- [ ] On-prem users visible with "Synced from Windows Server AD" label
- [ ] UPNs match (on-prem SAMAccountName@domain.com = Entra UPN)

**Common Issue:** UPN mismatch breaks SSO

**Fix:** Update on-prem AD UPN suffix to match verified Entra domain

```powershell
# On-prem AD PowerShell
Set-ADUser -Identity jdoe -UserPrincipalName jdoe@contoso.com
```

Force sync:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

See [[../Identity/sso-integration|SSO Integration]] for detailed SSO configuration.

## Pre-Deployment Checklist

Verify before proceeding to Step 2:

### Subscriptions & Licenses

- [ ] Azure subscription active with Owner/Contributor role
- [ ] Resource providers registered
- [ ] M365 E3/E5 licenses assigned to users
- [ ] Entra ID P1/P2 included or assigned
- [ ] Intune licenses verified

### Architectural Decisions

- [ ] Identity model chosen (Cloud-Only or Hybrid)
- [ ] SSO prerequisites identified
- [ ] Deployment scenario documented (pooled/personal/both)
- [ ] Region selected (Central US)

### Hybrid-Specific (If Applicable)

- [ ] Entra Connect installed and syncing
- [ ] Seamless SSO configured
- [ ] UPNs verified and matching
- [ ] VPN/ExpressRoute to on-prem planned

## Verification

### License Verification (PowerShell)

```powershell
# Connect to Microsoft Graph
Connect-MgGraph -Scopes "User.Read.All", "Organization.Read.All"

# Check user licenses
Get-MgUserLicenseDetail -UserId "user@contoso.com" |
  Select-Object -ExpandProperty ServicePlans |
  Where-Object {$_.ServicePlanName -match "AAD|INTUNE|EXCHANGE|WINDOWS"}

# Expected output includes:
# - AAD_PREMIUM (Entra ID P1)
# - INTUNE_A (Intune Plan 1)
# - EXCHANGE_S_ENTERPRISE (Exchange Online)
# - WIN10_PRO_ENT_SUB (Windows 11 Enterprise)
```

### Subscription Verification (Azure CLI)

```bash
# List subscriptions
az account list --output table

# Check resource provider status
az provider list --query "[?namespace=='Microsoft.DesktopVirtualization']" --output table
```

## Next Steps

With licensing verified and architectural decisions made, proceed to resource group creation.

---

**Next:** [[02-resource-groups|Step 2: Resource Groups & Tagging]]

## Reference

- [[../Identity/entra-id-fundamentals|Entra ID Fundamentals]] - Detailed licensing and tenant architecture
- [[../Identity/entra-join-vs-hybrid-join|Entra Join vs Hybrid Join]] - Decision criteria and migration paths
- [[../Identity/sso-integration|SSO Integration]] - SSO configuration and troubleshooting
