# Azure Virtual Desktop Reference Documentation

Welcome to the Azure Reference guide. This documentation provides comprehensive resources for implementing Azure Virtual Desktop (AVD) in enterprise environments.

## Two-Tier Structure

This guide is organized around two complementary approaches:

### Deployment Playbook

The **[[Deployment/|Deployment]]** section provides a linear, step-by-step playbook for implementing Azure Virtual Desktop. This is the "golden path" - a structured 12-step process from initial planning through go-live. Each step includes:

- Portal-based instructions with exact navigation paths
- Lightweight checklists for quick reference
- Links to deeper technical reference material for complex topics
- Decision points for variant scenarios (hybrid, Intune-less configurations, etc.)

**Start here if:** You're implementing AVD and need a guided process to follow.

### Reference Topics

The **Reference** section contains deep-dive technical content organized by topic:

- **[[AVD/|AVD Architecture]]** - Host pools, session sizing, scaling plans, workspaces
- **[[Identity/|Identity]]** - Entra ID, conditional access, hybrid identity
- **[[Images/|Images]]** - Golden image creation, Azure Compute Gallery
- **[[Intune/|Intune]]** - Device management, configuration, compliance
- **[[Networking/|Networking]]** - VNets, NSGs, ExpressRoute, DNS
- **[[Operations/|Operations]]** - Capacity planning, cost management
- **[[Security/|Security]]** - Defender for Cloud, monitoring, audit logging
- **[[Storage/|Storage]]** - Azure Files, FSLogix, container optimization

Each reference page is self-contained and includes:

- What the component is and why it exists
- When and where to use it
- Step-by-step configuration in the Azure Portal
- Best practices with reasoning
- Troubleshooting common issues

**Start here if:** You need detailed technical information about a specific Azure component.

## Scenario Context

> **Medium Enterprise Environment**
>
> This documentation is written for a typical medium enterprise scenario: 200 users, mix of pooled and personal desktops, Office 365 and Adobe Creative Suite applications. Configuration examples and best practices reflect this profile, though guidance applies across scales.

## Quick Start

1. **New to AVD?** Start with [[Deployment/prerequisites-planning|Prerequisites and Planning]]
2. **Need specific technical details?** Jump to the relevant Reference topic
3. **Troubleshooting an issue?** Check the Common Issues section in the relevant Reference page

## Navigation

This is a complete, searchable documentation site. Use:

- The folder navigation on the left to browse by topic
- The search function to find specific information
- Wikilinks throughout the documentation to jump between related topics

---

Published at: https://aidrak.github.io/quartz-azure-docs/
