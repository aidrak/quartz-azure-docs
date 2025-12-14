# Reference Library

## Overview

The Reference Library provides deep-dive technical documentation for Azure Virtual Desktop components and supporting services. Each page is self-contained and can be read independently of others.

**How Reference relates to Deployment:**
- The [[Deployment/index|Deployment Guide]] is a linear playbook with step-by-step instructions
- Deployment steps link **to** Reference pages for detailed explanations
- Reference pages explain the "what" and "why" behind each component
- Use Reference when you need to understand design decisions, troubleshooting, or best practices

## Topic Areas

### [[Reference/AVD/index|AVD]]
Azure Virtual Desktop architecture, including host pools, session hosts, workspaces, and scaling plans. Core AVD concepts and configuration patterns.

### [[Reference/Identity/index|Identity]]
Entra ID (Azure Active Directory) setup, user management, groups, Conditional Access policies, and authentication flows for AVD.

### [[Reference/Images/index|Images]]
Golden image creation, customization, Azure Compute Gallery (Shared Image Gallery), image versioning, and distribution strategies.

### [[Reference/Networking/index|Networking]]
Virtual networks, subnets, network security groups (NSGs), DNS configuration, VPN Gateway, and ExpressRoute for hybrid connectivity.

### [[Reference/Storage/index|Storage]]
Azure Files for profile storage, FSLogix configuration, performance tiers, and file share security and permissions.

### [[Reference/Intune/index|Intune]]
Microsoft Intune device management, compliance policies, app deployment, configuration profiles, and Windows Update management for AVD session hosts.

### [[Reference/Security/index|Security]]
Microsoft Defender for Cloud, security baselines, monitoring with Log Analytics and Azure Monitor, threat protection, and security best practices.

### [[Reference/Operations/index|Operations]]
Capacity planning, cost management and optimization, scaling strategies, performance monitoring, and operational runbooks.

---

## Using This Library

Each Reference page follows this structure:
1. **What It Is** - Brief technical explanation
2. **When to Use It** - Use cases and decision criteria
3. **How to Set It Up** - Portal-based configuration steps
4. **Best Practices** - Recommendations with reasoning
5. **Common Issues** - Troubleshooting guide

Start with the topic area most relevant to your current task, or follow links from the [[Deployment/index|Deployment Guide]] as you work through implementation steps.
