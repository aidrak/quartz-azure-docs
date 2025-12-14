# Networking Reference

This section covers networking components for Azure Virtual Desktop deployments. These pages provide deep-dive technical guidance on VNets, subnets, security, connectivity, and network architecture patterns.

## What's Covered

Networking is the foundation of your AVD infrastructure. This reference section covers:

- Virtual Network (VNet) design and subnet planning
- Network Security Groups (NSGs) for traffic control
- Private connectivity to Azure services via private endpoints
- Hybrid connectivity using VPN Gateway or ExpressRoute
- DNS resolution for private endpoints
- Enterprise network patterns (hub-spoke topology)
- Advanced security with Azure Firewall

## Reference Pages

### Core Networking

[[../../Networking/vnet-design|VNet Design]]
Foundation of Azure networking. Covers address space planning, subnet segmentation, CIDR allocation, and sizing for AVD deployments. Learn when to use single VNet vs multiple VNets, and how to avoid IP conflicts.

[[../../Networking/nsg-best-practices|NSG Best Practices]]
Network Security Groups control traffic at Layer 4 (IP/port). Essential reading for understanding AVD's required outbound connectivity, security rule design, service tags, and troubleshooting blocked traffic. Includes real-world NSG configurations for session hosts and private endpoints.

[[../../Networking/private-endpoints|Private Endpoints]]
Secure access to Azure PaaS services (Azure Files, Key Vault, SQL) using private IPs within your VNet. Critical for FSLogix profile storage. Covers DNS integration, subnet requirements, and disabling public access.

[[../../Networking/private-dns-zones|Private DNS Zones]]
Enable name resolution for private endpoints. Learn how Azure resolves FQDNs to private IPs, VNet linking, conditional forwarding for hybrid scenarios, and troubleshooting DNS issues.

### Hybrid Connectivity

[[../../Networking/vpn-gateway|VPN Gateway]]
Site-to-Site and Point-to-Site VPN connectivity between Azure and on-premises networks. Covers gateway SKUs, IPsec configuration, shared keys, and when to use VPN vs ExpressRoute. Essential for hybrid AVD deployments with on-prem Active Directory.

[[../../Networking/expressroute|ExpressRoute]]
Private, dedicated connectivity for enterprise AVD. High bandwidth (up to 100 Gbps), low latency (<10ms), and predictable performance. Learn circuit provisioning, peering types, gateway SKUs, and cost comparison with VPN.

### Advanced Architecture

[[../../Networking/hub-spoke-topology|Hub-Spoke Topology]]
Enterprise network pattern for centralized security and shared services. Covers VNet peering, transitive routing, UDR configuration, and multi-environment isolation (prod/dev/test). Recommended for organizations with multiple AVD deployments.

[[../../Networking/azure-firewall|Azure Firewall]]
Cloud-native firewall with FQDN filtering, threat intelligence, and centralized logging. Learn when NSGs aren't sufficient, application rule design for AVD, UDR configuration, and required outbound FQDNs for session hosts.

## Related Quick-Deploy Steps

The Quick-Deploy playbook references these pages for detailed configuration:

- [[../../Quick-Deploy/04-networking|Step 04: Networking Setup]] - Walkthrough for creating VNets, subnets, NSGs, and VPN Gateway for AVD

## When to Use Each Component

**Start with these (required for all deployments):**
1. [[../../Networking/vnet-design|VNet Design]] - Plan your address space
2. [[../../Networking/nsg-best-practices|NSG Best Practices]] - Secure your subnets
3. [[../../Networking/private-dns-zones|Private DNS Zones]] - Required if using private endpoints

**Add these for hybrid connectivity:**
- [[../../Networking/vpn-gateway|VPN Gateway]] - Small to medium deployments needing on-prem access
- [[../../Networking/expressroute|ExpressRoute]] - Enterprise deployments requiring high bandwidth and low latency

**Consider these for advanced scenarios:**
- [[../../Networking/private-endpoints|Private Endpoints]] - Production deployments with strict security requirements
- [[../../Networking/hub-spoke-topology|Hub-Spoke Topology]] - Multiple environments or centralized security
- [[../../Networking/azure-firewall|Azure Firewall]] - FQDN filtering or centralized policy management

## Key Concepts

**Cloud-Only vs Hybrid:**
- Cloud-only deployments use Entra ID Join and don't need VPN/ExpressRoute
- Hybrid deployments require VPN Gateway or ExpressRoute for on-prem AD DS domain join

**Public vs Private Connectivity:**
- AVD control plane always uses public endpoints (secure by design)
- Azure Files for FSLogix should use private endpoints (blocks internet exposure)
- Session hosts need outbound internet for AVD agent, Windows Update, and activation

**Cost Considerations:**
- VNets and NSGs are free
- VPN Gateway: $30-300/month depending on SKU
- ExpressRoute: $50-$10,000+/month depending on bandwidth
- Private Endpoints: ~$7.30/month each
- Azure Firewall: ~$900-$1,500/month depending on SKU
- VNet Peering: ~$0.01/GB transferred

## Common Patterns

**Small Deployment (1-50 users):**
- Single VNet (10.0.0.0/16)
- NSGs for security (no Azure Firewall)
- VPN Gateway if hybrid (VpnGw1 SKU)
- Private endpoints for Azure Files

**Medium Deployment (50-200 users):**
- Single VNet or hub-spoke if multiple environments
- NSGs + Azure Firewall (Standard SKU)
- VPN Gateway (VpnGw2) or ExpressRoute (Local SKU)
- Private endpoints for all PaaS services

**Enterprise Deployment (200+ users):**
- Hub-spoke topology with zone-redundant services
- Azure Firewall Premium (TLS inspection, IDPS)
- ExpressRoute (Standard/Premium SKU) with VPN backup
- Private endpoints with centralized DNS management
- Multi-region for disaster recovery

## Troubleshooting Quick Reference

**Session hosts can't reach AVD control plane:**
- Check NSG allows outbound to WindowsVirtualDesktop service tag on port 443
- Verify UDR isn't blocking traffic to AVD endpoints
- Test: `Test-NetConnection -ComputerName rdbroker.wvd.microsoft.com -Port 443`

**Cannot access Azure Files:**
- Verify private endpoint DNS resolves to private IP (not public)
- Check NSG allows port 445 from session hosts to private endpoint subnet
- Test: `Test-NetConnection -ComputerName storageaccount.file.core.windows.net -Port 445`

**VPN tunnel down:**
- Verify shared key matches on both sides
- Check on-prem firewall allows UDP 500 and 4500
- Review connection status in Azure Portal

**High latency or packet loss:**
- Check VPN/ExpressRoute gateway SKU is appropriately sized
- Monitor bandwidth utilization (upgrade if consistently >80%)
- Review NSG flow logs for dropped packets

## Next Steps

1. Review [[../../Networking/vnet-design|VNet Design]] to plan your address space
2. Follow [[../../Quick-Deploy/04-networking|Step 04: Networking Setup]] to deploy networking components
3. Return to specific reference pages as needed during implementation
