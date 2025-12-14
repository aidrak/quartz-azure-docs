---
title: Variant Intune Integration
description: 
published: true
date: 2025-12-14T04:54:32.222Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:43.129Z
---

# Variant: Intune Integration

This variant extends the base deployment with Microsoft Intune for device management, enabling centralized configuration, compliance enforcement, and application deployment to AVD session hosts.

## When to Use This Variant

Use Intune integration when:
- Cloud-only management is preferred (no on-premises SCCM/ConfigMgr)
- Centralized configuration profiles for all session hosts
- Compliance policies with Conditional Access enforcement
- Application deployment via Intune (Win32 apps, Microsoft Store)
- Windows Update management with update rings
- Unified Endpoint Management across AVD and physical devices

## Pre-Deployment Checklist

- [ ] Microsoft Intune license (included in M365 E3/E5, EMS E3/E5)
- [ ] Entra ID Premium P1 (for Conditional Access, required for compliance enforcement)
- [ ] Session hosts Entra ID joined (or Hybrid joined with cloud sync)
- [ ] Dynamic device groups created (AVD-Devices-Pooled, AVD-Devices-Personal)
- [ ] Intune admin role assigned (Intune Administrator or higher)

## Step 1: Verify Device Enrollment

Session hosts should auto-enroll in Intune when Entra ID joined (if auto-enrollment is configured).

**Check Auto-Enrollment Setting:**
**Portal Path:** Entra Admin Center → Devices → Enrollment → Automatic Enrollment

**Configuration:**
| Setting | Value |
|---------|-------|
| **MDM User Scope** | All (or specific group including session hosts) |
| **MAM User Scope** | None (not applicable for Windows) |

**Verify Session Host Enrollment:**
**Portal Path:** Intune Admin Center → Devices → All Devices

Search for session host (e.g., `avd-pool-0`). Should show:
- **Managed by:** Intune
- **Compliance:** Compliant (or Not Evaluated if no policies yet)
- **OS:** Windows 11 Enterprise

**If Not Enrolled:**
1. RDP to session host
2. Open Settings → Accounts → Access work or school
3. Verify Entra ID connection shows "Connected to [tenant] Entra ID"
4. Click Info → Sync to trigger Intune enrollment

## Step 2: Create Configuration Profile for FSLogix

Deploy FSLogix settings via Intune instead of GPO or manual registry edits.

**Portal Path:** Intune Admin Center → Devices → Configuration Profiles → Create

**Configuration:**
| Setting | Value |
|---------|-------|
| **Platform** | Windows 10 and later |
| **Profile Type** | Settings Catalog |
| **Name** | AVD - FSLogix Profile Containers |

**Settings to Add (Search "FSLogix"):**
| Setting | Value |
|---------|-------|
| **Enabled** | Enabled |
| **VHDLocations** | `\\fslogix121025.file.core.windows.net\profiles` |
| **SizeInMBs** | 30000 |
| **IsDynamic** | Enabled |
| **VolumeType** | VHDX |
| **DeleteLocalProfileWhenVHDShouldApply** | Enabled |

**Assignment:**
- Include: AVD-Devices-Pooled (dynamic group)
- Exclude: None

## Step 3: Create Compliance Policy

Compliance policies define security requirements. Non-compliant devices can be blocked via Conditional Access.

**Portal Path:** Intune Admin Center → Devices → Compliance Policies → Create

**Configuration:**
| Setting | Value |
|---------|-------|
| **Platform** | Windows 10 and later |
| **Name** | AVD - Security Baseline Compliance |

**Compliance Settings:**
| Category | Setting | Value |
|----------|---------|-------|
| **Device Health** | Require BitLocker | Require |
| **Device Health** | Require Secure Boot | Require |
| **System Security** | Firewall | Require |
| **System Security** | Antivirus | Require |
| **System Security** | Antispyware | Require |

**Actions for Noncompliance:**
| Action | Schedule |
|--------|----------|
| Mark device noncompliant | Immediately |
| Send email to user | After 1 day |
| Block access (Conditional Access) | After 3 days |

**Assignment:**
- Include: AVD-Devices-All (all session hosts)

## Step 4: Create Conditional Access Policy for Compliance

Block non-compliant devices from accessing AVD.

**Portal Path:** Entra Admin Center → Protection → Conditional Access → Create new policy

**Configuration:**
| Setting | Value |
|---------|-------|
| **Name** | Require Compliant Device for AVD |
| **Users** | AVD-Users-Pooled, AVD-Users-Personal |
| **Cloud Apps** | Azure Virtual Desktop, Microsoft Remote Desktop |
| **Conditions** | Device platforms: Windows |
| **Grant** | Require device to be marked as compliant |
| **Enable Policy** | Report-only (initially) |

**Testing:**
1. Enable in Report-only mode
2. Monitor Sign-in logs for 7 days
3. Verify no unexpected blocks
4. Change to On after validation

## Step 5: Deploy Applications via Intune

Deploy LOB applications to session hosts using Intune instead of manual installation.

**Example: Deploy Adobe Acrobat Reader**

**Portal Path:** Intune Admin Center → Apps → All Apps → Add

**Configuration:**
| Setting | Value |
|---------|-------|
| **App Type** | Windows app (Win32) |
| **Name** | Adobe Acrobat Reader DC |
| **Publisher** | Adobe |

**App Package:**
1. Download Adobe Reader offline installer (.exe)
2. Use IntuneWinAppUtil.exe to create .intunewin package
3. Upload .intunewin file

**Install Command:**
```
AcroRdrDC2300820470_en_US.exe /sAll /rs /msi EULA_ACCEPT=YES
```

**Uninstall Command:**
```
msiexec /x {AC76BA86-7AD7-1033-7B44-AC0F074E4100} /qn
```

**Detection Rule:**
- Rule Type: File
- Path: C:\Program Files\Adobe\Acrobat DC\Acrobat
- File: Acrobat.exe
- Detection Method: File or folder exists

**Assignment:**
- Required: AVD-Devices-All

## Step 6: Configure Windows Update Rings

Control Windows Update behavior for AVD session hosts.

**Important for Pooled AVD:** Consider updating golden image instead of individual hosts. Update rings are more useful for personal AVD.

**Portal Path:** Intune Admin Center → Devices → Windows Update → Update Rings → Create

**Configuration:**
| Setting | Value |
|---------|-------|
| **Name** | AVD - Production Ring |
| **Servicing Channel** | General Availability Channel |
| **Deferral - Quality Updates** | 7 days |
| **Deferral - Feature Updates** | 30 days |
| **Pause Quality Updates** | No |
| **Auto-Restart** | Required with notification |
| **Active Hours** | 8:00 AM - 6:00 PM |

**Assignment:**
- Include: AVD-Devices-Personal (personal desktops only)
- Exclude: AVD-Devices-Pooled (update via golden image)

## Step 7: Create Device Configuration for RDP Settings

Apply RDP-specific settings (timezone redirection, clipboard, etc.).

**Portal Path:** Intune Admin Center → Devices → Configuration Profiles → Create

**Configuration:**
| Setting | Value |
|---------|-------|
| **Platform** | Windows 10 and later |
| **Profile Type** | Settings Catalog |
| **Name** | AVD - RDP Session Settings |

**Settings (Search "Remote Desktop"):**
| Setting | Value |
|---------|-------|
| Allow time zone redirection | Enabled |
| Allow audio redirection | Enabled |
| Allow clipboard redirection | Enabled |
| Allow drive redirection | Disabled |
| Allow printer redirection | Enabled |

**Assignment:**
- Include: AVD-Devices-All

## Step 8: Monitor Deployment Status

Verify profiles and policies are applying successfully.

**Profile Deployment Status:**
**Portal Path:** Intune Admin Center → Devices → Configuration Profiles → [Select Profile] → Device Status

Expected: All session hosts show "Succeeded"

**Compliance Status:**
**Portal Path:** Intune Admin Center → Devices → Monitor → Device Compliance

Expected: Session hosts show "Compliant"

**App Installation Status:**
**Portal Path:** Intune Admin Center → Apps → Monitor → App Install Status

Expected: Adobe Reader shows "Installed" on all targeted devices

## Validation

- [ ] Session hosts appear in Intune Admin Center
- [ ] Configuration profiles show "Succeeded" deployment status
- [ ] Compliance policies evaluate session hosts as "Compliant"
- [ ] Applications installed successfully (verify in Apps & Features on session host)
- [ ] FSLogix profiles mount correctly (user logon creates VHD)
- [ ] Conditional Access (if enabled) allows compliant devices

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Device not appearing in Intune | Auto-enrollment not configured | Verify MDM user scope includes device users in Entra ID |
| Configuration profile not applying | Assignment filter mismatch | Check dynamic group membership, verify device meets filter criteria |
| Device shows non-compliant | Security setting not met | Check which compliance rule failed, remediate (enable BitLocker, etc.) |
| App stuck in "Installing" | Installation timeout or error | Check Intune Management Extension logs on device: C:\ProgramData\Microsoft\IntuneManagementExtension\Logs |
| Policy conflict error | Multiple profiles configuring same setting | Use Intune troubleshooting blade to identify conflicting policies |

## Reference

- **Concept:** [[Intune Prerequisites for AVD|Intune Prerequisites]] - Licensing, enrollment
- **Concept:** [[device-configuration-profiles]] - Settings Catalog
- **Concept:** [[Compliance Policies for AVD|Compliance Policies]] - Security requirements
- **Concept:** [[Application Deployment with Intune|App Deployment]] - Win32, MSIX, Microsoft Store

## Deployment Complete

Congratulations! You have completed the AVD Implementation Guide with optional Hybrid Connectivity and Intune Integration variants.

**Summary of Deployed Components:**
- Resource Group: RG-Azure-VDI-01
- Virtual Network: vnet-avd (10.0.0.0/16)
- Host Pools: hp-pooled-prod1, hp-personal-prod1
- Workspace: ws-avd-prod
- Session Host: avd-pool-0 (and additional as needed)
- Storage: fslogix121025 (Azure Files Premium)
- Monitoring: log-avd-prod (Log Analytics)
- VPN Gateway: vpngw-avd (if hybrid)
- Intune Profiles: FSLogix, RDP Settings, Compliance

**Next Steps:**
1. Deploy additional session hosts as user count grows
2. Update golden image monthly with Windows patches
3. Monitor AVD Insights for performance issues
4. Review security posture with Defender for Cloud
5. Train users on AVD client installation and connection