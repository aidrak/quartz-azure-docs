---
title: AVD Security Baseline
description: 
published: true
date: 2025-12-14T04:53:36.161Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:48.044Z
---

# AVD Security Baseline

The Azure Virtual Desktop security baseline provides a comprehensive set of recommendations from Microsoft to secure your AVD environment against modern threats. Implementing these controls reduces attack surface, protects sensitive data, and ensures compliance with industry standards. This page covers Microsoft's security baseline, CIS benchmarks, and practical implementation priorities for MSP environments.

## What is the AVD Security Baseline

The AVD security baseline is Microsoft's official security configuration guidance for Azure Virtual Desktop deployments. It's built on the Azure Security Benchmark framework and tailored specifically for virtual desktop infrastructure. The baseline addresses unique AVD risks like data exfiltration through client devices, session hijacking, and lateral movement within host pools.

Key areas covered:
- **Session security**: Screen capture protection, watermarking, clipboard controls
- **Network isolation**: Private endpoints, network segmentation, NSG rules
- **Identity & access**: MFA enforcement, conditional access, privileged access workstations
- **Data protection**: Encryption at rest/transit, FSLogix profile security
- **Monitoring**: Diagnostic logging, security alerts, audit trails

The baseline is regularly updated to address emerging threats and integrates with tools like Microsoft Defender for Cloud to provide automated security assessments.

## Microsoft Security Baseline for AVD

Microsoft's baseline recommendations are prioritized by impact and ease of implementation:

### Critical Controls (Implement First)

**Screen Capture Protection**
- Prevents users from capturing screenshots of remote sessions
- Protects against data leaks via screen recording tools
- Configured via RDP properties on host pool
- **Setting**: `screen capture protection:i:1`

**Session Watermarking**
- Displays user identity and connection details as overlay
- Deters unauthorized sharing of session content
- QR code contains connection metadata for forensics
- **Setting**: Enable via host pool properties (Preview feature)

**Clipboard Redirection Control**
- Limits copy/paste between local and remote sessions
- Reduces data exfiltration risk
- Balance security vs user productivity
- **Recommended**: Disable for high-security workloads, enable one-way (remote-to-local only) for general use

**Drive Redirection Restrictions**
- Prevents mapping local drives into remote sessions
- Blocks file transfers outside controlled channels
- **Recommended**: Disable unless business justification exists
- Alternative: Use OneDrive/SharePoint with DLP policies

### Session Timeout Controls

Automatic logoff prevents abandoned sessions from becoming security risks:

```yaml
# GPO Settings (Computer Configuration → Policies → Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host → Session Time Limits)

- Set time limit for disconnected sessions: 4 hours
- Set time limit for active but idle sessions: 2 hours
- End session when time limits are reached: Enabled
```

**Rationale**: Reduces window for session hijacking and conserves resources.

### Network Isolation

**Private Endpoints for AVD Resources**
- Host pools, workspaces, and storage accounts use private IPs
- Traffic stays within Azure backbone (no internet exposure)
- Requires Azure Virtual Network with proper subnets

**Network Segmentation**
- Separate subnets for session hosts, domain controllers, and management
- NSG rules to enforce least-privilege network access
- Example from our deployment (RG-Azure-VDI-01):

```bash
# Session host subnet: 10.0.1.0/24
# Allow RDP from Azure Virtual Desktop service tag only
# Deny direct internet access (force through Azure Firewall/NAT Gateway)
```

**Service Tags in NSG Rules**
- Use `AzureVirtualDesktop` service tag instead of IP ranges
- Automatically updated as Microsoft adds new AVD endpoints

## CIS Benchmarks for Windows 11

The Center for Internet Security (CIS) provides hardening guidelines for Windows 11 session hosts. Key recommendations:

### Level 1 Benchmarks (Minimal Performance Impact)

1. **Account Policies**
   - Enforce password history: 24 passwords
   - Maximum password age: 60 days
   - Minimum password length: 14 characters
   - Password complexity: Enabled

2. **Audit Policies**
   - Audit account logon events: Success and Failure
   - Audit logon events: Success and Failure
   - Audit object access: Failure
   - Audit policy change: Success and Failure

3. **User Rights Assignment**
   - Deny log on through Remote Desktop Services: Guests
   - Allow log on through Remote Desktop Services: Only authorized groups

4. **Security Options**
   - Interactive logon: Do not display last user name: Enabled
   - Network security: LAN Manager authentication level: Send NTLMv2 response only
   - User Account Control: Run all administrators in Admin Approval Mode: Enabled

### Level 2 Benchmarks (May Impact Usability)

- Disable unnecessary services (Print Spooler, Remote Registry)
- Enable Windows Defender Application Control (WDAC)
- Restrict anonymous access to named pipes and shares

**Implementation Approach**: Apply Level 1 to all session hosts. Evaluate Level 2 controls per workload (e.g., stricter for finance applications).

## Attack Surface Reduction (ASR)

Microsoft Defender ASR rules block common attack vectors. Recommended rules for AVD:

```powershell
# Enable ASR rules via Intune or GPO
# Rule: Block executable content from email client and webmail
Add-MpPreference -AttackSurfaceReductionRules_Ids BE9BA2D9-53EA-4CDC-84E5-9B1EEEE46550 -AttackSurfaceReductionRules_Actions Enabled

# Rule: Block Office applications from creating executable content
Add-MpPreference -AttackSurfaceReductionRules_Ids 3B576869-A4EC-4529-8536-B80A7769E899 -AttackSurfaceReductionRules_Actions Enabled

# Rule: Block credential stealing from Windows local security authority subsystem (lsass.exe)
Add-MpPreference -AttackSurfaceReductionRules_Ids 9e6c4e1f-7d60-472f-ba1a-a39ef669e4b2 -AttackSurfaceReductionRules_Actions Enabled

# Rule: Block untrusted and unsigned processes from USB
Add-MpPreference -AttackSurfaceReductionRules_Ids b2b3f03d-6a65-4f7b-a9c7-1c7ef74a9ba4 -AttackSurfaceReductionRules_Actions Enabled
```

**Testing**: Start with Audit mode to identify false positives before enforcing.

## Credential Guard

Windows Credential Guard uses virtualization-based security (VBS) to protect NTLM password hashes and Kerberos tickets from memory-scraping attacks.

**Requirements**:
- UEFI firmware 2.3.1 or higher
- Secure Boot enabled
- TPM 2.0 (for full protection)
- Azure VM sizes with nested virtualization support (Dv3, Ev3, or later)

**Enable via Intune**:
1. Devices → Configuration profiles → Create profile
2. Platform: Windows 10 and later
3. Profile type: Templates → Endpoint protection
4. Device Guard → Credential Guard: Enable with UEFI lock

**Verification**:
```powershell
# On session host
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
# Check SecurityServicesRunning includes "1" (Credential Guard)
```

## Our Implementation Priorities

Based on our environment (RG-Azure-VDI-01, session host avd-pool-0), here's the phased approach:

### Phase 1: Foundation (Week 1)
- [x] Enable screen capture protection on all host pools
- [x] Configure session timeout policies (4h disconnect, 2h idle)
- [ ] Implement NSG rules with AzureVirtualDesktop service tag
- [ ] Enable diagnostic logging to log-avd-prod workspace

### Phase 2: Hardening (Week 2-3)
- [ ] Deploy CIS Level 1 benchmarks via GPO
- [ ] Enable ASR rules in Audit mode
- [ ] Configure private endpoints for storage accounts (FSLogix profiles)
- [ ] Implement conditional access policy requiring MFA for AVD

### Phase 3: Advanced (Week 4+)
- [ ] Enable Credential Guard on session hosts
- [ ] Deploy session watermarking (currently Preview)
- [ ] Disable clipboard/drive redirection for high-security users
- [ ] Implement just-in-time (JIT) access for administrative tasks

### Monitoring & Validation

```kql
// Query Log Analytics (log-avd-prod) for security events
WVDConnections
| where TimeGenerated > ago(24h)
| where State == "Connected"
| project TimeGenerated, UserName, ClientOS, ClientVersion, CorrelationId
| join kind=inner (
    WVDCheckpoints
    | where Name == "ScreenCaptureProtection"
    | project CorrelationId, ScreenCaptureEnabled = tostring(Parameters)
) on CorrelationId
| summarize ConnectionsWithProtection = countif(ScreenCaptureEnabled == "true"),
            TotalConnections = count() by bin(TimeGenerated, 1h)
```

## Best Practices

- **Defense in Depth** - No single control is foolproof. Layer screen capture protection with DLP policies, network isolation, and user training.

- **Least Privilege** - Apply stricter controls (clipboard disable, drive redirection block) only to roles that handle sensitive data. Balance security with user productivity.

- **Gradual Rollout** - Test ASR rules and CIS benchmarks in pilot group before production. Monitor helpdesk tickets for usability issues.

- **Automation** - Use Azure Policy to enforce security baselines across all host pools. Detect and remediate drift automatically.

- **Regular Reviews** - Security baselines evolve. Review Microsoft's guidance quarterly and adjust configurations accordingly.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Screen capture protection not working | RDP client version too old | Upgrade to Remote Desktop client 1.2.3317 or later (Windows), 10.8.2+ (macOS) |
| Users complain about session timeouts | Aggressive timeout policies | Review idle timeout (2h may be too short for some workflows). Consider user activity vs security risk. |
| Clipboard disabled breaks workflows | Blanket policy applied to all users | Use Intune filters to apply restrictive policies only to high-risk groups (finance, HR). |
| Credential Guard causes boot issues | Incompatible VM size or firmware | Verify VM supports nested virtualization (Dv3/Ev3 series). Check UEFI settings in Azure. |
| ASR rules block legitimate apps | Overly broad rules | Use Audit mode logs to identify exclusions. Add file/folder exclusions in Defender policy. |
| Private endpoints break connectivity | DNS misconfiguration | Ensure custom DNS servers resolve privatelink.wvd.microsoft.com to private IP. Test with `nslookup`. |

## Related Resources

- Microsoft Security Baseline: https://learn.microsoft.com/security/benchmark/azure/baselines/virtual-desktop-security-baseline
- CIS Windows 11 Benchmark: https://www.cisecurity.org/benchmark/microsoft_windows_desktop
- ASR Rules Reference: https://learn.microsoft.com/microsoft-365/security/defender-endpoint/attack-surface-reduction-rules-reference
- Credential Guard Documentation: https://learn.microsoft.com/windows/security/identity-protection/credential-guard/