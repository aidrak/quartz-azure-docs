---
title: ExpressRoute
description: 
published: true
date: 2025-12-14T04:53:20.289Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:52.769Z
---

# ExpressRoute

Azure ExpressRoute provides private, dedicated connectivity between on-premises networks and Azure datacenters, bypassing the public internet. Offers higher bandwidth, lower latency, and more reliable connections than VPN for enterprise AVD deployments.

## What is ExpressRoute

ExpressRoute creates a private connection between your on-premises infrastructure and Azure through a connectivity provider (e.g., AT&T, Verizon, Equinix). Key characteristics:

- **Private Connectivity:** Traffic never traverses public internet
- **Predictable Performance:** Dedicated bandwidth with SLA (99.95% availability)
- **Low Latency:** <10ms typical (vs 20-50ms for VPN)
- **High Bandwidth:** 50 Mbps to 100 Gbps (vs VPN's max 10 Gbps)
- **Global Reach:** Connect multiple on-prem sites through single ExpressRoute circuit

**Key Components:**
- **ExpressRoute Circuit:** The dedicated connection (provisioned by connectivity provider)
- **Service Provider:** Telecom/network provider (AT&T, Equinix, CenturyLink, etc.)
- **Peering:** BGP sessions for routing (Private, Microsoft, Public - deprecated)
- **ExpressRoute Gateway:** Virtual gateway in GatewaySubnet (similar to VPN Gateway)
- **Connection:** Links ExpressRoute Gateway to ExpressRoute Circuit

**Connectivity Models:**
1. **CloudExchange Co-location:** Your equipment in same datacenter as provider
2. **Point-to-Point Ethernet:** Dedicated line from your office to Azure
3. **Any-to-Any (IPVPN):** Connect via provider's MPLS network

## When to Use ExpressRoute

**Use ExpressRoute Instead of VPN When:**
- **Bandwidth >1 Gbps Required:** Large AVD deployments (500+ users), heavy multimedia workflows
- **Latency-Sensitive Applications:** CAD/CAM, video editing, real-time collaboration in AVD sessions
- **Compliance Requirements:** Data cannot traverse public internet (HIPAA, PCI-DSS, FedRAMP)
- **Predictable Performance Needed:** Mission-critical applications requiring guaranteed bandwidth
- **Multiple Azure Regions:** ExpressRoute Global Reach connects regions without separate circuits

**Stick with VPN When:**
- **Small Deployments:** <100 AVD users with light workloads
- **Budget Constraints:** ExpressRoute costs $50-$10,000+/month vs VPN Gateway $30-$300/month
- **Temporary Connectivity:** PoC, testing, short-term projects
- **Public Internet Acceptable:** No compliance restrictions

**Cost Comparison (Monthly):**
- **VPN Gateway (VpnGw1):** ~$30 + egress data transfer
- **ExpressRoute (50 Mbps Metered):** ~$50 + per-GB data transfer
- **ExpressRoute (1 Gbps Unlimited):** ~$5,000 (flat rate, no data charges)
- **ExpressRoute (10 Gbps Unlimited):** ~$40,000+

> **Warning:** ExpressRoute requires significant upfront planning and provisioning time (2-12 weeks). VPN can be deployed in hours.

## ExpressRoute vs VPN Comparison

| Feature | ExpressRoute | VPN Gateway |
|---------|--------------|-------------|
| **Bandwidth** | 50 Mbps - 100 Gbps | 100 Mbps - 10 Gbps |
| **Latency** | <10ms | 20-50ms |
| **SLA** | 99.95% | 99.9% (active-active) |
| **Encryption** | Not encrypted by default | AES-256 IPsec |
| **Connectivity** | Private, dedicated | Over public internet |
| **Setup Time** | 2-12 weeks | Hours to 1 day |
| **Cost (1 Gbps)** | ~$5,000/month | ~$140/month |
| **Best For** | Enterprise, high-bandwidth | SMB, remote access |

**When to Use Both (ExpressRoute + VPN):**
- **Redundancy:** VPN as failover if ExpressRoute circuit fails
- **Encryption Over ExpressRoute:** VPN tunnel over ExpressRoute for end-to-end encryption
- **Different Traffic Types:** ExpressRoute for production, VPN for admin access

## ExpressRoute Peering Types

### Private Peering (REQUIRED for AVD)

Connects to Azure VNets (IaaS resources like VMs, storage accounts with private endpoints):

- **Use Case:** AVD session hosts, file servers, VNets
- **Routing:** BGP exchange between on-prem and Azure VNets
- **Address Space:** On-prem and Azure VNet ranges (must not overlap)
- **Example Routes:**
  - On-prem advertises: 172.20.0.0/16
  - Azure advertises: 10.0.0.0/16 (vnet-avd)

### Microsoft Peering (Optional for AVD)

Connects to Microsoft PaaS services (Office 365, Dynamics 365, Azure public endpoints):

- **Use Case:** Access to Azure Storage public endpoints, Azure AD
- **Routing:** BGP exchange between on-prem and Microsoft public IPs
- **Not Needed for AVD:** AVD control plane uses public internet (no ExpressRoute benefit)

### Public Peering (DEPRECATED)

Replaced by Microsoft Peering. Do not use for new deployments.

## ExpressRoute Circuit SKUs

| SKU | Provider | Bandwidth Options | Billing Model |
|-----|----------|-------------------|---------------|
| **Local** | Any | 50 Mbps - 10 Gbps | Unlimited data, flat rate |
| **Standard** | Any | 50 Mbps - 10 Gbps | Metered or Unlimited |
| **Premium** | Any | 50 Mbps - 100 Gbps | Metered or Unlimited |

**Local SKU:**
- Lowest cost (~$50-$300/month)
- Only connects to Azure regions in same metro area (e.g., Dallas ExpressRoute → South Central US, North Central US)
- Sufficient for most single-region AVD deployments

**Standard SKU:**
- Connects to all Azure regions in same geopolitical area (e.g., US, Europe, Asia)
- ~$500-$5,000/month depending on bandwidth

**Premium SKU:**
- Global connectivity (any region worldwide)
- Required for ExpressRoute Global Reach (connect on-prem sites via Azure backbone)
- ~$1,000-$10,000+/month

**Metered vs Unlimited:**
- **Metered:** Pay per GB egress (~$0.02-$0.05/GB) - good for <1 TB/month
- **Unlimited:** Flat monthly rate regardless of data transfer - good for >5 TB/month

## How to Configure (High-Level)

> **Note:** ExpressRoute setup requires coordination with connectivity provider. Steps vary by provider.

### Step 1: Order ExpressRoute Circuit

1. **Choose Provider:** Select from Azure Portal → ExpressRoute → Supported Providers
   - Examples: Equinix, Verizon, AT&T, CenturyLink, Megaport
   - Check provider coverage in your location

2. **Select Bandwidth:** 50 Mbps, 100 Mbps, 200 Mbps, 500 Mbps, 1 Gbps, 2 Gbps, 5 Gbps, 10 Gbps

3. **Choose SKU:** Local, Standard, or Premium

4. **Create Circuit in Azure:**
   - Azure Portal → ExpressRoute Circuits → Create
   - Provider: Select your provider
   - Peering Location: Provider's datacenter (e.g., "Dallas - Equinix DA1")
   - Bandwidth: 1 Gbps
   - SKU: Local (or Standard/Premium)
   - Billing Model: Unlimited

5. **Retrieve Service Key:**
   - After circuit creation, Azure generates Service Key
   - Provide Service Key to connectivity provider

### Step 2: Provider Provisions Circuit

1. Contact connectivity provider with Service Key
2. Provider configures their equipment (can take 2-12 weeks)
3. Provider notifies you when circuit is "Provisioned"

### Step 3: Configure Private Peering

1. Navigate to ExpressRoute Circuit → Peerings → Add
2. **Peering Type:** Azure private
3. **Peer ASN:** Your BGP ASN (e.g., 65001)
4. **Primary Subnet:** /30 subnet for BGP peering (e.g., 192.168.1.0/30)
5. **Secondary Subnet:** /30 subnet for redundancy (e.g., 192.168.1.4/30)
6. **VLAN ID:** Provided by connectivity provider
7. **Shared Key:** Optional MD5 hash for BGP authentication

### Step 4: Create ExpressRoute Gateway

1. Ensure GatewaySubnet exists in vnet-avd (/27 or larger)

2. **Azure Portal → Virtual Network Gateways → Create**
   - Resource Group: RG-Azure-VDI-01
   - Name: ergw-avd
   - Region: East US
   - Gateway Type: ExpressRoute
   - SKU: Standard (or HighPerformance/UltraPerformance)
   - Virtual Network: vnet-avd

3. **Create** (takes 30-45 minutes)

### Step 5: Link Gateway to Circuit

1. Navigate to ExpressRoute Gateway → Connections → Add
2. **Name:** conn-ergw-to-circuit
3. **Connection Type:** ExpressRoute
4. **ExpressRoute Circuit:** Select your circuit
5. **Create**

### Azure CLI Example

```bash
# Create ExpressRoute Circuit
az network express-route create \
  --resource-group RG-Azure-VDI-01 \
  --name er-circuit-avd \
  --peering-location "Dallas - Equinix DA1" \
  --bandwidth 1000 \
  --provider "Equinix" \
  --sku-family UnlimitedData \
  --sku-tier Local

# Get Service Key (provide to connectivity provider)
az network express-route show \
  --resource-group RG-Azure-VDI-01 \
  --name er-circuit-avd \
  --query "serviceKey" \
  --output tsv

# Create ExpressRoute Gateway (after circuit provisioned)
az network public-ip create \
  --resource-group RG-Azure-VDI-01 \
  --name pip-ergw-avd \
  --allocation-method Static \
  --sku Standard

az network vnet-gateway create \
  --resource-group RG-Azure-VDI-01 \
  --name ergw-avd \
  --vnet vnet-avd \
  --public-ip-addresses pip-ergw-avd \
  --gateway-type ExpressRoute \
  --sku Standard

# Link Gateway to Circuit (replace {circuit-id} with your circuit resource ID)
az network vpn-connection create \
  --resource-group RG-Azure-VDI-01 \
  --name conn-ergw-to-circuit \
  --vnet-gateway1 ergw-avd \
  --express-route-circuit2 {circuit-id}
```

## ExpressRoute Gateway SKUs

| SKU | Bandwidth | Cost (approx/month) | Use Case |
|-----|-----------|---------------------|----------|
| **Standard** | 1 Gbps | ~$200 | Small to medium AVD (up to 200 users) |
| **HighPerformance** | 2 Gbps | ~$500 | Medium to large AVD (200-500 users) |
| **UltraPerformance** | 10 Gbps | ~$1,500 | Enterprise AVD (500+ users), heavy workloads |
| **ErGw1AZ** | 1 Gbps | ~$250 | Zone-redundant, small to medium AVD |
| **ErGw2AZ** | 2 Gbps | ~$625 | Zone-redundant, medium to large AVD |
| **ErGw3AZ** | 10 Gbps | ~$1,875 | Zone-redundant, enterprise AVD |

> **Note:** AZ SKUs provide zone redundancy (99.99% SLA vs 99.95% for non-AZ).

## Best Practices

**Design:**
- **Use ExpressRoute Local for Single Region:** Saves cost if AVD resources in one region
- **Redundant Circuits:** Deploy two circuits to different providers/peering locations for 99.99% SLA
- **Zone-Redundant Gateway:** Use ErGw1AZ or higher for production AVD
- **Right-Size Gateway:** Start with Standard, upgrade if bandwidth consistently >80%

**Security:**
- **Enable MACsec:** Layer 2 encryption for ExpressRoute Direct (100 Gbps ports)
- **VPN Over ExpressRoute:** For end-to-end encryption (adds latency)
- **NSG on Subnets:** ExpressRoute doesn't encrypt traffic, rely on NSGs for access control
- **Private Endpoints:** Use private endpoints for Azure PaaS (Storage, SQL) accessed via ExpressRoute

**Performance:**
- **BGP Tuning:** Adjust BGP weights to prefer primary circuit
- **Monitor Circuit Utilization:** Set alerts at 80% bandwidth threshold
- **QoS on On-Prem:** Prioritize AVD traffic (RDP, multimedia) over bulk transfers
- **FastPath:** Enable for direct datapath (bypasses gateway, lower latency)

**Cost Optimization:**
- **Unlimited vs Metered:** Use unlimited if >5 TB/month egress
- **Share Circuit:** Multiple VNets can use same ExpressRoute circuit (up to 10 VNets per Standard gateway)
- **Downgrade SKU:** If bandwidth usage low, downgrade from Premium to Standard

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **"Circuit status: Not provisioned"** | Connectivity provider hasn't completed setup | Contact provider with Service Key. Check circuit status in Azure Portal. Can take 2-12 weeks. |
| **"BGP sessions down"** | Incorrect peering config or on-prem routing issue | Verify VLAN ID, ASN, and subnets match provider's config. Check BGP status in Azure Portal → ExpressRoute Circuit → Peerings. |
| **"Can connect to Azure VMs but not PaaS services"** | Missing Microsoft Peering for public endpoints | Configure Microsoft Peering (or use private endpoints for PaaS services). AVD control plane uses public internet, not ExpressRoute. |
| **"Intermittent connectivity"** | BGP flapping or circuit redundancy issue | Check for duplicate BGP advertisements. Verify both primary and secondary peering links healthy. Review provider's circuit health. |
| **"High latency despite ExpressRoute"** | Traffic routing through VPN or suboptimal path | Verify effective routes on session host NIC show ExpressRoute gateway as next hop. Disable VPN if ExpressRoute is primary. |

## ExpressRoute + VPN Coexistence

Run ExpressRoute and VPN Gateway in parallel for redundancy:

**Architecture:**
- **Primary:** ExpressRoute for production AVD traffic
- **Backup:** VPN Gateway activates if ExpressRoute fails
- **Requirements:** GatewaySubnet must be /27 or larger

**Configuration:**

1. Deploy ExpressRoute Gateway (as shown above)
2. Deploy VPN Gateway in same vnet-avd:
   ```bash
   az network vnet-gateway create \
     --resource-group RG-Azure-VDI-01 \
     --name vpngw-avd-backup \
     --vnet vnet-avd \
     --public-ip-addresses pip-vpngw-backup \
     --gateway-type Vpn \
     --vpn-type RouteBased \
     --sku VpnGw1
   ```

3. Configure BGP on VPN Gateway with higher AS Path prepend (lower priority):
   - ExpressRoute routes have AS Path length 1 (preferred)
   - VPN routes prepended to AS Path length 3 (backup)

**Failover Behavior:**
- ExpressRoute fails → Azure automatically routes traffic through VPN
- Recovery time: 30-60 seconds (BGP reconvergence)

## Integration with AVD

**Use Cases:**
- **Hybrid Identity:** Session hosts domain-join to on-prem AD DS via ExpressRoute (low latency)
- **FSLogix Profiles:** Access on-prem file servers for profiles (better performance than VPN)
- **High-Bandwidth Apps:** CAD/CAM, video editing, large dataset transfers
- **Multi-Region AVD:** ExpressRoute Global Reach connects AVD deployments in multiple regions

**Network Flow Example:**

```
AVD User (Remote Location)
   |
   | HTTPS (443) - AVD Gateway (public internet)
   |
   v
AVD Session Host (10.0.1.5)
   |
   | ExpressRoute Private Peering
   |
   v
ExpressRoute Gateway (ergw-avd)
   |
   | Dedicated Line (not over internet)
   |
   v
ExpressRoute Circuit
   |
   | Provider's Network
   |
   v
On-Prem Network (172.20.0.0/16)
```

**Performance Benefits:**
- **RDP Latency:** <10ms (vs 30-50ms over VPN)
- **FSLogix Profile Load Time:** 5-10 seconds (vs 15-30 seconds over VPN)
- **File Server Throughput:** Up to 10 Gbps (vs 1 Gbps max for VPN)

## Validation Checklist

- [ ] ExpressRoute circuit provisioned by provider (status: Provisioned)
- [ ] Private peering configured (BGP sessions: Connected)
- [ ] ExpressRoute Gateway deployed in GatewaySubnet
- [ ] Connection created linking gateway to circuit
- [ ] BGP routes advertised from on-prem to Azure (verify effective routes on session host NIC)
- [ ] Can ping on-prem devices from Azure VMs (and vice versa)
- [ ] Latency <10ms between Azure and on-prem (Test-NetConnection or ping)
- [ ] Bandwidth usage monitored (set alerts at 80% threshold)

## Next Steps

1. **Monitor Performance:** Set up Azure Monitor for circuit bandwidth, BGP status
2. **Configure Routing:** Verify session hosts use ExpressRoute for on-prem traffic (see VNet Design page)
3. **Enable FastPath:** Bypass gateway for lower latency (requires UltraPerformance gateway)
4. **Plan for HA:** Deploy secondary circuit or VPN backup (see VPN Gateway page)