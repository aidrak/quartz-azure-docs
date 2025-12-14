---
title: Hub-Spoke Topology
description: 
published: true
date: 2025-12-14T04:53:21.930Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:54.389Z
---

# Hub-Spoke Topology

Hub-Spoke network topology centralizes shared services (firewall, VPN gateway, DNS) in a hub VNet while isolating workloads in spoke VNets. Recommended for enterprise AVD deployments with multiple environments or regions.

## What is Hub-Spoke Architecture

Hub-Spoke (also called star topology) is a network design pattern where:

- **Hub VNet:** Central VNet containing shared services accessible by all spokes
- **Spoke VNets:** Workload-specific VNets (e.g., AVD production, AVD dev, other applications)
- **VNet Peering:** Connects hub to each spoke (no direct spoke-to-spoke by default)
- **Transitive Routing:** Azure Firewall or NVA (Network Virtual Appliance) in hub enables spoke-to-spoke communication

**Key Components:**
- **Hub VNet (10.100.0.0/16):**
  - AzureFirewallSubnet (10.100.1.0/26) - Centralized security
  - GatewaySubnet (10.100.0.0/27) - VPN/ExpressRoute gateway
  - snet-shared-services (10.100.2.0/24) - DNS, monitoring, jump boxes

- **Spoke VNets:**
  - AVD Production VNet (10.0.0.0/16) - Production session hosts
  - AVD Dev VNet (10.1.0.0/16) - Development/test environment
  - Corporate Apps VNet (10.2.0.0/16) - Line-of-business applications

**Benefits:**
- **Centralized Security:** Single Azure Firewall for all spokes
- **Cost Savings:** Shared VPN Gateway/ExpressRoute (no per-spoke gateways)
- **Isolation:** Spoke workloads isolated unless explicitly routed through hub
- **Scalability:** Add spokes without touching hub configuration

## When to Use Hub-Spoke

**Use Hub-Spoke When:**
- **Multiple Environments:** Production, dev, test AVD in separate VNets
- **Shared Services Needed:** Single VPN Gateway, Azure Firewall, DNS servers for all workloads
- **Security Isolation:** Different compliance zones (e.g., PCI-DSS spoke, HIPAA spoke)
- **Multi-Region Deployments:** Hub in each region, spokes for workloads
- **Central IT Control:** Network team manages hub, app teams manage spokes

**Use Single VNet Instead When:**
- **Small Deployments:** Single AVD environment, no other workloads
- **Cost Sensitive:** VNet peering costs ~$0.01/GB (can add up for high traffic)
- **Simple Networking:** No need for centralized security or shared services
- **Single Subscription:** All resources in one subscription, no isolation requirements

**Hybrid Approach:**
- Start with single VNet (vnet-avd)
- Migrate to hub-spoke when adding second environment or centralized firewall

## Real-World Example: Hub-Spoke for AVD

**Hub VNet (10.100.0.0/16) in RG-Azure-Hub:**

| Subnet | CIDR | Purpose | Resources |
|--------|------|---------|-----------|
| AzureFirewallSubnet | 10.100.1.0/26 | Centralized firewall | azfw-hub |
| GatewaySubnet | 10.100.0.0/27 | VPN/ExpressRoute | vpngw-hub or ergw-hub |
| snet-shared-services | 10.100.2.0/24 | Jump boxes, DNS | vm-jumpbox-01, vm-dns-01 |
| AzureBastionSubnet | 10.100.3.0/26 | Azure Bastion | bastion-hub |

**Spoke VNets:**

**AVD Production (10.0.0.0/16) in RG-Azure-VDI-01:**
- snet-sessionhosts (10.0.1.0/24) - Production session hosts
- snet-privateendpoints (10.0.3.0/24) - Private endpoints for storage
- Peered to Hub: Yes, allow gateway transit

**AVD Development (10.1.0.0/16) in RG-Azure-VDI-Dev:**
- snet-sessionhosts-dev (10.1.1.0/24) - Dev/test session hosts
- snet-privateendpoints-dev (10.1.3.0/24) - Dev storage private endpoints
- Peered to Hub: Yes, allow gateway transit

**Corporate Apps (10.2.0.0/16) in RG-Azure-Apps:**
- snet-webservers (10.2.1.0/24) - Web tier
- snet-appservers (10.2.2.0/24) - Application tier
- snet-database (10.2.3.0/24) - Database tier
- Peered to Hub: Yes, allow gateway transit

**Traffic Flow Example:**

```
AVD Session Host (10.0.1.5)
   |
   | Needs to access on-prem file server (172.20.20.10)
   |
   v
UDR in spoke: 172.20.20.0/24 → Azure Firewall (10.100.1.4)
   |
   v
Peering: vnet-avd → vnet-hub
   |
   v
Azure Firewall (10.100.1.4) - Checks rules, allows traffic
   |
   v
VPN Gateway (vpngw-hub) in GatewaySubnet
   |
   | VPN tunnel to on-prem
   |
   v
On-Prem File Server (172.20.20.10)
```

## How to Configure

### Step 1: Create Hub VNet

```bash
# Create Hub VNet
az network vnet create \
  --resource-group RG-Azure-Hub \
  --name vnet-hub \
  --address-prefix 10.100.0.0/16 \
  --subnet-name snet-shared-services \
  --subnet-prefix 10.100.2.0/24

# Add AzureFirewallSubnet
az network vnet subnet create \
  --resource-group RG-Azure-Hub \
  --vnet-name vnet-hub \
  --name AzureFirewallSubnet \
  --address-prefix 10.100.1.0/26

# Add GatewaySubnet
az network vnet subnet create \
  --resource-group RG-Azure-Hub \
  --vnet-name vnet-hub \
  --name GatewaySubnet \
  --address-prefix 10.100.0.0/27

# Add AzureBastionSubnet (optional)
az network vnet subnet create \
  --resource-group RG-Azure-Hub \
  --vnet-name vnet-hub \
  --name AzureBastionSubnet \
  --address-prefix 10.100.3.0/26
```

### Step 2: Deploy Shared Services in Hub

```bash
# Deploy Azure Firewall (see Azure Firewall page for details)
az network firewall create \
  --resource-group RG-Azure-Hub \
  --name azfw-hub \
  --location eastus \
  --tier Standard

# Deploy VPN Gateway (see VPN Gateway page for details)
az network vnet-gateway create \
  --resource-group RG-Azure-Hub \
  --name vpngw-hub \
  --vnet vnet-hub \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1
```

### Step 3: Create Spoke VNets

```bash
# Create AVD Production Spoke
az network vnet create \
  --resource-group RG-Azure-VDI-01 \
  --name vnet-avd \
  --address-prefix 10.0.0.0/16 \
  --subnet-name snet-sessionhosts \
  --subnet-prefix 10.0.1.0/24

# Create AVD Dev Spoke
az network vnet create \
  --resource-group RG-Azure-VDI-Dev \
  --name vnet-avd-dev \
  --address-prefix 10.1.0.0/16 \
  --subnet-name snet-sessionhosts-dev \
  --subnet-prefix 10.1.1.0/24
```

### Step 4: Create VNet Peerings

**Hub-to-Spoke Peering (allow gateway transit):**

```bash
# Peer Hub → AVD Prod Spoke
az network vnet peering create \
  --resource-group RG-Azure-Hub \
  --name peer-hub-to-avd-prod \
  --vnet-name vnet-hub \
  --remote-vnet /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Network/virtualNetworks/vnet-avd \
  --allow-gateway-transit \
  --allow-forwarded-traffic
```

**Spoke-to-Hub Peering (use remote gateway):**

```bash
# Peer AVD Prod Spoke → Hub
az network vnet peering create \
  --resource-group RG-Azure-VDI-01 \
  --name peer-avd-prod-to-hub \
  --vnet-name vnet-avd \
  --remote-vnet /subscriptions/{sub}/resourceGroups/RG-Azure-Hub/providers/Microsoft.Network/virtualNetworks/vnet-hub \
  --use-remote-gateways \
  --allow-forwarded-traffic

# Repeat for AVD Dev Spoke
az network vnet peering create \
  --resource-group RG-Azure-Hub \
  --name peer-hub-to-avd-dev \
  --vnet-name vnet-hub \
  --remote-vnet /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-Dev/providers/Microsoft.Network/virtualNetworks/vnet-avd-dev \
  --allow-gateway-transit \
  --allow-forwarded-traffic

az network vnet peering create \
  --resource-group RG-Azure-VDI-Dev \
  --name peer-avd-dev-to-hub \
  --vnet-name vnet-avd-dev \
  --remote-vnet /subscriptions/{sub}/resourceGroups/RG-Azure-Hub/providers/Microsoft.Network/virtualNetworks/vnet-hub \
  --use-remote-gateways \
  --allow-forwarded-traffic
```

### Step 5: Configure UDRs for Spoke-to-Spoke via Firewall

```bash
# Create UDR for AVD Prod Spoke
az network route-table create \
  --resource-group RG-Azure-VDI-01 \
  --name rt-avd-prod-via-hub

# Route default traffic through firewall
az network route-table route create \
  --resource-group RG-Azure-VDI-01 \
  --route-table-name rt-avd-prod-via-hub \
  --name default-to-firewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.100.1.4  # Azure Firewall IP

# Route to AVD Dev Spoke through firewall (for spoke-to-spoke)
az network route-table route create \
  --resource-group RG-Azure-VDI-01 \
  --route-table-name rt-avd-prod-via-hub \
  --name to-avd-dev \
  --address-prefix 10.1.0.0/16 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.100.1.4

# Route to on-prem through firewall
az network route-table route create \
  --resource-group RG-Azure-VDI-01 \
  --route-table-name rt-avd-prod-via-hub \
  --name to-onprem \
  --address-prefix 172.20.20.0/24 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.100.1.4

# Associate UDR to session hosts subnet
az network vnet subnet update \
  --resource-group RG-Azure-VDI-01 \
  --vnet-name vnet-avd \
  --name snet-sessionhosts \
  --route-table rt-avd-prod-via-hub
```

## VNet Peering Settings Explained

**allow-gateway-transit (Hub side):**
- Allows spoke VNets to use hub's VPN Gateway/ExpressRoute Gateway
- Only set on hub-to-spoke peering
- Spoke VMs can reach on-prem without deploying per-spoke gateways

**use-remote-gateways (Spoke side):**
- Spoke uses hub's gateway for on-prem connectivity
- Requires `allow-gateway-transit` set on hub side
- Cannot be set if spoke already has its own gateway

**allow-forwarded-traffic (Both sides):**
- Allows traffic forwarded by Azure Firewall/NVA to pass through peering
- Required for spoke-to-spoke via hub firewall
- Set on both hub-to-spoke and spoke-to-hub peerings

**allow-vnet-access (Default: true):**
- Allows basic VNet-to-VNet connectivity
- Typically left enabled

## Best Practices

**Hub Design:**
- **Single Hub per Region:** Don't create multiple hubs in same region (adds complexity)
- **Large Address Space:** Use /16 for hub (10.100.0.0/16) to accommodate future subnets
- **Dedicated Resource Group:** RG-Azure-Hub (managed by network team)
- **Zone-Redundant Services:** Use AZ SKUs for firewall, gateways (99.99% SLA)

**Spoke Design:**
- **Consistent Addressing:** Use 10.x.0.0/16 pattern (10.0.x.x = prod, 10.1.x.x = dev)
- **Non-Overlapping CIDRs:** No overlap between hub and spoke address spaces
- **Per-Workload Spokes:** Separate AVD, web apps, databases into different spokes
- **RBAC per Spoke:** App teams own spoke resource groups, network team owns hub

**Routing:**
- **UDRs in Spokes, Not Hub:** Spokes route through firewall, hub doesn't need UDRs
- **BGP for On-Prem:** Use BGP on VPN/ExpressRoute for automatic route propagation
- **Avoid Spoke-to-Spoke Peering:** Route through hub firewall for centralized logging/control

**Cost Optimization:**
- **Peering Costs:** ~$0.01/GB in each direction (can be significant for high traffic)
- **Minimize Cross-Peering Traffic:** Keep storage in same spoke as compute
- **Shared Gateway:** One VPN/ExpressRoute gateway serves all spokes (vs per-spoke gateways)

## Transitive Routing (Spoke-to-Spoke)

By default, VNet peering is **non-transitive**:

```
Spoke A (peered to Hub) CANNOT directly reach Spoke B (peered to Hub)
```

**Enable Transitive Routing with Azure Firewall:**

1. **Deploy Azure Firewall in Hub** (10.100.1.4)
2. **Create UDR in Spoke A:**
   - Route: 10.1.0.0/16 (Spoke B CIDR) → Next Hop: 10.100.1.4 (Firewall)
3. **Create UDR in Spoke B:**
   - Route: 10.0.0.0/16 (Spoke A CIDR) → Next Hop: 10.100.1.4 (Firewall)
4. **Configure Firewall Network Rule:**
   - Allow: Spoke A (10.0.0.0/16) → Spoke B (10.1.0.0/16)

**Alternative: Azure Route Server (for NVA):**
- If using third-party firewall (Palo Alto, Fortinet) instead of Azure Firewall
- Route Server propagates BGP routes from NVA to spoke VNets
- More complex, use only if Azure Firewall insufficient

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **"Spoke cannot reach on-prem"** | `use-remote-gateways` not set or gateway not deployed | Verify hub has VPN/ExpressRoute gateway deployed. Set `use-remote-gateways` on spoke-to-hub peering. Check peering status is "Connected". |
| **"Spoke-to-spoke traffic not working"** | UDR missing or firewall blocking | Add UDR in each spoke routing to other spoke via firewall. Check Azure Firewall network rules allow spoke-to-spoke. |
| **"Peering status: Disconnected"** | Address space overlap or peering misconfigured | Verify no overlapping CIDRs between hub and spoke. Delete and recreate peering. Check peering exists on both sides (hub→spoke and spoke→hub). |
| **"High data transfer costs"** | Excessive cross-peering traffic | Move frequently accessed storage to same spoke as compute. Use private endpoints instead of routing through hub. Review traffic patterns with Network Watcher. |
| **"Cannot create peering"** | Spoke has gateway and `use-remote-gateways` set | Remove spoke's local gateway if using hub's gateway. Or unset `use-remote-gateways` to use spoke's local gateway. |

## Multi-Region Hub-Spoke

For AVD deployments across multiple Azure regions:

**Architecture:**

```
Hub East US (10.100.0.0/16)
   |
   +-- Spoke AVD Prod East (10.0.0.0/16)
   +-- Spoke AVD Dev East (10.1.0.0/16)

Hub West US (10.200.0.0/16)
   |
   +-- Spoke AVD Prod West (10.10.0.0/16)
   +-- Spoke AVD DR West (10.11.0.0/16)

Global VNet Peering: Hub East ↔ Hub West
```

**Benefits:**
- Users in East US connect to East hub (low latency)
- Users in West US connect to West hub
- Hub-to-hub peering enables cross-region DR failover

**Configuration:**

1. Deploy hub infrastructure in each region
2. Peer spokes to local hub (regional peering)
3. Peer hubs globally (global VNet peering)
4. Configure UDRs for cross-region traffic through hub-to-hub

**Cost:** Global VNet peering ~$0.05/GB (5x higher than regional peering)

## Integration with AVD

**Use Cases:**
- **Environment Separation:** Prod, dev, test AVD in separate spoke VNets
- **Security Zones:** High-security AVD spoke isolated from general-purpose spoke
- **Multi-Tenant:** Different customers in separate spokes (MSP scenario)
- **Centralized Management:** Single Azure Firewall for all AVD environments

**Example Spoke Layout:**

**AVD Production Spoke (10.0.0.0/16):**
- snet-sessionhosts (10.0.1.0/24) - Production session hosts
- snet-privateendpoints (10.0.3.0/24) - FSLogix storage private endpoints
- UDR: 0.0.0.0/0 → Azure Firewall in hub

**AVD Development Spoke (10.1.0.0/16):**
- snet-sessionhosts-dev (10.1.1.0/24) - Dev session hosts
- snet-privateendpoints-dev (10.1.3.0/24) - Dev storage
- UDR: 0.0.0.0/0 → Azure Firewall in hub

**Benefits:**
- Network isolation between prod and dev (must route through hub firewall)
- Single VPN Gateway in hub serves both environments
- Centralized logging of all traffic (firewall logs)

## Monitoring Hub-Spoke

**Network Watcher:**
- Topology view: Visualize hub and spoke relationships
- Connection Monitor: Test connectivity hub↔spoke, spoke↔on-prem
- NSG Flow Logs: Identify cross-peering traffic patterns

**Azure Monitor:**
- VNet Peering Metrics: BytesTransferred, PacketsTransferred
- Firewall Metrics: Throughput, RuleHit count
- Gateway Metrics: Bandwidth, BGP peer status

**Cost Management:**
- Tag spokes by environment (Prod, Dev, Test)
- Monitor VNet peering costs per spoke
- Identify high-traffic spokes for optimization

## Validation Checklist

- [ ] Hub VNet created with AzureFirewallSubnet and GatewaySubnet
- [ ] Shared services deployed in hub (firewall, gateway, Bastion)
- [ ] Spoke VNets created with non-overlapping address spaces
- [ ] Hub-to-spoke peering created with `allow-gateway-transit` and `allow-forwarded-traffic`
- [ ] Spoke-to-hub peering created with `use-remote-gateways` and `allow-forwarded-traffic`
- [ ] Peering status shows "Connected" on both sides
- [ ] UDRs in spokes route 0.0.0.0/0 to Azure Firewall
- [ ] Spoke VMs can reach on-prem resources via hub gateway
- [ ] Spoke-to-spoke traffic routes through hub firewall (if enabled)
- [ ] Azure Firewall logs show traffic from spokes

## Next Steps

1. **Deploy Hub Infrastructure:** Create hub VNet, deploy firewall and gateway (see Azure Firewall, VPN Gateway pages)
2. **Create Spoke VNets:** Deploy AVD production and dev spokes with proper addressing
3. **Configure Peerings:** Establish hub-spoke peerings with correct settings
4. **Set Up Routing:** Create UDRs in spokes routing through hub firewall
5. **Test Connectivity:** Verify spoke VMs can reach on-prem, internet, and other spokes (if allowed)