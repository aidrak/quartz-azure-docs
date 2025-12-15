# Intune Device Management

Microsoft Intune provides cloud-based device management for Azure Virtual Desktop session hosts. When AVD session hosts are Entra joined, they automatically enroll into Intune, enabling centralized configuration management, compliance enforcement, and application deployment without requiring on-premises infrastructure.

## What This Section Covers

This section provides in-depth reference material for managing AVD session hosts with Microsoft Intune:

- **Prerequisites and Licensing** - Intune licensing requirements, Entra ID configuration, and automatic enrollment setup
- **Configuration Management** - Deploying settings, policies, and FSLogix configurations using Settings Catalog
- **Compliance Enforcement** - Security baselines, BitLocker requirements, and Conditional Access integration
- **Application Deployment** - Win32 apps, Microsoft 365 Apps, and application management strategies
- **Windows Updates** - Update rings, feature update policies, and differentiated strategies for pooled vs personal AVD

## Reference Pages

### [[intune-prerequisites-for-avd|Intune Prerequisites for AVD]]

Covers licensing requirements, Entra ID configuration, and automatic enrollment for AVD session hosts.

**Key Topics:**
- Microsoft 365 E3/E5 vs standalone Intune licensing
- Entra Join vs Hybrid Join enrollment strategies
- MDM auto-enrollment configuration
- Dynamic device group creation
- Enrollment restrictions and verification

**When to Read:** Before deploying AVD session hosts. Essential for understanding licensing needs and enrollment setup.

### [[device-configuration-profiles|Device Configuration Profiles]]

Explains how to deploy settings and policies to session hosts using Intune Settings Catalog and templates.

**Key Topics:**
- Settings Catalog vs legacy templates (ADMX)
- FSLogix profile container configuration
- RDP properties (timezone redirection, clipboard, audio)
- OneDrive Known Folder Move
- Conflict resolution and troubleshooting
- Device group targeting strategies

**When to Read:** After session hosts are deployed and enrolled. Use this to configure FSLogix, RDP settings, and Windows features.

### [[compliance-policies-for-avd|Compliance Policies for AVD]]

Describes how to enforce security requirements and integrate with Conditional Access for Zero Trust security.

**Key Topics:**
- BitLocker encryption requirements
- Firewall and antivirus compliance checks
- Minimum OS version enforcement
- Grace periods and actions for non-compliance
- Conditional Access integration
- Report-only mode testing

**When to Read:** After configuration profiles are deployed. Essential for enforcing security baselines and blocking non-compliant devices.

### [[application-deployment-with-intune|Application Deployment with Intune]]

Covers Win32 app packaging, deployment strategies, and application management for AVD environments.

**Key Topics:**
- Win32 app packaging with IntuneWinAppUtil
- Microsoft 365 Apps with Shared Computer Activation
- Detection rules (file, registry, PowerShell script)
- Dependencies and supersedence relationships
- Pooled vs personal deployment strategies
- Assignment types (Required, Available, Uninstall)

**When to Read:** When deploying applications to personal AVD hosts. For pooled hosts, install apps in the golden image instead.

### [[remediation-scripts/index|Intune Proactive Remediation Scripts]]

Collection of PowerShell detection and remediation scripts for automatically fixing configuration drift on AVD session hosts.

**Included Scripts:**
- **Drive Mapping Task** - Detect and remediate scheduled task for network drive mapping
- **Notifications Enable** - Enable Windows notifications for specific users or groups
- **Office Shortcuts** - Deploy and manage Office application shortcuts on the taskbar

**Key Features:**
- Automatic detection of configuration drift
- Automatic remediation when issues detected
- Detailed Intune compliance reporting
- User-context execution for per-user settings

**When to Use:** Deploy as Intune Proactive Remediations to automatically detect and fix configuration drift on AVD session hosts. Useful for ensuring consistent user experience across pooled and personal hosts without full configuration profiles.

### [[windows-update-policies-for-avd|Windows Update Policies for AVD]]

Explains Windows Update for Business policies, update rings, and differentiated update strategies for pooled and personal AVD.

**Key Topics:**
- Quality update vs feature update deferral
- Update rings (pilot and production)
- Feature update policies (pin to specific Windows version)
- Driver update policies
- Golden image update process for pooled hosts
- Expedited updates for zero-day vulnerabilities

**When to Read:** After applications are deployed. Critical for understanding why pooled hosts should NOT use update rings (update golden image instead).

## Quick-Deploy Integration

These Intune reference pages support the following Quick-Deploy steps:

- **[[../Quick-Deploy/09-fslogix-configuration|Step 09: FSLogix Configuration]]** - Uses Settings Catalog to deploy FSLogix profile container settings
- **[[../Quick-Deploy/10-application-deployment|Step 10: Application Deployment]]** - Packages and deploys Win32 apps to personal hosts

## Key Concepts

### Pooled vs Personal Management Strategy

**Pooled Host Pools (Non-Persistent):**
- **Configuration Profiles:** Deploy FSLogix, RDP properties, and security settings via Intune
- **Applications:** Install in golden image, NOT via Intune (faster logon, consistent experience)
- **Windows Updates:** Update golden image monthly, redeploy session hosts (do NOT use update rings)
- **Compliance:** Enforce security baselines (BitLocker, firewall, antivirus)

**Personal Host Pools (Persistent):**
- **Configuration Profiles:** Deploy FSLogix, RDP properties, and security settings via Intune
- **Applications:** Deploy via Intune as Required or Available (flexible, automatic updates)
- **Windows Updates:** Use update rings with pilot and production phases
- **Compliance:** Enforce security baselines (BitLocker, firewall, antivirus)

### Intune vs Group Policy

**Intune:**
- Cloud-native, no domain controllers required
- Continuously enforced (re-applied every 8 hours)
- Settings Catalog provides modern UI for all Windows settings
- Integrates with Conditional Access for Zero Trust security
- Recommended for new AVD deployments

**Group Policy (GPO):**
- Requires on-premises Active Directory and domain controllers
- Applied at startup, logon, or manual refresh (gpupdate)
- Requires ADMX templates and domain controller infrastructure
- Use only for hybrid scenarios with existing on-premises AD

### Device Groups for AVD

Create these Entra ID dynamic groups for Intune targeting:

**AVD-Devices-All:**
```
(device.displayName -startsWith "hp-")
```

**AVD-Devices-Pooled:**
```
(device.displayName -contains "pooled")
```

**AVD-Devices-Personal:**
```
(device.displayName -contains "personal")
```

## Best Practices

1. **Use Entra Join for New Deployments** - Eliminates domain controllers, enables automatic Intune enrollment
2. **Settings Catalog for All New Profiles** - Modern, searchable, cloud-native approach (templates are legacy)
3. **Install Pooled Apps in Golden Image** - Faster logon, consistent experience, no per-session installation overhead
4. **Deploy Personal Apps via Intune** - Flexible deployment, automatic updates, user-specific assignments
5. **Test Conditional Access in Report-Only Mode** - Prevents accidental lockouts before enforcement
6. **Monitor Compliance and Deployment Weekly** - Catch errors early, ensure devices receive policies

## Common Workflows

### Initial Setup (Cloud-Native AVD)

1. Configure Intune auto-enrollment (Entra Admin Center → Mobility → MDM User Scope = All)
2. Create dynamic device groups (AVD-Devices-All, AVD-Devices-Pooled, AVD-Devices-Personal)
3. Deploy session hosts with Entra Join enabled (auto-enroll into Intune)
4. Create Settings Catalog profiles for FSLogix, RDP properties
5. Create compliance policy requiring BitLocker, firewall, antivirus
6. Create Conditional Access policy requiring compliant device for AVD access

### Application Deployment (Personal AVD)

1. Download application installer (Microsoft 365, Adobe Reader, etc.)
2. Package with IntuneWinAppUtil.exe
3. Upload to Intune as Win32 app
4. Configure detection rules (file, registry, or PowerShell script)
5. Assign to AVD-Devices-Personal or AVD-Personal-Users
6. Monitor deployment status (Apps → [App] → Device install status)

### Windows Update Management (Personal AVD)

1. Create pilot update ring (0-day quality deferral, 60-day feature deferral)
2. Create production update ring (14-day quality deferral, 180-day feature deferral)
3. Assign pilot ring to AVD-Devices-Pilot (2-5 test hosts)
4. Assign production ring to AVD-Devices-Personal
5. Monitor update compliance weekly (Reports → Windows Updates)

## Troubleshooting Quick Reference

| Issue | Check | Solution |
|-------|-------|----------|
| Session host not appearing in Intune | Auto-enrollment, licenses | Verify MDM auto-enrollment enabled, user has Intune license |
| FSLogix profile not loading | Registry, connectivity | Force Intune sync, verify storage account accessible |
| App installation fails | Detection rules, logs | Check detection script, review IntuneManagementExtension.log |
| Compliance shows "Not Evaluated" | Policy assignment | Assign compliance policy to device group, force sync |
| Update not installing | Update ring, deadline | Check deferral period, verify deadline not exceeded |

## Related Sections

- **[[../Identity/entra-id-device-join|Entra ID Device Join]]** - Understanding Entra Join vs Hybrid Join
- **[[../Storage/fslogix-profile-containers|FSLogix Profile Containers]]** - FSLogix architecture and sizing
- **[[../Images/golden-image-creation|Golden Image Creation]]** - Installing apps in pooled host images
- **[[../Security/conditional-access-for-avd|Conditional Access for AVD]]** - Zero Trust access controls
