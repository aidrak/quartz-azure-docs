---
title: Private DNS Zones
description: 
published: true
date: 2025-12-14T04:53:25.015Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:57.811Z
---

# Private DNS Zones

Private DNS zones enable name resolution for Azure services accessed via private endpoints, ensuring session hosts resolve FQDNs to private IPs instead of public IPs. Critical for FSLogix profile access and other private endpoint scenarios.

## What are Private DNS Zones

A Private DNS zone provides DNS name resolution within your VNets without exposing DNS records to the public internet. For private endpoints:

- **Resolves Azure Service FQDNs:** Converts `stfslogix01.file.core.windows.net` → private IP (10.0.3.4)
- **VNet Linked:** Must link the DNS zone to your VNet for resolution to work
- **Auto-Registration:** Optionally auto-registers VM names (not used for private endpoints)
- **Conditional Forwarding:** On-prem DNS can forward queries to Azure's DNS (168.63.129.16)

**Key Concepts:**
- **DNS Zone Name:** Must match privatelink.{service}.core.windows.net format
- **A Records:** Point service name to private endpoint IP (auto-created by DNS zone groups)
- **VNet Links:** Connect DNS zone to VNets requiring name resolution
- **Priority:** Private DNS zones take precedence over public DNS for linked VNets

**Why Needed:**
- Azure creates CNAME from public FQDN to privatelink FQDN
- Private DNS zone contains A record mapping privatelink FQDN to private IP
- Without private DNS zone, queries resolve to public IP (bypassing private endpoint)

## When to Use

**Always Use When:**
- Deploying private endpoints for any Azure PaaS service
- Session hosts need to access Azure Files, Key Vault, SQL via private IPs
- Security policies prohibit public DNS resolution for sensitive services

**Common Zones for AVD:**
- **privatelink.file.core.windows.net:** Azure Files (FSLogix profiles)
- **privatelink.blob.core.windows.net:** Blob storage (scripts, images)
- **privatelink.vaultcore.azure.net:** Key Vault (certificates, secrets)
- **privatelink.database.windows.net:** Azure SQL (application databases)

**Not Needed:**
- Resources without private endpoints
- Services accessed via public endpoints (AVD control plane, Azure AD)

## Real-World Example: privatelink.file.core.windows.net

Our production environment uses a private DNS zone for Azure Files private endpoints:

**Configuration:**
- **Zone Name:** privatelink.file.core.windows.net
- **Resource Group:** RG-Azure-VDI-01
- **VNet Links:**
  - vnet-avd-link (linked to vnet-avd, auto-registration disabled)
- **A Records (auto-created):**
  - stfslogix01 → 10.0.3.4 (private endpoint pe-fslogix-files)

**DNS Resolution Flow:**

```
1. Session host queries: stfslogix01.file.core.windows.net

2. Azure public DNS returns CNAME:
   stfslogix01.file.core.windows.net → stfslogix01.privatelink.file.core.windows.net

3. Private DNS zone (linked to vnet-avd) resolves:
   stfslogix01.privatelink.file.core.windows.net → 10.0.3.4

4. Session host connects to 10.0.3.4 (private endpoint)
```

**Verification from Session Host:**

```powershell
PS C:\> nslookup stfslogix01.file.core.windows.net
Server:  UnKnown
Address:  168.63.129.16

Non-authoritative answer:
Name:    stfslogix01.privatelink.file.core.windows.net
Address:  10.0.3.4
Aliases:  stfslogix01.file.core.windows.net
```

## How to Configure

### Azure Portal

**Azure Portal → Private DNS Zones → Create**

1. **Basics:**
   - Resource Group: RG-Azure-VDI-01
   - Name: privatelink.file.core.windows.net
   - (Must match exact format for service type)

2. **Review + Create**

3. **Link to VNet:**
   - Navigate to Private DNS Zone → Virtual network links → Add
   - Link Name: vnet-avd-link
   - Virtual Network: vnet-avd
   - Enable Auto Registration: No (not needed for private endpoints)

4. **Create A Record (if not auto-created):**
   - Navigate to Private DNS Zone → Record Sets → Add
   - Name: stfslogix01
   - Type: A
   - TTL: 1 Hour
   - IP Address: 10.0.3.4
   - (Usually auto-created when creating private endpoint with DNS zone group)

### Azure CLI

```bash
# Create private DNS zone
az network private-dns zone create \
  --resource-group RG-Azure-VDI-01 \
  --name privatelink.file.core.windows.net

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group RG-Azure-VDI-01 \
  --zone-name privatelink.file.core.windows.net \
  --name vnet-avd-link \
  --virtual-network vnet-avd \
  --registration-enabled false

# Manually create A record (if needed)
az network private-dns record-set a add-record \
  --resource-group RG-Azure-VDI-01 \
  --zone-name privatelink.file.core.windows.net \
  --record-set-name stfslogix01 \
  --ipv4-address 10.0.3.4

# Verify DNS resolution from session host (requires session host to be running)
# az vm run-command invoke --resource-group RG-Azure-VDI-01 --name vm-avd-01 --command-id RunPowerShellScript --scripts "nslookup stfslogix01.file.core.windows.net"
```

## Best Practices

**Zone Naming:**
- **Use Exact Azure Format:** Must be `privatelink.{service}.core.windows.net` for Azure to create CNAME
- **One Zone per Service Type:** Single zone supports multiple storage accounts (stfslogix01, stfslogix02 both use same zone)
- **Centralized Management:** Create zones in shared infrastructure resource group

**VNet Linking:**
- **Link to All Required VNets:** Hub-spoke topologies need zone linked to all spoke VNets accessing private endpoints
- **Disable Auto-Registration:** Not needed for private endpoints, only for VM name registration
- **Link Once, Use Many:** One VNet link supports all private endpoints in that VNet

**A Record Management:**
- **Prefer DNS Zone Groups:** Let Azure auto-create A records when creating private endpoints
- **Avoid Manual Records:** Manual records don't update if private endpoint IP changes
- **Monitor Orphaned Records:** Remove A records when private endpoints are deleted

**Integration with On-Premises:**
- **Conditional Forwarding:** Configure on-prem DNS to forward `privatelink.*.windows.net` to 168.63.129.16
- **Azure Private DNS Resolver:** For hybrid scenarios requiring on-prem to Azure DNS resolution
- **Split-Horizon DNS:** Different resolution for VNet clients vs on-prem clients (if needed)

## Common Private DNS Zone Names

| Azure Service | Private DNS Zone Name | Private Endpoint Sub-Resource |
|---------------|----------------------|-------------------------------|
| **Azure Files** | privatelink.file.core.windows.net | file |
| **Blob Storage** | privatelink.blob.core.windows.net | blob |
| **Table Storage** | privatelink.table.core.windows.net | table |
| **Queue Storage** | privatelink.queue.core.windows.net | queue |
| **Azure Key Vault** | privatelink.vaultcore.azure.net | vault |
| **Azure SQL Database** | privatelink.database.windows.net | sqlServer |
| **Azure Cosmos DB (SQL)** | privatelink.documents.azure.com | Sql |
| **Azure Container Registry** | privatelink.azurecr.io | registry |
| **Azure App Service** | privatelink.azurewebsites.net | sites |

> **Note:** Use lowercase for zone names. Azure Portal may auto-capitalize, but CLI requires lowercase.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **"nslookup returns public IP"** | Private DNS zone not linked to VNet or A record missing | Verify VNet link: `az network private-dns link vnet list --zone-name privatelink.file.core.windows.net --resource-group RG-Azure-VDI-01`. Check A records: `az network private-dns record-set a list --zone-name privatelink.file.core.windows.net --resource-group RG-Azure-VDI-01`. |
| **"Cannot resolve privatelink FQDN"** | DNS zone name incorrect or VNet not linked | Zone name must match exact format (e.g., `privatelink.file.core.windows.net`). Ensure VNet link exists and registration disabled. |
| **"Works from Azure VM but not on-prem"** | On-prem DNS not forwarding to Azure DNS | Configure conditional forwarder on on-prem DNS: Forward `privatelink.*.windows.net` to 168.63.129.16. Or deploy Azure Private DNS Resolver. |
| **"A record not created automatically"** | Private endpoint missing DNS zone group configuration | Add DNS zone group to private endpoint: `az network private-endpoint dns-zone-group create`. |
| **"Resolution intermittent"** | DNS cache on client or multiple DNS servers with different configs | Flush DNS cache on session host: `ipconfig /flushdns`. Verify all DNS servers have same forwarding config. |

## Integration with Hybrid DNS

For on-premises clients accessing Azure services via ExpressRoute or VPN:

**Option 1: Conditional Forwarding (Simple)**

Configure on-prem DNS servers to forward privatelink queries to Azure DNS:

```
Forwarder Configuration:
- Zone: privatelink.file.core.windows.net
- Forward to: 168.63.129.16 (Azure's recursive resolver)
```

**Limitations:**
- Requires VPN/ExpressRoute connectivity
- On-prem clients must be able to reach 168.63.129.16
- Each zone requires separate forwarder rule

**Option 2: Azure Private DNS Resolver (Recommended for Enterprise)**

Deploy Azure Private DNS Resolver in hub VNet:

1. **Inbound Endpoint:** On-prem DNS forwards queries to resolver (e.g., 10.0.5.4)
2. **Outbound Endpoint:** Azure VMs use resolver for on-prem domain queries
3. **Forwarding Ruleset:** Define which domains resolve via on-prem vs Azure DNS

**Benefits:**
- Centralized DNS resolution for hybrid environments
- No need to manage forwarder rules on every on-prem DNS server
- Supports custom DNS forwarding rules

**Cost:** ~$0.06/hour per endpoint (~$43/month) + query costs

## DNS Resolution Testing

**Test from Azure Session Host:**

```powershell
# Test DNS resolution
nslookup stfslogix01.file.core.windows.net
# Expected: 10.0.3.4 (private IP)

# Test with Resolve-DnsName (more details)
Resolve-DnsName -Name stfslogix01.file.core.windows.net -Type A
# Expected:
# Name: stfslogix01.privatelink.file.core.windows.net
# Type: A
# IPAddress: 10.0.3.4

# Test TCP connectivity
Test-NetConnection -ComputerName stfslogix01.file.core.windows.net -Port 445
# Expected: TcpTestSucceeded = True
```

**Test from On-Premises (via VPN):**

```powershell
# Test DNS resolution (should match Azure resolution if conditional forwarding configured)
nslookup stfslogix01.file.core.windows.net
# Expected: 10.0.3.4 (if on-prem DNS forwards to Azure)

# If returns public IP: On-prem DNS not forwarding privatelink queries
```

**Verify VNet Link:**

```bash
# List all VNet links for private DNS zone
az network private-dns link vnet list \
  --resource-group RG-Azure-VDI-01 \
  --zone-name privatelink.file.core.windows.net \
  --output table

# Expected output:
# Name            ResourceGroup    VirtualNetwork    ProvisioningState    RegistrationEnabled
# --------------  ---------------  ----------------  -------------------  ---------------------
# vnet-avd-link   RG-Azure-VDI-01  vnet-avd          Succeeded            False
```

## Validation Checklist

- [ ] Private DNS zone created with correct name (privatelink.{service}.core.windows.net)
- [ ] Private DNS zone linked to vnet-avd (auto-registration disabled)
- [ ] A record exists for each private endpoint (auto-created via DNS zone group)
- [ ] nslookup from session host returns private IP (not public IP)
- [ ] Test-NetConnection succeeds to private IP on required port
- [ ] On-prem DNS forwarding configured (if hybrid scenario)
- [ ] DNS cache flushed on session hosts after DNS changes (ipconfig /flushdns)

## Next Steps

1. **Create Private Endpoints:** Deploy private endpoints with DNS zone group integration (see Private Endpoints page)
2. **Test Resolution:** Verify DNS resolution from session hosts returns private IPs
3. **Configure On-Prem DNS:** Set up conditional forwarding if hybrid connectivity exists (see VPN Gateway page)
4. **Monitor DNS Queries:** Enable diagnostic logs on private DNS zone to track query patterns