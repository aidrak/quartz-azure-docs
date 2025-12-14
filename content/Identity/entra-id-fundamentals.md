---
title: Entra ID Fundamentals
description: 
published: true
date: 2025-12-14T04:52:53.735Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:47.959Z
---

# Entra ID Fundamentals

Microsoft Entra ID (formerly Azure Active Directory) is the foundation of identity and access management for Azure Virtual Desktop. Understanding Entra ID architecture, licensing, and identity types is critical for designing secure, scalable AVD deployments that meet enterprise requirements.

## What is Entra ID

**Entra ID** is Microsoft's cloud-based identity and access management service. It provides authentication, authorization, and directory services for Azure resources, Microsoft 365, and thousands of third-party SaaS applications.

### Tenants, Directories, and Subscriptions

These three concepts are often confused but have distinct relationships:

**Tenant:** A dedicated instance of Entra ID representing an organization. When you sign up for Microsoft cloud services, you create a tenant with a unique domain like `contoso.onmicrosoft.com`. The tenant is the top-level container for your organization's identity data.

**Directory:** The Entra ID directory is the actual database of users, groups, devices, and applications within a tenant. Most organizations have one directory per tenant, though complex enterprises may have multiple tenants for different business units or compliance requirements.

**Subscription:** An Azure subscription is a billing and resource container tied to a specific Entra ID tenant. You can have multiple subscriptions (dev, test, prod) all trusting the same Entra ID tenant for authentication. Users from the tenant can access resources in any subscription they have permissions for.

**Relationship Example:**
```
Contoso Organization
└── Entra ID Tenant (contoso.onmicrosoft.com)
    ├── Directory (users, groups, devices, apps)
    ├── Subscription 1 (Production)
    │   └── Resource Group: RG-Azure-VDI-01
    │       └── AVD Host Pools, VNets, etc.
    ├── Subscription 2 (Development)
    └── Subscription 3 (Test)
```

All three subscriptions trust the same Entra ID tenant, so users authenticate once and can access resources across subscriptions based on RBAC assignments.

## Licensing Requirements for AVD

AVD licensing follows a user-based model. Each user connecting to AVD must have an appropriate license, regardless of how many sessions they launch.

### Base Requirements

**Minimum License (per user):**
- Microsoft 365 E3/E5
- Microsoft 365 F3
- Microsoft 365 Business Premium
- Windows Enterprise E3/E5 (VDA)

These licenses grant the right to access Windows 10/11 Enterprise multi-session or personal desktops in AVD.

### Entra ID P1 vs P2 Feature Requirements

| Feature | Free | P1 | P2 |
|---------|------|----|----||
| Basic SSO | ✓ | ✓ | ✓ |
| Dynamic Groups | ✗ | ✓ | ✓ |
| Conditional Access | ✗ | ✓ | ✓ |
| MFA (included) | ✗ | ✓ | ✓ |
| Identity Protection | ✗ | ✗ | ✓ |
| Privileged Identity Management (PIM) | ✗ | ✗ | ✓ |
| Access Reviews | ✗ | ✗ | ✓ |

**For AVD deployments, Entra ID P1 is the practical minimum** because:
- Dynamic groups enable automated device and user management
- Conditional Access policies enforce security requirements
- MFA protects against credential compromise

**Entra ID P2 adds:**
- Identity Protection for risk-based Conditional Access (block sign-ins with high risk scores)
- PIM for just-in-time admin access
- Access Reviews for periodic certification of permissions

### Session Host Licensing

Session hosts (VMs) do NOT require individual Windows licenses if:
1. Users have eligible licenses (M365 E3/E5, etc.)
2. Session hosts are enrolled in Entra ID
3. VMs run Windows Enterprise multi-session or are accessed as personal desktops

If using third-party VDI (not AVD) or accessing via non-licensed users, you need RDS CALs or VDA licenses per device/user.

## User vs Device Identities

Entra ID manages two primary identity types, both critical for AVD:

### User Identities

**Cloud-Only Users:** Created directly in Entra ID (user@contoso.onmicrosoft.com or custom domain). No on-premises AD dependency.

**Synced Users:** Synchronized from on-premises Active Directory via Entra Connect. Have both an on-prem AD account and Entra ID representation. UPN must match for seamless SSO.

**Guest Users (B2B):** External identities invited to your tenant. Useful for contractors or partners who need AVD access. Licensed under External Identities pricing, not per-user AVD licensing.

**User Object Attributes:**
- `userPrincipalName`: Primary login identifier
- `objectId`: Unique GUID for the user in Entra ID
- `onPremisesSamAccountName`: Legacy AD username (if synced)
- `extensionAttributes`: Custom fields for dynamic group rules

### Device Identities

AVD session hosts are registered as devices in Entra ID. This enables:
- Conditional Access policies targeting device compliance
- Intune management and security baselines
- Device-based dynamic groups for automation
- SSO to Entra ID-joined session hosts

**Device Join Types:**
- **Entra Joined:** Device exists only in Entra ID (modern, cloud-only)
- **Hybrid Entra Joined:** Device exists in both on-prem AD and Entra ID (synced via Entra Connect)
- **Entra Registered:** Personal device with work account added (BYOD scenarios)

**AVD Session Host Device Object Attributes:**
- `displayName`: VM hostname (e.g., `avd-pool-prod1-0`)
- `deviceId`: Unique GUID in Entra ID
- `deviceOSType`: Windows
- `trustType`: AzureAd (Entra Joined) or ServerAd (Hybrid)
- `extensionAttribute1-15`: Custom fields for grouping

**Example Device Groups in RG-Azure-VDI-01:**
- `AVD-Devices-All`: All AVD session hosts
- `AVD-Devices-Pooled`: Devices for pooled host pools (hp-pooled-prod1)
- `AVD-Devices-Personal`: Devices for personal host pools (hp-personal-prod1)

These groups are populated dynamically based on naming conventions or extension attributes.

## Service Principals and Managed Identities

Automation and integrations require non-human identities for authentication.

### Service Principals

A **service principal** is an identity for an application or service that needs to authenticate to Azure. When you register an app in Entra ID, a service principal is created.

**Use Cases:**
- CI/CD pipelines deploying AVD infrastructure
- Monitoring tools querying AVD metrics
- Custom automation scripts managing host pools

**Authentication Methods:**
- Client secret (password) - expires, must be rotated
- Certificate - more secure, longer-lived
- Federated credentials (OIDC) - no secrets stored

**Example:** Our deployment engine uses a service principal with certificate-based authentication:
```bash
AZURE_CLIENT_ID="<app-id>"
AZURE_TENANT_ID="<tenant-id>"
AZURE_CERTIFICATE_PATH="/path/to/cert.pem"
AZURE_CERTIFICATE_THUMBPRINT="<thumbprint>"

az login --service-principal \
  --username "$AZURE_CLIENT_ID" \
  --tenant "$AZURE_CLIENT_ID" \
  --password "$AZURE_CERTIFICATE_PATH"
```

### Managed Identities

**Managed identities** are service principals managed automatically by Azure. No credentials to store or rotate.

**System-Assigned Managed Identity:**
- Created automatically when enabled on a resource (VM, Function App, etc.)
- Lifecycle tied to the resource (deleted when resource is deleted)
- One identity per resource

**User-Assigned Managed Identity:**
- Created as a standalone resource
- Can be assigned to multiple resources
- Lifecycle independent of resources using it

**AVD Use Case:** An Azure Automation runbook that scales host pools uses a system-assigned managed identity with the "Desktop Virtualization Contributor" role on the AVD resource group. No credentials stored in the runbook.

**Example PowerShell (running on a VM with managed identity):**
```powershell
Connect-AzAccount -Identity
Get-AzWvdSessionHost -ResourceGroupName "RG-Azure-VDI-01" -HostPoolName "hp-pooled-prod1"
```

The `-Identity` flag uses the VM's managed identity to authenticate without requiring credentials.

## Best Practices

- **Use Entra ID P1 minimum** - Dynamic groups and Conditional Access are essential for production AVD deployments, not optional features.
- **Implement device identities for session hosts** - Entra Join enables modern security policies and SSO without Hybrid Join complexity.
- **Prefer managed identities over service principals** - Eliminates credential management and rotation overhead for Azure-based automation.
- **Separate tenants only when required** - Multiple tenants add complexity. Use separate subscriptions within one tenant for department isolation unless compliance mandates tenant separation.
- **Document UPN requirements early** - On-prem AD UPN must match Entra ID UPN for seamless SSO. Mismatches cause authentication failures and are painful to fix post-migration.
- **Plan custom domains before onboarding users** - `user@contoso.onmicrosoft.com` works but `user@contoso.com` is more professional and simplifies federation.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Users can't sign in to AVD | Missing AVD licensing (M365 E3/E5) | Verify license assignment in Entra ID > Users > Licenses |
| Dynamic groups not populating | Entra ID Free or P1 license not assigned | Assign Entra ID P1/P2 license to tenant |
| Service principal authentication fails | Certificate expired or incorrect thumbprint | Regenerate certificate and update app registration |
| Session hosts not appearing in Entra ID | VM not Entra Joined during deployment | Redeploy with `--enable-entra-id-join` or join manually via Settings > Accounts |
| MFA not prompting for AVD users | Conditional Access policy not scoped to AVD app | Edit policy to include "Azure Virtual Desktop" app |

> **Note:** Entra ID free tier supports up to 50,000 objects (users + groups + devices). Most AVD deployments fit within this limit, but large enterprises may need to plan for directory quotas.

> **Warning:** Deleting an Entra ID tenant is irreversible and requires removing all subscriptions, users, and resources first. Never delete a production tenant without a complete backup plan and executive approval.