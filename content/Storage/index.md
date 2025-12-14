---
title: Storage Reference
description: Azure Files and FSLogix profile container configuration for AVD
published: true
date: 2025-12-14T00:00:00.000Z
tags:
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Storage Reference

This section covers cloud storage and profile management for Azure Virtual Desktop deployments, focusing on Azure Files as the storage backend and FSLogix for user profile roaming. Proper storage configuration is essential for user experience, data consistency, and session host performance.

## What's Covered

**Azure Files** provides the foundational cloud file share service that stores FSLogix profile containers and user data. **FSLogix** is Microsoft's profile management solution that delivers fast, consistent user experiences by mounting profiles as virtual disks rather than copying them at logon/logoff.

Key topics include storage tier selection, permission models using Entra ID authentication, advanced features like Cloud Cache for high availability, and systematic troubleshooting approaches for profile-related issues.

## Pages in This Section

### [[Storage/azure-files-overview|Azure Files Overview]]
Foundation of Azure Virtual Desktop storage. Covers what Azure Files is, storage tiers (Standard, Premium), authentication methods (Entra ID, Storage Keys), and protocol support (SMB, NFS). Essential for understanding the storage layer that hosts FSLogix profile containers.

### [[Storage/fslogix-profile-containers|FSLogix Profile Containers]]
The core profile management solution for AVD. Explains how FSLogix addresses traditional roaming profile limitations, how profile containers work using VHD mounting, and key concepts like user groups and profile locations. Start here to understand profile roaming fundamentals.

### [[Storage/storage-permissions-for-fslogix|Storage Permissions for FSLogix]]
Critical for secure, functional deployments. Details Entra ID-based authentication for Azure Files, RBAC roles, NTFS permissions, and the authentication flow from user logon through file access. Incorrect permissions cause most profile access failures.

### [[Storage/fslogix-cloud-cache|FSLogix Cloud Cache]]
Advanced feature for high availability and disaster recovery. Covers local caching on session hosts, replication to multiple storage providers, and use cases in multi-region deployments. For environments requiring zero-downtime profile access.

### [[Storage/troubleshooting-fslogix-profiles|Troubleshooting FSLogix Profiles]]
Systematic diagnosis of profile issues. Details FSLogix log locations and formats, common failure patterns (permissions, storage connectivity, VHD corruption), and step-by-step resolution approaches. Use when users experience slow logons, missing data, or logon failures.

## Related Quick-Deploy Steps

**[[../Quick-Deploy/05-storage-fslogix|Step 05 - Storage & FSLogix Setup]]**
Linear walkthrough for creating the Azure Files storage account and configuring FSLogix profile containers. Links to relevant reference pages for deeper understanding at each decision point.

**[[../Quick-Deploy/09-fslogix-configuration|Step 09 - FSLogix Configuration via Intune]]**
Deployment of FSLogix settings via Intune Settings Catalog to configure registry paths, user groups, and Cloud Cache options on session hosts.

## Implementation Path

1. Start with [[Storage/azure-files-overview|Azure Files Overview]] to select appropriate storage tiers
2. Follow [[../Quick-Deploy/05-storage-fslogix|Step 05]] for hands-on setup
3. Reference [[Storage/storage-permissions-for-fslogix|Storage Permissions for FSLogix]] when configuring access
4. Use [[../Quick-Deploy/09-fslogix-configuration|Step 09]] to deploy settings to session hosts
5. Consult [[Storage/troubleshooting-fslogix-profiles|Troubleshooting FSLogix Profiles]] if issues arise
6. For multi-region or high-availability requirements, evaluate [[Storage/fslogix-cloud-cache|FSLogix Cloud Cache]]

## Key Decisions

- **Storage Tier:** Standard (HDD, cost-effective) vs. Premium (SSD, high-performance) - see [[Storage/azure-files-overview|Azure Files Overview]]
- **Authentication:** Entra ID (recommended) vs. Storage account keys - see [[Storage/storage-permissions-for-fslogix|Storage Permissions for FSLogix]]
- **Profile Location:** Simple single share vs. user group separation - see [[Storage/fslogix-profile-containers|FSLogix Profile Containers]]
- **High Availability:** Standard FSLogix vs. Cloud Cache - see [[Storage/fslogix-cloud-cache|FSLogix Cloud Cache]]
