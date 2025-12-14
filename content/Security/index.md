---
title: Security
description: AVD security, threat protection, and compliance
published: true
---

# Security

The Security section covers threat protection, compliance, monitoring, and disaster recovery for Azure Virtual Desktop deployments. Topics include configuring Microsoft Defender for Cloud, setting up Azure Monitor alerting, implementing security baselines, and backing up critical workloads.

## What This Section Covers

- **Threat Protection**: Real-time threat detection and response using Microsoft Defender for Cloud
- **Security Baselines**: Microsoft's official security configuration guidance for AVD environments
- **Monitoring**: Alert rules, log analytics, and diagnostic logging to detect anomalies
- **Insights & Analytics**: Azure Monitor workbooks for security and performance visibility
- **Data Protection**: Backup strategies for personal desktops, user profiles, and FSLogix configurations
- **Compliance**: Mapping your environment to security frameworks and regulatory requirements

## Reference Pages

### [[microsoft-defender-for-cloud|Microsoft Defender for Cloud]]

Unified cloud security posture management (CSPM) and workload protection platform. Covers free vs paid plans, security posture assessment, vulnerability scanning, and threat detection for session hosts. Includes integration with AVD resources and automated response setup.

**Key Topics**: Secure Score, workload protection, regulatory compliance, paid plan options

### [[avd-security-baseline|AVD Security Baseline]]

Microsoft's official security configuration guidance for Azure Virtual Desktop. Covers critical controls (screen capture protection, watermarking, clipboard controls), network isolation, identity and access management, and monitoring requirements. Prioritized by implementation impact.

**Key Topics**: Session security, network segmentation, privileged access, data protection, CIS benchmarks

### [[azure-monitor-alerting|Azure Monitor Alerting]]

Proactive monitoring and alerting to maintain SLA commitments and respond to issues before users are affected. Covers metric alerts, log-based alerts, action groups, and recommended AVD-specific alert rules with thresholds.

**Key Topics**: Alert rules and thresholds, action groups, log queries, notification routing, AVD metrics

### [[log-analytics-workspace|Log Analytics Workspace]]

Centralized repository for collecting, analyzing, and querying logs from AVD resources, session hosts, and Azure services. Covers workspace setup, data collection configuration, KQL query basics, and retention policies.

**Key Topics**: Workspace configuration, data sources, retention, KQL queries, cost optimization

### [[avd-insights-azure-monitor-workbook|AVD Insights Azure Monitor Workbook]]

Interactive dashboards for visualizing AVD performance, user connection metrics, session host health, and capacity utilization. Pre-built workbook from Microsoft with customization options for your environment.

**Key Topics**: Performance metrics, user experience monitoring, capacity planning, custom visualizations

### [[backup-disaster-recovery|Backup & Disaster Recovery]]

Backup strategies for AVD environments including personal desktops, user profiles, and FSLogix configurations. Covers Azure Backup setup, recovery strategies, and business continuity planning for different host pool types.

**Key Topics**: What to back up, Azure Backup policies, FSLogix protection, disaster recovery architecture

## Related Deployment Steps

Security is integrated throughout the deployment process:

- [[Quick-Deploy/11-go-live-monitoring|Step 11: Go-Live Monitoring]] - Configure monitoring, alerts, and dashboards before going live to production
- [[Deployment/monitoring-setup|Deployment: Monitoring Setup]] - Initial monitoring configuration during deployment

## Getting Started

1. Start with [[microsoft-defender-for-cloud|Microsoft Defender for Cloud]] to assess your current security posture
2. Review [[avd-security-baseline|AVD Security Baseline]] to understand critical security controls
3. Configure [[azure-monitor-alerting|Azure Monitor Alerting]] for critical AVD metrics
4. Set up [[log-analytics-workspace|Log Analytics Workspace]] to centralize logs
5. Use [[avd-insights-azure-monitor-workbook|AVD Insights]] to monitor ongoing performance
6. Plan your backup strategy with [[backup-disaster-recovery|Backup & Disaster Recovery]]

## Key Principles

- **Defense in depth**: Layer controls across session security, network, identity, and monitoring
- **Continuous monitoring**: Detect issues early with alerting and log analysis
- **Automated response**: Use action groups and automation to respond to threats
- **Business continuity**: Balance security with availability through backup and disaster recovery planning
- **Compliance first**: Map security controls to regulatory requirements (HIPAA, PCI-DSS, ISO 27001)
