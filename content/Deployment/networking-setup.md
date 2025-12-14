---
title: Networking Setup
description: 
published: true
date: 2025-12-14T04:53:22.407Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:31.512Z
---

# Networking Setup

Create the virtual network infrastructure for AVD. This includes the VNet, subnets, and NSGs.

## Create Virtual Network

**Portal:** Azure Portal → Virtual Networks → Create

1. **Resource group:** RG-Azure-VDI-01
2. **Name:** `vnet-avd`
3. **Region:** Central US
4. **IPv4 address space:** `10.0.0.0/16`

## Create Subnets

Create purpose-specific subnets:

| Subnet Name | Address Range | Purpose | Size |
|-------------|---------------|---------|------|
| `GatewaySubnet` | 10.0.0.0/27 | VPN Gateway (if needed) | 32 IPs |
| `snet-sessionhosts` | 10.0.1.0/24 | AVD session hosts | 256 IPs |
| `snet-fileserver` | 10.0.2.0/24 | File servers (if needed) | 256 IPs |
| `snet-privateendpoints` | 10.0.3.0/24 | Private endpoints | 256 IPs |
| `snet-management` | 10.0.4.0/27 | Bastion, jump boxes | 32 IPs |

**Portal:** Virtual Network → Subnets → + Subnet

> **Note:** Azure reserves 5 IPs per subnet. A /24 gives 251 usable IPs.

## Create Network Security Groups

Create an NSG for each subnet:

**Portal:** Azure Portal → Network Security Groups → Create

### NSG: nsg-snet-sessionhosts

| Priority | Name | Direction | Action | Source | Destination | Port |
|----------|------|-----------|--------|--------|-------------|------|
| 100 | AllowRDP-Internal | Inbound | Allow | 10.0.4.0/27 | Any | 3389 |
| 200 | AllowHTTPS-Out | Outbound | Allow | Any | Internet | 443 |
| 210 | AllowKMS | Outbound | Allow | Any | 23.102.135.246 | 1688 |
| 220 | AllowDNS | Outbound | Allow | Any | Any | 53 |
| 4096 | DenyAllInbound | Inbound | Deny | Any | Any | * |

> **Note:** AVD session hosts need outbound HTTPS to Azure services. See [[nsg-best-practices]] for complete rule set.

### Associate NSGs to Subnets

**Portal:** NSG → Subnets → Associate → Select subnet

| NSG | Subnet |
|-----|--------|
| nsg-snet-sessionhosts | snet-sessionhosts |
| nsg-snet-privateendpoints | snet-privateendpoints |
| nsg-snet-management | snet-management |
| nsg-snet-fileserver | snet-fileserver |

## DNS Configuration

For cloud-only deployment, use Azure-provided DNS:

**Portal:** Virtual Network → DNS servers → Default (Azure-provided)

> **Decision Point:**
> - **Cloud-only:** Continue to Step 4
> - **Hybrid (custom DNS):** Configure DNS servers pointing to on-prem DC or Azure DNS Private Resolver

## Enable NSG Flow Logs (Recommended)

For troubleshooting network issues:

**Portal:** NSG → Monitoring → NSG flow logs → Create

- Storage account: `stavdflowlogs1523`
- Retention: 30 days

## Verification

- [ ] VNet created with correct address space
- [ ] All subnets created
- [ ] NSGs created and associated
- [ ] DNS configured
- [ ] Flow logs enabled

---

**Next:** [[identity-configuration|Step 4: Identity Configuration]]