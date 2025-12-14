---
title: Resource Group & Tagging
description: 
published: true
date: 2025-12-14T04:52:40.767Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:34.710Z
---

# Resource Group & Tagging

Proper resource organization is critical for management, billing, and RBAC. Create a dedicated resource group for AVD resources with consistent tagging.

## Create Resource Group

**Portal:** Azure Portal → Resource Groups → Create

1. **Subscription:** Select target subscription
2. **Resource group:** `RG-Azure-VDI-01` (or follow client naming convention)
3. **Region:** `Central US` (or client's preferred region)

> **Note:** All AVD resources should be in the same region for optimal performance. Cross-region deployments add latency.

## Naming Convention

Adopt a consistent naming pattern. Our standard:

| Resource Type | Pattern | Example |
|---------------|---------|---------|
| Resource Group | `RG-{Purpose}-{Sequence}` | RG-Azure-VDI-01 |
| Virtual Network | `vnet-{purpose}` | vnet-avd |
| Subnet | `snet-{purpose}` | snet-sessionhosts |
| NSG | `nsg-{subnet-name}` | nsg-snet-sessionhosts |
| Host Pool | `hp-{type}-{env}` | hp-pooled-prod1 |
| Workspace | `ws-{purpose}-{env}` | ws-avd-prod |
| Storage Account | `st{purpose}{random}` | fslogix121025 |

## Tagging Strategy

Apply tags to the resource group (inherited by resources):

| Tag | Value | Purpose |
|-----|-------|---------|
| `Environment` | Production | Identify environment |
| `Project` | AVD | Group related resources |
| `CostCenter` | IT-Infrastructure | Billing allocation |
| `Owner` | ITAdmin@company.com | Contact for issues |
| `CreatedDate` | 2024-12-14 | Track deployment date |

**Portal:** Resource Group → Tags → Add

## RBAC Assignment

Assign roles at the resource group level:

| Role | Assignee | Purpose |
|------|----------|---------|
| Owner | IT Admins | Full control |
| Desktop Virtualization Contributor | AVD Admins | Manage AVD resources |
| Reader | Helpdesk | View-only access |

**Portal:** Resource Group → Access control (IAM) → Add role assignment

## Resource Locks (Optional)

For production environments, consider adding a Delete lock:

**Portal:** Resource Group → Locks → Add

- **Lock name:** `PreventDeletion`
- **Lock type:** Delete

> **Warning:** Resource locks apply to all resources in the group. Document who can remove locks.

## Verification

- [ ] Resource group created in correct region
- [ ] Tags applied
- [ ] RBAC roles assigned
- [ ] Lock applied (if production)

---

**Next:** [[networking-setup|Step 3: Networking Setup]]