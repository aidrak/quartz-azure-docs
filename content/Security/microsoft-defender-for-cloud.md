---
title: Microsoft Defender for Cloud
description: 
published: true
date: 2025-12-14T04:53:43.298Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:54.866Z
---

# Microsoft Defender for Cloud

Microsoft Defender for Cloud (formerly Azure Security Center) is a unified cloud security posture management (CSPM) and cloud workload protection platform (CWPP). It continuously assesses your Azure environment against security best practices, identifies vulnerabilities, and provides actionable recommendations to improve your security posture. For AVD deployments, Defender for Cloud is essential for protecting session hosts, detecting threats, and maintaining compliance.

## What is Defender for Cloud

Defender for Cloud provides three core capabilities:

1. **Security Posture Management (CSPM)**: Evaluates your resources against security benchmarks (Azure Security Benchmark, CIS, PCI-DSS) and assigns a Secure Score. The score reflects how well you're meeting security recommendations.

2. **Workload Protection (CWPP)**: Advanced threat protection for specific resource types (VMs, storage, databases, containers). Requires paid plans but provides deep security telemetry and behavioral analytics.

3. **Regulatory Compliance**: Maps your environment to compliance frameworks (HIPAA, ISO 27001, NIST). Generates reports showing gaps and remediation steps.

**Integration with AVD**: Defender for Cloud monitors session hosts for vulnerabilities, missing patches, weak configurations, and active threats. It sends alerts to Azure Security Center dashboard and can trigger automated responses via Logic Apps or Azure Automation.

## Free Tier vs Paid Plans

Defender for Cloud has a foundational free tier enabled by default, with optional paid plans for enhanced protection:

### Free Tier (Enabled by Default)

**Included Features**:
- Secure Score calculation
- Security recommendations for all Azure resources
- Azure Policy integration
- Continuous assessment against Azure Security Benchmark
- Network map visualization
- Access to Defender for Cloud portal

**Limitations**:
- No threat detection or behavioral analytics
- No file integrity monitoring
- No just-in-time VM access
- No adaptive application controls
- Recommendations only (no automated remediation)

**Good for**: Small environments, cost-conscious deployments, compliance reporting without active threat protection.

### Paid Plans (Enable per Resource Type)

Plans are enabled per subscription and resource type. Relevant for AVD:

**Microsoft Defender for Servers** (Formerly Defender for VMs)
- **Cost**: ~$15/server/month (Plan 1) or ~$0.02/hour (Plan 2)
- **Features**:
  - File integrity monitoring (FIM)
  - Just-in-time (JIT) VM access
  - Adaptive application controls (allowlisting)
  - Threat detection with behavioral analytics
  - Integration with Microsoft Defender for Endpoint
  - Vulnerability assessment (Qualys or Microsoft Defender Vulnerability Management)

**Microsoft Defender for Storage**
- **Cost**: ~$10/storage account/month + $0.02/10k transactions
- **Features**:
  - Malware scanning (hash reputation)
  - Detection of unusual access patterns
  - Alerts on sensitive data exfiltration

**Recommended for AVD**: Enable Defender for Servers Plan 2 on session hosts. Enable Defender for Storage on FSLogix profile storage accounts.

## Secure Score Explained

Secure Score is a numerical representation of your security posture, calculated as:

```
Secure Score = (Points Earned / Total Available Points) × 100
```

Each security recommendation has a point value based on impact. Completing high-impact recommendations (e.g., "Enable MFA") adds more points than low-impact ones (e.g., "Remove unused NSG rules").

**Example from RG-Azure-VDI-01**:
- Total possible: 47 points
- Current score: 32 points (68%)
- Top recommendations:
  - Enable MFA for accounts with owner permissions (+10 points)
  - Install endpoint protection on VMs (+8 points)
  - Apply system updates (+6 points)

**How to Improve**:
1. Azure Portal → Microsoft Defender for Cloud → Secure Score
2. Expand recommendations by severity (High, Medium, Low)
3. Click recommendation → View affected resources → Apply remediation
4. Some recommendations have "Quick Fix" automation (one-click remediation)

**Tracking Progress**: Secure Score is measured over time. Use the Secure Score Over Time chart to demonstrate security improvements to stakeholders.

## Key Recommendations for AVD

Defender for Cloud generates 100+ security recommendations. These are the most critical for AVD environments:

### Enable Multi-Factor Authentication (MFA)

**Recommendation**: MFA should be enabled on accounts with owner/contributor permissions on your subscription.

**Why it Matters**: Compromised admin accounts can delete resources, modify NSG rules, or access user data. MFA blocks 99.9% of automated credential stuffing attacks.

**How to Fix**:
1. Azure Portal → Azure Active Directory → Users
2. Select user → Authentication methods → Require multi-factor authentication
3. **Better approach**: Use Conditional Access policy to enforce MFA for all users accessing AVD

```json
// Conditional Access Policy Example
{
  "displayName": "Require MFA for AVD Access",
  "conditions": {
    "applications": {
      "includeApplications": ["9cdead84-a844-4324-93f2-b2e6bb768d07"] // Azure Virtual Desktop
    },
    "users": {
      "includeGroups": ["all-avd-users-group-id"]
    }
  },
  "grantControls": {
    "operator": "AND",
    "builtInControls": ["mfa"]
  }
}
```

### Apply System Updates

**Recommendation**: System updates should be installed on your machines.

**Why it Matters**: Unpatched VMs are primary targets for ransomware and lateral movement attacks. CVEs like PrintNightmare and Zerologon are actively exploited.

**How to Fix**:
1. **Azure Update Manager** (recommended):
   - Azure Portal → Update Manager → Update Settings
   - Create schedule: Patch session hosts during maintenance window (e.g., Sundays 2 AM)
   - Enable automatic VM guest patching (reboots outside user hours)

2. **Windows Update for Business** (Intune):
   - Configure update rings: Defer feature updates 30 days, quality updates 7 days
   - Test patches on pilot group (10% of hosts) before broad deployment

**Monitoring**: Query Log Analytics for patch compliance:

```kql
Update
| where OSType == "Windows" and UpdateState == "Needed"
| summarize MissingUpdates=count() by Computer, Classification
| order by MissingUpdates desc
```

### Enable Endpoint Protection

**Recommendation**: Install endpoint protection solution on virtual machines.

**Why it Matters**: Session hosts are high-value targets (access to user data, credentials, network). Endpoint protection detects malware, ransomware, and fileless attacks.

**How to Fix**:
1. **Microsoft Defender for Endpoint** (included with Defender for Servers Plan 2):
   - Auto-deploys to Azure VMs when Defender for Servers is enabled
   - Provides EDR (Endpoint Detection and Response) capabilities
   - Integrates with Defender for Cloud alerts

2. **Third-party solutions** (CrowdStrike, SentinelOne):
   - Deploy via VM extensions or Intune
   - Ensure compatible with Azure VM agent

**Verification**:
```powershell
# Check Defender status on session host
Get-MpComputerStatus | Select-Object AntivirusEnabled, RealTimeProtectionEnabled, IoavProtectionEnabled
```

### Encrypt Virtual Machine Disks

**Recommendation**: Disk encryption should be applied on virtual machines.

**Why it Matters**: Prevents offline attacks where attacker gains access to VHD files (via snapshot or storage account breach). Protects user data in FSLogix profiles.

**How to Fix**:
1. **Azure Disk Encryption** (BitLocker on Windows):
   - Requires Key Vault for key storage
   - Encrypts OS and data disks

   ```bash
   az vm encryption enable \
     --resource-group RG-Azure-VDI-01 \
     --name avd-pool-0 \
     --disk-encryption-keyvault /subscriptions/.../vault588 \
     --volume-type All
   ```

2. **Encryption at Host** (alternative):
   - No Key Vault required
   - Encrypts VM cache and temp disks
   - Enable during VM creation: `--encryption-at-host true`

**Performance Impact**: Minimal (<5% CPU overhead). Test with representative workloads.

## Defender for Servers Plan

Defender for Servers has two tiers:

### Plan 1 (~$5/server/month)
- Threat detection and alerts
- Integration with Microsoft Defender for Endpoint (if licensed separately)
- Security alerts in Defender for Cloud
- Export to SIEM (Sentinel, Splunk)

**Use case**: You already have Defender for Endpoint licenses (e.g., Microsoft 365 E5).

### Plan 2 (~$15/server/month)
- **Everything in Plan 1, plus**:
- File Integrity Monitoring (FIM)
- Just-in-time VM access
- Adaptive application controls
- Adaptive network hardening
- Integrated vulnerability assessment (Qualys or Microsoft Defender Vulnerability Management)
- Docker host hardening (for container workloads)

**Use case**: Full security stack without separate licensing. Recommended for AVD session hosts.

**Enable Defender for Servers**:
1. Azure Portal → Microsoft Defender for Cloud → Environment settings
2. Select subscription → Defender plans
3. Toggle "Servers" to On → Select Plan 2 → Save
4. Defender for Endpoint auto-installs to Azure VMs within 24 hours

## Just-in-Time VM Access

JIT access reduces attack surface by blocking inbound RDP (port 3389) by default, opening it only when needed for administrative tasks.

**How it Works**:
1. Admin requests RDP access via Azure Portal
2. Approval granted (manual or automatic based on RBAC)
3. NSG rule temporarily allows admin's IP for 3 hours (configurable)
4. Rule auto-expires, port closes

**Configuration**:
1. Defender for Cloud → Workload protections → Just-in-time VM access
2. Enable on VMs → Configure policies:
   - RDP (3389): 3-hour access, require justification
   - SSH (22): Disable (not used for Windows)
   - Allowed source IPs: My IP, specific admin subnet

**Usage**:
```bash
# Request access via Azure CLI
az security jit-policy create \
  --location eastus \
  --name JIT-Policy-AVD \
  --resource-group RG-Azure-VDI-01 \
  --virtual-machines avd-pool-0 \
  --ports '[{"number":3389,"protocol":"TCP","allowedSourceAddressPrefixes":["203.0.113.5"],"maxRequestAccessDuration":"PT3H"}]'
```

**Best Practice**: Use JIT for management access to session hosts. End users connect via AVD gateway (no direct RDP to VMs).

## Adaptive Application Controls

Adaptive Application Controls (AAC) creates intelligent allowlists for session hosts, blocking unauthorized executables.

**How it Works**:
1. Defender for Cloud learns normal application behavior (30-day baseline)
2. Recommends allowlist rules (based on publisher, path, hash)
3. Enforces allowlist: Only approved apps can run
4. Alerts on violations

**Configuration**:
1. Defender for Cloud → Workload protections → Adaptive application controls
2. Select VM group (e.g., AVD session hosts)
3. Review recommended rules → Apply
4. Start in Audit mode (logs violations without blocking)
5. Switch to Enforce after 1-2 weeks of testing

**Example Rule**:
```json
{
  "path": "C:\\Program Files\\Microsoft Office\\root\\Office16\\WINWORD.EXE",
  "publisherName": "Microsoft Corporation",
  "action": "Allow"
}
```

**Use Case**: Prevent users from running unauthorized tools (Bitcoin miners, hacking tools) on session hosts.

## Workflow Automation for Remediation

Defender for Cloud can trigger automated responses to security alerts using Logic Apps.

**Example: Auto-remediate Weak NSG Rule**

1. **Trigger**: Defender for Cloud detects NSG rule allowing 0.0.0.0/0 on RDP port
2. **Action**: Logic App deletes the rule and sends notification to security team

**Setup**:
1. Defender for Cloud → Workflow automation → Add workflow automation
2. Trigger: Recommendation severity = High
3. Logic App: Call Azure Resource Manager API to delete NSG rule
4. Notification: Send email via Office 365 connector

**Sample Logic App**:
```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "triggers": {
      "When_a_Defender_for_Cloud_Recommendation_is_triggered": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            "connection": {
              "name": "@parameters('$connections')['ascassessment']['connectionId']"
            }
          },
          "method": "post",
          "path": "/triggers/recommendations"
        }
      }
    },
    "actions": {
      "Delete_NSG_Rule": {
        "type": "Http",
        "inputs": {
          "method": "DELETE",
          "uri": "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Network/networkSecurityGroups/NSG-AVD/securityRules/AllowRDPFromInternet?api-version=2021-02-01",
          "authentication": {
            "type": "ManagedServiceIdentity"
          }
        }
      }
    }
  }
}
```

## Best Practices

- **Enable Defender for Servers on All Session Hosts** - The cost (~$15/server/month) is justified by threat protection and compliance benefits. Factor into customer pricing.

- **Review Secure Score Weekly** - Assign responsibility to security team member. Track score trend over time. Celebrate improvements with team.

- **Use Quick Fix When Available** - Many recommendations have automated remediation. Click "Quick Fix" instead of manual steps. Saves time and reduces errors.

- **Prioritize High-Severity Recommendations** - Focus on recommendations with >5 Secure Score points. Low-priority items can wait until next quarter.

- **Integrate with Sentinel** - Stream Defender for Cloud alerts to Azure Sentinel for correlation with other security events (e.g., Azure AD sign-ins, firewall logs).

- **Test Adaptive Controls in Pilot** - Don't enable AAC in Enforce mode immediately. Run Audit mode for 2-4 weeks to identify false positives.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Secure Score not updating after remediation | Defender for Cloud rescans every 24 hours | Wait 24-48 hours. Force refresh: Settings → Trigger scan (if available). |
| Defender for Endpoint not installing | Older VM image without Azure VM agent | Install VM agent: Download from Microsoft, run installer, restart VM. Verify: `C:\WindowsAzure\GuestAgent` exists. |
| JIT access request denied | User lacks Security Reader + Contributor role | Assign "Security Admin" role or custom role with `Microsoft.Security/locations/jitNetworkAccessPolicies/initiate/action` permission. |
| AAC blocking legitimate app | App not in baseline or unsigned executable | Add exclusion: AAC → Edit group → Add path/publisher rule. Use hash-based rule for unsigned apps. |
| Workflow automation not triggering | Logic App connection expired | Reauthorize API connection: Logic App → API connections → ASC Assessment → Edit → Authorize. |
| High cost for Defender for Servers | All VMs enrolled including dev/test | Exclude non-production VMs: Defender for Cloud → Environment settings → Advanced settings → Exclusions. |

## Related Resources

- Defender for Cloud Overview: https://learn.microsoft.com/azure/defender-for-cloud/defender-for-cloud-introduction
- Secure Score Documentation: https://learn.microsoft.com/azure/defender-for-cloud/secure-score-security-controls
- Defender for Servers Pricing: https://azure.microsoft.com/pricing/details/defender-for-cloud/
- Just-in-Time Access Guide: https://learn.microsoft.com/azure/defender-for-cloud/just-in-time-access-usage
- Adaptive Application Controls: https://learn.microsoft.com/azure/defender-for-cloud/adaptive-application-controls