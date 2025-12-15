---
title: Step 04 - Networking Setup
description: Create VNet, subnets, NSGs, and configure network security for AVD deployment
published: true
date: 2025-12-14T00:00:00.000Z
tags: [Quick-Deploy, networking, security]
editor: markdown
dateCreated: 2025-12-14T00:00:00.000Z
---

# Step 04: Networking Setup

Create Virtual Network, subnets, and Network Security Groups. This step establishes network foundation for session hosts, storage, and management access.

## Prerequisites

- [ ] Resource Group created: `rg-avd-prod-01` (from Step 01)
- [ ] Naming conventions documented (from [[00-naming-conventions]])
- [ ] IP planning completed: 10.0.0.0/16 (no conflicts with on-premises or other VNets)
- [ ] Hybrid or cloud-only decision made

## Network Components

| Component | Name | CIDR |
|-----------|------|------|
| Virtual Network | `vnet-avd-prod-01` | 10.0.0.0/16 |
| Session Hosts Subnet | `snet-avd-prod-sessionhosts` | 10.0.1.0/24 |
| Private Endpoints Subnet | `snet-avd-prod-privateendpoints` | 10.0.2.0/24 |
| Gateway Subnet (optional) | `GatewaySubnet` | 10.0.3.0/27 |
| Session Hosts NSG | `nsg-avd-prod-sessionhosts` | Outbound: AVD, Azure, Internet, VNet |
| Private Endpoints NSG | `nsg-avd-prod-privateendpoints` | Inbound: SMB (445), HTTPS (443) from session hosts

---

## Create Virtual Network

**Portal:** Azure Portal → Virtual networks → + Create

| Setting | Value |
|---------|-------|
| Subscription | Your subscription |
| Resource Group | `rg-avd-prod-01` |
| VNet Name | `vnet-avd-prod-01` |
| Region | East US |
| IPv4 Address Space | `10.0.0.0/16` |

### Add Subnets

1. **Session Hosts Subnet**
   - Name: `snet-avd-prod-sessionhosts`
   - Address range: `10.0.1.0/24`
   - NSG: None (attach later)

2. **Private Endpoints Subnet**
   - Name: `snet-avd-prod-privateendpoints`
   - Address range: `10.0.2.0/24`
   - Private endpoint network policy: **Disabled**
   - NSG: None (attach later)

3. **Gateway Subnet (Optional - Hybrid Only)**
   - Name: `GatewaySubnet` (exact name required)
   - Address range: `10.0.3.0/27`
   - NSG: None

### Security & Tags

- Azure Bastion: Disabled
- Azure Firewall: Disabled
- DDoS Protection: Disabled
- Tags: Environment=Production, Workload=AVD, Department=IT

Click **Review + create** → **Create** (~2 minutes)

---

## Create Network Security Groups

**Portal:** Azure Portal → Network security groups → + Create

### NSG 1: Session Hosts

| Setting | Value |
|---------|-------|
| Resource Group | `rg-avd-prod-01` |
| Name | `nsg-avd-prod-sessionhosts` |
| Region | East US |

Click **Create**

### NSG 2: Private Endpoints

| Setting | Value |
|---------|-------|
| Resource Group | `rg-avd-prod-01` |
| Name | `nsg-avd-prod-privateendpoints` |
| Region | East US |

Click **Create**

---

## Configure NSG Rules - Session Hosts

**Portal:** Azure Portal → Network security groups → `nsg-avd-prod-sessionhosts`

### Outbound Rules

Navigate to **Outbound security rules** → **+ Add**

| Priority | Name | Source | Port | Destination | Action |
|----------|------|--------|------|-------------|--------|
| 100 | AllowAzureVirtualDesktopOutbound | Any | * | Service Tag: WindowsVirtualDesktop (443) | Allow |
| 110 | AllowAzureCloudOutbound | Any | * | Service Tag: AzureCloud (443) | Allow |
| 120 | AllowInternetOutbound | Any | * | Service Tag: Internet (80, 443) | Allow |
| 130 | AllowVnetOutbound | VirtualNetwork | * | VirtualNetwork (*) | Allow |

See [[../Networking/nsg-best-practices|NSG Best Practices]] for detailed rule design rationale.

### Inbound Rules

Navigate to **Inbound security rules** → **+ Add**

| Priority | Name | Source | Port | Action | Notes |
|----------|------|--------|------|--------|-------|
| 100 | AllowVnetInbound | VirtualNetwork | * | Allow | Health probes |
| 110 | AllowAzureLoadBalancerInbound | Service Tag: AzureLoadBalancer | * | Allow | LB health checks |
| 200 | AllowAdminRDPInbound (optional) | Your IP/32 | 3389 | Allow | Troubleshooting only - remove after use |

---

## Configure NSG Rules - Private Endpoints

**Portal:** Azure Portal → Network security groups → `nsg-avd-prod-privateendpoints`

### Inbound Rules

Navigate to **Inbound security rules** → **+ Add**

| Priority | Name | Source | Port | Action |
|----------|------|--------|------|--------|
| 100 | AllowSessionHostsSMBInbound | 10.0.1.0/24 | 445 | Allow |
| 110 | AllowSessionHostsHTTPSInbound | 10.0.1.0/24 | 443 | Allow |

### Outbound Rules

Navigate to **Outbound security rules** → **+ Add**

| Priority | Name | Destination | Port | Action |
|----------|------|-------------|------|--------|
| 100 | AllowStorageOutbound | Service Tag: Storage | 443 | Allow |

---

## Associate NSGs with Subnets

**Portal:** Azure Portal → Virtual networks → `vnet-avd-prod-01` → Subnets

| Subnet | NSG |
|--------|-----|
| `snet-avd-prod-sessionhosts` | `nsg-avd-prod-sessionhosts` |
| `snet-avd-prod-privateendpoints` | `nsg-avd-prod-privateendpoints` |

Click **Save** for each. (~5 minutes propagation time)

---

## VPN Gateway Setup (Optional - Hybrid Only)

> **Decision Point:** Cloud-only? Skip. Hybrid (on-prem AD)? Complete below.

For detailed VPN Gateway setup, see: [[../Networking/vpn-gateway|VPN Gateway Reference]]

**Quick Configuration:**

| Component | Name | Details |
|-----------|------|---------|
| VPN Gateway | `vpngw-avd-prod` | VPN type: Route-based, SKU: VpnGw1 |
| Local Network Gateway | `lgw-onprem` | Your on-prem VPN public IP + CIDR |
| VPN Connection | `conn-avd-to-onprem` | Site-to-Site IPsec, AES-256 encryption |

After deployment, verify: Portal → Virtual network gateways → `vpngw-avd-prod` → Connections → Status should show **Connected**

---

## Verification

**Portal:** Virtual networks → `vnet-avd-prod-01` → Subnets

- [ ] VNet created: 10.0.0.0/16
- [ ] Session hosts subnet: 10.0.1.0/24 → NSG: `nsg-avd-prod-sessionhosts`
- [ ] Private endpoints subnet: 10.0.2.0/24 → NSG: `nsg-avd-prod-privateendpoints`
- [ ] Gateway subnet (if hybrid): 10.0.3.0/27
- [ ] NSG rules deployed (check effective rules for conflicts)

**Test Session Host Connectivity:**
```powershell
Test-NetConnection -ComputerName rdbroker.wvd.microsoft.com -Port 443
# Expected: TcpTestSucceeded = True
```

---

## Troubleshooting

### Session hosts can't reach AVD control plane

**Symptom:** AVD agent shows "Not registered"

**Fix:**
1. Verify NSG outbound rule: Priority 100, Destination `WindowsVirtualDesktop`, Port 443
2. Check NSG effective rules for conflicts: Portal → NSG → Effective security rules
3. Test connectivity:
   ```powershell
   Test-NetConnection -ComputerName rdbroker.wvd.microsoft.com -Port 443
   ```

### Session hosts can't access Azure Files private endpoint

**Symptom:** FSLogix profile access fails

**Fix:**
1. Verify private endpoint subnet has network policy disabled: VNet → Subnets → `snet-avd-prod-privateendpoints` → Private endpoint network policy = **Disabled**
2. Verify NSG inbound rules allow SMB (445) from 10.0.1.0/24
3. Check session host can resolve storage account DNS: `nslookup storageaccount.blob.core.windows.net`

### VPN Gateway stuck in "Updating"

**Fix:**
1. Wait up to 90 minutes (slow deployment)
2. Verify `GatewaySubnet` exists and named exactly "GatewaySubnet"
3. If still stuck: Delete and recreate gateway

---

## Next Steps

**Next:** [[05-storage-setup|Step 05: Storage Setup (Azure Files & FSLogix)]]

## Related References

- [[../Networking/vnet-design|VNet Design]] - Address planning, subnet sizing
- [[../Networking/nsg-best-practices|NSG Best Practices]] - Rule design, service tags
- [[../Networking/vpn-gateway|VPN Gateway]] - Site-to-Site VPN, SKU selection
- [[../Networking/private-endpoints|Private Endpoints]] - Azure PaaS connectivity
