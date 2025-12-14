---
title: AVD Architecture Reference
description: Index of Azure Virtual Desktop architecture and component documentation
published: true
date: 2025-12-14
tags: avd, index
---

# AVD Architecture Reference

Welcome to the Azure Virtual Desktop architecture documentation. This section provides deep-dive references for understanding AVD components, design patterns, and best practices. Each page is self-contained and can be read independently.

## What's Covered Here

This reference documentation covers the technical architecture of Azure Virtual Desktop:

- **Core Components:** Understanding the control plane, data plane, and how connection brokering works
- **Host Pools:** The foundational building block of AVD—pooled vs personal, load balancing, and session management
- **Application Groups:** How applications and desktops are published and made available to users
- **Workspaces:** The user-facing interface that organizes and presents resources
- **Session Host Sizing:** VM selection, performance metrics, and right-sizing guidance
- **Scaling Plans:** Cost optimization through automated start/stop schedules

For step-by-step deployment instructions, see the [[Quick-Deploy/index]] section instead.

## Architecture Topics

### Foundational Concepts

**[[AVD Components Overview]]**
Overview of AVD architecture, control plane vs data plane, connection flow, and the division of responsibilities between Microsoft and customer-managed infrastructure.

**[[Host Pools Deep Dive]]**
Understanding host pools as the core building block. Covers pooled vs personal host pools, load balancing algorithms (breadth-first, depth-first, persistent), session limits, drain mode, and when to use each configuration.

**[[Application Groups]]**
How applications and desktops are published to users. Covers desktop vs RemoteApp application groups, user assignment through Entra ID security groups, and workspace registration.

**[[Workspaces]]**
The user-facing container that organizes application groups. Explains how users discover and connect to resources through the Remote Desktop client, and best practices for workspace naming and organization.

### Performance and Optimization

**[[Session Host Sizing]]**
Practical guidance on VM selection. Covers D-series (general purpose), E-series (memory-optimized), and F-series (compute-optimized) VMs, with sizing formulas for different user workload profiles, disk recommendations, and right-sizing based on performance metrics.

**[[Scaling Plans]]**
Automated cost optimization through schedule-based VM start/stop. Covers schedule phases (ramp-up, peak, ramp-down, off-peak), load balancing changes based on capacity, and how scaling plans can reduce compute costs by 40-60%.

## Getting Started

### For New Deployments

Follow the linear deployment playbook in [[Quick-Deploy/index]] for step-by-step instructions:

1. Start with [[Quick-Deploy/00-naming-conventions]]—establish consistent naming early
2. Work through prerequisites and licensing at [[Quick-Deploy/01-prerequisites-licensing]]
3. Progress sequentially through infrastructure setup
4. When you reach host pool creation at [[Quick-Deploy/06-host-pool-creation]], reference [[Host Pools Deep Dive]] for architectural decisions
5. At [[Quick-Deploy/07-session-hosts]], use [[Session Host Sizing]] to right-size your VMs
6. For application delivery at [[Quick-Deploy/08-app-groups-workspace]], see [[Application Groups]] and [[Workspaces]] for concepts

### For Understanding Architecture

1. Start with [[AVD Components Overview]] to understand the overall system
2. Deep-dive into specific components as needed using the topics above
3. Reference the [[Quick-Deploy]] section for Portal paths and step-by-step actions

### For Optimization

- Review [[Session Host Sizing]] to validate your current VM selection
- Implement [[Scaling Plans]] to optimize costs
- Check each reference page's "Best Practices" and "Common Issues" sections for troubleshooting

## Related Deployment Steps

These Quick-Deploy steps map to the architecture components documented here:

| Quick-Deploy Step | Related Reference |
|---|---|
| [[Quick-Deploy/06-host-pool-creation]] | [[Host Pools Deep Dive]] |
| [[Quick-Deploy/07-session-hosts]] | [[Session Host Sizing]] |
| [[Quick-Deploy/08-app-groups-workspace]] | [[Application Groups]], [[Workspaces]] |

## Key Architecture Principles

- **Host pools are foundational:** All other AVD resources (app groups, workspaces, scaling plans) build on host pools
- **Control plane is managed:** Microsoft handles connection brokering, authentication, and gateway services—you manage session hosts and supporting infrastructure
- **Application groups bridge compute and user experience:** They link host pools (compute) to workspaces (user interface)
- **Pooled vs personal is the primary decision:** This architectural choice drives choices in load balancing, session management, and cost structure
- **Scaling plans require planning:** Design your schedule phases based on actual user login patterns and peak demand

## Next Steps

- **Deploying AVD?** Start with [[Quick-Deploy/01-prerequisites-licensing]] for a linear playbook
- **Troubleshooting connectivity?** See [[AVD Components Overview#Connection Flow]]
- **Optimizing costs?** Review [[Scaling Plans]] for schedule configuration guidance
- **Sizing session hosts?** Use [[Session Host Sizing]] to calculate appropriate VM SKUs
