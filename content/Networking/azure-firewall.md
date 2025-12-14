---
title: Azure Firewall
description: 
published: true
date: 2025-12-14T04:53:18.588Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:51.195Z
---

# Azure Firewall

Azure Firewall is a cloud-native, stateful firewall with built-in high availability and FQDN filtering. Provides centralized network and application-level protection for AVD deployments when NSGs alone are insufficient.

## What is Azure Firewall

Azure Firewall is a managed, cloud-based network security service (FaaS - Firewall as a Service) that protects Azure VNet resources. Key features:

- **Stateful Packet Inspection:** Tracks connection state (like NSGs but more advanced)
- **FQDN Filtering:** Allow/deny by domain name (e.g., `*.microsoft.com`) instead of IP addresses
- **Application Rules:** Layer 7 filtering (HTTP/HTTPS based on URL, not just IP:port)
- **Network Rules:** Layer 4 filtering (IP:port, similar to NSG but centralized)
- **Threat Intelligence:** Auto-blocks traffic to/from known malicious IPs (Microsoft threat feed)
- **Built-in HA:** 99.99% SLA with zone redundancy
- **Centralized Logging:** All traffic logs to Log Analytics, Storage, or Event Hub

**Azure Firewall vs NSGs:**

| Feature | Azure Firewall | NSG |
|---------|----------------|-----|
| **Layer** | L3-L7 (Network, Application, FQDN) | L3-L4 (IP, Port) |
| **Filtering** | FQDN, URL path, TLS inspection | IP address, port, protocol |
| **Deployment** | Hub VNet (centralized) | Subnet or NIC (distributed) |
| **Cost** | ~$1.25/hour + data | Free |
| **Use Case** | Enterprise, centralized policy | Small to medium, per-subnet rules |

## When to Use Azure Firewall

**Use Azure Firewall When:**
- **FQDN Filtering Required:** Need to allow `*.wvd.microsoft.com` without tracking IP ranges
- **Centralized Policy:** Managing 10+ subnets/VNets with consistent rules
- **Threat Intelligence Needed:** Auto-blocking known malicious IPs critical for compliance
- **TLS Inspection Required:** Decrypt and inspect HTTPS traffic (Premium SKU)
- **Hub-Spoke Topology:** Centralized security in hub VNet for multiple spoke VNets
- **Forced Tunneling:** Route all internet traffic through on-prem firewall via Azure Firewall

**Use NSGs Instead When:**
- **Small Deployments:** 1-5 subnets with simple allow/deny rules
- **Budget Constraints:** Azure Firewall costs ~$900/month vs NSGs are free
- **IP-Based Rules Sufficient:** All required rules can be expressed as IP:port (no FQDNs)
- **Per-Subnet Isolation:** Different security policies per subnet (not centralized)

**Use Both (Recommended for Production):**
- **Azure Firewall:** Centralized FQDN filtering, threat intelligence
- **NSGs:** Additional subnet-level controls (defense in depth)

## Azure Firewall SKUs

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| **Throughput** | 250 Mbps | 30 Gbps | 30 Gbps |
| **Availability Zones** | No | Yes | Yes |
| **Network Rules** | Yes | Yes | Yes |
| **Application Rules (FQDN)** | Yes | Yes | Yes |
| **Threat Intelligence** | Alert Only | Alert + Deny | Alert + Deny |
| **TLS Inspection** | No | No | Yes |
| **IDPS (Intrusion Detection)** | No | No | Yes |
| **URL Filtering** | No | No | Yes |
| **Web Categories** | No | No | Yes (Social Media, Gambling, etc.) |
| **Cost (approx/month)** | ~$150 | ~$900 | ~$1,500 |

**Recommendations:**
- **Small AVD (<100 users):** NSGs only, skip Azure Firewall
- **Medium AVD (100-500 users):** Standard SKU if FQDN filtering needed
- **Enterprise AVD (500+ users):** Premium SKU for TLS inspection and IDPS

> **Note:** Basic SKU is preview-only and not recommended for production.

## Real-World Example: Azure Firewall for AVD

**Scenario:** Centralized security for AVD session hosts with FQDN-based outbound rules

**Configuration:**
- **Firewall Name:** azfw-hub
- **SKU:** Standard
- **Location:** East US
- **Subnet:** AzureFirewallSubnet (10.100.1.0/26 in hub VNet)
- **Public IP:** pip-azfw-hub (for outbound NAT)
- **Firewall Policy:** afwp-avd-outbound

**Routing:**
- Session host subnet (10.0.1.0/24) has UDR (User Defined Route):
  - Route: 0.0.0.0/0 → Next Hop: Azure Firewall (10.100.1.4)
  - Effect: All internet-bound traffic routes through firewall

**Application Rules (Allow Outbound HTTPS):**

| Priority | Rule Collection | Rule Name | Source | Destination FQDN | Action |
|----------|-----------------|-----------|--------|------------------|--------|
| 100 | AVD-Required | Allow-AVD-Control-Plane | 10.0.1.0/24 | *.wvd.microsoft.com, *.servicebus.windows.net | Allow |
| 110 | AVD-Required | Allow-Windows-Activation | 10.0.1.0/24 | kms.core.windows.net, azkms.core.windows.net | Allow |
| 120 | AVD-Required | Allow-Azure-Storage | 10.0.1.0/24 | *.blob.core.windows.net, *.table.core.windows.net | Allow |
| 200 | General-Internet | Allow-Windows-Update | 10.0.1.0/24 | *.windowsupdate.com, *.delivery.mp.microsoft.com | Allow |
| 300 | General-Internet | Allow-Certificate-Validation | 10.0.1.0/24 | *.digicert.com, *.verisign.com | Allow |

**Network Rules (Allow Outbound Non-HTTP):**

| Priority | Rule Collection | Rule Name | Source | Destination | Port/Protocol | Action |
|----------|-----------------|-----------|--------|-------------|---------------|--------|
| 100 | Time-Sync | Allow-NTP | 10.0.1.0/24 | * | 123/UDP | Allow |
| 110 | DNS | Allow-DNS | 10.0.1.0/24 | 168.63.129.16 | 53/UDP | Allow |

**Benefits:**
- Session hosts blocked from unapproved domains (e.g., social media, torrents)
- No need to track changing Microsoft IP ranges (FQDN rules auto-update)
- Threat intelligence blocks known malicious IPs
- All outbound traffic logged to Log Analytics for security audits

## How to Configure

### Azure Portal

**Step 1: Create Firewall Subnet**

1. Navigate to hub VNet → Subnets → Add Subnet
2. **Name:** AzureFirewallSubnet (EXACT name required)
3. **Address Range:** 10.100.1.0/26 (minimum /26)
4. **Save**

**Step 2: Create Firewall**

1. **Azure Portal → Firewalls → Create**
2. **Basics:**
   - Resource Group: RG-Azure-Hub
   - Name: azfw-hub
   - Region: East US
   - Availability Zone: Zone 1, 2, 3 (for zone redundancy)
   - Firewall SKU: Standard
   - Firewall Management: Use Firewall Policy (not classic rules)
   - Choose Virtual Network: vnet-hub
   - Public IP Address: Create new (pip-azfw-hub)

3. **Firewall Policy:**
   - Create New: afwp-avd-outbound
   - Policy Tier: Standard

4. **Review + Create** (takes 5-10 minutes)

**Step 3: Configure Firewall Policy Rules**

1. Navigate to Firewall Policy (afwp-avd-outbound) → Application Rules → Add Rule Collection
2. **Rule Collection:**
   - Name: AVD-Required-HTTPS
   - Priority: 100
   - Action: Allow

3. **Add Rules:**
   - Rule 1:
     - Name: Allow-AVD-Control-Plane
     - Source Type: IP Address
     - Source: 10.0.1.0/24
     - Protocol: HTTPS
     - Destination Type: FQDN
     - Destination: *.wvd.microsoft.com, *.servicebus.windows.net

   - Rule 2:
     - Name: Allow-Windows-Activation
     - Source: 10.0.1.0/24
     - Protocol: HTTPS
     - Destination Type: FQDN
     - Destination: kms.core.windows.net, azkms.core.windows.net

4. **Save**

**Step 4: Create UDR to Route Traffic Through Firewall**

1. **Azure Portal → Route Tables → Create**
2. **Basics:**
   - Resource Group: RG-Azure-VDI-01
   - Name: rt-session-hosts-via-firewall
   - Region: East US
   - Propagate Gateway Routes: No

3. **Add Route:**
   - Navigate to Route Table → Routes → Add
   - Route Name: default-to-firewall
   - Destination Type: IP Addresses
   - Destination IP Addresses/CIDR: 0.0.0.0/0
   - Next Hop Type: Virtual Appliance
   - Next Hop Address: 10.100.1.4 (Azure Firewall private IP)

4. **Associate to Subnet:**
   - Navigate to Route Table → Subnets → Associate
   - Virtual Network: vnet-avd
   - Subnet: snet-sessionhosts

### Azure CLI

```bash
# Create Firewall
az network firewall create \
  --resource-group RG-Azure-Hub \
  --name azfw-hub \
  --location eastus \
  --tier Standard \
  --enable-dns-proxy true

# Create Public IP for Firewall
az network public-ip create \
  --resource-group RG-Azure-Hub \
  --name pip-azfw-hub \
  --allocation-method Static \
  --sku Standard

# Assign Public IP to Firewall
az network firewall ip-config create \
  --resource-group RG-Azure-Hub \
  --firewall-name azfw-hub \
  --name azfw-ipconfig \
  --vnet-name vnet-hub \
  --public-ip-address pip-azfw-hub

# Get Firewall Private IP (for UDR)
FIREWALL_IP=$(az network firewall show \
  --resource-group RG-Azure-Hub \
  --name azfw-hub \
  --query "ipConfigurations[0].privateIPAddress" \
  --output tsv)

# Create Firewall Policy
az network firewall policy create \
  --resource-group RG-Azure-Hub \
  --name afwp-avd-outbound \
  --sku Standard

# Create Application Rule Collection
az network firewall policy rule-collection-group create \
  --resource-group RG-Azure-Hub \
  --policy-name afwp-avd-outbound \
  --name AVD-Required-Rules \
  --priority 100

az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group RG-Azure-Hub \
  --policy-name afwp-avd-outbound \
  --rule-collection-group-name AVD-Required-Rules \
  --name AVD-HTTPS \
  --collection-priority 100 \
  --action Allow \
  --rule-name Allow-AVD-Control-Plane \
  --rule-type ApplicationRule \
  --source-addresses 10.0.1.0/24 \
  --protocols Https=443 \
  --target-fqdns "*.wvd.microsoft.com" "*.servicebus.windows.net"

# Create UDR
az network route-table create \
  --resource-group RG-Azure-VDI-01 \
  --name rt-session-hosts-via-firewall

az network route-table route create \
  --resource-group RG-Azure-VDI-01 \
  --route-table-name rt-session-hosts-via-firewall \
  --name default-to-firewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FIREWALL_IP

# Associate UDR to Session Hosts Subnet
az network vnet subnet update \
  --resource-group RG-Azure-VDI-01 \
  --vnet-name vnet-avd \
  --name snet-sessionhosts \
  --route-table rt-session-hosts-via-firewall
```

## Best Practices

**Rule Organization:**
- **Use Firewall Policies:** Not classic rules (policies support inheritance, versioning)
- **Rule Collections by Purpose:** AVD-Required, Windows-Updates, LOB-Apps (separate collections)
- **Priority Ranges:** 100-199 critical, 200-299 standard, 300+ optional
- **FQDN Tags:** Use built-in tags (WindowsUpdate, AzureBackup) instead of manual FQDNs

**Performance:**
- **Right-Size SKU:** Standard sufficient for <10 Gbps, Premium for TLS inspection
- **Enable DNS Proxy:** Firewall resolves FQDNs instead of clients (better caching)
- **Monitor Throughput:** Set alerts at 80% of SKU limit
- **Minimize Rules:** Consolidate FQDNs into single rule (e.g., *.microsoft.com vs listing each subdomain)

**Security:**
- **Threat Intelligence:** Always enable in "Alert and Deny" mode
- **IDPS (Premium):** Enable for known exploit detection
- **Web Categories (Premium):** Block social media, gambling, adult content
- **Log All Traffic:** Send firewall logs to Log Analytics for SIEM integration

**Cost Optimization:**
- **Shared Firewall:** One firewall serves multiple spoke VNets (hub-spoke topology)
- **Scheduled Shutdown:** Stop/start firewall for non-production (not recommended for prod)
- **Review Logs:** Identify unused rules, clean up to improve performance
- **Cost:** ~$900/month (Standard) + egress data processing (~$0.016/GB)

## AVD Required FQDNs

Configure these application rules for AVD session hosts:

**Critical (AVD Won't Function Without These):**

```
# AVD Control Plane
*.wvd.microsoft.com
*.servicebus.windows.net
gcs.prod.monitoring.core.windows.net

# Azure Storage (agent downloads, diagnostics)
*.blob.core.windows.net
*.table.core.windows.net

# Windows Activation
kms.core.windows.net
azkms.core.windows.net

# Azure AD Authentication
login.microsoftonline.com
*.login.microsoftonline.com
```

**Recommended (Best User Experience):**

```
# Windows Update (use FQDN tag: WindowsUpdate)
*.windowsupdate.com
*.delivery.mp.microsoft.com

# Certificate Validation
*.digicert.com
*.verisign.com
ocsp.msocsp.com

# Microsoft Store
*.goods.microsoft.com
catalogv2.goods.microsoft.com
```

**Optional (Enhanced Features):**

```
# Diagnostic Data
*.events.data.microsoft.com

# OneDrive
*.onedrive.com
*.sharepoint.com

# Microsoft 365
*.office.com
*.office365.com
```

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **"Session hosts can't connect to AVD"** | UDR routing traffic to firewall but no allow rule | Verify UDR next hop is firewall IP. Check firewall logs for denied traffic. Add application rule allowing `*.wvd.microsoft.com`. |
| **"Asymmetric routing"** | Return traffic not routed through firewall | Ensure firewall has route back to session hosts. Check NSGs don't bypass firewall. Enable "BGP propagation" on UDR if using ExpressRoute. |
| **"Firewall blocking Azure Files private endpoint"** | UDR routing all traffic (including VNet) through firewall | Add UDR rule with higher priority: 10.0.0.0/16 → Virtual Network (bypasses firewall for intra-VNet traffic). Or add firewall network rule allowing VNet-to-VNet. |
| **"High latency after deploying firewall"** | All traffic hairpinning through firewall | Use FastPath for ExpressRoute. Add UDR exceptions for VNet-to-VNet traffic. Check firewall throughput (may need to upgrade SKU). |
| **"Cannot create firewall"** | AzureFirewallSubnet missing or wrong name/size | Create subnet named exactly "AzureFirewallSubnet" with minimum /26 CIDR. |

## Forced Tunneling

Route all internet traffic through on-prem firewall instead of Azure Firewall:

**Use Case:** Compliance requires all traffic inspected by on-prem appliance

**Configuration:**

1. Add UDR to AzureFirewallManagementSubnet (for firewall management traffic):
   ```bash
   az network route-table route create \
     --resource-group RG-Azure-Hub \
     --route-table-name rt-firewall-mgmt \
     --name default-to-onprem \
     --address-prefix 0.0.0.0/0 \
     --next-hop-type VirtualNetworkGateway  # Routes to VPN/ExpressRoute Gateway
   ```

2. Create firewall with forced tunneling enabled:
   ```bash
   az network firewall create \
     --resource-group RG-Azure-Hub \
     --name azfw-hub \
     --enable-forced-tunneling true
   ```

**Limitations:**
- Requires AzureFirewallManagementSubnet (/26) in addition to AzureFirewallSubnet
- Adds latency (traffic goes to on-prem then back to Azure)
- On-prem firewall must allow Azure management traffic

## Monitoring and Logging

**Enable Diagnostic Settings:**

1. Navigate to Azure Firewall → Diagnostic Settings → Add
2. **Logs:**
   - AzureFirewallApplicationRule
   - AzureFirewallNetworkRule
   - AzureFirewallDnsProxy
3. **Destination:**
   - Log Analytics Workspace (for queries)
   - Storage Account (for long-term retention)
4. **Retention:** 30-90 days

**Query Denied Traffic (Log Analytics):**

```kql
AzureDiagnostics
| where Category == "AzureFirewallApplicationRule" or Category == "AzureFirewallNetworkRule"
| where msg_s contains "Deny"
| project TimeGenerated, msg_s
| order by TimeGenerated desc
| take 100
```

**Top Blocked FQDNs:**

```kql
AzureDiagnostics
| where Category == "AzureFirewallApplicationRule"
| where msg_s contains "Deny"
| parse msg_s with * "FQDN: " fqdn " " *
| summarize count() by fqdn
| order by count_ desc
| take 20
```

## Validation Checklist

- [ ] AzureFirewallSubnet created (/26 or larger)
- [ ] Azure Firewall deployed in hub VNet (status: Succeeded)
- [ ] Firewall policy created with application/network rules
- [ ] UDR created routing 0.0.0.0/0 to firewall private IP
- [ ] UDR associated with session hosts subnet
- [ ] AVD required FQDNs allowed in application rules
- [ ] Diagnostic logs enabled and sent to Log Analytics
- [ ] Session hosts can reach AVD control plane (verify agent registration)
- [ ] Firewall logs reviewed for denied traffic (none expected for AVD)

## Next Steps

1. **Test Connectivity:** Deploy test session host, verify AVD agent registers
2. **Review Logs:** Identify blocked traffic, add rules as needed
3. **Enable Threat Intelligence:** Monitor for alerts on malicious IPs
4. **Optimize Rules:** Consolidate FQDNs, remove unused rules