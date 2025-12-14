---
title: Cost Optimization
description: Strategies to reduce Azure Virtual Desktop costs
published: true
date: 2025-12-14T04:53:32.793Z
tags: 
editor: markdown
dateCreated: 2025-12-14T04:44:31.878Z
---

# Cost Optimization

## What It Is
Strategies and configurations to minimize AVD spending while maintaining performance and user experience.

## Key Cost Drivers
1. **Compute (VMs)** - 60-80% of AVD costs
2. **Storage** - Profile disks, Azure Files
3. **Networking** - Egress, VPN Gateway
4. **Licensing** - Windows, Microsoft 365

## Cost Optimization Strategies

### 1. Right-Size Session Hosts

**Analyze Current Usage:**
**Portal:** Azure Monitor → Insights → Virtual Machines

Check these metrics:
- Average CPU utilization (target: 60-80%)
- Memory utilization
- Session density (users per host)

**Recommendations:**
| Workload | Recommended SKU | Users/Host |
|----------|-----------------|------------|
| Light (Office, web) | D2s_v5 | 8-12 |
| Medium (Light apps) | D4s_v5 | 6-10 |
| Heavy (CAD, dev) | D8s_v5 | 2-4 |

### 2. Use Scaling Plans

**Portal:** Azure Virtual Desktop → Scaling Plans

Configure:
- **Ramp-up:** Start VMs before business hours
- **Peak hours:** Maximum hosts available
- **Ramp-down:** Drain and deallocate after hours
- **Off-peak:** Minimum hosts (1-2 for availability)

**Example Schedule:**
| Phase | Time | Host % | Action |
|-------|------|--------|--------|
| Ramp-up | 7:00-9:00 | 25% → 100% | Start VMs |
| Peak | 9:00-17:00 | 100% | All running |
| Ramp-down | 17:00-19:00 | 100% → 25% | Drain, deallocate |
| Off-peak | 19:00-7:00 | 10% | Minimum hosts |

**Savings:** Up to 60% on compute costs

### 3. Reserved Instances

**Portal:** Azure Portal → Reservations → Add

- **1-year:** ~30% savings
- **3-year:** ~50% savings

**Best for:** Minimum baseline hosts that run 24/7

**Strategy:** Reserve baseline capacity, use pay-as-you-go for burst

### 4. Azure Hybrid Benefit

**Requirement:** Windows Server licenses with Software Assurance

**Savings:** Up to 40% on Windows VMs

**How to Enable:**
- During VM creation: Select "Yes" for Azure Hybrid Benefit
- Existing VMs: Update in VM Configuration

### 5. Storage Optimization

**FSLogix Profile Sizing:**
- Start with 10 GB per user
- Expand as needed (not shrink)
- Use Standard HDD for inactive profiles

**Azure Files Tier:**
| Tier | Use Case | Cost |
|------|----------|------|
| Premium | Active profiles, < 20ms latency | $$$ |
| Transaction Optimized | Mixed workloads | $$ |
| Hot | Archive profiles | $ |

### 6. Dev/Test Pricing

**Portal:** Azure Portal → Subscriptions → [Sub] → Properties

Enable Dev/Test pricing for non-production:
- 40-50% discount on Windows VMs
- Requires Visual Studio subscription

## Cost Monitoring

### Set Budget Alerts
**Portal:** Cost Management → Budgets → Add

1. Create monthly budget
2. Set alerts at 50%, 75%, 90%
3. Configure action groups for notifications

### Use Cost Analysis
**Portal:** Cost Management → Cost Analysis

Filter by:
- Resource group
- Tag (e.g., Environment: Production)
- Service (Virtual Machines, Storage)

## Quick Wins Checklist

- [ ] Enable Scaling Plans for all host pools
- [ ] Right-size VMs based on actual usage
- [ ] Purchase Reserved Instances for baseline
- [ ] Enable Azure Hybrid Benefit
- [ ] Set budget alerts
- [ ] Review and delete unused resources monthly
- [ ] Use Standard storage for non-critical workloads

## Related
- [[Scaling Plans]]
- [[Session Host Sizing]]
- [[Capacity Planning]]
