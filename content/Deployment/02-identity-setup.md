---
title: Step 02 - Identity Setup
description: Configure Entra ID users, groups, and security policies for AVD
published: true
date: 2025-12-14T00:00:00.000Z
tags:
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 02: Identity Setup

Configure Entra ID users, groups, Conditional Access policies, and SSO for Azure Virtual Desktop access. This step establishes the identity foundation for secure, automated user and device management.

## Example Scenario

This guide uses a 200-user organization with four departments:

| Department | User Count | Desktop Type | Example Users |
|------------|-----------|--------------|---------------|
| General | 100 | Pooled | general001-general100 |
| Finance | 50 | Pooled | finance001-finance050 |
| Creative | 30 | Personal | creative001-creative030 |
| Executives | 20 | Personal | exec001-exec020 |

**Grouping Strategy:**
- **AVD-Pooled-Users:** General + Finance (150 users) → Multi-session desktops
- **AVD-Personal-Users:** Creative + Executives (50 users) → Dedicated desktops

## Prerequisites

- [ ] Entra ID P1 or P2 licenses assigned (required for dynamic groups and Conditional Access)
- [ ] Global Administrator or User Administrator role
- [ ] Naming conventions documented (see [[prerequisites-planning]])

> **Note:** This step requires Entra ID P1 minimum. Verify licensing at **Entra Admin Center → Billing → Licenses**.

---

## Part 1: Create Test Users

Create sample users representing each department for testing and validation.

### Create Users via Portal

**Portal:** Entra Admin Center → Users → All users → New user

For each department, create representative users:

#### General Department Users (10 test users)

1. Click **+ New user → Create new user**
2. Fill in user details:
   - **User principal name:** `general001@yourdomain.com`
   - **Display name:** `General User 001`
   - **First name:** `General`
   - **Last name:** `User 001`
   - **Department:** `General`
   - **Job title:** `Knowledge Worker`
   - **Usage location:** `United States`
3. Set password:
   - **Password:** Generate automatically or set custom password
   - **Require password change:** ✓ (first sign-in)
4. Assignments:
   - **Licenses:** Assign Microsoft 365 E3 or E5
5. Click **Review + create**

Repeat for `general002` through `general010`.

#### Finance Department Users (5 test users)

Same process as General, but with:
- **User principal name:** `finance001@yourdomain.com` through `finance005@yourdomain.com`
- **Display name:** `Finance User 001` through `Finance User 005`
- **Department:** `Finance`
- **Job title:** `Financial Analyst`

#### Creative Department Users (5 test users)

- **User principal name:** `creative001@yourdomain.com` through `creative005@yourdomain.com`
- **Display name:** `Creative User 001` through `Creative User 005`
- **Department:** `Creative`
- **Job title:** `Designer`

#### Executive Users (3 test users)

- **User principal name:** `exec001@yourdomain.com` through `exec003@yourdomain.com`
- **Display name:** `Executive User 001` through `Executive User 003`
- **Department:** `Executives`
- **Job title:** `Executive`

### Bulk User Creation (Optional)

For creating 100+ users, use bulk import:

**Portal:** Entra Admin Center → Users → Bulk operations → Bulk create

1. Download CSV template
2. Fill template with user data (see example below)
3. Upload completed CSV
4. Review and create

**Example CSV format:**
```csv
Name [displayName] Required,User name [userPrincipalName] Required,Initial password [passwordProfile] Required,Department,Job title,Usage location Required
General User 001,general001@yourdomain.com,TempPass123!,General,Knowledge Worker,US
General User 002,general002@yourdomain.com,TempPass123!,General,Knowledge Worker,US
Finance User 001,finance001@yourdomain.com,TempPass123!,Finance,Financial Analyst,US
Creative User 001,creative001@yourdomain.com,TempPass123!,Creative,Designer,US
Executive User 001,exec001@yourdomain.com,TempPass123!,Executives,Executive,US
```

> **Warning:** Users created with initial passwords must change password on first sign-in. Communicate temporary credentials securely via password-protected email or separate channel.

---

## Part 2: Create Dynamic User Groups

Dynamic groups automatically assign users based on department attributes. This eliminates manual group management as users change departments.

### AVD-Pooled-Users

Users who access shared multi-session desktops (General + Finance departments).

**Portal:** Entra Admin Center → Groups → New group

1. **Group type:** Security
2. **Group name:** `AVD-Pooled-Users`
3. **Group description:** `Auto-assigned users for pooled AVD desktops (General and Finance departments)`
4. **Membership type:** Dynamic User
5. Click **Add dynamic query**
6. **Rule syntax:**
   ```
   (user.department -eq "General") -or (user.department -eq "Finance")
   ```
7. Click **Validate Rules** (optional but recommended)
   - Click **Add users** and search for a test user like `general001`
   - Verify the user appears in results
8. Click **Save**
9. Click **Create**

**Expected membership:** 150 users (100 General + 50 Finance) in production. Test group should show 15 users (10 General + 5 Finance).

> **Note:** Dynamic group processing takes 5-15 minutes for small groups. Check membership after 15 minutes.

### AVD-Personal-Users

Users who need dedicated personal desktops (Creative + Executives).

**Portal:** Entra Admin Center → Groups → New group

1. **Group type:** Security
2. **Group name:** `AVD-Personal-Users`
3. **Group description:** `Auto-assigned users for personal AVD desktops (Creative and Executives departments)`
4. **Membership type:** Dynamic User
5. Click **Add dynamic query**
6. **Rule syntax:**
   ```
   (user.department -eq "Creative") -or (user.department -eq "Executives")
   ```
7. Validate rules with `creative001` or `exec001`
8. Click **Save** and **Create**

**Expected membership:** 50 users (30 Creative + 20 Executives) in production. Test group should show 8 users (5 Creative + 3 Executives).

### AVD-Users-Admins

IT administrators who manage AVD infrastructure.

**Portal:** Entra Admin Center → Groups → New group

1. **Group type:** Security
2. **Group name:** `AVD-Users-Admins`
3. **Group description:** `IT administrators with AVD management access`
4. **Membership type:** Assigned (manual membership)
5. Click **No members selected**
6. Search for and add your admin account(s)
7. Click **Create**

> **Note:** Admin groups use assigned membership (not dynamic) to ensure tight control over privileged access.

---

## Part 3: Create Dynamic Device Groups

Dynamic device groups automatically organize session hosts based on naming conventions. These groups are used for Intune policies, Conditional Access, and automated management.

> **Important:** Device groups will remain empty until session hosts are deployed in Step 6. Create them now to establish the automation framework.

### AVD-Devices-Pooled

All session hosts in pooled (multi-session) host pools.

**Portal:** Entra Admin Center → Groups → New group

1. **Group type:** Security
2. **Group name:** `AVD-Devices-Pooled`
3. **Group description:** `Auto-assigned pooled AVD session hosts for Intune policies and compliance`
4. **Membership type:** Dynamic Device
5. Click **Add dynamic query**
6. **Rule syntax:**
   ```
   device.displayName -startsWith "avd-pool"
   ```
7. Click **Save** and **Create**

**Expected membership:** 0 now, 10+ after session host deployment. Matches naming convention: `avd-pool-prod1-0`, `avd-pool-prod1-1`, etc.

### AVD-Devices-Personal

All session hosts in personal (dedicated) host pools.

**Portal:** Entra Admin Center → Groups → New group

1. **Group type:** Security
2. **Group name:** `AVD-Devices-Personal`
3. **Group description:** `Auto-assigned personal AVD session hosts for Intune policies`
4. **Membership type:** Dynamic Device
5. Click **Add dynamic query**
6. **Rule syntax:**
   ```
   device.displayName -startsWith "avd-pers"
   ```
7. Click **Save** and **Create**

**Expected membership:** 0 now, 50+ after deployment. Matches naming: `avd-pers-prod1-0`, `avd-pers-prod1-1`, etc.

### AVD-Devices-All

All AVD session hosts (both pooled and personal) for organization-wide policies.

**Portal:** Entra Admin Center → Groups → New group

1. **Group type:** Security
2. **Group name:** `AVD-Devices-All`
3. **Group description:** `All AVD session hosts for organization-wide Intune baselines and monitoring`
4. **Membership type:** Dynamic Device
5. Click **Add dynamic query**
6. **Rule syntax:**
   ```
   (device.displayName -startsWith "avd-pool") -or (device.displayName -startsWith "avd-pers")
   ```
7. Click **Save** and **Create**

**Expected membership:** 0 now, 60+ after deployment (10 pooled + 50 personal).

> **Note:** Dynamic device groups populate automatically when session hosts join Entra ID during deployment. Processing time: 5-15 minutes per device.

**See:** [[dynamic-groups|Dynamic Groups Reference]] for advanced rules and troubleshooting.

---

## Part 4: Configure Conditional Access

Conditional Access enforces security policies for AVD connections. These policies require MFA and compliant devices before granting access.

> **Prerequisite:** Entra ID P1 or P2 license required for Conditional Access.

### Policy 1: AVD - Require MFA for All Users

Enforce multi-factor authentication for all AVD connections.

**Portal:** Entra Admin Center → Protection → Conditional Access → Policies → + New policy

1. **Name:** `AVD - Require MFA for All Users`
2. **Assignments → Users:**
   - **Include:** Select groups → Add:
     - `AVD-Pooled-Users`
     - `AVD-Personal-Users`
     - `AVD-Users-Admins`
   - **Exclude:** Select users → Add break-glass admin account (emergency access)
3. **Assignments → Target resources:**
   - **Select what this policy applies to:** Cloud apps
   - **Include:** Select apps
   - Search for and add:
     - `Azure Virtual Desktop` (App ID: 9cdead84-a844-4324-93f2-b2e6bb768d07)
     - `Microsoft Remote Desktop` (App ID: a4a365df-50f1-4397-bc59-1a1564b8bb9c)
4. **Assignments → Conditions:**
   - **Locations:** Not configured
   - **Client apps:** Configure = Yes
     - Select: `Browser`, `Mobile apps and desktop clients`
5. **Access controls → Grant:**
   - Select: `Grant access`
   - Check: `Require multifactor authentication`
   - **For multiple controls:** Require all the selected controls
6. **Session:** Not configured
7. **Enable policy:** Report-only (start in test mode)
8. Click **Create**

> **Important:** Start with "Report-only" mode. Monitor sign-in logs for 3-7 days before switching to "On" to avoid blocking users unexpectedly.

### Policy 2: AVD - Require Compliant Device for Pooled Users

Ensure only Intune-managed, compliant devices can access pooled desktops.

**Portal:** Entra Admin Center → Protection → Conditional Access → Policies → + New policy

1. **Name:** `AVD - Require Compliant Device for Pooled Users`
2. **Assignments → Users:**
   - **Include:** Select groups → `AVD-Pooled-Users`
   - **Exclude:** Break-glass admin
3. **Assignments → Target resources:**
   - Include: `Azure Virtual Desktop` and `Microsoft Remote Desktop`
4. **Assignments → Conditions:**
   - **Client apps:** Browser, Mobile apps and desktop clients
5. **Access controls → Grant:**
   - Select: `Grant access`
   - Check: `Require device to be marked as compliant`
   - Check: `Require multifactor authentication`
   - **For multiple controls:** Require all the selected controls
6. **Enable policy:** Report-only
7. Click **Create**

> **Note:** This policy requires Intune device compliance policies to be configured first. The compliance check will fail until session hosts are enrolled in Intune (covered in Step 8).

### Policy 3: AVD - Block Access from Untrusted Locations (Optional)

Restrict AVD access to known safe locations (office IPs, VPN ranges).

**Create Named Locations First:**

**Portal:** Entra Admin Center → Protection → Conditional Access → Named locations → + Countries location

1. Click **+ IP ranges location**
2. **Name:** `Corporate Office Networks`
3. Add IP ranges:
   - Office 1: `203.0.113.0/24`
   - Office 2: `198.51.100.0/24`
   - VPN Range: `10.20.0.0/16`
4. Click **Create**

**Create Conditional Access Policy:**

**Portal:** Entra Admin Center → Protection → Conditional Access → Policies → + New policy

1. **Name:** `AVD - Block Access from Untrusted Locations`
2. **Assignments → Users:**
   - **Include:** Select groups → `AVD-Pooled-Users`, `AVD-Personal-Users`
   - **Exclude:** Break-glass admin
3. **Assignments → Target resources:**
   - Include: `Azure Virtual Desktop` and `Microsoft Remote Desktop`
4. **Assignments → Conditions → Locations:**
   - **Configure:** Yes
   - **Include:** Any location
   - **Exclude:** Selected locations → `Corporate Office Networks`
5. **Access controls → Grant:**
   - Select: `Block access`
6. **Enable policy:** Report-only (test carefully before enabling)
7. Click **Create**

> **Warning:** Location-based blocking can lock out remote workers. Test thoroughly in report-only mode and verify VPN ranges are included in named locations before enabling.

### Test Conditional Access Policies

**Portal:** Entra Admin Center → Protection → Conditional Access → What If

1. **User:** Select test user (e.g., `general001`)
2. **Cloud apps:** `Azure Virtual Desktop`
3. **IP address:** Enter test IP (optional)
4. **Device platform:** Windows
5. **Client apps:** Mobile apps and desktop clients
6. Click **What If**
7. Review **Policies that will apply** section

Expected result: MFA policy applies, shows "Report-only" status.

### Monitor Report-Only Policies

**Portal:** Entra Admin Center → Monitoring → Sign-in logs

1. Filter by:
   - **Application:** Azure Virtual Desktop
   - **Date range:** Last 7 days
2. Click on a sign-in event
3. Navigate to **Conditional Access** tab
4. Review which policies applied and their results

**After 3-7 days of monitoring:**
- If no unexpected failures, switch policies from "Report-only" to "On"
- Edit each policy → Enable policy → Change to "On" → Save

**See:** [[conditional-access|Conditional Access Reference]] for advanced scenarios and troubleshooting.

---

## Part 5: Enable SSO

Single Sign-On eliminates repeated credential prompts when users connect to AVD.

### Configure SSO on Host Pool (Placeholder)

> **Note:** SSO configuration requires the host pool to be created first. This section is a placeholder - actual SSO enablement occurs in Step 4 (Host Pool Deployment).

**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → [Host Pool Name] → Properties

1. Navigate to **RDP Properties** tab
2. Under **Connection information**, find **Microsoft Entra single sign-on**
3. Set to: `Connections will use Microsoft Entra authentication for SSO`
4. Click **Save**

**Additional RDP properties for Entra Joined hosts:**
```
targetisaadjoined:i:1
```

### Enable Seamless SSO for Hybrid Environments (Optional)

If using Hybrid Entra Joined session hosts with on-premises AD:

**Portal:** Entra Admin Center → Hybrid management → Microsoft Entra Connect → Azure AD Connect

1. Launch Entra Connect on your sync server
2. Select **Change user sign-in**
3. Check **Enable single sign-on**
4. Enter on-premises domain admin credentials
5. Complete wizard

**Verification:**
```powershell
# On domain-joined client
whoami /upn
# Should show: user@yourdomain.com

# Test SSO
Get-AzureADTenantDetail
# Should authenticate without prompting for credentials
```

> **Decision Point:**
> - **Cloud-only (Entra Joined hosts):** SSO works automatically with Entra authentication. No additional configuration needed.
> - **Hybrid (Hybrid Entra Joined hosts):** Follow Seamless SSO setup above to enable on-prem AD integration.

**See:** [[sso-integration|SSO Integration Reference]] for detailed configuration and troubleshooting.

---

## Verification Checklist

Run these checks to confirm identity setup is complete:

### User Groups

**Portal:** Entra Admin Center → Groups

- [ ] `AVD-Pooled-Users` exists with 150 members (test: 15 users)
- [ ] `AVD-Personal-Users` exists with 50 members (test: 8 users)
- [ ] `AVD-Users-Admins` exists with admin members
- [ ] All dynamic groups show **Rule processing status: Succeeded**

**Verify dynamic group processing:**
1. Open `AVD-Pooled-Users`
2. Expand **Dynamic membership rules**
3. Check **Rule processing status** = `Succeeded`
4. Click **Members** → Verify test users appear (wait 15 minutes if empty)

### Device Groups

**Portal:** Entra Admin Center → Groups

- [ ] `AVD-Devices-Pooled` created (0 members expected at this step)
- [ ] `AVD-Devices-Personal` created (0 members expected)
- [ ] `AVD-Devices-All` created (0 members expected)
- [ ] All dynamic device rules validated

> **Note:** Device groups will populate automatically when session hosts are deployed in Step 6.

### Conditional Access Policies

**Portal:** Entra Admin Center → Protection → Conditional Access → Policies

- [ ] `AVD - Require MFA for All Users` created in Report-only mode
- [ ] `AVD - Require Compliant Device for Pooled Users` created in Report-only mode
- [ ] Optional: Location-based blocking policy created (if applicable)
- [ ] All policies tested with "What If" tool
- [ ] Break-glass admin excluded from all policies

**Test with What If:**
1. Go to **What If** tool
2. User: `general001`
3. Cloud apps: `Azure Virtual Desktop`
4. Verify policies apply correctly

### SSO Configuration

- [ ] SSO documentation reviewed
- [ ] SSO will be enabled after host pool creation (Step 4)
- [ ] For hybrid: Seamless SSO configured on Entra Connect (if applicable)

### Test User Access

Create a test sign-in to validate MFA:

1. Open browser in incognito/private mode
2. Navigate to: `https://client.wvd.microsoft.com/arm/webclient`
3. Sign in as `general001@yourdomain.com`
4. Verify MFA prompt appears (if policy enabled)
5. Expected result: No desktops shown yet (normal - host pools not created)

---

## Troubleshooting

### Issue: Dynamic groups are empty after 30 minutes

**Symptom:** AVD-Pooled-Users shows 0 members

**Cause:**
- Department attribute not set on users
- Rule syntax error
- Entra ID P1 license not assigned

**Fix:**
1. Check user attributes: Entra Admin Center → Users → Select `general001` → Properties → Department field
2. Verify department = `General` (case-sensitive)
3. Check group rule processing: Groups → AVD-Pooled-Users → Dynamic membership rules → Rule processing status
4. If status = `Failed`, validate rule syntax
5. Confirm Entra ID P1 license: Billing → Licenses

### Issue: Conditional Access policy not applying

**Symptom:** Users bypass MFA when connecting to AVD

**Cause:**
- Policy still in Report-only mode
- Wrong cloud app selected
- User not in assigned group

**Fix:**
1. Verify policy is "On" not "Report-only"
2. Check Target resources includes both:
   - Azure Virtual Desktop (9cdead84-a844-4324-93f2-b2e6bb768d07)
   - Microsoft Remote Desktop (a4a365df-50f1-4397-bc59-1a1564b8bb9c)
3. Verify user is member of AVD-Pooled-Users or AVD-Personal-Users
4. Test with "What If" tool

### Issue: MFA prompts on every connection

**Symptom:** Users must complete MFA every time they open AVD, even minutes apart

**Cause:**
- Sign-in frequency set too low
- Persistent browser session disabled

**Fix:**
1. Edit Conditional Access policy → Session controls
2. Sign-in frequency: Set to 8 hours (or 24 hours for low-risk users)
3. Persistent browser session: Enable
4. Save policy

### Issue: "You don't have permission" when creating groups

**Symptom:** Error creating dynamic groups in Entra portal

**Cause:**
- Insufficient permissions
- Entra ID P1 license not assigned to tenant

**Fix:**
1. Verify role: User must have Global Administrator or Groups Administrator role
2. Check licensing: Entra Admin Center → Billing → Licenses → Verify Entra ID P1 or P2

---

## Next Steps

**Identity setup is complete.** Dynamic groups will automatically populate as users are created and session hosts are deployed.

**Next:** [[03-networking-setup|Step 03: Networking Setup]]

---

## Related Reference Pages

- [[entra-id-fundamentals|Entra ID Fundamentals]] - Understanding tenants, licensing, and identity types
- [[dynamic-groups|Dynamic Groups]] - Advanced rules, troubleshooting, processing time
- [[conditional-access|Conditional Access]] - Policy design patterns and best practices
- [[sso-integration|SSO Integration]] - Seamless SSO for cloud and hybrid scenarios
- [[mfa-for-avd|MFA for AVD]] - Multi-factor authentication configuration
- [[rbac-for-avd|RBAC for AVD]] - Role assignments for AVD access (configured in Step 4)
