---
title: NSG Best Practices
description: 
published: true
date: 2025-12-14T04:53:23.452Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:56.135Z
---

# NSG Best Practices

Network Security Groups (NSGs) are the primary firewall mechanism for controlling traffic to and from Azure Virtual Desktop resources. Understanding NSG rules, flow evaluation, and AVD-specific requirements is critical for secure and functional deployments.

## What are Network Security Groups

A Network Security Group (NSG) is a firewall that contains security rules to allow or deny network traffic. NSGs operate at Layer 4 (TCP/UDP) and can be associated with:

- **Subnets:** Rules apply to all resources in the subnet (RECOMMENDED for AVD)
- **Network Interfaces:** Rules apply to a specific VM's NIC (use sparingly)

**Key Concepts:**
- **Security Rules:** Allow or Deny rules with priority (100-4096, lower = higher priority)
- **Default Rules:** Azure provides default rules (priority 65000+) that cannot be deleted
- **Stateful:** Return traffic is automatically allowed (no need for explicit allow rules)
- **Direction:** Inbound (traffic TO resources) and Outbound (traffic FROM resources)

**Rule Evaluation Order:**
1. Rules evaluated by priority (lowest number first)
2. First matching rule applies (processing stops)
3. If no custom rules match, default rules apply

## When to Use NSGs

**Always Use:**
- AVD session host subnets (control internet egress and lateral movement)
- Subnets containing sensitive data (file servers, databases)
- Management subnets (restrict administrative access)

**Consider Alternatives:**
- **Azure Firewall:** When you need application-layer filtering (FQDN rules, TLS inspection)
- **Application Security Groups (ASGs):** When managing hundreds of VMs with dynamic membership
- **Service Endpoints:** When securing access to Azure PaaS services

**Not Needed:**
- GatewaySubnet (VPN Gateway manages its own security)
- AzureBastionSubnet (Bastion has built-in security)

## Real-World Example: AVD NSGs

Our production environment uses 4 NSGs aligned with subnet boundaries:

### nsg-snet-sessionhosts

**Applied To:** snet-sessionhosts (10.0.1.0/24)

**Inbound Rules:**

| Priority | Name | Source | Dest Port | Protocol | Action | Purpose |
|----------|------|--------|-----------|----------|--------|----------|
| 100 | AllowVnetInbound | VirtualNetwork | * | * | Allow | Internal VNet communication |
| 200 | AllowAzureLoadBalancerInbound | AzureLoadBalancer | * | * | Allow | Health probes |
| 65000 | DenyAllInbound | * | * | * | Deny | Default deny (Azure managed) |

**Outbound Rules:**

| Priority | Name | Destination | Dest Port | Protocol | Action | Purpose |
|----------|------|-------------|-----------|----------|--------|----------|
| 100 | AllowAzureCloudOutbound | AzureCloud | * | TCP | Allow | AVD service endpoints |
| 110 | AllowInternetOutbound | Internet | 80,443 | TCP | Allow | Windows Updates, KMS activation |
| 120 | AllowVnetOutbound | VirtualNetwork | * | * | Allow | Access to file servers, private endpoints |
| 65000 | DenyAllOutbound | * | * | * | Deny | Default deny (Azure managed) |

> **Critical:** AVD session hosts MUST have outbound internet access on ports 80/443 for the AVD control plane and Windows activation.

### nsg-snet-privateendpoints

**Applied To:** snet-privateendpoints (10.0.3.0/24)

**Inbound Rules:**

| Priority | Name | Source | Dest Port | Protocol | Action | Purpose |
|----------|------|--------|-----------|----------|--------|----------|
| 100 | AllowSessionHostsInbound | 10.0.1.0/24 | 445 | TCP | Allow | SMB access from session hosts |
| 110 | AllowManagementInbound | 10.0.4.0/27 | 445,443 | TCP | Allow | Admin access to storage |
| 65000 | DenyAllInbound | * | * | * | Deny | Default deny |

**Outbound Rules:**

| Priority | Name | Destination | Dest Port | Protocol | Action | Purpose |
|----------|------|-------------|-----------|----------|--------|----------|
| 100 | AllowAzureStorageOutbound | Storage | * | TCP | Allow | Private endpoint backend communication |
| 65000 | DenyAllOutbound | * | * | * | Deny | Default deny |

### nsg-snet-management

**Applied To:** snet-management (10.0.4.0/27)

**Inbound Rules:**

| Priority | Name | Source | Dest Port | Protocol | Action | Purpose |
|----------|------|--------|-----------|----------|--------|----------|
| 100 | AllowCorpVPNInbound | 172.20.20.0/24 | 3389,22 | TCP | Allow | RDP/SSH from VPN |
| 65000 | DenyAllInbound | * | * | * | Deny | Default deny |

**Outbound Rules:**

| Priority | Name | Destination | Dest Port | Protocol | Action | Purpose |
|----------|------|-------------|-----------|----------|--------|----------|
| 100 | AllowVnetOutbound | VirtualNetwork | * | * | Allow | Manage all subnets |
| 110 | AllowInternetOutbound | Internet | 80,443 | TCP | Allow | Download tools, updates |
| 65000 | DenyAllOutbound | * | * | * | Deny | Default deny |

## How to Configure

### Azure Portal

**Azure Portal → Network Security Groups → Create**

1. **Basics:**
   - Resource Group: RG-Azure-VDI-01
   - Name: nsg-snet-sessionhosts
   - Region: East US (must match VNet region)

2. **Review + Create**

3. **Add Inbound Security Rule:**
   - Navigate to NSG → Inbound security rules → Add
   - Source: Service Tag → VirtualNetwork
   - Source port ranges: *
   - Destination: Any
   - Service: Custom
   - Destination port ranges: *
   - Protocol: Any
   - Action: Allow
   - Priority: 100
   - Name: AllowVnetInbound

4. **Add Outbound Security Rule:**
   - Navigate to NSG → Outbound security rules → Add
   - Source: Any
   - Destination: Service Tag → AzureCloud
   - Destination port ranges: *
   - Protocol: TCP
   - Action: Allow
   - Priority: 100
   - Name: AllowAzureCloudOutbound

5. **Associate with Subnet:**
   - Navigate to NSG → Subnets → Associate
   - Virtual Network: vnet-avd
   - Subnet: snet-sessionhosts

### Azure CLI

```bash
# Create NSG
az network nsg create \
  --resource-group RG-Azure-VDI-01 \
  --name nsg-snet-sessionhosts

# Add outbound rule for AVD service
az network nsg rule create \
  --resource-group RG-Azure-VDI-01 \
  --nsg-name nsg-snet-sessionhosts \
  --name AllowAzureCloudOutbound \
  --priority 100 \
  --direction Outbound \
  --source-address-prefixes "*" \
  --destination-address-prefixes AzureCloud \
  --destination-port-ranges "*" \
  --protocol Tcp \
  --access Allow

# Add outbound rule for internet (Windows Update, KMS)
az network nsg rule create \
  --resource-group RG-Azure-VDI-01 \
  --nsg-name nsg-snet-sessionhosts \
  --name AllowInternetOutbound \
  --priority 110 \
  --direction Outbound \
  --source-address-prefixes "*" \
  --destination-address-prefixes Internet \
  --destination-port-ranges 80 443 \
  --protocol Tcp \
  --access Allow

# Associate with subnet
az network vnet subnet update \
  --resource-group RG-Azure-VDI-01 \
  --vnet-name vnet-avd \
  --name snet-sessionhosts \
  --network-security-group nsg-snet-sessionhosts
```

## Best Practices

**Rule Design:**
- **Use Service Tags:** Prefer `AzureCloud`, `Storage`, `Internet` over IP ranges (Azure manages IP updates)
- **Deny by Default:** Remove unnecessary Allow rules, rely on explicit denies
- **Lowest Priority for Allow:** Start custom allow rules at 100-200, leave room for exceptions at 50-99
- **Document Rules:** Use descriptive names (AllowAzureCloudOutbound vs Rule1)

**AVD-Specific Requirements:**
- **REQUIRED Outbound:** AzureCloud (AVD control plane), Internet:443 (Windows activation, Azure Storage)
- **REQUIRED Outbound Ports:** TCP 443 to *.wvd.microsoft.com, gcs.prod.monitoring.core.windows.net
- **Optional Outbound:** TCP 1688 for KMS activation (usually via Internet:443)
- **Inbound:** Typically NONE - AVD uses reverse connect (session hosts initiate outbound connections)

**Performance:**
- **Associate with Subnets, Not NICs:** Centralized management, better scale
- **Minimize Rules:** Each rule adds processing overhead (keep under 200 rules per NSG)
- **Use ASGs for Large Deployments:** Application Security Groups reduce rule count for hundreds of VMs

**Security:**
- **Explicit Deny Rules:** Add explicit deny rules for sensitive ports (RDP 3389, SSH 22) at priority 4096
- **Audit Regularly:** Review NSG rules quarterly, remove unused rules
- **Enable NSG Flow Logs:** Critical for troubleshooting and compliance (see section below)

## NSG Flow Logs

NSG Flow Logs record all traffic allowed/denied by NSG rules. Essential for troubleshooting and security monitoring.

**Enable Flow Logs:**

1. **Prerequisites:**
   - Storage account (e.g., stavdlogs01)
   - Network Watcher enabled in region

2. **Configuration:**
   - Navigate to NSG → NSG Flow Logs → Create
   - Select NSG: nsg-snet-sessionhosts
   - Flow Logs Version: Version 2 (includes flow direction)
   - Storage Account: stavdlogs01
   - Retention: 30 days
   - Traffic Analytics: Enabled (optional, uses Log Analytics Workspace)

**Analyzing Flow Logs:**

```bash
# Download flow logs from storage account
az storage blob download \
  --account-name stavdlogs01 \
  --container-name insights-logs-networksecuritygroupflowevent \
  --name "<blob-path>" \
  --file flowlogs.json

# Example log entry (JSON):
{
  "time": "2025-12-14T10:30:00.0000000Z",
  "flowLogResourceID": "/subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Network/networkSecurityGroups/nsg-snet-sessionhosts",
  "macAddress": "000D3AF8801A",
  "category": "NetworkSecurityGroupFlowEvent",
  "flowLogVersion": 2,
  "flowRecords": {
    "rule": "DefaultRule_AllowInternetOutBound",
    "flows": [{
      "sourceAddress": "10.0.1.5",
      "destinationAddress": "13.107.42.16",
      "destinationPort": "443",
      "protocol": "T",
      "trafficDecision": "A",
      "flowState": "B"
    }]
  }
}
```

**Traffic Analytics (Optional):**
- Visualizes flow logs in Azure Monitor
- Identifies top talkers, blocked traffic, malicious IPs
- Cost: ~$2/GB ingested + Log Analytics storage

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **"AVD session hosts can't connect to control plane"** | NSG blocking outbound to AzureCloud or Internet | Add outbound allow rule for AzureCloud service tag on TCP 443. Verify with `Test-NetConnection -ComputerName rdbroker.wvd.microsoft.com -Port 443` from session host. |
| **"Users can't access FSLogix profiles"** | NSG blocking session hosts from reaching private endpoint | Add rule allowing 10.0.1.0/24 → 10.0.3.0/24 on port 445 (SMB). Check flow logs for denied traffic. |
| **"Windows activation failing (0x80070005)"** | NSG blocking outbound to KMS servers | Add outbound allow rule for Internet on TCP 1688 or ensure TCP 443 to Internet is allowed (modern KMS uses HTTPS). |
| **"NSG changes not taking effect"** | NSG rule priority conflict or cache delay | Check rule priority (lower number = higher priority). Wait 5-10 minutes for rule propagation. Restart VM NIC if persistent. |
| **"Cannot create private endpoint"** | Subnet has NSG with network policies enabled | Disable network policies on subnet: `az network vnet subnet update --disable-private-endpoint-network-policies true` |

## AVD Required Outbound URLs

These FQDNs require outbound HTTPS (port 443) access from session hosts:

**AVD Core Services:**
- `*.wvd.microsoft.com` - AVD control plane
- `*.servicebus.windows.net` - Diagnostics and health monitoring
- `gcs.prod.monitoring.core.windows.net` - Geneva monitoring agent

**Azure Services:**
- `*.blob.core.windows.net` - Azure Storage (agent downloads, diagnostics)
- `*.table.core.windows.net` - Azure Table Storage (diagnostics)

**Windows Services:**
- `login.microsoftonline.com` - Azure AD authentication
- `kms.core.windows.net` - Windows activation (KMS)

**Optional (Enable for Best Experience):**
- `*.events.data.microsoft.com` - Windows diagnostic data
- `*.sfx.ms` - Microsoft Store updates
- `catalogv2.goods.microsoft.com` - Microsoft Store

> **Note:** If using Azure Firewall instead of NSGs, you can use FQDN-based application rules instead of IP-based network rules.

## Integration with Private Endpoints

When using private endpoints (e.g., pe-fslogix for Azure Files), NSGs must allow traffic to private endpoint IPs:

**Scenario:** Session hosts need to access Azure Files via private endpoint pe-fslogix

1. **Private Endpoint IP:** 10.0.3.4 (in snet-privateendpoints)
2. **Required NSG Rule on nsg-snet-sessionhosts:**
   - Source: 10.0.1.0/24 (session hosts)
   - Destination: 10.0.3.0/24 (private endpoints subnet)
   - Port: 445 (SMB)
   - Action: Allow

3. **Required NSG Rule on nsg-snet-privateendpoints:**
   - Source: 10.0.1.0/24 (session hosts)
   - Destination: 10.0.3.0/24 (private endpoints subnet)
   - Port: 445 (SMB)
   - Action: Allow

> **Warning:** Private endpoints bypass service firewalls (e.g., Azure Files firewall). Rely on NSGs and private DNS zones for security.

## Validation Checklist

- [ ] NSG created for each subnet (except GatewaySubnet, AzureBastionSubnet)
- [ ] NSGs associated with subnets (not NICs)
- [ ] Outbound rule allows AzureCloud service tag on TCP 443
- [ ] Outbound rule allows Internet on TCP 80,443
- [ ] Inbound rules follow least-privilege (no 0.0.0.0/0 allow rules)
- [ ] NSG Flow Logs enabled for troubleshooting
- [ ] Rules use service tags instead of hardcoded IPs
- [ ] Rules have descriptive names
- [ ] Priority ranges organized (100-200 allow, 4000+ deny)

## Next Steps

1. **Test Connectivity:** Deploy test session host, verify AVD agent registration
2. **Enable Flow Logs:** Configure NSG Flow Logs for security monitoring
3. **Configure Private Endpoints:** Create private endpoints with appropriate NSG rules (see Private Endpoints page)
4. **Review Logs:** Use Traffic Analytics to identify unexpected traffic patterns