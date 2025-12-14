---
title: Variant Hybrid Connectivity
description: 
published: true
date: 2025-12-14T04:54:30.966Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:41.441Z
---

# Variant: Hybrid Connectivity

This variant extends the base deployment with site-to-site VPN or ExpressRoute connectivity to on-premises networks, enabling access to on-prem resources from AVD session hosts.

## When to Use This Variant

Use hybrid connectivity when:
- Session hosts need access to on-premises file servers, databases, or applications
- Users need to access on-prem intranet sites from AVD desktops
- Domain controllers are on-premises (Hybrid AD Join scenario)
- DNS resolution for on-prem resources is required
- Compliance requires traffic to stay on private networks (no internet egress for internal traffic)

## Pre-Deployment Checklist

- [ ] On-premises network requirements documented (IP ranges, DNS, bandwidth)
- [ ] VPN device available and compatible (Cisco, Palo Alto, Fortinet, etc.)
- [ ] GatewaySubnet created in vnet-avd (10.0.0.0/27)
- [ ] Public IP for VPN Gateway
- [ ] Shared key for IPsec tunnel
- [ ] On-prem DNS server IPs
- [ ] Firewall rules approved for required ports

## Our Deployed Configuration

**VPN Gateway:** `vpngw-avd`
**Local Network Gateway:** `lgw-home`
**Connection:** `conn-home`
**On-Premises Network:** 172.20.20.0/24

## Step 1: Create VPN Gateway

If not already created during network setup, deploy the VPN Gateway.

**Portal Path:** Azure Portal → Virtual Network Gateways → Create

**Configuration:**
| Setting | Value | Rationale |
|---------|-------|-----------|
| **Name** | `vpngw-avd` | Matches naming convention |
| **Region** | `Central US` | Same as VNet |
| **Gateway Type** | VPN | Site-to-site VPN |
| **VPN Type** | Route-based | Required for most modern VPN devices |
| **SKU** | VpnGw1 | Up to 650 Mbps, sufficient for most AVD |
| **Generation** | Generation1 | Use Gen2 for higher throughput |
| **Virtual Network** | vnet-avd | |
| **Gateway Subnet** | 10.0.0.0/27 (GatewaySubnet) | Pre-created |
| **Public IP** | Create new: `pip-vpngw-avd` | |

**Deployment Time:** 30-45 minutes (Azure provisions gateway VMs)

**Azure CLI:**
```bash
# Create public IP
az network public-ip create \
  --resource-group RG-Azure-VDI-01 \
  --name pip-vpngw-avd \
  --allocation-method Static \
  --sku Standard

# Create VPN Gateway
az network vnet-gateway create \
  --resource-group RG-Azure-VDI-01 \
  --name vpngw-avd \
  --vnet vnet-avd \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1 \
  --public-ip-address pip-vpngw-avd \
  --no-wait
```

## Step 2: Create Local Network Gateway

The Local Network Gateway represents your on-premises network in Azure.

**Portal Path:** Azure Portal → Local Network Gateways → Create

**Configuration:**
| Setting | Value |
|---------|-------|
| **Name** | `lgw-home` |
| **Resource Group** | `RG-Azure-VDI-01` |
| **Region** | `Central US` |
| **IP Address** | Your on-prem VPN device public IP |
| **Address Space** | 172.20.20.0/24 (on-prem network CIDR) |

**Add Additional Address Spaces:**
If on-prem has multiple subnets, add each:
- 172.20.20.0/24 (LAN)
- 172.20.21.0/24 (DMZ)
- 10.1.0.0/16 (Data Center)

**Azure CLI:**
```bash
az network local-gateway create \
  --resource-group RG-Azure-VDI-01 \
  --name lgw-home \
  --gateway-ip-address <your-onprem-public-ip> \
  --local-address-prefixes 172.20.20.0/24
```

## Step 3: Create VPN Connection

Connect the VPN Gateway to the Local Network Gateway.

**Portal Path:** Azure Portal → Virtual Network Gateways → vpngw-avd → Connections → Add

**Configuration:**
| Setting | Value |
|---------|-------|
| **Name** | `conn-home` |
| **Connection Type** | Site-to-site (IPsec) |
| **Virtual Network Gateway** | vpngw-avd |
| **Local Network Gateway** | lgw-home |
| **Shared Key (PSK)** | Strong key (32+ characters) |
| **IKE Protocol** | IKEv2 (preferred) |

**Azure CLI:**
```bash
az network vpn-connection create \
  --resource-group RG-Azure-VDI-01 \
  --name conn-home \
  --vnet-gateway1 vpngw-avd \
  --local-gateway2 lgw-home \
  --shared-key "YourSuperSecureSharedKey123!@#"
```

## Step 4: Configure On-Premises VPN Device

Configure your on-prem VPN device to establish the tunnel. Azure provides device-specific configuration scripts.

**Download Configuration Script:**
1. Azure Portal → Virtual Network Gateways → vpngw-avd → Connections → conn-home
2. Click "Download configuration"
3. Select your device vendor (Cisco, Palo Alto, Fortinet, etc.)
4. Download and apply configuration to on-prem device

**Key Parameters for Manual Configuration:**
| Parameter | Value |
|-----------|-------|
| Azure VPN Gateway IP | (Public IP of pip-vpngw-avd) |
| Shared Key (PSK) | (Same as configured in Azure) |
| Azure VNet CIDR | 10.0.0.0/16 |
| IKE Version | IKEv2 |
| Phase 1 (IKE) | AES256, SHA256, DH Group 2 |
| Phase 2 (IPsec) | AES256, SHA256, PFS Group 2 |

**Verify Connection Status:**
```bash
az network vpn-connection show \
  --resource-group RG-Azure-VDI-01 \
  --name conn-home \
  --query connectionStatus
```

Expected output: `"Connected"`

## Step 5: Configure DNS for Hybrid Resolution

Session hosts need to resolve on-premises DNS names (e.g., fileserver.corp.local).

**Option A: Custom DNS Servers on VNet**

**Portal Path:** Azure Portal → Virtual Networks → vnet-avd → DNS Servers

**Configuration:**
| Setting | Value |
|---------|-------|
| **DNS Servers** | Custom |
| **Server 1** | 172.20.20.10 (on-prem DNS) |
| **Server 2** | 168.63.129.16 (Azure DNS for Azure services) |

**Important:** Include Azure DNS (168.63.129.16) to resolve Azure Private DNS zones (e.g., privatelink.file.core.windows.net for FSLogix).

**Option B: Azure DNS Private Resolver (Advanced)**

For complex hybrid DNS scenarios, deploy Azure DNS Private Resolver to forward queries between Azure and on-premises.

## Step 6: Test Connectivity from Session Host

RDP to a session host and verify connectivity to on-premises resources.

**Test 1: Ping On-Prem Server**
```powershell
Test-Connection -ComputerName 172.20.20.10 -Count 4
```
Expected: Replies from on-prem server

**Test 2: Resolve On-Prem DNS Name**
```powershell
Resolve-DnsName fileserver.corp.local
```
Expected: Returns on-prem server IP

**Test 3: Access On-Prem File Share**
```powershell
Get-ChildItem "\\fileserver.corp.local\shared"
```
Expected: Lists files on on-prem server

## Step 7: Update NSG Rules

If connectivity fails, verify NSG rules allow traffic to on-premises networks.

**Add Outbound Rule to nsg-snet-sessionhosts:**
| Setting | Value |
|---------|-------|
| **Priority** | 200 |
| **Name** | AllowOnPremOutbound |
| **Source** | 10.0.1.0/24 (session hosts) |
| **Destination** | 172.20.20.0/24 (on-prem) |
| **Port** | * (or specific ports like 445, 389, 3389) |
| **Action** | Allow |

## ExpressRoute Alternative

For production environments with high bandwidth requirements or latency-sensitive applications, consider ExpressRoute instead of VPN.

**ExpressRoute vs VPN:**
| Aspect | VPN Gateway | ExpressRoute |
|--------|-------------|--------------|
| Bandwidth | Up to 10 Gbps (VpnGw5) | Up to 100 Gbps |
| Latency | Variable (internet) | Predictable (private) |
| Cost | $150-$1,500/month | $300-$10,000+/month |
| Setup Time | Minutes | Weeks (provider circuit) |
| Use Case | Dev/test, small AVD | Production, large AVD |

**ExpressRoute Setup:** Contact Azure partner (AT&T, Equinix, Megaport) to provision circuit, then connect to vnet-avd using ExpressRoute Gateway.

## Validation

- [ ] VPN connection shows "Connected" status in Azure Portal
- [ ] Session hosts can ping on-premises server IP
- [ ] Session hosts can resolve on-premises DNS names
- [ ] Users can access on-premises file shares from AVD desktop
- [ ] Traffic flows over VPN (check on-prem firewall logs)

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Connection stuck in "Connecting" | On-prem device not configured or firewall blocking | Verify on-prem VPN config, check firewall allows UDP 500, 4500 |
| Ping fails to on-prem | Routing issue or NSG blocking | Check effective routes on session host NIC, add NSG rule for on-prem CIDR |
| DNS resolution fails | VNet using Azure DNS only | Configure custom DNS servers on VNet (include on-prem DNS) |
| Slow file access over VPN | Bandwidth insufficient | Upgrade VPN Gateway SKU or consider ExpressRoute |
| Connection drops intermittently | IPsec SA lifetime mismatch | Align IKE/IPsec timers on both Azure and on-prem devices |

## Reference

- **Concept:** [[vpn-gateway]] - Detailed gateway configuration
- **Concept:** [[Expressroute]] - Private connectivity option
- **Concept:** [[private-dns-zones]] - Hybrid DNS resolution

## Next Step

→ Return to main deployment guide or proceed to [Variant: Intune Integration](variant-intune-integration) if managing devices with Intune.