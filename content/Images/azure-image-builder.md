---
title: Azure Image Builder
description: 
published: true
date: 2025-12-14T04:53:05.441Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:19.281Z
---

# Azure Image Builder

Azure Image Builder (AIB) is Microsoft's native service for automating the creation of customized virtual machine images. Instead of manually building and sysprepping VMs every month, Image Builder uses declarative JSON templates to reproducibly install software, apply configurations, and distribute images to Azure Compute Galleries. For production AVD environments, AIB is the recommended approach—it eliminates human error, provides full audit trails, and integrates seamlessly with CI/CD pipelines.

## What is Azure Image Builder

Azure Image Builder is a managed Azure service that orchestrates the entire image build lifecycle:
1. Provisions a temporary build VM in a managed resource group
2. Applies customizations (PowerShell scripts, Windows Update, file copies, etc.)
3. Generalizes the VM with sysprep automatically
4. Captures the image to Azure Compute Gallery or as a managed image/VHD
5. Cleans up the build VM and all temporary resources

All build operations are defined in an **Image Template** JSON file, which acts as infrastructure-as-code for your golden images. Templates are versioned in source control (Git), enabling change tracking, peer review, and rollback capabilities that manual processes cannot provide.

Image Builder runs in Microsoft's Azure backend—you don't manage the build VM, install agents, or troubleshoot sysprep failures manually. Build logs are streamed to Azure Storage, and the service handles retries, error recovery, and parallel builds across regions.

## Benefits Over Manual Capture

### Repeatability
Manual image builds suffer from "works on my machine" syndrome. Different technicians install software in different orders, skip steps, or apply inconsistent configurations. With Image Builder, the same template produces identical images every time—whether it's run by Alice, Bob, or a scheduled GitHub Action.

**Example Scenario:** Your FSLogix agent is mysteriously missing from session hosts deployed in March but present in February builds. With manual captures, you rely on memory and incomplete documentation. With Image Builder, you diff the Git commit history to see exactly when `Install-FSLogix.ps1` was removed from the template.

### Version Control
Image Builder templates are JSON files stored in Git. Every change has a commit hash, author, timestamp, and description. You can branch for testing (feature/win11-26h2), tag releases (v1.0.2), and revert to known-good configurations with `git checkout`.

Manual processes rely on documentation (often outdated) or tribal knowledge ("I think we started including Adobe in version 1.0.1?"). Automated processes have single-source-of-truth templates that answer "What's in this image?" definitively.

### Automation
Trigger builds automatically on a schedule (monthly patch Tuesday), when code changes (CI/CD pipeline), or via API calls (ServiceNow change requests). No need to remember to update images manually—the system enforces the cadence.

**Integration Examples:**
- **Azure DevOps Pipeline:** Trigger AIB template submission when `image-template.json` changes in main branch
- **GitHub Actions:** Run nightly builds to catch Windows Update regressions early
- **Azure Logic Apps:** Trigger build on second Tuesday of each month (Patch Tuesday + 24 hours)

### Audit and Compliance
Image Builder logs every action to Azure Storage (customizer execution, stdout/stderr, sysprep logs). Compliance audits can trace exactly what was installed, when, by whom, and with what result. This level of auditability is impossible with manual RDP-based builds.

Manual builds: "I think Jim built the image on January 15th using the checklist... probably."
Image Builder: "Template commit SHA abc123 by alice@company.com on 2025-01-15T14:32:00Z, build run ID xyz789, logs at <storage URL>, SUCCEEDED."

### Parallel Builds
Image Builder can build multiple image variants simultaneously. Need both multi-session and single-session versions? Two templates run in parallel, completing in the time it takes to build one manually.

### Error Handling
If a manual build fails during app installation, you must troubleshoot, fix, and restart from scratch. Image Builder retries transient failures (network timeouts, Windows Update hiccups) and provides granular error messages pointing to the exact customizer step that failed.

## Image Template Structure

An Image Template is a JSON document with four main sections: **source**, **customize**, **distribute**, and **build properties**.

### Source

Defines the base image to start from. Options:
- **Azure Marketplace image:** Start from Microsoft's official Windows 11 multi-session
- **Existing Compute Gallery image:** Build on top of a previous version (layered builds)
- **Managed image:** Legacy option (not recommended)

**Example (Marketplace):**

```json
{
  "source": {
    "type": "PlatformImage",
    "publisher": "MicrosoftWindowsDesktop",
    "offer": "Windows-11",
    "sku": "win11-25h2-avd",
    "version": "latest"
  }
}
```

**Example (Existing Gallery Image):**

```json
{
  "source": {
    "type": "SharedImageVersion",
    "imageVersionId": "/subscriptions/{subscriptionId}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Compute/galleries/avd_gallery/images/Win11_Multi_25H2_Gen2/versions/1.0.1"
  }
}
```

**Best Practice:** Use Marketplace for major OS version changes (Win10 → Win11, 24H2 → 25H2). Use existing gallery images for incremental updates (monthly patches, minor app updates) to reduce build time.

### Customizers

Customizers are the actions applied to the base image. They execute in order, like a script. Common types:

#### PowerShell Customizer

Runs inline PowerShell code or script files from GitHub/Azure Storage.

**Inline Example:**

```json
{
  "type": "PowerShell",
  "name": "Install-FSLogix",
  "inline": [
    "Write-Host '[START] Installing FSLogix'",
    "$url = 'https://aka.ms/fslogix-latest'",
    "Invoke-WebRequest -Uri $url -OutFile C:\\Temp\\FSLogix.zip",
    "Expand-Archive -Path C:\\Temp\\FSLogix.zip -DestinationPath C:\\Temp\\FSLogix",
    "Start-Process -FilePath 'C:\\Temp\\FSLogix\\x64\\Release\\FSLogixAppsSetup.exe' -ArgumentList '/install', '/quiet', '/norestart' -Wait",
    "Write-Host '[SUCCESS] FSLogix installed'"
  ],
  "runElevated": true,
  "runAsSystem": false
}
```

**Script File Example:**

```json
{
  "type": "PowerShell",
  "name": "Install-M365Apps",
  "scriptUri": "https://raw.githubusercontent.com/company/avd-scripts/main/Install-Office365.ps1",
  "runElevated": true
}
```

**Parameters:**
- `runElevated: true` - Runs as Administrator (required for most installations)
- `runAsSystem: true` - Runs as SYSTEM account (use sparingly, for low-level configurations)
- `validExitCodes: [0, 3010]` - Accept reboot codes as success (3010 = reboot required)

#### File Customizer

Copies files from a URL to the build VM.

```json
{
  "type": "File",
  "name": "Copy-CompanyCA",
  "sourceUri": "https://company.blob.core.windows.net/certs/CompanyRootCA.cer",
  "destination": "C:\\Temp\\CompanyRootCA.cer"
}
```

**Use Cases:** Certificate files, application installers, configuration templates.

#### Windows Update Customizer

Applies Windows Updates during the build (recommended for monthly patch cycles).

```json
{
  "type": "WindowsUpdate",
  "searchCriteria": "IsInstalled=0",
  "filters": [
    "exclude:$_.Title -like '*Preview*'"
  ],
  "updateLimit": 100
}
```

**Parameters:**
- `searchCriteria: "IsInstalled=0"` - Only install missing updates
- `filters` - Exclude preview updates, specific KB numbers, or categories
- `updateLimit` - Max updates to install in one pass (prevents timeout on major feature updates)

**Warning:** Windows Update customizer adds 20-40 minutes to build time. Consider running monthly update builds separately from application update builds.

#### Windows Restart Customizer

Reboots the build VM (required after some updates or app installations).

```json
{
  "type": "WindowsRestart",
  "restartCheckCommand": "echo 'Reboot complete'",
  "restartTimeout": "10m"
}
```

**Best Practice:** Add a restart after Windows Update customizer and after any app that requires a reboot (e.g., .NET Framework).

### Distribute

Defines where to publish the finished image. Options:
- **Azure Compute Gallery:** Recommended for AVD (supports versioning, replication, RBAC)
- **Managed Image:** Legacy single-region image
- **VHD:** Export as VHD file to storage account (rare, for hybrid scenarios)

**Compute Gallery Example:**

```json
{
  "distribute": [
    {
      "type": "SharedImage",
      "galleryImageId": "/subscriptions/{subscriptionId}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Compute/galleries/avd_gallery/images/Win11_Multi_25H2_Gen2",
      "runOutputName": "Win11_Multi_25H2_1.0.2",
      "artifactTags": {
        "source": "Azure Image Builder",
        "buildDate": "2025-01-20",
        "patchTuesday": "Jan-2025"
      },
      "replicationRegions": [
        "East US 2",
        "West US 2"
      ],
      "storageAccountType": "Premium_LRS",
      "excludeFromLatest": false
    }
  ]
}
```

**Parameters:**
- `galleryImageId` - Target image definition (must exist before running template)
- `runOutputName` - Friendly name for this build (appears in AIB logs)
- `artifactTags` - Metadata tags for the image version
- `replicationRegions` - Where to replicate the image (must be supported by gallery)
- `storageAccountType` - `Standard_LRS` (cheap), `Premium_LRS` (fast), or `Standard_ZRS` (zone-redundant)
- `excludeFromLatest: false` - This version becomes "latest" for deployments

### Build Properties

Global settings for the build process.

```json
{
  "buildTimeoutInMinutes": 240,
  "vmProfile": {
    "vmSize": "Standard_D4s_v5",
    "osDiskSizeGB": 127,
    "userAssignedIdentities": [
      "/subscriptions/{subscriptionId}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aib-identity"
    ],
    "vnetConfig": {
      "subnetId": "/subscriptions/{subscriptionId}/resourceGroups/RG-Network/providers/Microsoft.Network/virtualNetworks/VNet-AVD/subnets/ImageBuild"
    }
  }
}
```

**Parameters:**
- `buildTimeoutInMinutes` - Max build duration (default 240 = 4 hours). Increase if installing large apps.
- `vmSize` - Build VM size (larger = faster builds, but higher cost)
- `osDiskSizeGB` - OS disk size (127GB for Windows 11, 256GB if installing large apps like Visual Studio)
- `userAssignedIdentities` - Managed identity for AIB to access storage, galleries, etc.
- `vnetConfig.subnetId` - Deploy build VM in a specific subnet (required for private builds, air-gapped networks)

## Our Image Templates

### AVD-Win11-25H2-Multi-Template-01

**Purpose:** Production multi-session Windows 11 image for pooled host pools.

**Template Highlights:**

```json
{
  "type": "Microsoft.VirtualMachineImages/imageTemplates",
  "apiVersion": "2022-07-01",
  "location": "East US 2",
  "identity": {
    "type": "UserAssigned",
    "userAssignedIdentities": {
      "/subscriptions/{subscriptionId}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aib-identity": {}
    }
  },
  "properties": {
    "source": {
      "type": "PlatformImage",
      "publisher": "MicrosoftWindowsDesktop",
      "offer": "Windows-11",
      "sku": "win11-25h2-avd",
      "version": "latest"
    },
    "customize": [
      {
        "type": "PowerShell",
        "name": "Install-FSLogix",
        "scriptUri": "https://company.blob.core.windows.net/scripts/Install-FSLogix.ps1",
        "runElevated": true
      },
      {
        "type": "PowerShell",
        "name": "Install-M365Apps",
        "scriptUri": "https://company.blob.core.windows.net/scripts/Install-Office365.ps1",
        "runElevated": true
      },
      {
        "type": "PowerShell",
        "name": "Install-AdobeReader",
        "scriptUri": "https://company.blob.core.windows.net/scripts/Install-AdobeReader.ps1",
        "runElevated": true
      },
      {
        "type": "WindowsUpdate",
        "searchCriteria": "IsInstalled=0",
        "filters": ["exclude:$_.Title -like '*Preview*'"],
        "updateLimit": 100
      },
      {
        "type": "WindowsRestart",
        "restartTimeout": "10m"
      },
      {
        "type": "PowerShell",
        "name": "Run-VDOT",
        "scriptUri": "https://company.blob.core.windows.net/scripts/Apply-VDOTOptimizations.ps1",
        "runElevated": true
      }
    ],
    "distribute": [
      {
        "type": "SharedImage",
        "galleryImageId": "/subscriptions/{subscriptionId}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Compute/galleries/avd_gallery/images/Win11_Multi_25H2_Gen2",
        "runOutputName": "Win11-Multi-Build",
        "replicationRegions": ["East US 2", "West US 2"],
        "storageAccountType": "Premium_LRS"
      }
    ],
    "buildTimeoutInMinutes": 240,
    "vmProfile": {
      "vmSize": "Standard_D4s_v5",
      "osDiskSizeGB": 127
    }
  }
}
```

**Software Included:**
- Windows 11 Enterprise multi-session 25H2 (marketplace base)
- FSLogix 2.9.9000+
- Microsoft 365 Apps for Enterprise (Monthly Enterprise Channel)
- Adobe Acrobat Reader DC
- Latest Windows security updates
- VDOT optimizations (disabled unnecessary services, scheduled tasks)

**Build Time:** ~90 minutes (30 min base provisioning + 40 min Windows Update + 20 min app installs)

**Trigger:** Manually via Azure CLI or automated via Azure DevOps pipeline on Git commit to `main` branch.

### AVD-Win11-25H2-Single-Template-01

**Purpose:** Production single-session Windows 11 image for personal host pools.

**Differences from Multi-Session Template:**
- **Source SKU:** `win11-25h2-ent` (standard Windows 11 Enterprise, not multi-session)
- **M365 Apps Configuration:** `SharedComputerLicensing=0` (not required for single-user VMs)
- **Distribution Target:** `Win11_Single_25H2_Gen2` image definition

**Use Case:** Personal desktops for executives, developers, or users requiring persistent local profiles (not using FSLogix redirection).

## Triggering Builds

### Manual (Azure CLI)

```bash
# Create or update the image template
az image builder create \
  --resource-group RG-Azure-VDI-01 \
  --name AVD-Win11-25H2-Multi-Template-01 \
  --image-template template.json

# Start the build
az image builder run \
  --resource-group RG-Azure-VDI-01 \
  --name AVD-Win11-25H2-Multi-Template-01

# Monitor build progress
az image builder show-runs \
  --resource-group RG-Azure-VDI-01 \
  --name AVD-Win11-25H2-Multi-Template-01

# Check final status
az image builder show \
  --resource-group RG-Azure-VDI-01 \
  --name AVD-Win11-25H2-Multi-Template-01 \
  --query "lastRunStatus"
```

**Expected Output:**
```json
{
  "endTime": "2025-01-20T16:45:32Z",
  "message": "Image build succeeded",
  "runState": "Succeeded",
  "startTime": "2025-01-20T15:15:00Z"
}
```

### Automated (Azure DevOps Pipeline)

**Pipeline YAML:**

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - image-templates/multi-session.json
      - scripts/*.ps1

pool:
  vmImage: 'ubuntu-latest'

variables:
  resourceGroup: 'RG-Azure-VDI-01'
  templateName: 'AVD-Win11-25H2-Multi-Template-01'
  templateFile: 'image-templates/multi-session.json'

steps:
- task: AzureCLI@2
  displayName: 'Update Image Template'
  inputs:
    azureSubscription: 'Azure-AVD-Prod'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az image builder create \
        --resource-group $(resourceGroup) \
        --name $(templateName) \
        --image-template $(templateFile)

- task: AzureCLI@2
  displayName: 'Trigger Image Build'
  inputs:
    azureSubscription: 'Azure-AVD-Prod'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az image builder run \
        --resource-group $(resourceGroup) \
        --name $(templateName)

- task: AzureCLI@2
  displayName: 'Wait for Build Completion'
  inputs:
    azureSubscription: 'Azure-AVD-Prod'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      while true; do
        status=$(az image builder show \
          --resource-group $(resourceGroup) \
          --name $(templateName) \
          --query "lastRunStatus.runState" -o tsv)

        if [ "$status" == "Succeeded" ]; then
          echo "Build succeeded"
          exit 0
        elif [ "$status" == "Failed" ]; then
          echo "Build failed"
          exit 1
        else
          echo "Build in progress... ($status)"
          sleep 60
        fi
      done
```

**Trigger Logic:** When `image-templates/multi-session.json` or any PowerShell script in `scripts/` changes on the `main` branch, the pipeline automatically rebuilds the image.

### Scheduled (Azure Logic App)

**Use Case:** Rebuild images monthly on Patch Tuesday (second Tuesday of the month).

**Logic App Workflow:**
1. **Trigger:** Recurrence - Monthly, day 8-14, Tuesday only (covers all possible Patch Tuesday dates)
2. **Action:** HTTP - POST to Azure Image Builder API to start template run
3. **Action:** Wait until build completes (poll API every 15 minutes)
4. **Action:** Send email notification to AVD admins with build status

## Build Logs and Troubleshooting

### Accessing Build Logs

Image Builder streams logs to a storage account in the managed resource group (auto-created during builds).

**Find the Managed Resource Group:**

```bash
# List all managed RGs (prefixed with IT_)
az group list --query "[?starts_with(name, 'IT_')].name" -o table

# Example output: IT_RG-Azure-VDI-01_AVD-Win11-25H2-Multi-Template-01_12345678-abcd-1234-abcd-123456789abc
```

**Download Customizer Logs:**

```bash
# Set variables
RG="IT_RG-Azure-VDI-01_AVD-Win11-25H2-Multi-Template-01_12345678-abcd-1234-abcd-123456789abc"
STORAGE_ACCOUNT=$(az storage account list --resource-group $RG --query "[0].name" -o tsv)

# List log blobs
az storage blob list \
  --account-name $STORAGE_ACCOUNT \
  --container-name packerlogs \
  --output table

# Download specific customizer log
az storage blob download \
  --account-name $STORAGE_ACCOUNT \
  --container-name packerlogs \
  --name "customization.log" \
  --file customization.log
```

**Log Contents:**
- `customization.log` - All customizer stdout/stderr output
- `Sysprep.log` - Sysprep execution logs (if build reaches sysprep phase)
- `builder.log` - Image Builder service logs (provisioning, cleanup)

### Common Build Failures

| Issue | Cause | Solution |
|-------|-------|----------|
| **Build fails with "Marketplace image not found"** | Incorrect publisher/offer/sku in source section | Verify marketplace image exists: `az vm image list --publisher MicrosoftWindowsDesktop --offer Windows-11 --all`. Use exact case-sensitive values. |
| **PowerShell customizer fails with exit code 1** | Script error, network timeout, or permission issue | Download `customization.log` and search for the customizer name. Check for error messages. Add `Write-Host` statements to scripts for debugging. |
| **Windows Update customizer times out** | Too many updates (feature update + monthly patches) | Split into two builds: base image with feature update (quarterly), monthly update builds with just security patches. Increase `buildTimeoutInMinutes` to 360. |
| **Build succeeds but image missing from gallery** | Distribution `galleryImageId` incorrect or RBAC issue | Verify image definition exists. Ensure AIB managed identity has `Contributor` role on gallery. Check AIB run output for distribution errors. |
| **Build VM fails to provision** | Quota limit reached or VM size unavailable in region | Check quota: `az vm list-usage --location eastus2`. Request quota increase if needed. Try smaller VM size (`Standard_D2s_v5`). |
| **Customizer fails with "Access denied"** | `runElevated: false` but needs admin rights | Set `runElevated: true` on PowerShell customizer. Verify script doesn't require SYSTEM account (use `runAsSystem: true` only if necessary). |
| **Sysprep fails during build** | Pending updates, AppX packages, or corrupted system | Add Windows Update customizer before sysprep. Use VDOT to remove AppX packages. Check `Sysprep.log` in packerlogs for specific error. |
| **Image replication fails** | Invalid target region or storage type | Ensure target regions are enabled in Azure subscription. Verify gallery supports ZRS if using `Standard_ZRS`. Check Service Health for regional outages. |
| **Build succeeds but takes 4+ hours** | Excessive customizers or network-bound downloads | Optimize scripts: cache downloads in Azure Blob Storage instead of re-downloading from internet. Combine multiple small PowerShell customizers into one. |
| **Managed identity permission errors** | AIB identity missing RBAC on resources | Assign roles: `Contributor` on gallery, `Reader` on source image, `Storage Blob Data Reader` on script storage account. |

### Debugging Tips

**Enable verbose logging in PowerShell customizers:**

```json
{
  "type": "PowerShell",
  "name": "Debug-Install",
  "inline": [
    "$VerbosePreference = 'Continue'",
    "$DebugPreference = 'Continue'",
    "Write-Host '[DEBUG] Starting installation'",
    "# ... rest of script"
  ],
  "runElevated": true
}
```

**Test scripts locally before adding to template:**
- RDP to a fresh marketplace VM
- Run each customizer script manually
- Fix errors, then update template JSON

**Use `validExitCodes` for apps that return non-zero success codes:**

```json
{
  "type": "PowerShell",
  "name": "Install-App",
  "scriptUri": "https://...",
  "runElevated": true,
  "validExitCodes": [0, 3010]  // 3010 = reboot required (success)
}
```

**Check Azure Activity Log for AIB API errors:**

```bash
az monitor activity-log list \
  --resource-group RG-Azure-VDI-01 \
  --resource-type Microsoft.VirtualMachineImages/imageTemplates \
  --start-time 2025-01-20T00:00:00Z \
  --query "[?contains(status.value, 'Failed')]"
```

## Best Practices

**Store templates in Git** - Treat image templates as infrastructure-as-code. Version control enables rollback, peer review, and audit trails. Use branches for testing experimental customizers.

**Modularize PowerShell scripts** - Don't embed 500 lines of PowerShell in `inline` JSON. Use `scriptUri` to reference external scripts stored in Azure Blob Storage or GitHub. This improves readability and allows script reuse across multiple templates.

**Use managed identities, not service principals** - AIB supports user-assigned managed identities for RBAC. This eliminates secret management and rotation overhead compared to service principal credentials.

**Tag image versions with build metadata** - Include tags like `buildDate`, `templateVersion`, `gitCommitSHA` in the `artifactTags` section. This links deployed images back to source code for troubleshooting.

**Test in non-prod gallery first** - Create a separate `avd_gallery_dev` for testing template changes. Only promote validated templates to the production gallery.

**Implement approval gates in CI/CD** - Don't auto-deploy to production. Require manual approval after dev/test builds succeed. Use Azure DevOps Environments or GitHub branch protection rules.

**Monitor build costs** - Each build consumes compute (VM hours), storage (replication), and networking (cross-region traffic). For high-frequency builds, optimize VM size and replica counts to balance speed vs. cost.

**Set realistic timeouts** - Default 240 minutes is sufficient for typical builds. Increase to 360 for feature update builds or large app suites. Don't set to 1440 (24 hours) "just in case"—fix slow customizers instead.

**Cache downloads in Azure Storage** - Instead of `Invoke-WebRequest https://external.com/app.exe`, upload installers to Azure Blob Storage. This reduces build failures from external site downtime and improves download speeds.

**Use Windows Update customizer sparingly** - Only run monthly for security patches. Don't include in every build—it adds 30-60 minutes. For non-security app updates, use PowerShell customizers.

**Exclude preview updates** - `filters: ["exclude:$_.Title -like '*Preview*'"]` prevents untested preview patches from entering production images.

## Next Steps

- **[[image-versioning]]** - Strategies for managing multiple image versions
- **[[golden-image-process]]** - Manual process (useful for understanding what AIB automates)
- **[[azure-compute-gallery]]** - Gallery structure and configuration