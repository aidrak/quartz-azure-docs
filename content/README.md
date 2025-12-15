# Azure Virtual Desktop Reference Documentation

Comprehensive implementation and reference documentation for Azure Virtual Desktop (AVD) with Intune, FSLogix, and Entra ID.

## Published Documentation

📖 **View the published documentation**: https://aidrak.github.io/quartz-azure-docs/

The documentation is automatically published to the public site whenever changes are pushed to this repository.

## What's Included

### Deployment Playbook
Linear step-by-step guides for deploying a complete AVD environment:
- Prerequisites & Planning
- Identity Configuration
- Networking Setup
- Golden Image Creation
- Storage & FSLogix Setup
- Intune Configuration (Win32 apps, FSLogix policies, device settings)
- Host Pool Creation & Session Hosts
- Application Groups & Workspace
- Go-Live Monitoring & Insights

### Reference Documentation
Deep-dive reference pages organized by topic:
- **AVD** - Host pools, session host sizing, scaling plans, workspaces
- **Identity** - Entra ID, RBAC, conditional access, dynamic groups
- **Intune** - Device configuration, compliance policies, application deployment, remediation scripts
- **Storage** - FSLogix architecture, Azure Files, Entra Kerberos authentication
- **Networking** - VNets, NSGs, private endpoints, DNS
- **Security** - Windows security baselines, monitoring, Log Analytics
- **Operations** - User acceptance testing, troubleshooting, runbooks

## Key Topics

### Azure Files Entra Kerberos for FSLogix
Complete guide to configuring Entra Kerberos authentication on Azure Files for FSLogix profile containers. Includes:
- 8-part detailed setup guide
- Multiple implementation methods (Portal, PowerShell, CLI)
- Comprehensive troubleshooting (9 common issues)
- Complete verification and testing procedures

See: [[Storage/azure-files-entra-kerberos|Azure Files Entra Kerberos Configuration]]

### Intune Proactive Remediation Scripts
Collection of PowerShell scripts for automatically detecting and fixing configuration drift:
- Drive Mapping Task
- Notifications Enable
- Office Shortcuts

See: [[Intune/remediation-scripts/index|Intune Proactive Remediation Scripts]]

## How to Use

### For Deployment
1. Start with [[Deployment/00-naming-conventions|Step 00: Naming Conventions & Planning]]
2. Follow steps sequentially through the Deployment folder
3. Link to Reference pages for detailed information on specific topics

### For Reference
- Use topic folders (AVD, Identity, Intune, etc.) for deep dives
- Each reference page is self-contained and readable independently
- Link between related pages for cross-topic understanding

## Architecture

This documentation uses the "Playbook + Reference" pattern:

**Deployment (Playbook)**:
- Linear step-by-step guides
- Lightweight checklists with portal paths
- Links TO reference pages (no content duplication)

**Reference (Topic Folders)**:
- Deep-dive, self-contained pages
- Each page readable without context from others
- Format: What → Why/When → How → Best Practices → Troubleshooting

## Contributing

This documentation is maintained in Obsidian and synced to Git for backup and version control.

- **Edit in Obsidian**: Changes are automatically synced via Obsidian Sync
- **Commit to Git**: Use `git add .` and `git commit -m "..."` to create checkpoints
- **Publish**: Push to `main` branch; GitHub Actions automatically builds and deploys to the public site

### Naming Conventions

**Markdown Files**:
- Use lowercase with hyphens: `azure-files-entra-kerberos.md`
- One file per topic

**Wikilinks** (internal links):
```markdown
# Basic link
[[Page Name]]

# Link with display text
[[page-name|Display Text]]

# Cross-folder link
[[Networking/vpn-gateway|VPN Gateway]]
```

**Commit Messages**:
- Use conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`
- Example: `feat: add Entra Kerberos configuration guide for FSLogix`

## Repository Structure

```
Azure-Reference/
├── README.md                        # This file
├── Deployment/                      # Linear deployment steps (Step 00-11)
│   ├── 00-naming-conventions.md
│   ├── 01-prerequisites-licensing.md
│   └── ... (through Step 11)
├── AVD/                             # AVD architecture and sizing
├── Identity/                        # Entra ID, RBAC, conditional access
├── Intune/                          # Device management and remediation scripts
├── Storage/                         # FSLogix, Azure Files, Entra Kerberos
├── Networking/                      # VNets, NSGs, private endpoints
├── Security/                        # Windows baselines, monitoring, alerts
├── Operations/                      # Testing, troubleshooting, runbooks
├── Images/                          # Golden image process
└── Screenshots/                     # Images for documentation
```

## Key Resources

### Microsoft Learn
- [Azure Virtual Desktop Documentation](https://learn.microsoft.com/en-us/azure/virtual-desktop/)
- [Intune Device Management](https://learn.microsoft.com/en-us/mem/intune/)
- [Azure Files with Entra ID](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-active-directory-overview)
- [FSLogix Documentation](https://learn.microsoft.com/en-us/fslogix/)

### Recommended Deployment Path
1. **Planning Phase**: Read Step 00, 01, 02
2. **Infrastructure Phase**: Steps 03-05 (Networking, Images, Storage)
3. **Configuration Phase**: Steps 06-07 (Intune setup, FSLogix policies)
4. **Deployment Phase**: Steps 08-10 (Host pools, session hosts, app groups)
5. **Operations Phase**: Step 11 (Monitoring and go-live)

## Last Updated

- **Storage & FSLogix**: December 15, 2025 - Comprehensive Entra Kerberos guide added
- **Intune**: December 15, 2025 - Remediation scripts integrated
- **Deployment**: December 15, 2025 - Pooled-only FSLogix configuration

## License

This documentation is provided as-is for reference and educational purposes.
