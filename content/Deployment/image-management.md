---
title: Image Management
description: 
published: true
date: 2025-12-14T04:53:02.173Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:28.352Z
---

# Image Management

Set up Azure Compute Gallery and create or import golden images for session hosts. A well-maintained golden image is the foundation of a reliable AVD deployment.

## Create Azure Compute Gallery

**Portal:** Azure Portal → Azure Compute Gallery → Create

- **Resource group:** RG-Azure-VDI-01
- **Name:** `avd_gallery`
- **Region:** Central US

## Create Image Definition

**Portal:** Gallery → Image definitions → Create

### For Multi-Session (Pooled Desktops)

- **Image definition name:** `Win11_Multi_25H2_Gen2`
- **OS type:** Windows
- **OS state:** Generalized
- **VM generation:** Gen 2
- **Publisher:** Internal
- **Offer:** Windows11-MultiSession
- **SKU:** 25H2

### For Single-Session (Personal Desktops)

- **Image definition name:** `Win11_Single_25H2_Gen2`
- **OS type:** Windows
- **OS state:** Generalized
- **VM generation:** Gen 2
- **Publisher:** Internal
- **Offer:** Windows11-Enterprise
- **SKU:** 25H2

> **Note:** Gen 2 VMs provide faster boot times and support for larger disks. Always use Gen 2 for new deployments.

## Build Golden Image (Manual Method)

### Step 1: Create Base VM

**Portal:** Virtual Machines → Create

- **Image:** Windows 11 Enterprise multi-session + Microsoft 365 Apps, version 24H2
- **Size:** Standard_D4s_v4 (for image building)
- **Virtual network:** vnet-avd
- **Subnet:** snet-management (temporary)

### Step 2: Install Software

Connect via RDP or Bastion and install:

1. **FSLogix Agent** (latest from Microsoft)
2. **Microsoft 365 Apps** (if not in base image)
3. **Company LOB applications**
4. **Monitoring agents** (Azure Monitor Agent)
5. **Any required drivers or tools**

### Step 3: Optimize and Configure

```powershell
# Download and run Virtual Desktop Optimization Tool
# https://github.com/The-Virtual-Desktop-Team/Virtual-Desktop-Optimization-Tool

# Disable unnecessary services
# Configure regional settings
# Remove AppX packages
```

### Step 4: Sysprep and Capture

```powershell
# Run from C:\Windows\System32\Sysprep
sysprep.exe /generalize /oobe /shutdown /mode:vm
```

**Portal:** VM → Capture → Create image

- **Share to Azure Compute Gallery:** Yes
- **Target gallery:** avd_gallery
- **Image definition:** Win11_Multi_25H2_Gen2
- **Version number:** 1.0.0

> **Decision Point:**
> - **Manual image:** Follow steps above
> - **Automated (AIB):** See [[azure-image-builder]] for automated pipelines

## Verification

- [ ] Compute Gallery created
- [ ] Image definitions created (multi + single session)
- [ ] At least one image version available
- [ ] Image replication completed

---

**Next:** [[host-pool-deployment|Step 7: Host Pool Deployment]]