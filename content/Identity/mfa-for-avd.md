---
title: MFA for AVD
description: Configure Multi-Factor Authentication for Azure Virtual Desktop
published: true
date: 2025-12-14T04:52:56.963Z
tags: 
editor: markdown
dateCreated: 2025-12-14T04:44:03.169Z
---

# MFA for AVD

## What It Is
Multi-Factor Authentication adds a second verification step beyond passwords for AVD access.

## When to Use It
- All production AVD environments (security best practice)
- Compliance requirements (HIPAA, SOC 2, etc.)
- Access from untrusted networks
- Privileged user accounts

## MFA Methods Supported
- Microsoft Authenticator app (recommended)
- FIDO2 security keys
- Windows Hello for Business
- SMS/Voice (not recommended for high security)
- Hardware OATH tokens

## How to Set It Up

### Option 1: Per-User MFA (Simple)
**Portal:** Entra Admin Center → Users → Per-user MFA

1. Select users
2. Enable MFA
3. Users register at next sign-in

### Option 2: Conditional Access MFA (Recommended)
**Portal:** Entra Admin Center → Protection → Conditional Access

1. Create new policy
2. Name: "AVD - Require MFA"
3. Users: All users (exclude break-glass accounts)
4. Target resources:
   - Azure Virtual Desktop (app ID: 9cdead84-a844-4324-93f2-b2e6bb768d07)
   - Microsoft Remote Desktop (app ID: a4a365df-50f1-4397-bc59-1a1564b8bb9c)
5. Grant: Require multifactor authentication
6. Enable policy

### Enable Passwordless (Optional)
**Portal:** Entra Admin Center → Protection → Authentication methods

1. Enable Microsoft Authenticator
2. Configure for passwordless
3. Target AVD user groups

## Best Practices
- Use Conditional Access over per-user MFA
- Deploy Authenticator app via Intune
- Configure number matching to prevent MFA fatigue
- Create break-glass accounts excluded from MFA
- Consider location-based policies (trusted office IPs)

## Common Issues

### Issue: MFA prompt every connection
**Symptom:** Users prompted for MFA repeatedly
**Cause:** No persistent session configured
**Fix:** Configure sign-in frequency in Conditional Access session controls

### Issue: MFA registration not completing
**Symptom:** Users stuck in registration loop
**Cause:** Combined registration not enabled
**Fix:** Enable combined security information registration in Entra ID

### Issue: Authenticator notifications not received
**Symptom:** Push notifications don't arrive
**Cause:** Battery optimization blocking app
**Fix:** Exclude Authenticator from battery optimization on mobile device

## Related
- [[Conditional Access]]
- [[SSO Integration]]
