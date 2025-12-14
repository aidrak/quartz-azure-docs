---
title: Operations Reference
description: Operational guidance for Azure Virtual Desktop management
published: true
date: 2025-12-14
tags: [reference, operations, capacity, cost, maintenance]
---

# Operations Reference

This section provides operational guidance for managing, monitoring, and optimizing Azure Virtual Desktop deployments after go-live. Topics include capacity planning, cost optimization, scaling strategies, and ongoing maintenance procedures.

## What's Covered

**Capacity Planning** - Sizing and scaling methodologies to ensure adequate resources for current and future user demand, including VM SKU selection, user density calculations, and growth planning.

**Cost Optimization** - Strategies to reduce Azure spending while maintaining performance, including right-sizing, scaling plans, reserved instances, and storage tier optimization.

**Maintenance & Operations** - Ongoing operational tasks such as golden image updates, session host management, monitoring reviews, and incident response procedures.

## Reference Pages

### [[capacity-planning|Capacity Planning]]
Comprehensive sizing methodology for AVD deployments. Includes workload profiles, VM SKU recommendations, user density calculations, storage planning formulas, and network bandwidth requirements. Use this to determine the right number and size of session hosts for your environment.

**Key Topics:**
- User concurrency calculations
- Workload profiles (light, medium, heavy)
- Session host sizing formulas
- FSLogix storage capacity planning
- Network bandwidth requirements

### [[cost-optimization|Cost Optimization]]
Cost reduction strategies and best practices for AVD. Covers the primary cost drivers (compute, storage, networking) and actionable optimization techniques including scaling plans, reserved instances, Azure Hybrid Benefit, and storage tier selection.

**Key Topics:**
- Right-sizing session hosts
- Scaling plan configuration for cost savings
- Reserved instances and Azure Hybrid Benefit
- Storage tier optimization
- Budget alerts and cost monitoring

## Related Quick-Deploy Steps

The Operations reference pages support these deployment steps:

### [[../../Quick-Deploy/11-go-live-monitoring|Step 11: Go-Live & Monitoring Setup]]
Complete operational readiness guide covering Log Analytics workspace deployment, AVD Insights configuration, alert rule creation, scaling plan setup, and operational runbooks. This step bridges deployment and ongoing operations.

**What it covers:**
- Log Analytics workspace and diagnostic settings
- AVD Insights workbook deployment
- Azure Monitor alert rules
- Scaling plan configuration for pooled host pools
- User acceptance testing procedures
- Operational runbooks (add users, troubleshoot connections, update images)
- Weekly capacity review process

## Operational Workflows

### Daily Tasks
- Review AVD Insights Overview dashboard (5 minutes)
- Check fired alerts and resolve critical issues
- Verify scaling plan executed correctly (session hosts started/stopped as scheduled)

### Weekly Tasks
- Capacity trend review (session count, CPU, memory utilization)
- Scaling plan performance analysis
- Plan session host additions if approaching capacity
- Review cost analysis in Azure Cost Management

### Monthly Tasks
- Update golden images (Windows updates, application updates)
- Alert rule tuning (adjust thresholds, reduce false positives)
- Generate management reports (uptime, user count, cost trends)
- Review and optimize storage tiers

### Quarterly Tasks
- Disaster recovery testing
- Security posture review (Conditional Access, MFA adoption)
- User satisfaction surveys
- Capacity forecasting for next 6-12 months

## Best Practices

**Establish Baselines** - During the first 2-4 weeks of production, collect baseline metrics for CPU utilization, memory usage, session density, connection quality, and user satisfaction. Use these baselines to set alert thresholds and identify anomalies.

**Monitor Proactively** - Use AVD Insights and Azure Monitor alerts to detect issues before users report them. Configure alerts for session host availability, connection failures, high CPU/memory, low disk space, and FSLogix profile failures.

**Plan for Growth** - Review capacity trends monthly and forecast 6-12 months ahead. Ensure you have time to procure additional resources, update scaling plans, and test capacity changes before hitting limits.

**Optimize Continuously** - Cost optimization is ongoing, not one-time. Review compute usage monthly, adjust scaling plans based on actual usage patterns, right-size VMs that are consistently over or under-utilized, and leverage Reserved Instances for baseline capacity.

**Document Everything** - Maintain operational runbooks for common tasks (add users, add session hosts, troubleshoot connections, update images). This reduces mean time to resolution (MTTR) and enables team members to respond to incidents effectively.

## Common Issues

### Issue: Running Out of Capacity
**Symptom:** Users receive "No resources available" errors during peak hours.

**Cause:** Concurrent user count exceeds available session capacity.

**Fix:**
1. Immediately: Manually start additional session hosts or increase max session limits temporarily
2. Short-term: Adjust scaling plan to start more hosts during peak hours
3. Long-term: Deploy additional session hosts, review capacity planning calculations

### Issue: High Costs
**Symptom:** Monthly Azure bill exceeds budget.

**Cause:** Over-provisioned session hosts running 24/7, inefficient scaling plans, or inappropriate VM SKUs.

**Fix:**
1. Review Cost Analysis to identify top spending resources
2. Enable scaling plans if not already configured
3. Right-size VMs based on actual CPU/memory utilization
4. Purchase Reserved Instances for baseline capacity
5. Enable Azure Hybrid Benefit if eligible
6. Review storage tiers and downgrade non-critical workloads

### Issue: Poor User Experience
**Symptom:** Users report slow performance, lag, or connection quality issues.

**Cause:** Overloaded session hosts, network latency, or under-provisioned VMs.

**Fix:**
1. Check AVD Insights Host Diagnostics for CPU/memory utilization
2. Verify RTT and input delay metrics in Connection Performance tab
3. Reduce max session limits if hosts consistently >80% CPU
4. Add more session hosts or resize to larger VM SKU
5. Review network path for latency (check ExpressRoute/VPN if hybrid)

## Troubleshooting Resources

For detailed troubleshooting of specific issues:
- Connection failures: See [[../../Quick-Deploy/11-go-live-monitoring#runbook-3-troubleshoot-user-connection-failure|Runbook 3: Troubleshoot User Connection Failure]]
- Session host management: See [[../../Quick-Deploy/11-go-live-monitoring#runbook-2-add-new-session-host-to-pooled-host-pool|Runbook 2: Add New Session Host]]
- Golden image updates: See [[../../Quick-Deploy/11-go-live-monitoring#runbook-4-update-golden-image|Runbook 4: Update Golden Image]]
- Monitoring issues: See [[../../Quick-Deploy/11-go-live-monitoring#troubleshooting|Step 11 Troubleshooting Section]]

## Next Steps

After reviewing this operational guidance:

1. **Establish Monitoring** - Complete [[../../Quick-Deploy/11-go-live-monitoring|Step 11]] to deploy Log Analytics, AVD Insights, and alerting
2. **Baseline Capacity** - Use [[capacity-planning|Capacity Planning]] formulas to validate your current deployment sizing
3. **Optimize Costs** - Apply [[cost-optimization|Cost Optimization]] strategies to reduce spending
4. **Create Runbooks** - Document your specific operational procedures based on the examples in Step 11
5. **Schedule Reviews** - Set up recurring calendar events for daily, weekly, and monthly operational tasks
