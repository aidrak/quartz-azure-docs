---
title: Identity & Access Management Reference
description: Microsoft Entra ID, authentication, and access control for Azure Virtual Desktop
published: true
date: 2025-12-14
tags:
  - identity
  - entra-id
  - authentication
  - conditional-access
editor: markdown
---

# Identity & Access Management

This section covers Microsoft Entra ID (formerly Azure Active Directory) configuration for Azure Virtual Desktop deployments. Identity is the foundation of AVD security, encompassing user authentication, device identity, access policies, and role-based permissions.

## What's Covered

The Identity reference pages provide deep-dive technical content on:

- **Entra ID Fundamentals** - Tenant architecture, licensing requirements, user and device identities, service principals, and managed identities
- **Authentication Methods** - Single Sign-On (SSO), Multi-Factor Authentication (MFA), and passwordless authentication
- **Access Control** - Conditional Access policies, role-based access control (RBAC), and security group management
- **Device Identity** - Entra Join vs Hybrid Entra Join decision criteria, device registration, and join methods
- **Automation** - Dynamic groups for automatic membership management based on user and device attributes

## Reference Pages

### Core Identity Concepts

#### [[../../Identity/entra-id-fundamentals|Entra ID Fundamentals]]
Understanding tenants, subscriptions, licensing tiers (Free/P1/P2), user identities (cloud-only vs synced), device identities, service principals, and managed identities. Essential reading for AVD architects making licensing and identity model decisions.

#### [[../../Identity/entra-join-vs-hybrid-join|Entra Join vs Hybrid Join]]
Decision criteria for cloud-only (Entra Join) vs hybrid (Hybrid Entra Join) device identity. Covers authentication flows, on-premises resource access, network requirements, and migration paths from hybrid to cloud-only deployments.

### Access Control & Security

#### [[../../Identity/conditional-access|Conditional Access]]
Policy-based access control that evaluates user, location, device, and risk signals before granting AVD access. Includes policy patterns for requiring MFA, enforcing device compliance, blocking untrusted locations, and implementing risk-based authentication.

#### [[../../Identity/mfa-for-avd|MFA for AVD]]
Multi-factor authentication configuration for Azure Virtual Desktop. Covers supported MFA methods (Authenticator app, FIDO2, Windows Hello), Conditional Access integration, and troubleshooting common MFA prompting issues.

#### [[../../Identity/rbac-for-avd|RBAC for AVD]]
Azure Role-Based Access Control for AVD resource management. Explains AVD-specific built-in roles (Desktop Virtualization Contributor, Desktop Virtualization User, etc.), role assignment scopes, and separation of administrative vs end-user access.

### Authentication & Sign-In

#### [[../../Identity/sso-integration|SSO Integration]]
Single Sign-On configuration that eliminates double authentication prompts for AVD connections. Covers SSO prerequisites, host pool enablement, Entra ID configuration, and RDP client requirements.

### Automation & Management

#### [[../../Identity/dynamic-groups|Dynamic Groups]]
Automatic group membership management using attribute-based rules. Essential for scaling AVD deployments with zero-touch device assignment, automated user categorization, and policy enforcement. Includes rule syntax, AVD naming convention patterns, and processing time considerations.

## Related Quick-Deploy Steps

When implementing the concepts from these reference pages, follow the linear deployment playbook:

### [[../../Quick-Deploy/01-prerequisites-licensing|Step 01: Prerequisites & Licensing]]
Verify Azure subscriptions, validate Microsoft 365 and Entra ID licensing requirements, decide on cloud-only vs hybrid identity model, and determine SSO configuration approach.

**References from this step:**
- [[../../Identity/entra-id-fundamentals|Entra ID Fundamentals]] for licensing details
- [[../../Identity/entra-join-vs-hybrid-join|Entra Join vs Hybrid Join]] for identity model decision
- [[../../Identity/sso-integration|SSO Integration]] for SSO planning

### [[../../Quick-Deploy/02-identity-setup|Step 02: Identity Setup]]
Create test users, configure dynamic user and device groups, implement Conditional Access policies (MFA, device compliance, location-based access), and prepare for SSO enablement.

**References from this step:**
- [[../../Identity/dynamic-groups|Dynamic Groups]] for automated group membership
- [[../../Identity/conditional-access|Conditional Access]] for policy design patterns
- [[../../Identity/mfa-for-avd|MFA for AVD]] for authentication methods
- [[../../Identity/rbac-for-avd|RBAC for AVD]] for role assignments (configured after host pool creation)
- [[../../Identity/sso-integration|SSO Integration]] for configuration details

## Key Architectural Decisions

Before proceeding with AVD deployment, make these identity-related decisions:

1. **Cloud-Only vs Hybrid Identity**
   - Entra Join: Modern, cloud-first approach with no on-premises AD dependency
   - Hybrid Join: Maintains on-premises AD for legacy app and file share access
   - See [[../../Identity/entra-join-vs-hybrid-join|Entra Join vs Hybrid Join]] for detailed comparison

2. **Licensing Requirements**
   - Minimum: Entra ID P1 (required for Conditional Access and dynamic groups)
   - Recommended: Entra ID P2 (adds Identity Protection and Privileged Identity Management)
   - See [[../../Identity/entra-id-fundamentals|Entra ID Fundamentals]] for feature comparison

3. **Authentication Strategy**
   - SSO: Enable for seamless user experience
   - MFA: Required for production environments
   - Passwordless: Optional advanced security feature
   - See [[../../Identity/sso-integration|SSO Integration]] and [[../../Identity/mfa-for-avd|MFA for AVD]]

4. **Access Control Model**
   - Conditional Access: Define security policies (MFA, device compliance, location)
   - RBAC: Separate administrative access from end-user access
   - See [[../../Identity/conditional-access|Conditional Access]] and [[../../Identity/rbac-for-avd|RBAC for AVD]]

## Best Practices Summary

- Use Entra ID P1 minimum for production AVD deployments (Conditional Access and dynamic groups are essential, not optional)
- Default to Entra Join for new deployments unless you have documented requirements for on-premises resource access
- Implement Conditional Access policies in report-only mode first, monitor for 1-2 weeks before enforcement
- Use dynamic groups for session host management and user assignment to eliminate manual group maintenance
- Assign RBAC roles to Entra ID groups, never to individual users (except break-glass scenarios)
- Create break-glass emergency access accounts excluded from all Conditional Access policies
- Enable SSO to reduce credential prompts and improve user experience
- Require MFA for all AVD connections from external networks

## Common Patterns

### Production AVD Identity Configuration

Typical identity setup for a production AVD environment with 200 users:

**User Groups (Dynamic):**
- `avd-users-pooled` - Automatic assignment based on department (General, Finance)
- `avd-users-personal` - Automatic assignment for power users (Creative, Executives)
- `avd-users-admin` - Manual assignment for IT administrators

**Device Groups (Dynamic):**
- `avd-devices-all` - All session hosts (naming: vm-pooled-*, vm-personal-*)
- `avd-devices-pooled` - Multi-session hosts only (naming: vm-pooled-*)
- `avd-devices-personal` - Single-session hosts only (naming: vm-personal-*)

**Conditional Access Policies:**
- Policy 1: Require MFA for all AVD users
- Policy 2: Require compliant device for pooled users
- Policy 3: Block access from untrusted locations (optional)

**RBAC Assignments:**
- Desktop Virtualization Contributor: `avd-users-admin` group (resource group scope)
- Desktop Virtualization User: `avd-users-pooled` and `avd-users-personal` groups (application group scope)

See [[../../Quick-Deploy/02-identity-setup|Step 02: Identity Setup]] for implementation details.

## Troubleshooting Quick Links

**Dynamic Groups Not Populating:**
- Verify Entra ID P1/P2 license assigned
- Check rule syntax with "Validate rules" feature
- Allow 5-15 minutes for small groups, up to 24 hours for large groups
- See [[../../Identity/dynamic-groups|Dynamic Groups]] troubleshooting section

**Users Can't Connect to AVD:**
- Verify Desktop Virtualization User role assigned at application group scope
- Check Conditional Access policies not blocking access
- Confirm MFA registration completed
- See [[../../Identity/rbac-for-avd|RBAC for AVD]] and [[../../Identity/conditional-access|Conditional Access]]

**SSO Not Working:**
- Verify session hosts are Entra Joined or Hybrid Entra Joined
- Confirm RDP client version 1.2.3317 or later
- Check host pool SSO setting enabled
- Ensure `targetisaadjoined:i:1` in RDP properties
- See [[../../Identity/sso-integration|SSO Integration]] troubleshooting

**MFA Prompting Every Connection:**
- Configure sign-in frequency in Conditional Access session controls
- Enable persistent browser session for web client users
- Adjust token lifetime policies
- See [[../../Identity/mfa-for-avd|MFA for AVD]] common issues

## Additional Resources

**Microsoft Documentation:**
- [Entra ID Documentation](https://learn.microsoft.com/entra/identity/)
- [Conditional Access Overview](https://learn.microsoft.com/entra/identity/conditional-access/)
- [Azure RBAC Documentation](https://learn.microsoft.com/azure/role-based-access-control/)
- [AVD Authentication Documentation](https://learn.microsoft.com/azure/virtual-desktop/authentication)

**Related Sections:**
- [[../../Security/|Security]] - Defender for Endpoint, security baselines, monitoring
- [[../../Networking/|Networking]] - VPN, VNets, private endpoints for secure connectivity
- [[../../Intune/|Intune]] - Device management and compliance policies
