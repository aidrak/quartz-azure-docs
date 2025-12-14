---
title: VNet Design
description: 
published: true
date: 2025-12-14T04:53:28.090Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:01.183Z
---

# VNet Design

Virtual Network (VNet) design is the foundation of your Azure Virtual Desktop infrastructure. Proper address space planning, subnet segmentation, and sizing ensures scalability, security, and compatibility with on-premises networks.

## What is VNet Design

A Virtual Network (VNet) is an isolated network in Azure that enables resources to securely communicate with each other, the internet, and on-premises networks. VNet design involves:

- **Address Space Planning:** Selecting CIDR ranges that don't conflict with on-premises networks or other Azure VNets
- **Subnet Segmentation:** Dividing the VNet into logical subnets based on workload types and security boundaries
- **Reserved Addresses:** Understanding that Azure reserves 5 IP addresses in each subnet (.0, .1, .2, .3, and .255)
- **Future Growth:** Planning for expansion without requiring readdressing

## When to Use Specific Subnet Strategies

**Single Large Subnet (NOT Recommended for AVD):**
- Simple deployments with no network segmentation requirements
- Non-production environments

**Multiple Purpose-Specific Subnets (RECOMMENDED for AVD):**
- Production AVD deployments requiring security boundaries
- Environments with different workload types (session hosts, file servers, management)
- Compliance requirements for network segmentation
- When using NSGs for traffic control between tiers

**Hub-Spoke Topology:**
- Multi-region deployments
- Shared services (Azure Firewall, VPN Gateway)
- Enterprise environments with multiple workloads beyond AVD