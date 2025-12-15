---
title: Step 03 - Azure Compute Gallery & Golden Images
description: Create and manage Golden Images using Azure Virtual Desktop Custom Image Templates.
published: true
date: 2025-12-14T00:00:00.000Z
tags: [Quick-Deploy, images, gallery, golden-image, AIB]
---

# Step 03: Azure Compute Gallery & Golden Images

Create "Golden Images" using **Custom Image Templates** (Azure Image Builder).

## Purpose
The purpose of this step is to create **Golden Images**, which are used for repeatedly creating new VMs.
*   **Automation:** We use Custom Image Templates to avoid manually logging into servers. The template handles spinning up a VM, running updates/optimizations, and shutting it down.
*   **Reusability:** Templates are stagnant but reusable. To update from 25H2 to a newer version, you simply update the template and re-run it.
*   **Cleanliness:** Ensures a clean build every time without "dirty" artifacts from manual sessions.

## Prerequisites
- [ ] **Azure Compute Gallery** created (e.g., `gal-avd-prod`).
- [ ] **User Managed Identity** created (e.g., `id-avd-imagebuilder`) with Contributor access to the resource group.

---

## Create Custom Image Template

**Navigation:** Search **Azure Virtual Desktop** -> **Custom image templates** -> **+ Add custom image template**.

### Tab 1: Basics

| Setting | Value | Why / Notes |
| :--- | :--- | :--- |
| **Import from existing template** | **No** | We are creating a new fresh template. |
| **Template name** | `tmp-avd-multisession` | Use a name that is copy/paste friendly. |
| **Subscription** | *Select yours* | Target subscription for the template resource. |
| **Resource group** | `rg-avd-prod-01` | Use your AVD infrastructure group. |
| **Location** | **East US** | Match your gallery/AVD region. |
| **Managed Identity** | `id-avd-imagebuilder` | Must be the User Assigned Identity created in prerequisites. |

![[Pasted image 20251214165337.png]]

### Tab 2: Source Image

| Setting | Value | Why / Notes |
| :--- | :--- | :--- |
| **Image source** | **Platform Image (Marketplace)** | We start fresh from an official Microsoft image. |
| **Image type** | *Windows 11 Enterprise multi-session* | Select "multi-session" for pooled, standard Enterprise for personal. |
| **SKU** | `win11-25h2-avd` | Verify the specific version (25H2 or later). |

![[Pasted image 20251214165344.png]]

### Tab 3: Distribution Targets

| Setting | Value | Why / Notes |
| :--- | :--- | :--- |
| **Managed Image** | **No** | This is for legacy, static single-VM captures. We are building a versioned Gallery Image. |
| **Shared Image** | **Yes** | Delivers the build to the Azure Compute Gallery. |
| **Gallery** | `gal-avd-prod-01` | Select your created gallery. |
| **Image definition** | `win11-multisession` | Select the definition created earlier. |
| **Gallery image definition run output name** | *Any Name* | This name **does not matter**; it is an internal artifact name. |
| **Exclude from latest** | **No** | **Critical:** If "Yes", host pools won't see this image when deploying. |

![[Pasted image 20251214165210.png]]

### Tab 4: Build Properties

| Setting | Value | Why / Notes |
| :--- | :--- | :--- |
| **Build timeout** | `120` | Default is usually fine. |
| **VM size** | `Standard_D4s_v5` | Faster VMs build images quicker. |
| **Staging resource group** | **Leave Blank** | **Important:** Let Azure create its own temporary RG. |
| **Staging VNet** | **Leave Blank** | Let Azure create its own temporary VNet. |

> **Note:** When the build is done, the auto-created staging resource group will only contain a log file. You can verify the build and then safely delete this group.

![[Pasted image 20251214165243.png]]

### Tab 5: Customizations

Define what happens inside the VM during the build.

| Customizer | Action | Notes |
| :--- | :--- | :--- |
| **PowerShell / Shell** | **Add** | Add scripts here if needed (e.g., VDOT). |
| **Windows Update** | **Add** | Always add to ensure latest patches. |
| **Restart** | **Add** | Add if a script requires a reboot. |

**Optimization Configuration:**

*   **Script Selection:** Apply defaults.
    *   **Exception:** Uncheck "Enable Kerberos" and "Enable Kerberos Entra ID".
*   **Windows Optimizations:**
    *   **Check All** boxes **EXCEPT** "Network Optimizations".
    *   *Warning:* "Remove OneDrive" is checked by default; uncheck if you need OneDrive.
    *   *Note:* The "LGPO" optimization disables notifications.
*   **Exclusions (Do NOT install here):**
    *   **FSLogix:** Install separately (requires specific path).
    *   **Session Timeouts:** Configure via policy separately.

![[Pasted image 20251214165251.png]]

![[Pasted image 20251214165307.png]]

### Tab 6: Review + Create
Validation will check identity permissions. Click **Create**.

---

## Run the Build
Creating the template only *defines* it. You must **Run** it to generate the image.
1.  Select the template.
2.  Click **Start build**.
3.  Wait for completion (image will appear in your Azure Compute Gallery).

---

## Troubleshooting Build Failures

If the deployment fails, the error message in the notification might be vague.

1.  **Locate the Staging Resource Group:** Look for a resource group named similar to `IT_<ResourceGroup>_<TemplateName>_...` (this is the one you left blank in Step 4).
2.  **Find the Log File:** Inside that group, there will be a storage account containing a container named `packerlogs`.
3.  **Analyze:** Download the `customization.log`. It contains the step-by-step console output of the build process (PowerShell errors, timeout details, etc.).

