---
title: Images Index
description: Overview of Azure Virtual Desktop golden image management and creation
published: true
date: 2025-12-14T04:53:00.000Z
tags:
editor: markdown
dateCreated: 2025-12-14T04:53:00.000Z
---

# Images

This section covers the creation, management, and deployment of golden images for Azure Virtual Desktop. Golden images are fully configured, generalized Windows installations that serve as templates for all session hosts in your host pools.

## What's Covered

- **Golden Image Creation** - Manual process for building and capturing customized Windows images
- **Azure Image Builder Automation** - Infrastructure-as-code templates for repeatable, auditable image builds
- **Azure Compute Gallery** - Centralized repository for storing, versioning, and distributing images across regions
- **Image Versioning Strategies** - Semantic versioning and rollout procedures for managing updates
- **Operating System Selection** - Choosing between Windows 11 multi-session and single-session editions

## Pages

### [[golden-image-process]]
Learn the manual process for creating a golden image. Covers VM provisioning, application installation, sysprep generalization, and capture to Azure Compute Gallery. Essential foundation knowledge for understanding automated build tools.

**Key Topics:**
- Creating base VMs from Azure Marketplace
- Installing applications and configuring settings
- Running sysprep to generalize the image
- Capturing to gallery with proper versioning

### [[azure-image-builder]]
Automate image creation using Azure Image Builder (AIB) and infrastructure-as-code JSON templates. Covers template structure, customizers, distribution configuration, CI/CD integration, and troubleshooting.

**Key Topics:**
- Image template structure (source, customize, distribute)
- PowerShell, Windows Update, and file customizers
- Automated regional replication
- Integration with Azure DevOps and GitHub Actions
- Build logs and failure debugging

### [[azure-compute-gallery]]
Deep-dive into the hierarchical structure of Azure Compute Gallery (Gallery → Image Definition → Image Version). Covers RBAC, regional replication, replica counts, and best practices for enterprise image management.

**Key Topics:**
- Three-tier hierarchy and naming conventions
- Replication strategies and replica counts
- Cross-subscription and cross-region sharing
- Trusted Launch and Hyper-V generation configuration
- Storage costs and performance optimization

### [[image-versioning]]
Strategies for managing multiple image versions using semantic versioning (MAJOR.MINOR.PATCH). Covers when to create new versions, retention policies, rolling updates, and rollback procedures.

**Key Topics:**
- Semantic versioning schema (1.0.0, 1.0.1, etc.)
- Monthly security patches, quarterly feature updates, annual OS upgrades
- Drain mode strategy for updating pooled host pools
- Blue-green deployment for zero-downtime migrations
- Quick rollback and long-term recovery procedures

### [[multi-session-vs-single-session-windows]]
Comprehensive comparison of Windows 11 multi-session (for pooled host pools) versus single-session (for personal desktops). Covers licensing, performance, application compatibility, and cost implications.

**Key Topics:**
- Multi-session architecture and per-user licensing
- Single-session dedicated resources and higher costs
- Application compatibility challenges in multi-user environments
- Performance characteristics and user density recommendations
- Decision matrix for choosing the right OS edition

## Related Quick-Deploy Steps

- **[[Quick-Deploy/03-image-gallery]]** - Step-by-step deployment of Azure Compute Gallery and initial image definition creation

## Typical Workflow

1. **Design** - Determine OS edition (multi/single-session) based on user personas and app compatibility
2. **Build** - Create golden image manually (for learning) or use Azure Image Builder (for production)
3. **Capture** - Generalize with sysprep and capture to Azure Compute Gallery with semantic version
4. **Version** - Maintain multiple versions following MAJOR.MINOR.PATCH strategy
5. **Distribute** - Roll out updated images to session hosts using drain mode or blue-green deployment
6. **Retire** - Delete old versions per retention policy, keeping at least 2 previous versions for rollback

## Key Concepts

**Golden Image** - A fully configured, sysprepped Windows installation that serves as a template for cloning to multiple session hosts. Contains OS, patches, applications, monitoring agents, and optimizations.

**Semantic Versioning** - Version numbering scheme (MAJOR.MINOR.PATCH) that communicates change scope. PATCH for security updates, MINOR for new features, MAJOR for OS version changes.

**Sysprep (System Preparation)** - Microsoft's tool that removes computer-specific information (SID, computer name, drivers) to generalize a Windows installation for cloning.

**Azure Compute Gallery** - Enterprise image management service providing hierarchical organization (Gallery → Definition → Version), regional replication, RBAC, and version lifecycle policies.

**FSLogix** - Microsoft's profile virtualization solution that stores user profiles in containers on Azure Files rather than on session host OS disks, enabling stateless sessions and profile portability.

**Image Builder** - Managed Azure service that automates image creation from JSON templates, handles customization (PowerShell, Windows Update), and distributes to galleries automatically.

## Best Practices

- Always apply Windows updates before sysprep to prevent generalization failures
- Use semantic versioning consistently (don't skip version numbers or deviate from MAJOR.MINOR.PATCH)
- Keep at least 3 image versions available (latest, N-1, N-2) for rollback capability
- Test new versions in dev/UAT environments before production deployment
- Implement PATCH versions monthly (Patch Tuesday), MINOR versions quarterly, MAJOR versions annually
- Use Azure Image Builder for production environments, manual builds for learning and troubleshooting
- Tag image versions with metadata (BuildDate, ApprovedBy, GitCommitSHA) for auditability
- Deploy multi-session images for general office workers (cost-optimized)
- Deploy single-session images only for users requiring dedicated resources or app-incompatible applications

## Performance Considerations

**Multi-Session Windows (Pooled Host Pools):**
- User Density: 10-20 users per D4s_v5 VM depending on workload intensity
- CPU, RAM, and disk IOPS are shared; careful capacity planning essential
- Autoscaling can adjust session host count based on active sessions
- Login time: 20-30 seconds (existing session infrastructure)

**Single-Session Windows (Personal Host Pools):**
- One user per VM; no resource contention
- Can deallocate when unused (save 80% compute cost)
- Login time: 60-90 seconds if VM deallocated (VM startup + RDP), 15-20 seconds if running
- 10x higher infrastructure cost than multi-session for same user count
