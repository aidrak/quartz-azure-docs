---
title: VPN Gateway
description: 
published: true
date: 2025-12-14T04:53:29.658Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:02.715Z
---

# VPN Gateway

VPN Gateway enables secure connectivity between Azure VNets and on-premises networks using Site-to-Site (S2S) VPN or Point-to-Site (P2S) VPN. Essential for hybrid AVD deployments requiring access to on-premises Active Directory, file servers, or admin access.

## What is VPN Gateway

Azure VPN Gateway is a specific type of virtual network gateway that sends encrypted traffic between Azure and on-premises locations over the public internet. Key features:

- **Site-to-Site (S2S):** Connects entire on-prem network to Azure VNet (for offices, datacenters)
- **Point-to-Site (P2S):** Connects individual clients to Azure VNet (for remote workers)
- **VNet-to-VNet:** Connects multiple Azure VNets (alternative to VNet peering)
- **ExpressRoute Coexistence:** Can run VPN Gateway alongside ExpressRoute for redundancy

**Key Components:**
- **VPN Gateway:** Virtual appliance deployed in GatewaySubnet
- **Local Network Gateway:** Represents on-prem network (public IP, address spaces)
- **Connection:** Links VPN Gateway to Local Network Gateway
- **Shared Key:** Pre-shared key (PSK) for IPsec tunnel authentication

**Encryption:**
- IPsec/IKE protocol (industry standard)
- AES-256 encryption (default)
- SHA-256 for integrity

## When to Use

**Use Site-to-Site VPN When:**
- AVD session hosts need on-prem AD DS domain join
- FSLogix profiles stored on on-prem file servers
- Applications require connectivity to on-prem SQL, web services
- Admins manage AVD from on-premises network
- Small to medium bandwidth requirements (<10 Gbps)

**Use Point-to-Site VPN When:**
- Remote admins need secure access to Azure management VMs
- Individual users need temporary access to Azure resources
- Testing connectivity before deploying S2S VPN

**Use ExpressRoute Instead When:**
- High bandwidth requirements (>10 Gbps)
- Low latency critical (<10ms)
- Compliance requires private connectivity (no internet traversal)
- Predictable network performance needed

## Real-World Example: vpngw-avd

Our production environment uses S2S VPN to connect Azure AVD to home office:

**Configuration:**
- **VPN Gateway Name:** vpngw-avd
- **SKU:** VpnGw1 (650 Mbps, $0.04/hour = ~$29/month)
- **Gateway Type:** VPN
- **VPN Type:** Route-based (supports IKEv2, required for P2S)
- **Generation:** Generation1
- **Active-Active:** Disabled (single gateway instance)
- **Subnet:** GatewaySubnet (10.0.0.0/27)
- **Public IP:** pip-vpngw-avd (static)

**Local Network Gateway:**
- **Name:** lgw-home
- **IP Address:** 98.203.45.67 (home router public IP)
- **Address Spaces:** 172.20.20.0/24 (home network range)

**Connection:**
- **Name:** conn-avd-to-home
- **Connection Type:** Site-to-Site (IPsec)
- **Shared Key:** (32-character random string)
- **IKE Protocol:** IKEv2
- **Encryption:** AES256

**Routing:**
- Session hosts (10.0.1.0/24) can reach home network (172.20.20.0/24)
- Home devices can reach Azure management subnet (10.0.4.0/27)
- BGP not enabled (static routes)

## How to Configure

### Azure Portal - Site-to-Site VPN

**Step 1: Create GatewaySubnet (if not exists)**

1. Navigate to vnet-avd → Subnets → Add Subnet
2. **Name:** GatewaySubnet (EXACT name required)
3. **Address Range:** 10.0.0.0/27 (minimum /29, recommend /27)
4. **Save**

**Step 2: Create VPN Gateway**

1. **Azure Portal → Virtual Network Gateways → Create**
2. **Basics:**
   - Resource Group: RG-Azure-VDI-01
   - Name: vpngw-avd
   - Region: East US (must match VNet)
   - Gateway Type: VPN
   - VPN Type: Route-based
   - SKU: VpnGw1 (or higher for more throughput)
   - Generation: Generation1
   - Virtual Network: vnet-avd (GatewaySubnet auto-selected)

3. **Public IP Address:**
   - Create new: pip-vpngw-avd
   - SKU: Standard
   - Assignment: Static

4. **Review + Create** (takes 30-45 minutes to deploy)

**Step 3: Create Local Network Gateway**

1. **Azure Portal → Local Network Gateways → Create**
2. **Basics:**
   - Resource Group: RG-Azure-VDI-01
   - Name: lgw-home
   - Region: East US
   - Endpoint: IP address
   - IP Address: 98.203.45.67 (your on-prem public IP)
   - Address Spaces: 172.20.20.0/24 (on-prem network)

3. **Review + Create**

**Step 4: Create Connection**

1. Navigate to vpngw-avd → Connections → Add
2. **Basics:**
   - Name: conn-avd-to-home
   - Connection Type: Site-to-Site (IPsec)
   - Virtual Network Gateway: vpngw-avd (auto-selected)
   - Local Network Gateway: lgw-home

3. **Settings:**
   - Shared Key (PSK): Generate 32-character random string (save securely!)
   - IKE Protocol: IKEv2
   - Enable BGP: Unchecked (unless using dynamic routing)

4. **Create**

**Step 5: Configure On-Prem VPN Device**

Configure your on-prem router/firewall with:
- **Remote Gateway IP:** pip-vpngw-avd public IP (from Azure Portal)
- **Local Gateway IP:** 98.203.45.67 (your public IP)
- **Remote Networks:** 10.0.0.0/16 (Azure VNet)
- **Local Networks:** 172.20.20.0/24 (on-prem)
- **Shared Key:** (same PSK from Step 4)
- **IKE Version:** IKEv2
- **Encryption:** AES-256
- **Hash:** SHA-256

### Azure CLI

```bash
# Create Public IP for VPN Gateway
az network public-ip create \
  --resource-group RG-Azure-VDI-01 \
  --name pip-vpngw-avd \
  --allocation-method Static \
  --sku Standard

# Create VPN Gateway (takes 30-45 minutes)
az network vnet-gateway create \
  --resource-group RG-Azure-VDI-01 \
  --name vpngw-avd \
  --vnet vnet-avd \
  --public-ip-addresses pip-vpngw-avd \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1 \
  --no-wait

# Create Local Network Gateway
az network local-gateway create \
  --resource-group RG-Azure-VDI-01 \
  --name lgw-home \
  --gateway-ip-address 98.203.45.67 \
  --local-address-prefixes 172.20.20.0/24

# Create Connection (replace SHARED_KEY with your generated key)
az network vpn-connection create \
  --resource-group RG-Azure-VDI-01 \
  --name conn-avd-to-home \
  --vnet-gateway1 vpngw-avd \
  --local-gateway2 lgw-home \
  --location eastus \
  --shared-key "YOUR-32-CHARACTER-SHARED-KEY"
```

## VPN Gateway SKUs

| SKU | Throughput | S2S Tunnels | P2S Connections | Cost (approx) |
|-----|------------|-------------|-----------------|---------------|
| **Basic** | 100 Mbps | 10 | 128 | ~$0.015/hour (~$11/month) |
| **VpnGw1** | 650 Mbps | 30 | 250 | ~$0.04/hour (~$29/month) |
| **VpnGw2** | 1 Gbps | 30 | 500 | ~$0.19/hour (~$139/month) |
| **VpnGw3** | 1.25 Gbps | 30 | 1000 | ~$0.39/hour (~$285/month) |
| **VpnGw1AZ** | 650 Mbps | 30 | 250 | ~$0.06/hour (~$44/month) |

**Recommendations:**
- **Small AVD (1-50 users):** VpnGw1 (650 Mbps sufficient)
- **Medium AVD (50-200 users):** VpnGw2 (1 Gbps)
- **Large AVD (200+ users):** Consider ExpressRoute instead
- **High Availability:** Use AZ SKUs (zone-redundant) in production

> **Note:** Basic SKU does not support IKEv2 or P2S VPN. Use VpnGw1 minimum for modern deployments.

## Best Practices

**Design:**
- **Use Route-Based VPN:** Policy-based VPN has limitations (single tunnel, no IKEv2)
- **Active-Active for HA:** Deploy two gateway instances for 99.99% SLA (requires VpnGw1 or higher)
- **GatewaySubnet Sizing:** Use /27 or /26 (allows for future active-active or ExpressRoute coexistence)
- **Static Public IP:** Use Standard SKU public IP (required for zone-redundant gateways)

**Security:**
- **Strong Shared Keys:** Use 32+ character random strings (not passwords)
- **IKEv2 Protocol:** More secure than IKEv1, supports mobility
- **Disable Unused Tunnels:** Remove connections to decommissioned sites
- **Monitor Connection Health:** Set up alerts for VPN disconnects

**Performance:**
- **Right-Size SKU:** VpnGw1 sufficient for <500 Mbps, upgrade if consistently saturated
- **BGP for Multi-Site:** Use BGP if connecting 3+ sites (automatic failover)
- **Connection Mode:** Use "Default" (Azure route-based), not "InitiatorOnly"
- **Avoid Forced Tunneling:** Unless required by compliance (adds latency)

**Cost Optimization:**
- **Delete Unused Gateways:** VPN Gateway charges per hour even if no traffic
- **Combine Tunnels:** Use single VPN Gateway for multiple sites (up to 30 tunnels)
- **Data Transfer:** Ingress free, egress charged (~$0.05/GB)

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **"Connection status: Not connected"** | Shared key mismatch or on-prem config error | Verify shared key matches on both sides (case-sensitive). Check on-prem VPN device logs. Ensure public IPs correct. |
| **"Tunnel connects but no traffic"** | Routing issue or firewall blocking | Verify on-prem routes include Azure VNet (10.0.0.0/16). Check NSGs allow traffic from on-prem (172.20.20.0/24). Test with `tracert` from both sides. |
| **"Intermittent disconnects"** | ISP changing public IP or NAT-T issues | Use dynamic DNS if public IP changes. Enable NAT Traversal (NAT-T) on on-prem device. Check for UDP 500/4500 blocks. |
| **"Cannot create VPN Gateway"** | GatewaySubnet missing or wrong name | Create subnet named exactly "GatewaySubnet" (case-sensitive). Minimum /29, recommend /27. |
| **"Slow performance"** | VPN Gateway SKU too small or encryption overhead | Upgrade to higher SKU (VpnGw2/VpnGw3). Test bandwidth without encryption (local test). Check for packet loss with `ping -n 100`. |

## Point-to-Site (P2S) VPN

For remote admin access to Azure management VMs:

**Configuration Steps:**

1. **Enable P2S on VPN Gateway:**
   - Navigate to vpngw-avd → Point-to-site configuration
   - Address Pool: 192.168.100.0/24 (non-overlapping with VNet/on-prem)
   - Tunnel Type: IKEv2 and OpenVPN (SSL)
   - Authentication Type: Azure certificate or Azure AD

2. **Generate Certificates (if using certificate auth):**
   ```powershell
   # Generate root certificate
   $cert = New-SelfSignedCertificate -Type Custom -KeySpec Signature `
     -Subject "CN=P2S-Root-Cert" -KeyExportPolicy Exportable `
     -HashAlgorithm sha256 -KeyLength 2048 `
     -CertStoreLocation "Cert:\CurrentUser\My" `
     -KeyUsageProperty Sign -KeyUsage CertSign

   # Generate client certificate
   New-SelfSignedCertificate -Type Custom -KeySpec Signature `
     -Subject "CN=P2S-Client-Cert" -KeyExportPolicy Exportable `
     -HashAlgorithm sha256 -KeyLength 2048 `
     -CertStoreLocation "Cert:\CurrentUser\My" `
     -Signer $cert -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2")
   ```

3. **Upload Root Certificate to Azure:**
   - Export root cert public key (.cer)
   - Upload to Azure Portal → VPN Gateway → P2S configuration

4. **Download VPN Client:**
   - Click "Download VPN client" in Azure Portal
   - Extract and install on admin workstation
   - Connect using client certificate

## Monitoring and Troubleshooting

**Check Connection Status:**

```bash
# View connection status
az network vpn-connection show \
  --resource-group RG-Azure-VDI-01 \
  --name conn-avd-to-home \
  --query "connectionStatus"

# Expected: "Connected"
```

**VPN Gateway Diagnostics:**

```bash
# View gateway metrics (bandwidth, packet count)
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/RG-Azure-VDI-01/providers/Microsoft.Network/virtualNetworkGateways/vpngw-avd \
  --metric "AverageBandwidth"

# Download VPN diagnostic logs
az network vnet-gateway vpn-client generate \
  --resource-group RG-Azure-VDI-01 \
  --name vpngw-avd
```

**Test Connectivity:**

```powershell
# From Azure session host to on-prem device
Test-NetConnection -ComputerName 172.20.20.10 -Port 445
# Expected: TcpTestSucceeded = True

# From on-prem to Azure management VM
Test-NetConnection -ComputerName 10.0.4.5 -Port 3389
# Expected: TcpTestSucceeded = True
```

## Integration with AVD

**Use Cases:**
- **Hybrid Identity:** Session hosts domain-join to on-prem AD DS via VPN
- **FSLogix Profiles:** Access on-prem file servers for profiles (not recommended, use Azure Files)
- **Line-of-Business Apps:** AVD users access on-prem SQL, ERP systems
- **Admin Access:** IT staff manage Azure VMs from on-prem workstations

**Network Flow Example:**

```
AVD User (Remote Location)
   |
   | HTTPS (443) - AVD Gateway
   |
   v
AVD Session Host (10.0.1.5)
   |
   | VPN Tunnel (IPsec)
   |
   v
VPN Gateway (vpngw-avd)
   |
   | Over Internet (encrypted)
   |
   v
On-Prem VPN Device (98.203.45.67)
   |
   | Local Network
   |
   v
On-Prem File Server (172.20.20.10)
```

## Validation Checklist

- [ ] GatewaySubnet created with /27 or larger CIDR
- [ ] VPN Gateway deployed (status: Succeeded)
- [ ] Public IP assigned to VPN Gateway (static)
- [ ] Local Network Gateway configured with on-prem public IP and address spaces
- [ ] Connection created with shared key matching on-prem config
- [ ] On-prem VPN device configured with Azure gateway IP and shared key
- [ ] Connection status shows "Connected" in Azure Portal
- [ ] Can ping on-prem device from Azure VM (and vice versa)
- [ ] NSGs allow traffic between Azure and on-prem subnets
- [ ] Routes propagated correctly (check effective routes on session host NIC)

## Next Steps

1. **Test Connectivity:** Ping and traceroute between Azure and on-prem
2. **Configure Routing:** Ensure session hosts can reach on-prem resources (see NSG Best Practices)
3. **Domain Join Session Hosts:** Join AVD VMs to on-prem AD DS (see Identity Setup chapter)
4. **Monitor Performance:** Set up alerts for VPN gateway bandwidth and connection status