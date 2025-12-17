---
title: Intune Proactive Remediation Scripts for AVD
description: Collection of PowerShell detection and remediation scripts for Azure Virtual Desktop configuration management
published: true
date: 2025-12-15T00:00:00.000Z
tags: [Intune, Proactive Remediation, PowerShell, AVD, configuration]
---

# Intune Proactive Remediation Scripts

Collection of PowerShell scripts for Intune Proactive Remediations. These scripts automatically detect and remediate configuration drift on Azure Virtual Desktop session hosts.

## What are Proactive Remediations?

**Intune Proactive Remediations** are managed scripts that:
1. **Detect** - Run a detection script to check device configuration status
2. **Remediate** - Automatically run a remediation script if detection fails
3. **Report** - Log success/failure and configuration drift in Intune console

**Portal**: Intune Admin Center → Devices → Compliance → Proactive Remediations

---

## Included Scripts

### Drive Mapping Task

**Purpose**: Create and maintain a scheduled task that maps network drives at user logon.

**Folder**: `Drive Mapping Task/`

**Scripts**:
- `Detect-DriveMappingTask.ps1` - Checks if drive mapping scheduled task exists
- `Remediate-DriveMappingTask.ps1` - Creates/updates the drive mapping scheduled task
- `Fix-DriveMappingTask.ps1` - Alternative remediation script

**Use Case**: Ensure consistent network drive mapping across all AVD users without requiring persistent profile containers.

**Detection Checks**:
- Scheduled task exists (name matches)
- Task is enabled
- Task points to correct script/batch file
- Task runs at user logon

**Remediation Actions**:
- Creates scheduled task if missing
- Updates task configuration if incorrect
- Enables task if disabled

---

### Notifications Enable

**Purpose**: Enable Windows notification system for specific users or application contexts.

**Folder**: `Notifications Enable/`

**Scripts**:
- `Detect-NotificationsEnabled.ps1` - Checks notification configuration in registry
- `Remediate-NotificationsEnabled.ps1` - Enables notifications by modifying registry
- `Fix-NotificationsEnabled.ps1` - Alternative remediation script

**Use Case**: Ensure Windows notifications and toast notifications work across all users, even in managed environments where notifications may be disabled by default.

**Detection Checks**:
- Registry key exists: `HKCU:\Software\Microsoft\Windows\CurrentVersion\Notifications\Settings`
- Notification toggles enabled in registry
- Action Center accessible

**Remediation Actions**:
- Creates registry paths if missing
- Sets notification values to "enabled"
- Enables Action Center access

---

### Office Shortcuts

**Purpose**: Deploy and maintain Office application shortcuts on the Windows taskbar.

**Folder**: `Office Shortcuts/`

**Scripts**:
- `Detect-OfficeShortcuts.ps1` - Checks if Office apps are pinned to taskbar
- `Remediate-OfficeShortcuts.ps1` - Pins Office apps (Word, Excel, Outlook, PowerPoint) to taskbar
- Additional remediation variants

**Use Case**: Standardize taskbar across all users with consistent Office application access, without requiring full taskbar customization policies.

**Detection Checks**:
- Word pinned to taskbar
- Excel pinned to taskbar
- Outlook pinned to taskbar
- PowerPoint pinned to taskbar (optional)

**Remediation Actions**:
- Pins each Office app to taskbar if missing
- Handles both classic Office and Office 365 Click-to-Run versions
- Works with machine-wide and user-specific installations

---

## How to Deploy

### Deploy as Proactive Remediation in Intune

1. **Intune Admin Center** → **Devices** → **Compliance** → **Proactive Remediations** → **+ Create**

2. **General**:
   - **Name**: e.g., "AVD - Drive Mapping Task"
   - **Description**: What the script does

3. **Detection script**:
   - Copy-paste content from `Detect-*.ps1` file

4. **Remediation script**:
   - Copy-paste content from `Remediate-*.ps1` file

5. **Settings**:
   - **Run this script using the logged-in credentials**: Yes (for user-context settings)
   - **Enforce script signature check**: No (unless signed)
   - **Run script in 64-bit PowerShell context**: Yes (for compatibility)

6. **Assignments**:
   - Assign to `avd-devices-all` or specific device group

7. **Review + Create**

### Monitor Results

**Portal**: Intune Admin Center → Devices → Proactive Remediations → [Script Name]

**Monitor**:
- **Device Check-in status**: Shows devices that have run detection
- **Detection failures**: Devices where detection failed (remediation ran)
- **Remediation status**: Whether remediation succeeded/failed

---

## Best Practices

1. **Use User-Context Execution** - For settings that apply per-user (notifications, taskbar, shortcuts)
2. **Test with Pilot Group** - Deploy to small test group first (avd-devices-pilot)
3. **Check Detection Scripts** - Review what the detection script checks before deploying
4. **Monitor Compliance Weekly** - Review proactive remediation status in Intune
5. **Version Your Scripts** - Add comments with version numbers and modification dates
6. **Log Output** - Add logging to scripts for troubleshooting (Write-Output statements)

---

## File Naming Convention

Scripts follow this naming convention:

- `Detect-<Feature>.ps1` - Detection script (run first, determines if remediation needed)
- `Remediate-<Feature>.ps1` - Remediation script (fixes the issue if detection fails)
- `Fix-<Feature>.ps1` - Alternative/variant remediation script

---

## Customization

Each script can be customized:

1. **Detection Logic**: Modify the detection checks in `Detect-*.ps1`
   - Change registry paths
   - Check different file locations
   - Add additional validation

2. **Remediation Actions**: Modify remediation steps in `Remediate-*.ps1`
   - Change created values
   - Update shortcuts
   - Adjust paths for your environment

3. **Add Logging**: Insert logging statements for troubleshooting:
   ```powershell
   Add-Content -Path "C:\Windows\Temp\Remediation.log" -Value "$(Get-Date): Detection failed, running remediation..."
   ```

---

## Script Output and Exit Codes

**Proactive Remediation Exit Codes**:

- **0 (Success)**: Detection passed (device compliant, no remediation needed)
- **1 (Failure)**: Detection failed (device non-compliant, remediation will run)

**Detection Script Output**:
```powershell
# Example: Task exists, so exit 0 (compliant)
exit 0

# Example: Task missing, so exit 1 (non-compliant, trigger remediation)
exit 1
```

**Remediation Script Output**:
```powershell
# On success:
Write-Output "Drive mapping task created successfully"
exit 0

# On failure:
Write-Output "Failed to create drive mapping task"
exit 1
```

---

## Related References

- [[../intune-prerequisites-for-avd|Intune Prerequisites for AVD]] - Setup and enrollment
- [[../device-configuration-profiles|Device Configuration Profiles]] - Alternative: Settings Catalog approach
- [[../../Deployment/06-intune-configuration|Step 06: Intune Configuration]] - Deployment playbook
- [Microsoft Documentation: Proactive Remediations](https://learn.microsoft.com/en-us/mem/intune/configuration/remediations)

---

## Script Directory

```
remediation-scripts/
├── Drive Mapping Task/
│   ├── Detect-DriveMappingTask.ps1
│   ├── Remediate-DriveMappingTask.ps1
│   └── Fix-DriveMappingTask.ps1
├── Notifications Enable/
│   ├── Detect-NotificationsEnabled.ps1
│   ├── Remediate-NotificationsEnabled.ps1
│   └── Fix-NotificationsEnabled.ps1
├── Office Shortcuts/
│   ├── Detect-OfficeShortcuts.ps1
│   └── Remediate-OfficeShortcuts.ps1
└── index.md (this file)
```

---

**Last Updated**: December 15, 2025
