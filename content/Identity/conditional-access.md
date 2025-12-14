---
title: Conditional Access
description: Configure Conditional Access policies for Azure Virtual Desktop
published: true
date: 2025-12-14T04:52:52.634Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:44.750Z
---

# Conditional Access

## What It Is
Conditional Access is Azure's policy-based access control engine that evaluates signals (user, location, device, application) to make automated access decisions. It acts as the gatekeeper that enforces security policies before granting access to resources like Azure Virtual Desktop.

Policies follow an if-then model: **if** a user wants to access a resource, **then** they must complete specific requirements (MFA, compliant device, approved location, etc.).

## When to Use It
- **Require MFA for AVD connections** - Add second factor authentication for remote desktop access
- **Block access from untrusted locations** - Prevent connections from geographic regions or IP ranges outside your organization
- **Require compliant devices** - Ensure only Intune-managed, policy-compliant devices can connect
- **Enforce session controls** - Limit sign-in frequency or implement persistent sessions for specific user groups
- **Step-up authentication** - Apply stricter controls for sensitive applications within AVD
- **Risk-based access** - Block or require additional verification based on Azure AD Identity Protection risk scores

## How to Set It Up

### Prerequisites
- **Entra ID P1 or P2 license** - Conditional Access requires premium licensing
- **Global Administrator or Conditional Access Administrator role** - Required to create/modify policies
- **Security Reader role (minimum)** - For viewing and testing policies in report-only mode

### Create AVD Conditional Access Policy
**Portal:** Entra Admin Center → Protection → Conditional Access → Policies

1. **Create new policy**
   - Click "+ New policy"
   - Name: "AVD - Require MFA and Compliant Device"

2. **Assignments - Users**
   - Include: Select specific groups (e.g., "SG-AVD-Users")
   - Exclude: Emergency access accounts (break-glass accounts)

3. **Assignments - Target resources**
   - Select what this policy applies to: "Cloud apps"
   - Include: Select apps
   - Search for and select:
     - **Azure Virtual Desktop** (app ID: 9cdead84-a844-4324-93f2-b2e6bb768d07)
     - **Microsoft Remote Desktop** (app ID: a4a365df-50f1-4397-bc59-1a1564b8bb9c)

   > **Note:** You must target BOTH applications to cover all AVD connection methods (web client, desktop client, mobile clients)

4. **Conditions - Locations** (optional)
   - Configure: Yes
   - Include: Any location
   - Exclude: Selected locations (configure named locations first)
   - To create named locations: Entra Admin Center → Protection → Conditional Access → Named locations

5. **Conditions - Device platforms** (optional)
   - Configure: Yes
   - Include: Select device platforms
   - Select: Windows (primary AVD client platform)

6. **Conditions - Client apps**
   - Configure: Yes
   - Select: Browser, Mobile apps and desktop clients

7. **Access controls - Grant**
   - Select: "Grant access"
   - Check: "Require multifactor authentication"
   - Check: "Require device to be marked as compliant"
   - For multiple controls: "Require all the selected controls"

8. **Session controls** (optional)
   - Sign-in frequency: Configure based on security requirements
     - Persistent browser session: On/Off based on user experience vs. security needs

9. **Enable policy**
   - Start with: "Report-only" (test mode)
   - Review sign-in logs for 1-2 weeks
   - Switch to: "On" after validation

10. **Create**

### Testing Conditional Access Policies

**Portal:** Entra Admin Center → Protection → Conditional Access → What If

1. Select user to test
2. Select cloud app (Azure Virtual Desktop)
3. Configure conditions (location, device platform)
4. Click "What If"
5. Review which policies would apply

## Best Practices

- **Start with report-only mode** - Always test policies in report-only mode first, review sign-in logs for at least a week to identify potential issues before enforcement

- **Create separate policies for different user populations** - Don't create one massive policy; split by role (admins vs. standard users), location (on-premises vs. remote), or sensitivity level

- **Use named locations for office IPs** - Define trusted IP ranges as named locations, then exclude them from MFA requirements for office-based connections

- **Implement break-glass accounts** - Create 2-3 emergency access accounts excluded from ALL Conditional Access policies, store credentials in a physical safe

- **Layer policies strategically** - Apply broader baseline policies to all users (e.g., require MFA), then add specific policies for high-risk scenarios

- **Use groups for assignments** - Assign policies to dynamic or assigned Entra ID groups, never to individual users (except break-glass exclusions)

- **Document policy intent** - Use descriptive policy names that explain what they do (e.g., "AVD-Require-MFA-External-Users" not "Policy-001")

- **Monitor sign-in logs regularly** - Review Entra ID → Sign-in logs → Conditional Access tab weekly to catch blocked users or policy conflicts

- **Consider device compliance requirements carefully** - Requiring compliant devices is strong security but can block BYOD scenarios; create separate policies for personal vs. corporate devices if needed

- **Test with real users before rollout** - Use pilot groups to validate policies with actual users before organization-wide deployment

## Common Issues

### Issue: Users blocked unexpectedly
**Symptom:** Users receive "Access blocked" errors when connecting to AVD, sign-in logs show Conditional Access failures

**Cause:**
- Policy too restrictive (e.g., requiring compliant device for unmanaged BYOD users)
- Policy misconfigured (e.g., wrong app selected, missing exclusions)
- Multiple conflicting policies applying simultaneously

**Fix:**
1. Check sign-in logs: Entra Admin Center → Monitoring → Sign-in logs
2. Filter by user, look for "Failure" status
3. Click failed sign-in → Conditional Access tab → Review which policy blocked access
4. Options:
   - Temporarily switch policy to report-only mode
   - Add user to excluded group while investigating
   - Adjust policy conditions or grant controls
   - Use "What If" tool to simulate user's scenario

### Issue: MFA prompt every session
**Symptom:** Users prompted for MFA on every AVD connection, even minutes apart

**Cause:**
- Sign-in frequency set too aggressively (e.g., 1 hour)
- Persistent browser session disabled
- Token lifetime policies conflicting

**Fix:**
1. Review policy session controls
2. Adjust sign-in frequency to reasonable interval (4-8 hours for remote workers, 24 hours for office workers)
3. Enable persistent browser session if using web client
4. Check token lifetime policies: Entra Admin Center → Protection → Conditional Access → Session

### Issue: Policy not applying to AVD connections
**Symptom:** Users bypass MFA requirement when connecting to AVD

**Cause:**
- Wrong cloud app selected (missing "Microsoft Remote Desktop" app)
- Client app types not configured correctly
- Policy disabled or still in report-only mode

**Fix:**
1. Edit policy → Target resources
2. Ensure BOTH apps selected:
   - Azure Virtual Desktop (9cdead84-a844-4324-93f2-b2e6bb768d07)
   - Microsoft Remote Desktop (a4a365df-50f1-4397-bc59-1a1564b8bb9c)
3. Verify client apps include "Mobile apps and desktop clients"
4. Check policy is "On" not "Report-only"

### Issue: Compliant device check fails for Entra Joined machines
**Symptom:** Entra Joined, Intune-managed devices marked as non-compliant

**Cause:**
- Device compliance policy evaluation lag
- Device not synced to Entra ID recently
- Intune compliance policy misconfigured

**Fix:**
1. On affected device: Settings → Accounts → Access work or school → Info → Sync
2. Force Intune sync: Company Portal → Settings → Sync
3. Wait 5-10 minutes for compliance evaluation
4. Verify device compliance: Intune Admin Center → Devices → Windows → [Device] → Device compliance
5. Check compliance policy: Intune Admin Center → Devices → Compliance policies

### Issue: Break-glass account also blocked
**Symptom:** Emergency access account cannot sign in during outage

**Cause:** Break-glass account not properly excluded from Conditional Access policies

**Fix:**
1. This is a critical issue - use Azure CLI or PowerShell with saved credentials to remediate
2. Create dedicated Entra ID group: "Exclude-ConditionalAccess-BreakGlass"
3. Add emergency accounts to this group
4. Edit ALL Conditional Access policies → Assignments → Users → Exclude → Select break-glass group
5. Document break-glass account credentials offline (physical secure location)

## Related Resources
- [[entra-id-fundamentals]] - Understanding the identity foundation
- [[prerequisites|Intune Prerequisites]] - Setting up device compliance
- [[defender|AVD Security]] - Broader AVD security architecture
