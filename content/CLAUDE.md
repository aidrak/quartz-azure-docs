# Azure Reference Documentation

## Project Purpose
Azure Virtual Desktop implementation and reference documentation, maintained in Obsidian.
Published at: **https://aidrak.github.io/quartz-azure-docs/**

## How It Works
- **Obsidian Sync** handles real-time sync across devices
- **Git backup** to private repo `github.com/aidrak/azure-reference`
- **Quartz** publishes to GitHub Pages automatically on push

## Folder Structure

```
Azure-Reference/
├── CLAUDE.md                    # This file
├── AVD/                         # AVD Architecture
│   ├── host-pools-deep-dive.md
│   ├── session-host-sizing.md
│   ├── scaling-plans.md
│   └── workspaces.md
├── Deployment/                  # Linear deployment playbook
│   ├── prerequisites-planning.md
│   ├── resource-group-tagging.md
│   ├── identity-configuration.md
│   ├── networking-setup.md
│   └── ...
├── Identity/                    # Entra ID, Conditional Access
├── Images/                      # Golden Image, Azure Compute Gallery
├── Intune/                      # Device management
├── Networking/                  # VNets, NSGs, VPN, DNS
├── Operations/                  # Capacity, Cost
├── Security/                    # Defender, Monitoring
└── Storage/                     # Azure Files, FSLogix
```

## Book Architecture

**Deployment (playbook):**
- Linear step-by-step guides - the "golden path"
- Lightweight checklists with Portal paths
- Links TO reference pages, never duplicates explanations
- Decision points for variants (hybrid, no Intune, etc.)

**Reference (topic folders):**
- Deep-dive, self-contained pages
- Each page readable without context from others
- Format: What → Why/When → How (Portal) → Best Practices → Troubleshooting

## Wikilink Format

Use Obsidian wikilinks for internal links:

```markdown
# Basic link
[[Page Name]]

# Link with display text
[[page-name|Display Text]]

# Cross-folder link
[[Networking/vpn-gateway|VPN Gateway]]
```

## Formatting Standards

**Playbook pages:**
```markdown
# Step X: [Component]

**Portal:** Azure Portal → Service → Blade

1. Action one
2. Action two
3. Configure per [[Reference Page]]

> **Decision Point:**
> - Standard: Continue to Step X+1
> - Hybrid: See [[vpn-gateway|VPN Gateway]] first
```

**Reference pages:**
```markdown
# [Topic]

## What It Is
Brief explanation.

## When to Use It
- Use cases and decision criteria

## How to Set It Up
**Portal:** Azure Portal → Service → Blade
1. Steps here

## Best Practices
- Recommendation with reasoning

## Common Issues
### Issue: [Problem]
**Symptom:** What you see
**Cause:** Why
**Fix:** Resolution
```

### Callouts
- `> **Note:**` - Tips
- `> **Warning:**` - Gotchas
- `> **Decision Point:**` - Variant paths

## Git Backup Workflow

```bash
# From Azure-Reference folder
git add .
git commit -m "feat: add new section on X"
git push
```

Commit messages: Use conventional commits (`feat:`, `fix:`, `docs:`, etc.)

## Publishing to Web

Changes pushed to `azure-reference` repo need to be synced to Quartz:

```bash
cd /mnt/cache_pool/development/quartz-azure-docs
git add .
git commit -m "Update content"
git push
```

GitHub Actions auto-builds and deploys to `aidrak.github.io/quartz-azure-docs`.

## Key Principles

1. **Folder structure = site navigation** - Organize folders as you want the site structured
2. **Don't duplicate** - Playbook links to Reference
3. **Self-contained Reference** - Each page readable standalone
4. **Real examples** - Use actual resource names from test environment
5. **Portal-first** - CLI only when Portal can't do it
6. **Explain WHY** - Best practices include reasoning
7. **Wikilinks** - Use `[[PageName]]` format for internal links

## Task Master AI Instructions
**Import Task Master's development workflow commands and guidelines, treat as if import is in the main CLAUDE.md file.**
@./.taskmaster/CLAUDE.md
