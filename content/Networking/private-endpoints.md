---
title: Private Endpoints
description: 
published: true
date: 2025-12-14T04:53:26.594Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:44:59.493Z
---

# Private Endpoints

Private Endpoints bring Azure PaaS services (Storage, Key Vault, SQL) into your VNet with private IP addresses, eliminating public internet exposure. Essential for secure AVD deployments using FSLogix profiles on Azure Files.

## What are Private Endpoints

A Private Endpoint is a network interface that connects you privately to an Azure PaaS service using a private IP address from your VNet. Benefits:

- **No Public Exposure:** Service accessible only from your VNet (or peered VNets)
- **Private IP Address:** Azure assigns an IP from your subnet (e.g., 10.0.3.4)
- **DNS Integration:** Private DNS zones resolve service FQDNs to private IPs
- **Bypasses Service Firewalls:** Traffic doesn't traverse Azure's public backbone

**Key Concepts:**
- **Private Link Service:** The Azure service being connected to (e.g., Azure Files)
- **Private Endpoint:** The network interface in YOUR VNet
- **Private DNS Zone:** Resolves `storageaccount.file.core.windows.net` → private IP
- **VNet Linking:** Connects private DNS zone to your VNet for name resolution

**Traffic Flow Example:**
1. Session host (10.0.1.5) requests `stfslogix01.file.core.windows.net`
2. Private DNS zone resolves to 10.0.3.4 (private endpoint IP)
3. Traffic stays within VNet, reaches Azure Files backend
4. No internet traversal, no public IP exposure

## When to Use Private Endpoints

**Always Use For:**
- **FSLogix Profile Storage:** Azure Files shares for user profiles (security requirement)
- **Key Vault:** Storing certificates, secrets for AVD (no public access)
- **Azure SQL:** If using SQL for application databases
- **Storage Accounts:** Blob storage for scripts, images (if sensitive)

**Consider Alternatives:**
- **Service Endpoints:** If you need multi-region access or lower cost (but traffic uses Azure backbone, not VNet)
- **Public Access + Firewall:** Small non-production environments (NOT recommended for production AVD)

**When NOT Needed:**
- **Azure AD:** Authentication happens via public endpoints (uses Conditional Access for security)
- **AVD Control Plane:** Microsoft-managed service with public endpoints
- **Public-Facing Web Apps:** When users access from internet

## Real-World Example: pe-fslogix

Our production environment uses private endpoints for FSLogix profile storage:

**Configuration:**
- **Private Endpoint Name:** pe-fslogix-files
- **Target Resource:** stfslogix01 (Azure Storage Account)
- **Target Sub-Resource:** file (Azure Files)
- **Subnet:** snet-privateendpoints (10.0.3.0/24)
- **Private IP:** 10.0.3.4 (Azure-assigned)
- **DNS Zone:** privatelink.file.core.windows.net

**Architecture:**

```
Session Host (10.0.1.5)
   |
   | Request: \\stfslogix01.file.core.windows.net\profiles
   |
   v
Private DNS Zone (privatelink.file.core.windows.net)
   |
   | Resolves to: 10.0.3.4
   |
   v
Private Endpoint (pe-fslogix-files)
   |
   | Network Interface: 10.0.3.4 in snet-privateendpoints
   |
   v
Azure Files Backend (stfslogix01)
```

**Benefits:**
- Session hosts access profiles via SMB without internet traversal
- Storage account firewall blocks all public access
- NSG rules control which subnets can reach private endpoint
- Complies with security policies requiring no public storage access

## How to Configure

### Azure Portal

**Azure Portal → Storage Account → Networking → Private Endpoint Connections → Private Endpoint**

1. **Basics:**
   - Resource Group: RG-Azure-VDI-01
   - Name: pe-fslogix-files
   - Region: East US (must match VNet region)

2. **Resource:**
   - Connection Method: Connect to an Azure resource in my directory
   - Subscription: [Your Subscription]
   - Resource Type: Microsoft.Storage/storageAccounts
   - Resource: stfslogix01
   - Target Sub-Resource: file

3. **Virtual Network:**
   - Virtual Network: vnet-avd
   - Subnet: snet-privateendpoints (10.0.3.0/24)
   - Private IP Configuration: Dynamically allocate IP address

4. **DNS:**
   - Integrate with Private DNS Zone: Yes
   - Private DNS Zone: privatelink.file.core.windows.net
   - (Azure creates zone if it doesn't exist)

5. **Review + Create**

6. **Disable Public Access (CRITICAL):**
   - Navigate to Storage Account → Networking
   - Firewalls and Virtual Networks → Enabled from selected virtual networks and IP addresses
   - Virtual Networks: Remove all entries (rely on private endpoint only)
   - Exceptions: Uncheck "Allow Azure services on the trusted services list to access this storage account" (unless needed for backups)
   - Save

### Azure CLI

```bash
# Get storage account resource ID
STORAGE_ID=$(az storage account show \
  --resource-group RG-Azure-VDI-01 \
  --name stfslogix01 \
  --query id \
  --output tsv)

# Get subnet ID
SUBNET_ID=$(az network vnet subnet show \
  --resource-group RG-Azure-VDI-01 \
  --vnet-name vnet-avd \
  --name snet-privateendpoints \
  --query id \
  --output tsv)

# Create private endpoint
az network private-endpoint create \
  --resource-group RG-Azure-VDI-01 \
  --name pe-fslogix-files \
  --vnet-name vnet-avd \
  --subnet snet-privateendpoints \
  --private-connection-resource-id $STORAGE_ID \
  --group-id file \
  --connection-name pe-fslogix-connection

# Create private DNS zone (if not exists)
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

# Create DNS zone group (auto-creates A record)
az network private-endpoint dns-zone-group create \
  --resource-group RG-Azure-VDI-01 \
  --endpoint-name pe-fslogix-files \
  --name default \
  --private-dns-zone privatelink.file.core.windows.net \
  --zone-name privatelink.file.core.windows.net

# Disable public access to storage account
az storage account update \
  --resource-group RG-Azure-VDI-01 \
  --name stfslogix01 \
  --default-action Deny
```

## Best Practices

**Subnet Design:**
- **Dedicated Subnet:** Use snet-privateendpoints for all private endpoints (easier NSG management)
- **Disable Network Policies:** Required for private endpoint creation: `az network vnet subnet update --disable-private-endpoint-network-policies true`
- **Sufficient IP Space:** Use /24 subnet - each private endpoint consumes 1 IP, plan for growth

**DNS Configuration:**
- **Always Use Private DNS Zones:** Manual DNS records are error-prone and don't auto-update
- **One Zone per Service Type:** privatelink.file.core.windows.net for all storage accounts using Azure Files
- **Link to All VNets:** If using hub-spoke, link private DNS zone to all spoke VNets
- **Verify Resolution:** Test with `nslookup stfslogix01.file.core.windows.net` from session host (should return 10.0.3.x)

**Security:**
- **Disable Public Access:** Set storage account firewall to Deny (selected networks only)
- **NSG Rules:** Explicitly allow session hosts → private endpoint subnet on required ports (445 for SMB)
- **RBAC:** Use Azure RBAC instead of storage account keys (assign "Storage File Data SMB Share Contributor" to users)
- **Audit Logs:** Enable diagnostic settings to monitor private endpoint access

**Cost Optimization:**
- **Consolidate Endpoints:** One private endpoint can serve multiple subnets in the same VNet
- **Share DNS Zones:** Use same private DNS zone across multiple storage accounts (e.g., stfslogix01 and stfslogix02 both use privatelink.file.core.windows.net)
- **Cost:** ~$0.01/hour per private endpoint (~$7.30/month) + data processing fees

## DNS Integration Explained

**Without Private Endpoint:**
```bash
# Public DNS resolution
nslookup stfslogix01.file.core.windows.net
# Returns: 52.239.x.x (public IP)
```

**With Private Endpoint (Correctly Configured):**
```bash
# Private DNS resolution
nslookup stfslogix01.file.core.windows.net
# Returns:
# Name: stfslogix01.privatelink.file.core.windows.net
# Address: 10.0.3.4
```

**How It Works:**
1. Azure creates CNAME record in public DNS: `stfslogix01.file.core.windows.net` → `stfslogix01.privatelink.file.core.windows.net`
2. Private DNS zone `privatelink.file.core.windows.net` contains A record: `stfslogix01` → `10.0.3.4`
3. VNet-linked clients resolve via private DNS zone, get private IP
4. External clients (internet) cannot resolve `stfslogix01.privatelink.file.core.windows.net`, cannot reach service

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **"Cannot create private endpoint"** | Subnet has network policies enabled | Disable policies: `az network vnet subnet update --disable-private-endpoint-network-policies true --name snet-privateendpoints --vnet-name vnet-avd --resource-group RG-Azure-VDI-01` |
| **"Name resolution returns public IP"** | Private DNS zone not linked to VNet or missing A record | Verify VNet link exists: `az network private-dns link vnet list --zone-name privatelink.file.core.windows.net --resource-group RG-Azure-VDI-01`. Create DNS zone group if A record missing. |
| **"Access denied to storage account"** | Public access disabled but no private endpoint route | Check NSG allows traffic to 10.0.3.0/24 on port 445. Verify private endpoint NIC is healthy. Test with `Test-NetConnection -ComputerName 10.0.3.4 -Port 445`. |
| **"Private endpoint IP not responding"** | NSG blocking traffic or storage account firewall misconfigured | Disable storage account firewall temporarily for testing. Check NSG flow logs for denied traffic. Verify private endpoint NIC is attached and provisioned. |
| **"Certificate errors when accessing storage"** | Custom DNS servers not forwarding to Azure DNS (168.63.129.16) | Configure on-prem DNS servers to forward privatelink.file.core.windows.net queries to 168.63.129.16 (Azure's recursive resolver). Or use Azure Private DNS Resolver. |

## Private Endpoints for Common Services

### Azure Files (FSLogix Profiles)

- **Target Sub-Resource:** file
- **Private DNS Zone:** privatelink.file.core.windows.net
- **Required Port:** 445 (SMB)
- **Example:** pe-fslogix-files → stfslogix01

### Azure Key Vault

- **Target Sub-Resource:** vault
- **Private DNS Zone:** privatelink.vaultcore.azure.net
- **Required Port:** 443 (HTTPS)
- **Example:** pe-keyvault → kv-avd-prod

### Azure SQL Database

- **Target Sub-Resource:** sqlServer
- **Private DNS Zone:** privatelink.database.windows.net
- **Required Port:** 1433 (TDS)
- **Example:** pe-sql-avd → sql-avd-prod

### Azure Blob Storage

- **Target Sub-Resource:** blob
- **Private DNS Zone:** privatelink.blob.core.windows.net
- **Required Port:** 443 (HTTPS)
- **Example:** pe-scripts-blob → stavdscripts

## Validation Checklist

- [ ] Private endpoint created in snet-privateendpoints
- [ ] Target sub-resource correct (file for Azure Files, vault for Key Vault)
- [ ] Private DNS zone created (privatelink.{service}.core.windows.net)
- [ ] Private DNS zone linked to vnet-avd
- [ ] A record exists in private DNS zone (auto-created by DNS zone group)
- [ ] NSG allows session hosts (10.0.1.0/24) → private endpoints (10.0.3.0/24) on required port
- [ ] Storage account public access disabled (Networking → Firewalls → Deny)
- [ ] DNS resolution returns private IP from session host (nslookup test)
- [ ] Network connectivity test succeeds (Test-NetConnection)
- [ ] Service accessible from session host (SMB mount, SQL connection, etc.)