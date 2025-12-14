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
Single Sign-On (SSO) allows users to authenticate once and access AVD without additional password prompts.

## When to Use It
- Improve user experience (no double authentication)
- Reduce helpdesk password reset tickets
- Required for passwordless authentication scenarios

## Prerequisites
- Entra ID joined or Hybrid Entra ID joined session hosts
- Windows 10/11 multi-session or Windows 11 Enterprise single-session
- Entra ID P1 license (for Conditional Access integration)

## How to Set It Up

### Enable SSO on Host Pool
**Portal:** Azure Portal → Azure Virtual Desktop → Host pools → [Your Pool] → Properties

1. Navigate to RDP Properties tab
2. Under Connection information, find "Microsoft Entra single sign-on"
3. Set to "Connections will use Microsoft Entra authentication for SSO"
4. Save changes

### Configure Entra ID for SSO
**Portal:** Entra Admin Center → Applications → Enterprise applications

1. Search for "Azure Virtual Desktop" and "Microsoft Remote Desktop"
2. For each app:
   - Properties → Assignment required = Yes (optional, for restricted access)
   - Users and groups → Add your AVD users

### RDP Client Configuration
Users need Windows Remote Desktop client (MSRDC) version 1.2.3317 or later

## Best Practices
- Test with pilot group first
- Combine with Conditional Access for security
- Use Entra ID joined hosts for best SSO experience
- Deploy latest Remote Desktop client via Intune

## Common Issues

### Issue: SSO not working, still prompted for credentials
**Symptom:** Users see Windows credential prompt
**Cause:** Hybrid joined hosts or client version too old
**Fix:** Verify host join type, update RD client

### Issue: "Your session couldn't be connected" error
**Symptom:** Connection fails after authentication
**Cause:** Missing targetisaadjoined RDP property
**Fix:** Add "targetisaadjoined:i:1" to custom RDP properties

## Related
- [[Conditional Access]]
- [[Entra ID Fundamentals]]
