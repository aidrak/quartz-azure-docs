---
title: Azure Compute Gallery
description: 
published: true
date: 2025-12-14T04:53:05.441Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:17.647Z
---

# Azure Compute Gallery

Azure Compute Gallery (formerly Shared Image Gallery) is Microsoft's centralized repository for managing and distributing virtual machine images across your organization. For Azure Virtual Desktop deployments, the Compute Gallery is essential for maintaining consistent, version-controlled golden images that can be replicated across multiple regions and shared with specific teams or subscriptions.

## What is Azure Compute Gallery

Azure Compute Gallery provides a structured, enterprise-grade approach to image management in Azure. Unlike standalone managed images or VHDs, the Compute Gallery introduces a three-tier hierarchical model that separates the logical definition of an image from its physical versions, enabling sophisticated version control, regional replication, and granular access control.

The Compute Gallery acts as a container for image definitions, which in turn contain versioned snapshots of fully configured virtual machines. This architecture allows you to maintain multiple generations of an image (for example, monthly security updates) while preserving the ability to roll back to previous versions if issues arise.

For MSPs managing multiple client environments, a single Compute Gallery can host image definitions for different customer workloads, with RBAC policies ensuring that each client can only access their authorized images. The gallery also supports cross-subscription and cross-tenant sharing, making it ideal for hub-and-spoke topologies where a central IT team maintains images for multiple business units.

## Key Concepts

### Three-Tier Hierarchy

The Compute Gallery uses a three-level structure that separates image metadata from actual image data:

**Gallery (Top Level):** The container resource that holds all your image definitions. A gallery exists in a specific Azure region and resource group, but its contents can be replicated to other regions. Our example gallery `avd_gallery` resides in the `RG-Azure-VDI-01` resource group.

**Image Definition (Middle Level):** A logical grouping that defines the characteristics shared by all versions of an image. This includes the operating system type (Windows/Linux), OS state (generalized/specialized), Hyper-V generation (Gen1/Gen2), and publisher information. For example, our `Win11_Multi_25H2_Gen2` definition specifies a generalized Windows 11 multi-session Gen2 image published internally.

**Image Version (Bottom Level):** A specific snapshot captured at a point in time. Versions follow semantic versioning (major.minor.patch) and contain the actual disk data. Our `Win11_Multi_25H2_Gen2` definition has three versions: 1.0.0 (initial release), 1.0.1 (first update), and 1.0.2 (latest). Each version can be replicated to different regions independently.

### Replication and Regional Availability

One of the Compute Gallery's most powerful features is automatic regional replication. When you create an image version, you specify target regions and replica counts. Azure handles the background replication, ensuring that session hosts in any region can quickly deploy from a local copy of the image rather than transferring gigabytes across regions.

**Replica Count:** Each region can host multiple replicas of an image version. Higher replica counts improve deployment parallelism—if you're spinning up 50 session hosts simultaneously, 5 replicas prevent throttling. The formula is roughly: `replicas = ceil(concurrent_deployments / 10)`.

**Replication Latency:** Initial replication typically completes within 30-60 minutes for a 127GB Windows 11 image, but can vary based on region pair distances. Check replication status before attempting deployments to new regions.

**Storage Costs:** Each replica consumes standard storage in the target region. A 127GB image with 3 replicas in 2 regions = ~762GB of billable storage. Zone-redundant storage (ZRS) increases costs but improves resilience.

### RBAC and Access Control

Compute Galleries support Azure RBAC at multiple scopes, enabling fine-grained access control:

**Gallery Level:** Assign `Reader` role to allow users to browse all image definitions and versions. Assign `Contributor` to allow creating new definitions and versions.

**Image Definition Level:** Restrict specific teams to only their relevant images. For example, the finance department can only see `Win11_Finance_Apps_Gen2`.

**Image Version Level:** Lock down production images to prevent accidental deletion. Assign `Reader` on version 1.0.2 (production) but `Contributor` on 1.0.3 (testing).

**Sharing Across Subscriptions:** Use community galleries (public sharing), direct RBAC sharing (private, controlled), or managed application galleries (ISV scenarios). For MSP environments, direct RBAC sharing to client subscriptions is most common.

### Hyper-V Generation

All modern Azure VMs use Gen2 Hyper-V firmware, which provides UEFI boot, Secure Boot, and vTPM support. Our image definitions (`Win11_Multi_25H2_Gen2` and `Win11_Single_25H2_Gen2`) are Gen2-only.

**Why Gen2 Matters:**
- **Security:** Trusted Launch requires Gen2 (vTPM for BitLocker, Secure Boot for rootkit protection)
- **Performance:** UEFI boot is faster than legacy BIOS
- **VM Sizes:** Newer Azure VM SKUs (Dv5, Ev5, Dasv5) only support Gen2
- **Migration Warning:** You cannot convert a Gen1 image to Gen2—you must rebuild from scratch

**Gen1 Legacy:** Only needed for ancient applications with BIOS dependencies or specific VM sizes (A-series). Avoid for new deployments.

## How to Configure Azure Compute Gallery

### Creating a Compute Gallery (Portal)

**Azure Portal Path:** Azure Portal → Create a resource → Compute Gallery

1. **Basics Tab:**
   - **Subscription:** Select your Azure subscription
   - **Resource Group:** Use your AVD resource group (e.g., `RG-Azure-VDI-01`)
   - **Name:** Follow naming convention `<prefix>_gallery` (e.g., `avd_gallery`). Must be unique within the resource group. Lowercase, alphanumeric, underscores only.
   - **Region:** Choose your primary AVD region (e.g., East US 2). The gallery metadata lives here, but image versions can replicate anywhere.
   - **Description (optional):** "Production AVD golden images for multi and single-session deployments"

2. **Sharing Tab:**
   - **Default:** RBAC (Role-based access control) - keeps sharing within your tenant via standard Azure roles
   - **Private:** Share with specific subscriptions or tenants (requires manual approval)
   - **Community:** Public sharing (not recommended for enterprise images)

3. **Tags (recommended):**
   - `Environment: Production`
   - `Workload: AVD`
   - `ManagedBy: Infrastructure-Team`
   - `CostCenter: IT-Operations`

4. **Review + Create:** Deployment completes in under 30 seconds.

### Creating an Image Definition (Portal)

**Azure Portal Path:** Azure Portal → Compute Galleries → `avd_gallery` → Create → VM image definition

1. **Basics Tab:**
   - **VM image definition name:** Use descriptive naming: `Win11_Multi_25H2_Gen2` or `Win11_Single_25H2_Gen2`
   - **Region:** Inherits from gallery (read-only)
   - **Publisher:** `Internal` (or your company name)
   - **Offer:** `Windows-11-AVD` (logical grouping of related SKUs)
   - **SKU:** `25H2-Multi-Session` or `25H2-Single-Session`

2. **Operating System Tab:**
   - **OS type:** Windows
   - **OS state:** Generalized (sysprep has been run)
   - **VM generation:** Gen 2 (required for Trusted Launch and modern VM sizes)
   - **Security type:**
     - **Standard:** Basic security (no Secure Boot/vTPM)
     - **Trusted Launch:** Recommended for AVD (enables Secure Boot, vTPM, and integrity monitoring)
     - **Confidential:** For sensitive workloads requiring encrypted memory (AMD SEV-SNP)

3. **Recommended configuration:**
   - **vCPUs:** 4-8 (guides VM size selection during deployment)
   - **Memory:** 16-32GB (appropriate for AVD session hosts)
   - **Description:** "Windows 11 Enterprise multi-session 25H2 with FSLogix, M365 Apps, and monitoring agents"

4. **Lifecycle Policy (optional):**
   - **Enable version expiration:** Set older versions to expire after 90 days
   - **Enable safe delete:** Prevent deletion if a VM is using this image

5. **Review + Create:** Definition is created instantly (no image data yet).

### Creating an Image Version (Manual Capture)

**Prerequisite:** A generalized VM (sysprep completed, VM deallocated)

**Azure Portal Path:** Azure Portal → Compute Galleries → `avd_gallery` → Image Definition → Create version

1. **Basics Tab:**
   - **Version number:** Semantic versioning `1.0.0` (first release), `1.0.1` (patch), `1.1.0` (minor update), `2.0.0` (major change)
   - **Exclude from latest:** Uncheck (this version becomes the default for deployments)
   - **Source:**
     - **Disks and/or snapshots:** Recommended (select OS disk from deallocated VM)
     - **Managed image:** Legacy option
     - **Version:** Clone from existing version
     - **Storage blob:** For offline imports

2. **Source Details:**
   - **Source type:** Managed disk
   - **OS disk:** Select the generalized VM's OS disk (e.g., `AVD-Template-01_OsDisk`)
   - **Data disks (optional):** Select if your image includes additional drives

3. **Replication Tab:**
   - **Default replica count:** 3 (supports ~30 concurrent deployments per region)
   - **Target regions:** Add all regions where you'll deploy session hosts
     - **East US 2:** 3 replicas (primary)
     - **West US 2:** 2 replicas (DR site)
   - **Storage account type:**
     - **Standard HDD:** Cheapest ($0.05/GB/month), slower replication
     - **Premium SSD:** Fastest replication, 3x cost ($0.15/GB/month)
     - **Zone-redundant storage:** Adds 25% cost but protects against zone failures

4. **Encryption Tab:**
   - **Default:** Platform-managed keys (recommended for most scenarios)
   - **Customer-managed keys:** For regulatory compliance (requires Azure Key Vault)

5. **Review + Create:** Replication begins immediately. Monitor progress in the gallery overview.

### Creating an Image Version (Azure Image Builder)

For automated, repeatable builds, use Azure Image Builder templates instead of manual capture. See the [[azure-image-builder]] page for detailed configuration.

## Our Environment Examples

### Gallery Structure: `avd_gallery`

```
avd_gallery (RG-Azure-VDI-01, East US 2)
├── Win11_Multi_25H2_Gen2
│   ├── 1.0.0 (Initial release - Nov 2024)
│   ├── 1.0.1 (Security updates - Dec 2024)
│   └── 1.0.2 (Application updates - Jan 2025)
└── Win11_Single_25H2_Gen2
    └── 1.0.0 (Initial release - Nov 2024)
```

### Image Definition Details

**Win11_Multi_25H2_Gen2:**
- **Publisher:** Internal
- **Offer:** Windows-11-AVD
- **SKU:** 25H2-Multi-Session
- **OS:** Windows 11 Enterprise multi-session (build 26100)
- **Generation:** Gen2 (UEFI, Secure Boot enabled)
- **Security Type:** Trusted Launch
- **Use Case:** Pooled host pools (multiple users per session host)
- **Replicated To:** East US 2, West US 2
- **Template Source:** AVD-Win11-25H2-Multi-Template-01

**Win11_Single_25H2_Gen2:**
- **Publisher:** Internal
- **Offer:** Windows-11-AVD
- **SKU:** 25H2-Single-Session
- **OS:** Windows 11 Enterprise (build 26100)
- **Generation:** Gen2 (UEFI, Secure Boot enabled)
- **Security Type:** Trusted Launch
- **Use Case:** Personal host pools (one user per session host)
- **Replicated To:** East US 2
- **Template Source:** AVD-Win11-25H2-Single-Template-01

## Naming Conventions

Consistent naming prevents confusion in large environments with multiple image definitions.

### Gallery Naming

**Pattern:** `<prefix>_gallery` or `acg-<workload>-<environment>`

**Examples:**
- `avd_gallery` (our environment)
- `acg-avd-prod`
- `acg-vdi-shared`

**Avoid:** Spaces, special characters, excessively long names (50 char limit)

### Image Definition Naming

**Pattern:** `<OS>_<Edition>_<Version>_<Gen>`

**Examples:**
- `Win11_Multi_25H2_Gen2` (our multi-session image)
- `Win11_Single_25H2_Gen2` (our single-session image)
- `Win10_Multi_22H2_Gen2` (legacy Windows 10)
- `Ubuntu_2204_LTS_Gen2` (Linux example)

**Components:**
- **OS:** Win11, Win10, RHEL, Ubuntu
- **Edition:** Multi (multi-session), Single (single-session), Ent (Enterprise), Pro
- **Version:** 25H2, 24H2, 2204 (Ubuntu 22.04)
- **Gen:** Gen2 (always use Gen2 for new images)

### Version Number Strategy

**Semantic Versioning:** `MAJOR.MINOR.PATCH`

- **MAJOR (1.x.x → 2.x.x):** OS version change (Win10 → Win11, 24H2 → 25H2)
- **MINOR (1.0.x → 1.1.x):** Application updates, new software installed
- **PATCH (1.0.0 → 1.0.1):** Security updates, configuration tweaks, hotfixes

**Examples:**
- `1.0.0` - Initial Windows 11 25H2 image with FSLogix and M365 Apps
- `1.0.1` - January 2025 security updates applied
- `1.0.2` - February 2025 security updates + FSLogix 2.9.9000 upgrade
- `1.1.0` - Added Adobe Acrobat and company VPN client
- `2.0.0` - Upgraded to Windows 11 26H2

## Best Practices

**Use Gen2 for all new images** - Gen1 is legacy and blocks Trusted Launch security features. All modern Azure VM sizes require Gen2. The minimal storage savings of Gen1 (no EFI partition) are not worth the technical debt.

**Enable Trusted Launch** - Adds Secure Boot, vTPM, and boot integrity monitoring with negligible performance impact. Required for compliance frameworks (CIS, NIST) and protects against bootkits and rootkits.

**Replicate to DR regions proactively** - Don't wait for a disaster to discover that your images aren't available in the failover region. Add your secondary region during initial version creation, not during an outage.

**Set replica counts based on concurrency** - If you deploy 20 session hosts simultaneously, use at least 2 replicas (rule of thumb: 1 replica per 10 concurrent deployments). Monitor for throttling errors during large-scale deployments and increase replicas if needed.

**Tag image versions with deployment metadata** - Add tags like `BuildDate: 2025-01-15`, `ApprovedBy: John.Doe`, `PatchTuesday: Jan-2025`. This makes auditing and change tracking much easier than relying on descriptions.

**Implement a retention policy** - Keep the latest 3-5 versions and delete older ones unless they're pinned as "known good" rollback points. Each version consumes storage costs across all replicated regions.

**Use separate galleries for dev/test/prod** - Prevents accidental deployment of untested images to production. Apply stricter RBAC and lifecycle policies to the production gallery.

**Document what's in each version** - Maintain a changelog in the image definition's description or in an external system (Azure DevOps, ServiceNow). Include software versions, applied patches, and configuration changes.

**Test replication before deployments** - After creating a new version, verify that replication to all target regions completes successfully. Check the gallery overview for "Replication state: Succeeded" before attempting deployments.

**Don't delete the source VM immediately** - Keep the generalized VM for 24-48 hours after creating an image version, in case you need to recapture or troubleshoot. Once the version is validated, you can safely delete the source.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Image version creation fails with "Disk not found"** | Source VM is still running or not deallocated | Stop and deallocate the VM (`az vm deallocate`) before capturing. Verify VM status is "Stopped (deallocated)" not just "Stopped". |
| **Deployments fail with "Image not found in region"** | Replication to target region incomplete or failed | Check gallery → image definition → version → "Replication state". Wait for "Succeeded" status or manually trigger replication to missing region. |
| **New VMs fail to boot with "No bootable device"** | Image is Gen2 but VM created as Gen1 (or vice versa) | Ensure VM deployment specifies `--gen2` flag or verify ARM template uses `"generation": "V2"`. Cannot convert between generations—must recreate VM. |
| **"Image version cannot be deleted" error** | VMs or VM scale sets are using this version | Run `az sig image-version show --expand ReplicationStatus` to find resources using the version. Deallocate or delete dependent VMs first. |
| **Replication stuck at "In Progress" for hours** | Network throttling, large image size, or regional capacity issues | Check Azure Service Health for regional outages. For images >500GB, replication can take 4+ hours. Consider using Premium SSD storage accounts for faster replication. |
| **High storage costs for gallery** | Too many versions or excessive replica counts | Audit versions: delete unused versions, reduce replicas in low-usage regions. Enable lifecycle policies to auto-expire old versions. |
| **RBAC sharing not working across subscriptions** | Missing `Microsoft.Compute/galleries/share/action` permission | Ensure the target subscription has at least `Reader` role on the gallery resource. For cross-tenant sharing, use Azure Lighthouse or direct gallery sharing. |
| **Image version deploy slower than expected** | Replicas exhausted due to high concurrency | Increase replica count in the target region. Monitor Azure Metrics for "Gallery Image Version Replica Throttling" alerts. |
| **Cannot create Trusted Launch VMs from image** | Image definition has `securityType: Standard` | Trusted Launch must be configured at image definition creation. Cannot change security type retroactively—create new definition. |
| **Sysprep failed, cannot generalize VM** | Pending Windows updates, failed apps, or incorrect unattend.xml | Review `C:\Windows\System32\Sysprep\Panther\setuperr.log` for errors. Ensure all Windows updates completed before sysprep. See [[golden-image-process]] for sysprep best practices. |

## Next Steps

- **[[golden-image-process]]** - Learn how to manually create and capture golden images
- **[[azure-image-builder]]** - Automate image builds with repeatable templates
- **[[image-versioning]]** - Strategies for version management and rollback procedures