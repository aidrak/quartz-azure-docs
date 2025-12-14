---
title: Capacity Planning
description: Size and plan Azure Virtual Desktop deployments
published: true
date: 2025-12-14T04:53:31.237Z
tags: 
editor: markdown
dateCreated: 2025-12-14T04:44:30.272Z
---

# Capacity Planning

## What It Is
Methodology for determining the right number and size of session hosts for your AVD deployment.

## Key Inputs

### 1. User Count and Concurrency
- **Total users:** All users with AVD access
- **Concurrent users:** Peak simultaneous connections (typically 60-80% of total)
- **Growth projection:** Expected user growth over 12-24 months

### 2. Workload Profile

| Profile | Description | Example Apps | vCPU/User | RAM/User |
|---------|-------------|--------------|-----------|----------|
| Light | Basic office tasks | Office 365, web browsing, email | 0.5 | 2 GB |
| Medium | Standard knowledge worker | Office + line-of-business apps | 1 | 4 GB |
| Heavy | Power users | Development, CAD, data analysis | 2+ | 8+ GB |

### 3. Session Host Sizing

**Recommended VM SKUs:**

| SKU | vCPU | RAM | Light Users | Medium Users | Heavy Users |
|-----|------|-----|-------------|--------------|-------------|
| D2s_v5 | 2 | 8 GB | 4-6 | 2-4 | 1-2 |
| D4s_v5 | 4 | 16 GB | 8-12 | 4-8 | 2-4 |
| D8s_v5 | 8 | 32 GB | 16-24 | 8-16 | 4-8 |
| D16s_v5 | 16 | 64 GB | 32-48 | 16-32 | 8-16 |

**Formula:**
```
Hosts Needed = Concurrent Users / Users Per Host
Add 20% buffer for maintenance/failover
```

## Sizing Calculator

### Example: 500 User Deployment

**Inputs:**
- Total users: 500
- Concurrent rate: 70%
- Workload: Medium
- Target VM: D8s_v5

**Calculation:**
```
Concurrent users = 500 × 0.70 = 350 users
Users per D8s_v5 (medium) = 12
Hosts needed = 350 / 12 = 29.2 → 30 hosts
With 20% buffer = 30 × 1.2 = 36 hosts
```

**Recommendation:** 36 × D8s_v5 session hosts

## Storage Planning

### FSLogix Profile Containers

**Profile Size Guidelines:**
| User Type | Profile Size | Notes |
|-----------|--------------|-------|
| Standard | 10-20 GB | Office docs, settings |
| Power User | 30-50 GB | Large data sets |
| Developer | 50-100 GB | Code repos, tools |

**Storage Account Sizing:**
```
Total Storage = Users × Profile Size × 1.5 (growth buffer)
Example: 500 users × 20 GB × 1.5 = 15 TB
```

**IOPS Requirements:**
- Standard workload: 10 IOPS per user
- Heavy workload: 20+ IOPS per user
- Use Premium Azure Files for < 10ms latency requirements

### Azure Files Share Limits
| Tier | Max Size | Max IOPS | Max Throughput |
|------|----------|----------|----------------|
| Premium | 100 TB | 100,000 | 10 GB/s |
| Standard | 100 TB | 20,000 | 300 MB/s |

## Network Planning

### Bandwidth Requirements

| Activity | Bandwidth/User |
|----------|----------------|
| Idle | 10-20 Kbps |
| Office work | 200-500 Kbps |
| Video playback | 1-5 Mbps |
| Teams video call | 2-4 Mbps |

**Total Bandwidth:**
```
Peak Bandwidth = Concurrent Users × Average Bandwidth × 1.3 (overhead)
Example: 350 users × 1 Mbps × 1.3 = 455 Mbps
```

### Latency Targets
| Metric | Target | Acceptable |
|--------|--------|------------|
| RTT to Azure | < 100ms | < 150ms |
| Within Azure | < 5ms | < 10ms |

## Host Pool Configuration

### Breadth-First vs Depth-First

**Breadth-First (Recommended):**
- Distributes users evenly across all hosts
- Better performance, higher availability
- Use for: Production workloads

**Depth-First:**
- Fills one host before moving to next
- Better for cost optimization with scaling plans
- Use for: Dev/test, cost-sensitive environments

### Max Session Limits
Set per host pool to prevent overloading:
```
Max Sessions = Host vCPU / vCPU per User
Example: D8s_v5 (8 vCPU) / 1 vCPU per user = 8 max sessions
```

## Capacity Planning Checklist

- [ ] Survey users for workload requirements
- [ ] Determine concurrent user percentage
- [ ] Select appropriate VM SKU
- [ ] Calculate host count with buffer
- [ ] Plan FSLogix storage capacity and IOPS
- [ ] Verify network bandwidth to Azure
- [ ] Configure max session limits
- [ ] Plan for scaling (manual or automatic)

## Monitoring and Adjustment

After deployment, monitor:
- **CPU:** Target 60-80% average utilization
- **Memory:** Target < 80% utilization
- **User density:** Actual vs. planned sessions/host
- **User experience:** Connection quality, latency

Adjust capacity quarterly based on metrics.

## Related
- [[Session Host Sizing]]
- [[Scaling Plans]]
- [[Cost Optimization]]
