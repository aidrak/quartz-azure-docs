---
title: SSO Integration
description: Configure Single Sign-On for Azure Virtual Desktop
published: true
date: 2025-12-14T04:53:00.060Z
tags: 
editor: markdown
dateCreated: 2025-12-14T04:44:06.466Z
---

# SSO Integration

## What It Is
Single Sign-On (SSO) allows users to authenticate once and access AVD without additional password prompts. When you enable single sign-on using Microsoft Entra ID, users authenticate to Windows using an Entra ID token, enabling seamless sign-in without additional password prompts. This also enables passwordless authentication (FIDO2, passkeys) and third-party identity providers that federate with Entra ID.

## When to Use It
- Seamless user experience (no double authentication)
- Passwordless authentication scenarios (FIDO2, passkeys)
- External identities (required for external user access)
- Consistent Entra ID-based authentication across AVD and on-premises resources

**Prerequisite:** Session hosts must be Entra ID joined or Entra ID hybrid joined. Session hosts joined only to on-premises AD or Azure AD DS are not supported.

For comprehensive details, see [Configure single sign-on for Azure Virtual Desktop](https://learn.microsoft.com/en-us/azure/virtual-desktop/configure-single-sign-on).

## How to Set It Up

### Prerequisites

Before enabling SSO, ensure:

- **Session hosts:** Windows 11 Enterprise (KB5018418+), Windows 10 Enterprise (KB5018410+), or Windows Server 2022 (KB5018421+)
- **Join type:** Entra ID joined or Entra ID hybrid joined (not AD DS or Azure AD DS only)
- **RDP client:** Windows App or Remote Desktop client (Windows 10+, macOS 10.8.2+, iOS 10.5.1+, Android 10.0.16+)
- **Microsoft Entra role:** Application Administrator or Cloud Application Administrator to configure tenant-wide settings

### 1. Enable Microsoft Entra Authentication for RDP (Tenant-Wide)

You must enable RDP authentication for the Windows Cloud Login service principal tenant-wide. This step must be completed before host pool configuration.

**CLI:** Use Microsoft Graph PowerShell

```powershell
# Import modules
Import-Module Microsoft.Graph.Authentication
Import-Module Microsoft.Graph.Applications

# Connect to Microsoft Graph with required scopes
Connect-MgGraph -Scopes "Application.Read.All","Application-RemoteDesktopConfig.ReadWrite.All"

# Get the Windows Cloud Login service principal ID
$WCLspId = (Get-MgServicePrincipal -Filter "AppId eq '270efc09-cd0d-444b-a71f-39af4910ec45'").Id

# Enable RDP authentication
if ((Get-MgServicePrincipalRemoteDesktopSecurityConfiguration -ServicePrincipalId $WCLspId) -ne $true) {
    Update-MgServicePrincipalRemoteDesktopSecurityConfiguration -ServicePrincipalId $WCLspId -IsRemoteDesktopProtocolEnabled
}

# Verify RDP is enabled
Get-MgServicePrincipalRemoteDesktopSecurityConfiguration -ServicePrincipalId $WCLspId
```

Expected output: `IsRemoteDesktopProtocolEnabled: True`

### 2. Hide Consent Prompt (Optional but Recommended)

By default, users see a consent dialog on first connection. Hide this by adding session hosts to a trusted device group.

**Step A:** Create a dynamic group for session hosts

**Portal:** Entra Admin Center → Groups → New group

1. **Group type:** Security
2. **Group name:** `avd-sessionhosts-trusteddevices`
3. **Membership type:** Dynamic Device
4. **Add dynamic query:** `(device.displayName -startsWith "sh-pool") -or (device.displayName -startsWith "sh-pers")`
5. Click **Create**

> **Note:** Adjust the display name pattern to match your session host naming convention. Wait 5-15 minutes for group to populate.

**Step B:** Add group to Windows Cloud Login service principal

```powershell
# Get the session host group object ID
$tdg = New-Object -TypeName Microsoft.Graph.PowerShell.Models.MicrosoftGraphTargetDeviceGroup
$tdg.Id = (Get-MgGroup -Filter "displayName eq 'avd-sessionhosts-trusteddevices'").Id
$tdg.DisplayName = "avd-sessionhosts-trusteddevices"

# Add the group to the service principal
New-MgServicePrincipalRemoteDesktopSecurityConfigurationTargetDeviceGroup -ServicePrincipalId $WCLspId -BodyParameter $tdg
```

### 3. Create Kerberos Server Object (If Applicable)

**Required if:**
- Session hosts are Entra ID hybrid joined, OR
- Session hosts are Entra ID joined AND your environment has on-premises AD domain controllers (for accessing on-premises resources)

**Not required:** Cloud-only Entra ID joined hosts with no on-premises AD access.

See [Enable passwordless security key sign-in to on-premises resources](https://learn.microsoft.com/en-us/azure/virtual-desktop/configure-single-sign-on#create-a-kerberos-server-object) for detailed Kerberos server creation steps.

### 4. Review Conditional Access Policies

When SSO is enabled, a new Entra ID app authenticates users. Review existing Conditional Access policies:

**Portal:** Entra Admin Center → Protection → Conditional Access → Policies

- Ensure your Conditional Access policies don't block Windows Cloud Login authentication
- If you require MFA, test with target users to ensure acceptable experience
- Consider using Conditional Access to require device compliance or location-based policies

See [[Conditional Access]] for details on policy configuration.

### 5. Enable SSO on Host Pool

**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → `hp-pooled` or `hp-personal` → Properties

1. Navigate to **RDP Properties** tab
2. Under "Microsoft Entra single sign-on", select **Connections will use Microsoft Entra authentication for SSO**
3. Click **Save**

**Alternative (PowerShell):**

```powershell
# Set enablerdsaadauth property to enable SSO
Update-AzWvdHostPool -ResourceGroupName "RG-Azure-VDI-01" `
  -HostPoolName "hp-pooled" `
  -CustomRdpProperty "enablerdsaadauth:i:1"
```

### 6. Configure Session Lock Behavior (Optional)

**Default behavior:** When session locks (user lock or policy), the session disconnects and users see a reconnect dialog. This supports passwordless authentication.

**Alternative behavior:** Show remote lock screen instead of disconnecting. Configure if needed:

**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → `hp-pooled` → Properties → RDP Properties

Add custom RDP property: `rdpcam:s:TRUE` (to maintain session lock)

### 7. Assign Users to AVD Application

**Portal:** Entra Admin Center → Enterprise applications → Azure Virtual Desktop

1. Go to **Properties**
2. Set **Assignment required** to **Yes** (restricts access to assigned users only)
3. Go to **Users and groups**
4. Add the user groups: `avd-users-pooled`, `avd-users-personal`
5. Verify Conditional Access policies don't block SSO

### 8. Deploy Updated RDP Client

**RDP Client Requirements:**
- Windows: Remote Desktop client (built-in) or Windows App
- macOS: Remote Desktop client version 10.8.2+
- iOS: Remote Desktop app version 10.5.1+
- Android: Remote Desktop app version 10.0.16+

Deploy via Intune or direct installation. Users with older clients will fall back to standard password authentication.

## Best Practices

- **Tenant-wide enablement first:** Always run the RDP authentication enablement script before configuring host pools
- **Test with pilot group:** Create a test host pool and pilot user group to verify SSO works before production rollout
- **Use trusted device groups:** Hide consent dialogs by configuring dynamic device groups for session hosts
- **Review Conditional Access:** Test MFA policies with real users to ensure acceptable experience
- **Kerberos for hybrid environments:** Create Kerberos server objects if using hybrid-joined hosts or accessing on-premises resources
- **Deploy latest RDP clients:** Ensure all users have updated Remote Desktop clients (Windows App preferred for best experience)
- **Monitor session locks:** Test session lock behavior with users; default disconnection supports passwordless authentication but may surprise users
- **External identities:** If supporting external users, SSO must be enabled and Kerberos configured for on-premises resource access

## Common Issues

### Issue: "RDP authentication is not enabled" error
**Symptom:** Connection fails with error during SSO attempt
**Cause:** Tenant-wide RDP authentication not enabled on Windows Cloud Login service principal
**Fix:** Run Step 1 (Enable Microsoft Entra Authentication for RDP) PowerShell script to enable `isRemoteDesktopProtocolEnabled`

### Issue: Consent dialog appears every connection
**Symptom:** Users prompted to approve connection on each login
**Cause:** Session hosts not added to trusted device group
**Fix:** Create dynamic device group (Step 2) and add to Windows Cloud Login service principal

### Issue: "The specific session doesn't exist" error (Hybrid-joined hosts)
**Symptom:** Error on connection attempt for hybrid-joined session hosts
**Cause:** Kerberos server object not created
**Fix:** Create Kerberos server object following [Microsoft documentation](https://learn.microsoft.com/en-us/azure/virtual-desktop/configure-single-sign-on#create-a-kerberos-server-object)

### Issue: SSO not working, still prompted for credentials
**Symptom:** Connection succeeds but users see credential prompt inside session
**Cause:**
- Older RDP client version
- Session hosts not Entra ID joined (still on AD DS only)
- SSO not enabled on host pool (enablerdsaadauth property missing)
**Fix:**
1. Verify session hosts are Entra ID joined: `dsregcmd /status`
2. Update RDP client to latest version
3. Verify host pool has "Connections will use Microsoft Entra authentication for SSO" enabled
4. Check Conditional Access policies not blocking Windows Cloud Login app

### Issue: Session disconnects when locked
**Symptom:** Session disconnects when user locks computer or timeout policy engages
**Cause:** Default behavior (supports passwordless authentication)
**Fix:** If remote lock screen is preferred, add `rdpcam:s:TRUE` to custom RDP properties (Step 6)

## References

- [Configure single sign-on for Azure Virtual Desktop](https://learn.microsoft.com/en-us/azure/virtual-desktop/configure-single-sign-on) - Official Microsoft documentation
- [Enable passwordless security key sign-in to on-premises resources](https://learn.microsoft.com/en-us/azure/virtual-desktop/configure-single-sign-on#create-a-kerberos-server-object) - Kerberos server object creation
- [[Conditional Access]] - Policy configuration for SSO scenarios
- [[Entra ID Fundamentals]] - Identity and authentication overview
