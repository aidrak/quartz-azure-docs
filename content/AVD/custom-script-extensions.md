---
title: Custom Script Extensions for AVD Session Host Deployment
description: Use custom PowerShell scripts from GitHub for automated post-deployment configuration on AVD session hosts
published: true
date: 2025-12-15T00:00:00.000Z
tags: [AVD, Custom Scripts, PowerShell, Post-Deployment, Automation, GitHub]
---

# Custom Script Extensions for AVD Session Host Deployment

Custom Script Extensions automate post-deployment configuration on AVD session hosts by downloading and executing PowerShell scripts during VM provisioning. Scripts run during deployment, before AVD Agent installation, making them ideal for software installation, system configuration, and security hardening.

**Script Repository:** [github.com/aidrak/azure-scripts](https://github.com/aidrak/azure-scripts) - Publicly accessible PowerShell scripts for AVD deployments

---

## What It Is

**Custom Script Extension** is an Azure VM feature that:
1. **Downloads** a script from a public repository (GitHub raw content URL) during VM creation
2. **Executes** the script under `LocalSystem` account (elevated privileges)
3. **Completes** before AVD Agent installation and host pool registration
4. **Logs** output to VM extension logs for troubleshooting

**Execution Flow:**
```
VM Creation → Custom Script Runs → AVD Agent Installs → Session Host Registers
```

**Execution Context:**
- Account: `SYSTEM` (elevated, full privileges)
- Timeout: 90 minutes maximum
- Working directory: `C:\Packages\Plugins\Microsoft.Compute.CustomScriptExtension\1.*\Downloads\<n>`
- Logs: `C:\WindowsAzure\Logs\Plugins\Microsoft.Compute.CustomScriptExtension\`

---

## When to Use It

### Ideal Use Cases

✅ **Software Installation**
- LOB applications (not in golden image)
- Drivers and firmware updates
- Browser extensions
- Development tools

✅ **System Configuration**
- Windows features (Remote Assistance, etc.)
- Registry settings (time zone, RDP properties, etc.)
- Windows Defender exclusions for FSLogix
- Event log forwarding

✅ **Monitoring & Logging**
- Install monitoring agents (Microsoft Defender, Application Insights)
- Configure event log forwarding
- Deploy custom performance counters

✅ **Post-Image Customization**
- Apply environment-specific settings
- Join custom storage accounts
- Configure region-specific resources
- Apply per-host hardening

### Not Ideal For

❌ **Applications already in golden image** - Slower deployment (install during image creation instead)
❌ **User-context settings** - Script runs as SYSTEM, use Intune for user context
❌ **VM restarts** - Extension times out if script restarts VM
❌ **Interactive operations** - No user interaction support
❌ **Frequent changes** - Requires VM redeployment to change script

---

## How to Set It Up

### Step 1: Find Script in GitHub Repository

All AVD deployment scripts are hosted in the public GitHub repository:

**Repository:** [github.com/aidrak/azure-scripts](https://github.com/aidrak/azure-scripts)

**Available scripts:**
- Browse the repository for available scripts
- Each script has documentation describing what it configures
- Scripts are maintained in version control with change history

### Step 2: Get Raw GitHub URL

For any script in the repository:

**URL Format:**
```
https://raw.githubusercontent.com/aidrak/azure-scripts/main/<script-path>
```

**Example:**
```
https://raw.githubusercontent.com/aidrak/azure-scripts/main/Deploy-AVDHost.ps1
```

**How to Get Raw URL from GitHub:**
1. Navigate to the script file in GitHub
2. Click **Raw** button (top right of file)
3. Copy the URL from your browser address bar
4. Use this URL in Step 3

### Step 3: Reference Script During Session Host Deployment

**Azure Portal - During Step 09 (Session Host Provisioning):**

1. Navigate to **Azure Virtual Desktop** → **Host pools** → `hp-pooled-prod` → **Session hosts** → **+ Add**
2. Fill in all standard deployment settings (name, image, size, network, etc.)
3. Scroll down to **Custom configuration** section
4. In **Custom configuration script URL**, paste the GitHub raw URL:
   ```
   https://raw.githubusercontent.com/aidrak/azure-scripts/main/Deploy-AVDHost.ps1
   ```
5. Continue with deployment → **Review + create** → **Create**

---

## ARM Template Example

```json
{
  "type": "Microsoft.Compute/virtualMachines/extensions",
  "name": "[concat(parameters('vmName'), '/CustomScriptExtension')]",
  "apiVersion": "2021-03-01",
  "location": "[resourceGroup().location]",
  "properties": {
    "publisher": "Microsoft.Compute",
    "type": "CustomScriptExtension",
    "typeHandlerVersion": "1.10",
    "autoUpgradeMinorVersion": true,
    "protectedSettings": {
      "fileUris": [
        "https://raw.githubusercontent.com/aidrak/azure-scripts/main/Deploy-AVDHost.ps1"
      ],
      "commandToExecute": "powershell.exe -ExecutionPolicy Unrestricted -File Deploy-AVDHost.ps1"
    }
  }
}
```

---

## Best Practices

### Script Design

1. **Idempotent Execution**
   - Scripts should handle multiple executions gracefully
   - Check if action already completed before executing
   ```powershell
   # Example: Check if software already installed
   if ((Get-Command 7z.exe -ErrorAction SilentlyContinue) -eq $null) {
       # Install 7-Zip
   }
   ```

2. **Comprehensive Logging**
   - Log all actions to file (Azure portal shows only last 4,096 bytes)
   - Use timestamped entries for troubleshooting
   ```powershell
   function Write-Log {
       param([string]$Message)
       $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
       "$timestamp - $Message" | Out-File "C:\AVDSetup.log" -Append
   }
   ```

3. **Proper Error Handling**
   - Use `$ErrorActionPreference = "Stop"` to fail fast
   - Catch exceptions and log them
   - Return proper exit codes (0 = success, 1 = failure)

4. **No User Interaction**
   - ❌ Don't use `Read-Host`, `Read-Menu`, or interactive prompts
   - ❌ Don't require manual input
   - Use silent installers (`/S`, `/quiet`, etc.)

5. **No VM Restarts**
   - ❌ Don't restart the VM from the script
   - Extension times out waiting for VM to come back online
   - If restart needed, mark it as pending and let deployment complete

### Security

1. **No Secrets in Scripts**
   - Never hardcode passwords, API keys, or connection strings
   - Use Azure Key Vault with managed identity if secrets needed
   - Access via managed identity: `Get-AzKeyVaultSecret`

2. **Apply Network Security**
   - Verify outbound HTTPS access to GitHub (github.com, raw.githubusercontent.com)
   - Check NSG rules allow outbound to required domains
   - Verify firewall not blocking script download

3. **Script Source Control**
   - All scripts in GitHub repository
   - Version controlled with change history
   - Reviewed before merging to main branch
   - Pull requests for any changes

### Script Management

1. **Version Your Scripts**
   - Add version number in script comments
   - Use consistent naming conventions
   - Maintain changelog in repository

2. **Use Source Control**
   - Store scripts in Git repository
   - Easy to rollback to previous versions
   - Track changes and approvals
   - Collaboration and code review

3. **Documentation**
   - README for each script explaining what it does
   - Comments for non-obvious logic
   - Parameter documentation

---

## Troubleshooting

### Issue: Script Never Executes

**Symptom:** VM deployed but log file not created, no evidence script ran

**Causes & Fixes:**

1. **GitHub URL incorrect or inaccessible**
   - Test URL in browser on the VM or local machine - should download file
   - Verify format: `https://raw.githubusercontent.com/aidrak/azure-scripts/main/<script>`
   - Check for typos in username, repo, or filename
   - Verify network allows outbound to raw.githubusercontent.com (GitHub CDN)

2. **Network blocked**
   - Verify NSG allows outbound HTTPS (port 443) to raw.githubusercontent.com
   - Check if Azure Firewall blocking GitHub access
   - Test from VM: `Test-NetConnection -ComputerName raw.githubusercontent.com -Port 443`

3. **Extension not configured**
   - Verify script URL in host pool deployment settings
   - Re-run deployment with correct URL

**Check Extension Logs on VM:**
```powershell
# RDP into VM and run:
Get-Content "C:\WindowsAzure\Logs\Plugins\Microsoft.Compute.CustomScriptExtension\*\CommandExecution.log"

# Or view last 20 lines:
Get-Content "C:\WindowsAzure\Logs\Plugins\Microsoft.Compute.CustomScriptExtension\*\CommandExecution.log" -Tail 20
```

---

### Issue: Script Execution Failed

**Symptom:** Extension status shows "Provisioning failed" or "Transitioned to failed state"

**Causes & Fixes:**

1. **PowerShell execution policy**
   - Command line should use: `-ExecutionPolicy Unrestricted`
   ```powershell
   powershell.exe -ExecutionPolicy Unrestricted -File Deploy-AVDHost.ps1
   ```

2. **Script syntax error**
   - Test script locally on Windows VM first
   - Check logs for specific PowerShell error message
   - Validate JSON in ARM template

3. **Script times out**
   - Default timeout: 90 minutes
   - Long-running operations may exceed limit
   - Break into smaller, faster scripts if needed
   - Increase timeout via extension handler settings

4. **Insufficient privileges**
   - Script runs as SYSTEM (should have full privileges)
   - If permission denied, check file/registry permissions
   - Verify Windows Defender/antivirus not blocking script execution

**Check Script Output:**
```powershell
# View extension status and error message
Get-AzVm -ResourceGroupName "rg-avd-prod-01" -Name "vm-pool-001" `
    -Status | Select-Object -ExpandProperty Extensions

# View detailed logs
Get-Content "C:\WindowsAzure\Logs\Plugins\Microsoft.Compute.CustomScriptExtension\*\*.log"
```

---

### Issue: Download Failed (403, 404)

**Symptom:** Extension logs show "403 Forbidden" or "404 Not Found" when downloading script

**Causes & Fixes:**

1. **URL not found**
   - Test URL in browser - should show raw script content
   - Verify script exists in repository at that path
   - Check branch name (main, develop, etc.)
   - Verify no typos in path

2. **Network blocking GitHub**
   - Firewall rule blocking raw.githubusercontent.com
   - Proxy intercepting HTTPS
   - ISP/network provider blocking GitHub CDN
   - Test from VM: `Invoke-WebRequest -Uri "https://raw.githubusercontent.com/aidrak/azure-scripts/main/Deploy-AVDHost.ps1"`

3. **Repository private or deleted**
   - Verify repository is public: [github.com/aidrak/azure-scripts](https://github.com/aidrak/azure-scripts)
   - Check if script file still exists

---

## Creating Custom Scripts

Scripts in the `aidrak/azure-scripts` repository follow this template:

```powershell
<#
.SYNOPSIS
    Brief description of what the script does

.DESCRIPTION
    Longer description explaining the purpose and what operations are performed

.NOTES
    Author: Your Name
    Date: YYYY-MM-DD
    Version: 1.0
#>

# Set strict error handling
$ErrorActionPreference = "Stop"
$ProgressPreference = "SilentlyContinue"

# Log function
function Write-Log {
    param([string]$Message, [string]$Level = "INFO")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logEntry = "[$timestamp] [$Level] $Message"
    Write-Output $logEntry

    # Also log to file for post-deployment review
    $logPath = "C:\AVDDeployment.log"
    Add-Content -Path $logPath -Value $logEntry -ErrorAction SilentlyContinue
}

Write-Log "=== Script Starting ==="

try {
    # Your configuration logic here

    Write-Log "=== Script Completed Successfully ===" -Level "SUCCESS"
    exit 0
}
catch {
    Write-Log "Error: $_" -Level "ERROR"
    exit 1
}
```

---

## Related References

- [[../Deployment/09-session-hosts|Step 09: Session Host Provisioning]] - Where scripts are configured
- [[../Images/golden-image-process|Golden Image Creation]] - Alternative: Install software in image
- [[../Intune/application-deployment-with-intune|Application Deployment with Intune]] - Alternative: Deploy apps via Intune
- [[../Intune/remediation-scripts/index|Intune Proactive Remediation Scripts]] - Continuous configuration management
- [Azure Custom Script Extension for Windows - Microsoft Learn](https://learn.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-windows)
- [GitHub aidrak/azure-scripts Repository](https://github.com/aidrak/azure-scripts)

---

**Last Updated:** December 15, 2025
