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

Create the network foundation for Azure Virtual Desktop: Virtual Network (VNet), subnets, Network Security Groups (NSGs), and security rules. This step establishes secure network segmentation and connectivity for session hosts, storage, and management access.

## Example Scenario

Following the 200-user deployment from previous steps, we'll create:

| Component | Name | CIDR / Details |
|-----------|------|----------------|
| **Resource Group** | `rg-avd-prod-eastus-01` | Created in Step 01 |
| **Virtual Network** | `vnet-avd-prod-eastus-01` | 10.0.0.0/16 (65,536 IPs) |
| **Session Hosts Subnet** | `snet-avd-prod-sessionhosts` | 10.0.1.0/24 (251 usable IPs) |
| **Private Endpoints Subnet** | `snet-avd-prod-privateendpoints` | 10.0.2.0/24 (251 usable IPs) |
| **Gateway Subnet (optional)** | `GatewaySubnet` | 10.0.3.0/27 (27 usable IPs) |
| **Session Hosts NSG** | `nsg-avd-prod-sessionhosts` | Controls session host traffic |
| **Private Endpoints NSG** | `nsg-avd-prod-privateendpoints` | Controls storage traffic |

## Prerequisites

- [ ] Resource Group created: `rg-avd-prod-eastus-01` (from Step 01)
- [ ] Naming conventions documented (from [[00-naming-conventions]])
- [ ] IP address planning confirmed (no conflicts with on-premises networks)
- [ ] Decision made on hybrid connectivity (VPN Gateway needed or cloud-only)

**Duration:** ~45 minutes (VNet and NSGs deploy quickly; VPN Gateway adds 30-45 minutes if needed)

---

## Part 1: IP Address Planning

Before creating resources, confirm your IP address allocation strategy.

### Address Space Allocation

**Production VNet: 10.0.0.0/16 (65,536 total IPs)**

| Subnet | CIDR | Usable IPs | Purpose | Expected Usage |
|--------|------|------------|---------|----------------|
| `snet-avd-prod-sessionhosts` | 10.0.1.0/24 | 251 | Pooled + Personal session hosts | 10 pooled + 50 personal = 60 VMs |
| `snet-avd-prod-privateendpoints` | 10.0.2.0/24 | 251 | Private endpoints (storage, Key Vault) | 3-5 endpoints |
| `GatewaySubnet` (optional) | 10.0.3.0/27 | 27 | VPN/ExpressRoute Gateway | 2-4 gateway instances |
| Reserved for future growth | 10.0.4.0/22 | ~1,000 | Additional subnets (monitoring, management) | As needed |

> **Note:** Azure reserves 5 IPs in each subnet (.0, .1, .2, .3, .255), reducing usable addresses. Example: /24 subnet has 256 IPs - 5 reserved = 251 usable.

### IP Conflict Check

Verify 10.0.0.0/16 does NOT conflict with:
- On-premises networks (e.g., if home/office uses 10.x.x.x, choose different range like 172.16.0.0/16)
- Other Azure VNets in your subscription
- VPN client address pools

**If conflicts exist:** Adjust to alternate ranges:
- **Option 1:** 172.16.0.0/16 (private class B)
- **Option 2:** 192.168.0.0/16 (private class C, larger range)

**For this guide, we'll use 10.0.0.0/16 (cloud-only deployment with no conflicts).**

---

## Part 2: Create Virtual Network

**Portal:** Azure Portal → Virtual networks → + Create

### Basics Tab

1. **Subscription:** Select your Azure subscription
2. **Resource group:** `rg-avd-prod-eastus-01`
3. **Virtual network name:** `vnet-avd-prod-eastus-01`
4. **Region:** East US

Click **Next: IP Addresses**

### IP Addresses Tab

1. **IPv4 address space:**
   - Remove default if present (delete existing 10.x.x.x/16)
   - Click **Add IPv4 address space**
   - Enter: `10.0.0.0/16`

2. **Add subnet: snet-avd-prod-sessionhosts**
   - Click **+ Add subnet**
   - **Subnet name:** `snet-avd-prod-sessionhosts`
   - **Subnet address range:** `10.0.1.0/24`
   - **NAT Gateway:** None
   - **Network security group:** None (will attach later)
   - **Route table:** None
   - Click **Add**

3. **Add subnet: snet-avd-prod-privateendpoints**
   - Click **+ Add subnet**
   - **Subnet name:** `snet-avd-prod-privateendpoints`
   - **Subnet address range:** `10.0.2.0/24`
   - **NAT Gateway:** None
   - **Network security group:** None (will attach later)
   - **Route table:** None
   - **Private endpoint network policy:** Disabled (required for private endpoints)
   - Click **Add**

4. **Add GatewaySubnet (Optional - for VPN Gateway):**
   > **Decision Point:** Only create if planning hybrid connectivity (VPN or ExpressRoute).

   - Click **+ Add subnet**
   - **Subnet name:** `GatewaySubnet` (EXACT name required - no variations)
   - **Subnet address range:** `10.0.3.0/27`
   - **NAT Gateway:** None
   - **Network security group:** None (gateways manage their own security)
   - **Route table:** None
   - Click **Add**

Click **Next: Security**

### Security Tab

1. **Azure Bastion:** Disabled (optional; adds ~$140/month)
2. **Azure Firewall:** Disabled (NSGs sufficient for most deployments)
3. **DDoS Protection:** Disabled (Standard tier protection enabled by default)

Click **Next: Tags**

### Tags Tab

Add organizational tags:

| Name | Value |
|------|-------|
| `Environment` | `Production` |
| `Workload` | `AVD` |
| `Department` | `IT` |
| `CostCenter` | `12345` |

Click **Review + create**

### Review and Create

1. Verify settings:
   - VNet address space: 10.0.0.0/16
   - Session hosts subnet: 10.0.1.0/24
   - Private endpoints subnet: 10.0.2.0/24
   - Gateway subnet (if hybrid): 10.0.3.0/27

2. Click **Create**

**Deployment time:** ~2 minutes

---

## Part 3: Create Network Security Groups

NSGs control traffic to/from subnets. We'll create two NSGs:
1. **nsg-avd-prod-sessionhosts** - For session host subnet
2. **nsg-avd-prod-privateendpoints** - For private endpoints subnet

### Create NSG for Session Hosts

**Portal:** Azure Portal → Network security groups → + Create

#### Basics Tab

1. **Subscription:** Select your subscription
2. **Resource group:** `rg-avd-prod-eastus-01`
3. **Name:** `nsg-avd-prod-sessionhosts`
4. **Region:** East US (must match VNet region)

#### Tags Tab

Add tags (same as VNet):
- Environment: Production
- Workload: AVD
- Department: IT

Click **Review + create** → **Create**

**Deployment time:** ~30 seconds

### Create NSG for Private Endpoints

**Portal:** Azure Portal → Network security groups → + Create

#### Basics Tab

1. **Subscription:** Select your subscription
2. **Resource group:** `rg-avd-prod-eastus-01`
3. **Name:** `nsg-avd-prod-privateendpoints`
4. **Region:** East US

#### Tags Tab

Add same tags as above.

Click **Review + create** → **Create**

---

## Part 4: Configure NSG Rules - Session Hosts

Configure outbound and inbound rules for session hosts.

**Portal:** Azure Portal → Network security groups → `nsg-avd-prod-sessionhosts`

### Outbound Security Rules

Session hosts require outbound access to AVD services, Azure Storage, Windows Update, and authentication.

**Navigate to: Outbound security rules → + Add**

#### Rule 1: Allow AVD Control Plane

1. **Source:** Any
2. **Source port ranges:** *
3. **Destination:** Service Tag
4. **Destination service tag:** `WindowsVirtualDesktop`
5. **Service:** Custom
6. **Destination port ranges:** `443`
7. **Protocol:** TCP
8. **Action:** Allow
9. **Priority:** 100
10. **Name:** `AllowAzureVirtualDesktopOutbound`
11. **Description:** `Required for AVD control plane communication`

Click **Add**

#### Rule 2: Allow Azure Cloud Services

1. **Source:** Any
2. **Source port ranges:** *
3. **Destination:** Service Tag
4. **Destination service tag:** `AzureCloud`
5. **Service:** HTTPS
6. **Destination port ranges:** `443`
7. **Protocol:** TCP
8. **Action:** Allow
9. **Priority:** 110
10. **Name:** `AllowAzureCloudOutbound`
11. **Description:** `Required for Azure services (Storage, Monitor, etc.)`

Click **Add**

#### Rule 3: Allow Internet (Windows Update & KMS Activation)

1. **Source:** Any
2. **Source port ranges:** *
3. **Destination:** Service Tag
4. **Destination service tag:** `Internet`
5. **Service:** Custom
6. **Destination port ranges:** `80,443`
7. **Protocol:** TCP
8. **Action:** Allow
9. **Priority:** 120
10. **Name:** `AllowInternetOutbound`
11. **Description:** `Required for Windows Update and KMS activation`

Click **Add**

#### Rule 4: Allow VNet (Internal Communication)

1. **Source:** VirtualNetwork
2. **Source port ranges:** *
3. **Destination:** VirtualNetwork
4. **Destination port ranges:** *
5. **Protocol:** Any
6. **Action:** Allow
7. **Priority:** 130
8. **Name:** `AllowVnetOutbound`
9. **Description:** `Allow communication within VNet (FSLogix, private endpoints)`

Click **Add**

> **Note:** Azure's default deny rule (priority 65001) automatically blocks all other outbound traffic. No explicit deny rule needed.

### Inbound Security Rules

Session hosts use reverse connect (outbound-initiated), so minimal inbound rules required.

**Navigate to: Inbound security rules → + Add**

#### Rule 1: Allow VNet Inbound (Load Balancer Health Probes)

1. **Source:** VirtualNetwork
2. **Source port ranges:** *
3. **Destination:** VirtualNetwork
4. **Destination port ranges:** *
5. **Protocol:** Any
6. **Action:** Allow
7. **Priority:** 100
8. **Name:** `AllowVnetInbound`
9. **Description:** `Allow internal VNet communication and health probes`

Click **Add**

#### Rule 2: Allow Azure Load Balancer

1. **Source:** Service Tag
2. **Source service tag:** `AzureLoadBalancer`
3. **Source port ranges:** *
4. **Destination:** Any
5. **Destination port ranges:** *
6. **Protocol:** Any
7. **Action:** Allow
8. **Priority:** 110
9. **Name:** `AllowAzureLoadBalancerInbound`
10. **Description:** `Required for AVD load balancer health checks`

Click **Add**

#### Optional Rule 3: Allow Management Access (Only if Needed)

> **Warning:** Only add if you need direct RDP access to session hosts for troubleshooting. Not required for normal AVD operations.

1. **Source:** IP Addresses
2. **Source IP addresses/CIDR ranges:** `YOUR-ADMIN-IP/32` (replace with your public IP or VPN range)
3. **Source port ranges:** *
4. **Destination:** VirtualNetwork
5. **Service:** RDP
6. **Destination port ranges:** `3389`
7. **Protocol:** TCP
8. **Action:** Allow
9. **Priority:** 200
10. **Name:** `AllowAdminRDPInbound`
11. **Description:** `Allow admin RDP access from specific IP (temporary)`

Click **Add**

> **Best Practice:** Remove this rule after troubleshooting. Use Azure Bastion for secure admin access instead.

**See:** [[nsg-best-practices|NSG Best Practices]] for detailed rule explanations and security hardening.

---

## Part 5: Configure NSG Rules - Private Endpoints

Configure rules for private endpoint subnet (Azure Files, Key Vault, etc.).

**Portal:** Azure Portal → Network security groups → `nsg-avd-prod-privateendpoints`

### Inbound Security Rules

**Navigate to: Inbound security rules → + Add**

#### Rule 1: Allow SMB from Session Hosts

1. **Source:** IP Addresses
2. **Source IP addresses/CIDR ranges:** `10.0.1.0/24` (session hosts subnet)
3. **Source port ranges:** *
4. **Destination:** VirtualNetwork
5. **Service:** Custom
6. **Destination port ranges:** `445`
7. **Protocol:** TCP
8. **Action:** Allow
9. **Priority:** 100
10. **Name:** `AllowSessionHostsSMBInbound`
11. **Description:** `Allow FSLogix profile access via Azure Files private endpoint`

Click **Add**

#### Rule 2: Allow HTTPS from Session Hosts

1. **Source:** IP Addresses
2. **Source IP addresses/CIDR ranges:** `10.0.1.0/24`
3. **Source port ranges:** *
4. **Destination:** VirtualNetwork
5. **Service:** HTTPS
6. **Destination port ranges:** `443`
7. **Protocol:** TCP
8. **Action:** Allow
9. **Priority:** 110
10. **Name:** `AllowSessionHostsHTTPSInbound`
11. **Description:** `Allow HTTPS to Key Vault and other private endpoints`

Click **Add**

### Outbound Security Rules

**Navigate to: Outbound security rules → + Add**

#### Rule 1: Allow Azure Storage

1. **Source:** VirtualNetwork
2. **Source port ranges:** *
3. **Destination:** Service Tag
4. **Destination service tag:** `Storage`
5. **Service:** HTTPS
6. **Destination port ranges:** `443`
7. **Protocol:** TCP
8. **Action:** Allow
9. **Priority:** 100
10. **Name:** `AllowStorageOutbound`
11. **Description:** `Allow private endpoint backend communication with Azure Storage`

Click **Add**

---

## Part 6: Associate NSGs with Subnets

Attach NSGs to their respective subnets to enforce security rules.

### Associate Session Hosts NSG

**Portal:** Azure Portal → Virtual networks → `vnet-avd-prod-eastus-01` → Subnets

1. Click on subnet: `snet-avd-prod-sessionhosts`
2. **Network security group:** Select `nsg-avd-prod-sessionhosts`
3. Click **Save**

**Propagation time:** ~5 minutes for rules to take effect

### Associate Private Endpoints NSG

**Portal:** Azure Portal → Virtual networks → `vnet-avd-prod-eastus-01` → Subnets

1. Click on subnet: `snet-avd-prod-privateendpoints`
2. **Network security group:** Select `nsg-avd-prod-privateendpoints`
3. Click **Save**

---

## Part 7: Enable NSG Flow Logs (Optional - Recommended)

NSG Flow Logs capture all traffic allowed/denied by NSG rules. Essential for troubleshooting and security auditing.

> **Cost:** ~$0.50/GB ingested + storage account costs (~$5-20/month for typical AVD deployment)

### Prerequisites

1. **Create storage account for logs:**
   - **Portal:** Storage accounts → + Create
   - **Name:** `stavdlogsprodeus01` (must be globally unique)
   - **Resource group:** `rg-avd-prod-eastus-01`
   - **Performance:** Standard
   - **Replication:** LRS (Locally-redundant storage)
   - Create account

2. **Ensure Network Watcher enabled:**
   - **Portal:** Network Watcher → Overview
   - Verify Network Watcher enabled in East US region
   - If not, click **Enable Network Watcher**

### Enable Flow Logs for Session Hosts NSG

**Portal:** Network Watcher → NSG flow logs → + Create

1. **Basics:**
   - **Subscription:** Select subscription
   - **Network security group:** `nsg-avd-prod-sessionhosts`
   - **Flow Logs Name:** `flowlog-nsg-sessionhosts`
   - **Storage account:** `stavdlogsprodeus01`
   - **Retention (days):** 30

2. **Configuration:**
   - **Flow Logs Version:** Version 2 (includes flow direction)
   - **Traffic Analytics:** Enabled (optional; uses Log Analytics)
   - **Traffic Analytics processing interval:** Every 10 mins
   - **Log Analytics Workspace:** Select existing or create new

3. Click **Review + create** → **Create**

Repeat for `nsg-avd-prod-privateendpoints` if needed.

**See:** [[nsg-best-practices#nsg-flow-logs|NSG Flow Logs Reference]] for analysis and troubleshooting.

---

## Part 8: VPN Gateway Setup (Optional - Hybrid Only)

> **Decision Point:**
> - **Cloud-only deployment (Entra ID Join):** Skip this section. Proceed to verification.
> - **Hybrid deployment (Hybrid Entra Join, on-prem AD):** Complete VPN Gateway setup below.

### When to Deploy VPN Gateway

Deploy VPN Gateway if:
- Session hosts need to domain-join on-premises Active Directory
- FSLogix profiles stored on on-prem file servers (not recommended; use Azure Files)
- AVD users need access to on-prem applications (SQL Server, ERP, etc.)
- IT admins manage AVD from on-premises network

**Alternative:** Use Azure ExpressRoute for higher bandwidth (>1 Gbps) and lower latency.

### VPN Gateway Configuration

For detailed VPN Gateway setup, see:

**Reference:** [[../Networking/vpn-gateway|VPN Gateway Deep Dive]]

**Quick Summary:**

1. **Create VPN Gateway:**
   - Name: `vpngw-avd-prod`
   - Gateway type: VPN
   - VPN type: Route-based
   - SKU: VpnGw1 (650 Mbps, ~$29/month)
   - Virtual network: `vnet-avd-prod-eastus-01`
   - Subnet: `GatewaySubnet` (must already exist)
   - Public IP: Create new `pip-vpngw-avd`
   - **Deployment time:** 30-45 minutes

2. **Create Local Network Gateway:**
   - Name: `lgw-onprem`
   - IP address: [Your on-premises VPN public IP]
   - Address space: [Your on-prem network CIDR, e.g., 192.168.1.0/24]

3. **Create VPN Connection:**
   - Name: `conn-avd-to-onprem`
   - Connection type: Site-to-Site (IPsec)
   - Virtual network gateway: `vpngw-avd-prod`
   - Local network gateway: `lgw-onprem`
   - Shared key: [Generate 32-character random string]

4. **Configure On-Premises VPN Device:**
   - Remote gateway IP: [pip-vpngw-avd public IP from Azure]
   - Remote networks: 10.0.0.0/16
   - Local networks: [Your on-prem CIDR]
   - Shared key: [Same as step 3]
   - Encryption: AES-256, IKEv2

5. **Verify Connection:**
   - Portal: Virtual network gateways → vpngw-avd-prod → Connections
   - Status should show: **Connected**

**See:** [[../Networking/vpn-gateway|VPN Gateway Reference]] for complete configuration, troubleshooting, and SKU selection.

---

## Verification Checklist

Confirm all networking components are deployed correctly:

### Virtual Network

**Portal:** Virtual networks → `vnet-avd-prod-eastus-01`

- [ ] VNet created with address space 10.0.0.0/16
- [ ] Subnet `snet-avd-prod-sessionhosts` exists (10.0.1.0/24)
- [ ] Subnet `snet-avd-prod-privateendpoints` exists (10.0.2.0/24)
- [ ] Optional: Subnet `GatewaySubnet` exists (10.0.3.0/27) if hybrid
- [ ] VNet provisioning state: **Succeeded**

**Check subnet details:**
1. Navigate to VNet → Subnets
2. Verify each subnet shows correct CIDR and NSG association

### Network Security Groups

**Portal:** Network security groups

- [ ] `nsg-avd-prod-sessionhosts` created
- [ ] `nsg-avd-prod-privateendpoints` created
- [ ] Session hosts NSG has 4 outbound rules (AVD, AzureCloud, Internet, VNet)
- [ ] Session hosts NSG has 2 inbound rules (VNet, AzureLoadBalancer)
- [ ] Private endpoints NSG has SMB + HTTPS inbound rules
- [ ] NSGs associated with correct subnets

**Test NSG rules:**
1. Navigate to NSG → Effective security rules
2. Verify no conflicts (lower priority = higher precedence)
3. Confirm default Azure rules present (priority 65000+)

### NSG Association

**Portal:** Virtual networks → `vnet-avd-prod-eastus-01` → Subnets

- [ ] `snet-avd-prod-sessionhosts` → NSG: `nsg-avd-prod-sessionhosts`
- [ ] `snet-avd-prod-privateendpoints` → NSG: `nsg-avd-prod-privateendpoints`

### NSG Flow Logs (Optional)

**Portal:** Network Watcher → NSG flow logs

- [ ] Flow log created for `nsg-avd-prod-sessionhosts`
- [ ] Storage account `stavdlogsprodeus01` configured
- [ ] Retention: 30 days
- [ ] Traffic Analytics: Enabled (if desired)

### VPN Gateway (Hybrid Only)

**Portal:** Virtual network gateways → `vpngw-avd-prod`

- [ ] VPN Gateway created (SKU: VpnGw1 or higher)
- [ ] Gateway status: **Succeeded**
- [ ] Public IP assigned
- [ ] Connection to on-prem shows: **Connected**
- [ ] Test connectivity: Ping on-prem device from Azure VM

**Test VPN connectivity:**
```powershell
# From Azure management VM or session host
Test-NetConnection -ComputerName [ON-PREM-IP] -Port 445
# Expected: TcpTestSucceeded = True
```

---

## Network Architecture Diagram

**Final Network Layout:**

```
┌─────────────────────────────────────────────────────────────────┐
│ Azure Virtual Network: vnet-avd-prod-eastus-01 (10.0.0.0/16)  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ ┌─────────────────────────────────────────────────────────┐   │
│ │ snet-avd-prod-sessionhosts (10.0.1.0/24)               │   │
│ │ NSG: nsg-avd-prod-sessionhosts                         │   │
│ │                                                         │   │
│ │ ┌──────────┐ ┌──────────┐ ┌──────────┐               │   │
│ │ │ Pooled   │ │ Personal │ │ Personal │  ... (60 VMs) │   │
│ │ │ VM 001   │ │ VM 011   │ │ VM 012   │               │   │
│ │ └──────────┘ └──────────┘ └──────────┘               │   │
│ └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│ ┌─────────────────────────────────────────────────────────┐   │
│ │ snet-avd-prod-privateendpoints (10.0.2.0/24)           │   │
│ │ NSG: nsg-avd-prod-privateendpoints                     │   │
│ │                                                         │   │
│ │ ┌──────────────┐ ┌──────────────┐                     │   │
│ │ │ pe-storage   │ │ pe-keyvault  │                     │   │
│ │ │ (Azure Files)│ │ (Key Vault)  │                     │   │
│ │ └──────────────┘ └──────────────┘                     │   │
│ └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│ ┌─────────────────────────────────────────────────────────┐   │
│ │ GatewaySubnet (10.0.3.0/27) - OPTIONAL                │   │
│ │                                                         │   │
│ │ ┌──────────────────┐                                   │   │
│ │ │ vpngw-avd-prod   │ ←───── VPN Tunnel ───→ On-Prem  │   │
│ │ │ (VPN Gateway)    │        (IPsec)                   │   │
│ │ └──────────────────┘                                   │   │
│ └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting

### Issue: Subnet creation fails with "Address space conflict"

**Symptom:** Error when adding subnet: "Address range overlaps with existing subnet"

**Cause:** CIDR ranges overlap with existing subnets or are outside VNet address space

**Fix:**
1. Verify VNet address space: 10.0.0.0/16 allows 10.0.0.0 through 10.0.255.255
2. Check existing subnets don't overlap:
   - Session hosts: 10.0.1.0/24 (10.0.1.0 - 10.0.1.255)
   - Private endpoints: 10.0.2.0/24 (10.0.2.0 - 10.0.2.255)
   - Gateway: 10.0.3.0/27 (10.0.3.0 - 10.0.3.31)
3. Use non-overlapping ranges

### Issue: Cannot associate NSG with subnet

**Symptom:** Error: "Subnet already has an NSG associated"

**Cause:** Subnet can only have one NSG

**Fix:**
1. Navigate to VNet → Subnets → [Subnet Name]
2. Remove existing NSG
3. Associate correct NSG

### Issue: Session hosts can't reach AVD control plane

**Symptom:** AVD agent shows "Not registered" or "Unavailable"

**Cause:** NSG blocking outbound to AVD services

**Fix:**
1. Verify outbound rule priority 100: Destination `WindowsVirtualDesktop`, Port 443, Action Allow
2. Check effective routes on session host NIC: Portal → Virtual machines → [VM] → Networking → Network Interface → Effective routes
3. Test connectivity from session host:
   ```powershell
   Test-NetConnection -ComputerName rdbroker.wvd.microsoft.com -Port 443
   # Expected: TcpTestSucceeded = True
   ```

### Issue: VPN Gateway stuck in "Updating" state

**Symptom:** VPN Gateway deployment shows "Updating" for over 60 minutes

**Cause:** Transient deployment issue

**Fix:**
1. Wait up to 90 minutes (gateway deployments can be slow)
2. If still stuck, delete gateway and recreate
3. Verify GatewaySubnet exists and is named exactly "GatewaySubnet"

### Issue: Private endpoint subnet blocking connections

**Symptom:** Session hosts can't connect to Azure Files via private endpoint

**Cause:** Private endpoint network policies enabled on subnet

**Fix:**
1. Navigate to VNet → Subnets → `snet-avd-prod-privateendpoints`
2. **Private endpoint network policies:** Set to **Disabled**
3. Click **Save**
4. Wait 5 minutes for propagation

---

## Cost Estimate

**Monthly networking costs for 200-user deployment:**

| Resource | Specification | Monthly Cost (USD) |
|----------|--------------|-------------------|
| Virtual Network | 10.0.0.0/16, 3 subnets | Free |
| Network Security Groups (2) | Standard rules | Free |
| NSG Flow Logs (optional) | 30-day retention, ~10 GB/month | ~$10 |
| VPN Gateway (optional) | VpnGw1, 650 Mbps | ~$29 |
| VPN Data Transfer (optional) | ~500 GB/month egress | ~$25 |
| **Total (Cloud-Only)** | | **$0-10** |
| **Total (Hybrid with VPN)** | | **$54-64** |

> **Note:** VNet and NSGs have no direct cost. You pay for resources deployed in them (VMs, gateways) and data transfer.

---

## Next Steps

**Networking foundation is complete.** You can now deploy storage, session hosts, and configure FSLogix profiles.

**Next:** [[05-storage-setup|Step 05: Storage Setup (Azure Files & FSLogix)]]

---

## Related Reference Pages

- [[../Networking/vnet-design|VNet Design Deep Dive]] - Address planning, subnet sizing, hub-spoke topologies
- [[../Networking/nsg-best-practices|NSG Best Practices]] - Security rule design, service tags, troubleshooting
- [[../Networking/vpn-gateway|VPN Gateway]] - Site-to-Site VPN, ExpressRoute, SKU selection
- [[../Networking/private-endpoints|Private Endpoints]] - Secure connectivity to Azure PaaS services
- [[../Operations/network-monitoring|Network Monitoring]] - Flow logs analysis, Traffic Analytics, alerts
