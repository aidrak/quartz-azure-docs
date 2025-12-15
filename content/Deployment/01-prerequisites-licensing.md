---
title: Step 01 - Prerequisites & Licensing
description: Verify subscriptions, licenses, and make key architectural decisions
published: true
date: 2025-12-14
tags:
editor: markdown
---

# Step 01: Prerequisites & Licensing

Verify Azure subscription access and licensing entitlements. AVD requires specific licenses for AVD access, Intune management, and Conditional Access policies.

## Prerequisites

- [ ] Active Azure subscription with **Owner** or **Contributor + User Access Administrator** role
- [ ] Global Administrator access to tenant
- [ ] Appropriate licensing assigned to users

---

## Part 1: Licensing Requirements (REQUIRED for ALL AVD users)

**All users in the AVD infrastructure need one license combination:**

- **Microsoft 365 E3** (includes Windows 11, Office, Entra ID Premium P1, Intune) - **RECOMMENDED**
- **OR Microsoft 365 E5** (adds Identity Protection, Privileged Identity Management)

**Why E3/E5 is mandatory:**
- **Windows 11:** AVD session host access (included in E3/E5)
- **Entra ID Premium P1:** Required for Conditional Access, Dynamic Groups, MFA (included in E3/E5)
- **Intune:** Device management and policy assignment. E3 also includes remediation scripts (included in E3/E5)

**200 users need M365 E3 or E5 licenses.** Verify at Microsoft 365 Admin Center → Users → [User] → Licenses

> **If any AVD user is missing E3/E5:** Conditional Access policies fail, Intune management is limited, AVD will not function properly.

---

## Part 2: Azure Subscription Setup

**Portal:** Azure Portal → Subscriptions

1. Confirm active subscription:
   - Subscription ID: `_______________________________`
   - Region: **Central US** (recommended)

2. Verify resource providers registered:
   - [ ] Microsoft.DesktopVirtualization
   - [ ] Microsoft.Compute
   - [ ] Microsoft.Network
   - [ ] Microsoft.Storage
   - [ ] Microsoft.Insights

   If not registered: Select each → **Register**

---

## Part 3: Make Architectural Decisions

### Decision: Identity Model (Cloud-Only vs Hybrid)

Choose your identity approach:

> **Decision Point:**
> - **Standard (Cloud-Only):** Entra Join only, no on-prem AD
>   - Use Azure Files (no on-prem shares)
>   - All apps cloud-native (Office 365, SaaS)
>   - **Continue below**
> - **Hybrid (Entra Join + on-prem):** If on-prem AD required
>   - See [[../Identity/entra-join-vs-hybrid-join|Entra Join vs Hybrid Join]] first

**This deployment:** Cloud-Only (Entra Join)

---

## Pre-Deployment Checklist

Before proceeding to Step 02:

**Licensing:**
- [ ] M365 E3 or E5 assigned to all 200 AVD users

**Subscription & Infrastructure:**
- [ ] Azure subscription active with Owner/Contributor role
- [ ] Resource providers registered (5 listed in Part 2)

**Architecture:**
- [ ] Identity model: **Cloud-Only (Entra Join)** selected
- [ ] Deployment scenario: **Pooled (150 users) + Personal (50 users)**
- [ ] Region: **Central US**

---

## Next Steps

**Licensing verified.** Ready for resource group and network setup.

**Next:** [[02-identity-setup|Step 02: Identity Setup]]

---

## Related Reference Pages

- [[../Identity/entra-id-fundamentals|Entra ID Fundamentals]] - Licensing tiers and requirements
- [[../Identity/entra-join-vs-hybrid-join|Entra Join vs Hybrid Join]] - Identity model decision matrix
- [[../Identity/sso-integration|SSO Integration]] - Single sign-on setup
