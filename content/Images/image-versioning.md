---
title: Image Versioning
description: 
published: true
date: 2025-12-14T04:53:07.105Z
tags: 
editor: markdown
dateCreated: 2025-12-14T03:45:24.106Z
---

# Image Versioning

Effective image version management is critical to maintaining a stable Azure Virtual Desktop environment while delivering timely security updates and new features. This page covers version numbering strategies, deciding when to create new versions, rolling out updates to session hosts, and implementing rollback procedures when new images introduce problems.

## Version Numbering Strategy

Azure Compute Gallery uses semantic versioning for image versions: `MAJOR.MINOR.PATCH`. This three-part numbering system communicates the scope of changes at a glance and helps operators decide when and how to deploy updates.

### Semantic Versioning: MAJOR.MINOR.PATCH

**Format:** `1.0.2` where:
- **MAJOR (1.x.x)** - Incompatible or breaking changes
- **MINOR (x.1.x)** - Backward-compatible new features or major updates
- **PATCH (x.x.2)** - Backward-compatible bug fixes and security patches

### MAJOR Version (1.x.x → 2.x.x)

Increment the major version when making changes that are not backward-compatible with existing deployments or that represent a fundamental shift in the image baseline.

**Examples:**
- **OS Version Upgrade:** Windows 10 22H2 → Windows 11 25H2 (different OS SKU, licensing, and feature set)
- **Architecture Change:** Windows 11 single-session → Windows 11 multi-session (changes licensing model and user experience)
- **Major Feature Update:** Windows 11 24H2 → Windows 11 25H2 (annual feature updates often introduce breaking changes)
- **Baseline Rebuild:** Completely rebuilding the image with a new application stack (e.g., migrating from Office 2016 to M365 Apps)

**Deployment Approach:**
- **Plan extensively:** Test for weeks in dev/UAT environments
- **Pilot rollout:** Deploy to a small subset of users (5-10%) before full deployment
- **Maintain previous version:** Keep MAJOR-1 version available for quick rollback if critical issues arise
- **Communication:** Notify users of significant changes (new UI, removed features, etc.)

**Our Example:** If we upgrade from Windows 11 25H2 to Windows 11 26H2, we'd go from `1.x.x` to `2.0.0`.

### MINOR Version (1.0.x → 1.1.x)

Increment the minor version when adding new functionality or making substantial updates that don't break compatibility with existing session hosts.

**Examples:**
- **New Applications:** Adding Adobe Acrobat, Microsoft Project, or company VPN client to the image
- **Application Major Versions:** Upgrading FSLogix 2.9.8 → FSLogix 2.9.9 (minor feature update)
- **Configuration Changes:** Enabling new group policies, adding monitoring agents, changing default browser settings
- **Optimization Updates:** Applying new VDOT optimizations, adjusting pagefile settings, or enabling performance tweaks
- **Certificate Updates:** Adding new root CA certificates for internal PKI changes

**Deployment Approach:**
- **Test in UAT:** 1-2 weeks of testing with representative user workloads
- **Gradual rollout:** Update one host pool at a time, starting with non-critical departments
- **Monitor metrics:** Track login times, application errors, and user support tickets during rollout
- **Validation criteria:** Ensure new features work, old features still work, no performance regressions

**Our Example:** Adding Microsoft Teams to an existing Windows 11 25H2 image: `1.0.2` → `1.1.0`.

### PATCH Version (1.0.0 → 1.0.1)

Increment the patch version for routine maintenance, security updates, and bug fixes that have minimal user impact.

**Examples:**
- **Monthly Security Updates:** Patch Tuesday Windows updates (most common patch version trigger)
- **Hotfixes:** Applying out-of-band security patches or critical bug fixes
- **Minor App Updates:** FSLogix 2.9.9000 → 2.9.9001 (patch release)
- **Configuration Tweaks:** Fixing a typo in a registry key, adjusting a service startup type, correcting a GPO setting
- **Dependency Updates:** Updating .NET Framework, Visual C++ Redistributables without feature changes

**Deployment Approach:**
- **Minimal testing:** 2-3 days in dev, quick UAT validation
- **Automated deployment:** Can be rolled out via Azure Image Builder on a monthly schedule
- **Low risk tolerance:** Patch versions should be boring—if they require extensive testing, they're probably MINOR changes
- **Fast rollback:** If issues arise, rollback to previous PATCH version within hours

**Our Example:** Applying January 2025 Patch Tuesday updates: `1.0.0` → `1.0.1`.

### Our Environment Versioning

**Win11_Multi_25H2_Gen2:**
- **1.0.0** (November 2024) - Initial Windows 11 25H2 multi-session image with FSLogix, M365 Apps, Adobe Reader, VDOT optimizations
- **1.0.1** (December 2024) - December Patch Tuesday security updates, FSLogix agent update
- **1.0.2** (January 2025) - January Patch Tuesday security updates, added company VPN client

**Win11_Single_25H2_Gen2:**
- **1.0.0** (November 2024) - Initial Windows 11 25H2 single-session image with FSLogix, M365 Apps, Adobe Reader

**Planned Future Versions:**
- **1.1.0** (February 2025) - Add Microsoft Project and Visio to both images (MINOR - new apps)
- **2.0.0** (Fall 2025) - Upgrade to Windows 11 26H2 (MAJOR - OS version change)

## When to Create New Versions

### Monthly: Security Updates

**Trigger:** Second Tuesday of each month (Patch Tuesday)

**Process:**
1. **Patch Tuesday + 1 day:** Microsoft releases cumulative updates. Wait 24 hours for community feedback on critical bugs.
2. **Wednesday/Thursday:** Trigger Azure Image Builder build with Windows Update customizer.
3. **Friday:** Test new image version in dev environment (smoke test: login, launch apps, check performance).
4. **Next Monday:** Deploy to UAT host pool for representative user testing.
5. **Following Week:** Roll out to production host pools incrementally (pilot → general availability).

**Version Example:** `1.0.1` → `1.0.2` (PATCH increment)

**Automation:** Use Azure DevOps or GitHub Actions to trigger AIB builds automatically on the second Tuesday. Include approval gates before production distribution.

**Exception:** If a critical zero-day vulnerability is announced mid-month, create an emergency patch version immediately (e.g., `1.0.2.1` or skip to `1.0.3`).

### Quarterly: Application Updates

**Trigger:** Major application version releases (Microsoft 365 Apps, FSLogix, LOB apps)

**Example Scenarios:**
- **FSLogix 2.9.9 → 2.9.10:** New features (Cloud Cache improvements, VHD compaction). Create MINOR version `1.0.x` → `1.1.0`.
- **M365 Apps Monthly Enterprise Channel:** Auto-updates via Click-to-Run, no image rebuild needed. If switching channels (Monthly → Semi-Annual), create MINOR version.
- **Adobe Acrobat DC 2023.006 → 2024.001:** Major version bump, test for compatibility issues. Create MINOR version.

**Process:**
1. Review vendor release notes for breaking changes
2. Test in isolated dev environment with regression testing
3. If no issues, schedule UAT deployment for 2 weeks
4. After UAT approval, create new image version
5. Roll out to production over 4 weeks (10% → 25% → 50% → 100%)

**Version Example:** `1.0.2` → `1.1.0` (MINOR increment)

### Annually: Major OS Updates

**Trigger:** Windows annual feature updates (e.g., Windows 11 25H2 → 26H2)

**Example Scenario:** Windows 11 26H2 releases in Fall 2025 with new UI changes, security features, and API updates.

**Process:**
1. **Month 1-2:** Create new image definition `Win11_Multi_26H2_Gen2` alongside existing `Win11_Multi_25H2_Gen2` (don't replace).
2. **Month 3-4:** Build version `2.0.0` (MAJOR increment) and deploy to dev environment. Extensive testing with all LOB apps.
3. **Month 5:** Deploy to UAT for representative user groups. Collect feedback on UI changes, performance, compatibility.
4. **Month 6:** Pilot production deployment (5-10% of users, non-critical departments).
5. **Month 7-12:** Gradual rollout to all production host pools. Monitor closely for regressions.
6. **Month 13+:** Retire old `Win11_Multi_25H2_Gen2` image definition (stop creating new versions, but keep last version for legacy hosts).

**Version Example:** `1.x.x` → `2.0.0` (MAJOR increment)

**Warning:** Never rush annual feature updates. Microsoft's recommendation is to wait 6 months after release before deploying to production AVD environments.

### Ad-Hoc: Configuration Changes

**Trigger:** Business requirements, security policy changes, or bug fixes

**Examples:**
- **New Company Certificate:** Internal CA root certificate expires, must add new cert. Create PATCH version `1.0.2` → `1.0.3`.
- **GPO Setting Change:** Disable OneDrive Files On-Demand due to user complaints. Create PATCH version.
- **Service Startup Fix:** Windows Search accidentally re-enabled, causing high CPU. Create PATCH version.
- **New Monitoring Agent:** Add Datadog or Splunk agent. Create MINOR version `1.0.3` → `1.1.0`.

**Decision Criteria:**
- **Does it add new functionality?** → MINOR version
- **Does it fix a bug or apply a patch?** → PATCH version
- **Does it fundamentally change the image baseline?** → MAJOR version

## Retention Policies

Keeping every image version forever consumes storage and complicates management. Implement lifecycle policies to automatically expire old versions.

### Keep Last N Versions

**Recommended Policy:** Retain the 3 most recent versions per image definition.

**Rationale:**
- **Latest (e.g., 1.0.2):** Current production version
- **Latest - 1 (e.g., 1.0.1):** Rollback target if latest has issues
- **Latest - 2 (e.g., 1.0.0):** Emergency fallback if both newer versions fail

**Older versions (1.0.-1, 1.0.-2, etc.):** Delete after 90 days unless explicitly pinned as "known good."

**Implementation (Azure CLI):**

```bash
# List all versions for an image definition
az sig image-version list \
  --resource-group rg-avd-prod-01 \
  --gallery-name gal-avd-prod-01 \
  --gallery-image-definition Win11_Multi_25H2_Gen2 \
  --query "sort_by([].{Name:name, Created:publishingProfile.publishedDate}, &Created)" \
  --output table

# Delete old versions (keep last 3)
# Manual script or Azure Automation Runbook
versions=$(az sig image-version list \
  --resource-group rg-avd-prod-01 \
  --gallery-name gal-avd-prod-01 \
  --gallery-image-definition Win11_Multi_25H2_Gen2 \
  --query "sort_by([].name, &@)" -o tsv)

# Get all versions except last 3
old_versions=$(echo "$versions" | head -n -3)

for version in $old_versions; do
  echo "Deleting version $version"
  az sig image-version delete \
    --resource-group rg-avd-prod-01 \
    --gallery-name gal-avd-prod-01 \
    --gallery-image-definition Win11_Multi_25H2_Gen2 \
    --gallery-image-version $version
done
```

### Exclude from Latest

For testing or known-bad versions, set `excludeFromLatest: true` to prevent accidental deployment.

**Use Case:** Version `1.0.3` has a critical bug discovered post-release. Mark it as excluded, and deployments automatically fall back to `1.0.2`.

**Azure CLI:**

```bash
az sig image-version update \
  --resource-group rg-avd-prod-01 \
  --gallery-name gal-avd-prod-01 \
  --gallery-image-definition Win11_Multi_25H2_Gen2 \
  --gallery-image-version 1.0.3 \
  --exclude-from-latest true
```

### End of Life Dates

Set automatic expiration dates on image versions to enforce retention policies.

**Example:** Set version `1.0.0` to expire 90 days after `1.0.3` is released.

**Azure Portal:**
1. Navigate to Compute Gallery → Image Definition → Image Version
2. Edit version properties
3. Set "End of life date" to `2025-04-15`
4. After this date, the version remains visible but is marked as deprecated in deployment UIs

**Azure CLI:**

```bash
az sig image-version update \
  --resource-group rg-avd-prod-01 \
  --gallery-name gal-avd-prod-01 \
  --gallery-image-definition Win11_Multi_25H2_Gen2 \
  --gallery-image-version 1.0.0 \
  --end-of-life-date 2025-04-15
```

## Rolling Out New Versions to Session Hosts

Creating a new image version is only half the battle—you must also update existing session hosts to use it.

### Pooled Host Pools (Drain Mode Strategy)

**Scenario:** You have 20 session hosts running image version `1.0.1` and want to upgrade to `1.0.2`.

**Process:**

1. **Set drain mode on subset of hosts:**

```bash
# Enable drain mode on 5 hosts (25% of fleet)
for i in {1..5}; do
  az desktopvirtualization sessionhost update \
    --resource-group rg-avd-prod-01 \
    --host-pool-name hp-pooled-prod \
    --name "AVD-SH-0$i.domain.com" \
    --allow-new-session false
done
```

**Drain mode:** Prevents new user logins but allows existing sessions to complete normally. Users drain off naturally over 24-48 hours.

2. **Wait for sessions to drain:**

```bash
# Monitor active sessions
az desktopvirtualization session-host list \
  --resource-group rg-avd-prod-01 \
  --host-pool-name hp-pooled-prod \
  --query "[?allowNewSession==\`false\`].{Name:name, Sessions:sessions}" \
  --output table
```

Once sessions reach 0, proceed to deletion.

3. **Delete drained session hosts:**

```bash
for i in {1..5}; do
  # Delete session host registration
  az desktopvirtualization sessionhost delete \
    --resource-group rg-avd-prod-01 \
    --host-pool-name hp-pooled-prod \
    --name "AVD-SH-0$i.domain.com" \
    --yes

  # Delete underlying VM
  az vm delete \
    --resource-group rg-avd-prod-01 \
    --name "AVD-SH-0$i" \
    --yes
done
```

4. **Deploy new session hosts from updated image:**

```bash
# Deploy 5 new session hosts using image version 1.0.2
az deployment group create \
  --resource-group rg-avd-prod-01 \
  --template-file deploy-session-hosts.json \
  --parameters \
    hostPoolName=hp-pooled-prod \
    vmCount=5 \
    vmNamePrefix=AVD-SH- \
    imageVersion=1.0.2 \
    vmSize=Standard_D4s_v5
```

5. **Verify new hosts are healthy:**

```bash
az desktopvirtualization session-host list \
  --resource-group rg-avd-prod-01 \
  --host-pool-name hp-pooled-prod \
  --query "[?status=='Available'].{Name:name, Image:imageVersion, Status:status}"
```

6. **Repeat for remaining hosts:** Continue draining and replacing in batches of 25% until all hosts are on version `1.0.2`.

**Total Time:** 1-2 weeks for full rollout (depending on drain speed and testing intervals between batches).

### Personal Host Pools (Scheduled Maintenance Windows)

**Scenario:** Personal desktops assigned to specific users. Can't delete without user coordination.

**Process:**

1. **Notify users of maintenance window:** Email users 1 week in advance: "Your AVD desktop will be refreshed on Saturday, Jan 25th, 2025 from 2:00 AM - 6:00 AM. Save all work and log off by Friday night."

2. **Schedule Azure Automation Runbook:**

```powershell
# Runbook: Update-PersonalDesktops
param (
    [string]$ResourceGroup = "rg-avd-prod-01",
    [string]$HostPoolName = "hp-personal-prod",
    [string]$NewImageVersion = "1.0.2"
)

# Get all session hosts
$sessionHosts = Get-AzWvdSessionHost -ResourceGroupName $ResourceGroup -HostPoolName $HostPoolName

foreach ($host in $sessionHosts) {
    # Extract VM name
    $vmName = ($host.Name -split '/')[1] -replace '\..*'

    # Delete session host registration
    Remove-AzWvdSessionHost -ResourceGroupName $ResourceGroup -HostPoolName $HostPoolName -Name $host.Name

    # Delete VM
    Remove-AzVM -ResourceGroupName $ResourceGroup -Name $vmName -Force

    # Redeploy from new image version
    # (Use ARM template or Azure CLI deployment)
}
```

3. **Run during maintenance window:** 2:00 AM Saturday (low usage time).

4. **Validate Monday morning:** Check that all users can log in and access their applications.

**Impact:** 4-hour downtime per user desktop. Acceptable for monthly updates if scheduled appropriately.

### Rolling Updates (Blue-Green Deployment)

**Advanced Strategy:** Maintain two host pools—one production (blue), one staging (green).

**Process:**

1. **Create staging host pool with new image version:**
   - Deploy `hp-pooled-prod-Green` with 5 session hosts on image `1.0.2`
   - Assign to a test application group with pilot users (5-10%)

2. **Monitor staging for 1 week:**
   - Track metrics: login failures, app crashes, performance issues
   - Collect user feedback

3. **Swap traffic to green:**
   - Reassign production application group to green host pool
   - Set blue host pool to drain mode

4. **Decommission blue host pool:**
   - After all users migrated, delete blue host pool and VMs

5. **Promote green to production:**
   - Rename `hp-pooled-prod-Green` → `hp-pooled-prod`

**Benefit:** Zero downtime, instant rollback capability (just swap back to blue).

**Cost:** 2x session host capacity during migration period.

## Rollback Procedures

### Quick Rollback (Within 24 Hours)

**Scenario:** Version `1.0.2` was deployed this morning, and users report widespread login failures.

**Immediate Action:**

1. **Stop deploying new hosts on 1.0.2:**

```bash
# If using Azure Image Builder, cancel in-progress builds
az image builder cancel \
  --resource-group rg-avd-prod-01 \
  --name AVD-Win11-25H2-Multi-Template-01
```

2. **Redeploy session hosts using previous version:**

```bash
# Deploy new hosts from version 1.0.1
az deployment group create \
  --resource-group rg-avd-prod-01 \
  --template-file deploy-session-hosts.json \
  --parameters imageVersion=1.0.1
```

3. **Drain and delete problematic hosts:**

```bash
# Set drain mode on all hosts running 1.0.2
az desktopvirtualization sessionhost update \
  --resource-group rg-avd-prod-01 \
  --host-pool-name hp-pooled-prod \
  --name "AVD-SH-01.domain.com" \
  --allow-new-session false

# Force logoff users (if critical issue)
az desktopvirtualization user-session delete \
  --resource-group rg-avd-prod-01 \
  --host-pool-name hp-pooled-prod \
  --session-host-name "AVD-SH-01.domain.com" \
  --user-session-id 1 \
  --force
```

4. **Mark version 1.0.2 as excluded from latest:**

```bash
az sig image-version update \
  --resource-group rg-avd-prod-01 \
  --gallery-name gal-avd-prod-01 \
  --gallery-image-definition Win11_Multi_25H2_Gen2 \
  --gallery-image-version 1.0.2 \
  --exclude-from-latest true
```

**Recovery Time:** 2-4 hours (drain existing sessions, deploy replacement hosts).

### Long-Term Rollback (After 1+ Weeks)

**Scenario:** Version `1.0.2` was deployed 2 weeks ago. A subtle bug is discovered that corrupts user profiles.

**Challenge:** Can't simply delete hosts—users have established sessions and data.

**Process:**

1. **Root cause analysis:** Determine what's wrong with `1.0.2` (bad FSLogix agent version? Registry corruption?).

2. **Create hotfix version `1.0.3`:**
   - Fix the issue in Azure Image Builder template
   - Build and test `1.0.3` thoroughly
   - Deploy to staging/UAT for validation

3. **Gradual rollout of 1.0.3:**
   - Use drain mode strategy to replace hosts incrementally
   - Monitor closely to ensure fix resolves the issue

4. **Alternative: Emergency rollback to 1.0.1:**
   - If hotfix fails, revert to last known-good version `1.0.1`
   - Communicate to users: "We're rolling back to the previous version to resolve stability issues"
   - Accept data loss risk (any profiles saved on `1.0.2` hosts may be incompatible)

**Recovery Time:** 1-2 weeks for full rollout and validation.

### Preventing Rollback Scenarios

**Pilot Deployments:** Always deploy to 5-10% of users first. Catch issues before full rollout.

**Automated Testing:** Use Azure DevTest Labs to spin up test VMs from new image versions and run automated UI tests (Selenium, Playwright) to verify critical applications work.

**Staging Environment:** Maintain a `HP-Pooled-UAT` host pool that mirrors production. Deploy new versions here first.

**Canary Metrics:** Monitor key metrics during rollout:
- Login success rate (should be >99.5%)
- Application launch failures (track Event Viewer errors)
- Profile load times (should be <15 seconds)
- CPU/memory usage (detect performance regressions)

If any metric degrades >5% from baseline, halt rollout and investigate.

## Best Practices

**Always tag image versions with build metadata** - Include tags like `BuildDate: 2025-01-20`, `PatchTuesday: Jan-2025`, `GitCommitSHA: abc123`. This links deployed images back to source code and documentation, making troubleshooting much easier.

**Test in dev/UAT before production** - Never deploy directly to production, no matter how minor the change. Even PATCH versions can introduce regressions. Budget 1 week for UAT testing on PATCH, 2 weeks for MINOR, 4+ weeks for MAJOR.

**Use drain mode, not forced logoffs** - Forcing users off their sessions creates support tickets and data loss. Drain mode respects user workflows and reduces friction.

**Keep at least 2 previous versions available** - Storage costs for 2 extra image versions (~$10-20/month) are trivial compared to the cost of a failed rollback. Delete aggressively after N+2, but always keep a safety buffer.

**Document version contents** - Maintain a changelog (markdown file, Azure DevOps work items, or Bookstack page) listing what changed in each version. Example: "1.0.2: January 2025 Patch Tuesday (KB5034203), FSLogix 2.9.9001 hotfix, disabled Windows Search service."

**Automate patch version deployments** - Use Azure Image Builder + Azure DevOps to build and deploy PATCH versions automatically on Patch Tuesday +3 days. Requires mature testing/validation pipelines.

**Plan MAJOR version migrations as projects** - Annual feature updates (25H2 → 26H2) are projects, not maintenance tasks. Allocate 3-6 months, assign project manager, budget for testing resources.

**Monitor rollout metrics closely** - During rollout, check Azure Monitor dashboards hourly (first 24h), then daily. Set up alerts for login failure rate >2%, profile load time >30sec, or session host unavailable >10 min.

**Communicate changes to users** - Even minor updates can confuse users ("Why does my desktop look different?"). Send emails before MINOR/MAJOR rollouts explaining changes.

**Use semantic versioning consistently** - Don't deviate from MAJOR.MINOR.PATCH. Consistency helps everyone (admins, users, auditors) understand the scope of changes instantly.

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **New image version deploys but apps missing** | Sysprep removed app during generalization, or installation failed in build | Review Azure Image Builder logs. Check if app was in customizer list. Verify app supports multi-user mode (some apps break in generalized images). |
| **Session hosts fail to domain join after update** | Incorrect domain join credentials or domain controller unavailable | Verify domain join credentials in Key Vault. Check DNS resolution from session hosts to domain. Review Azure Activity Log for ARM deployment errors. |
| **Users report "This app can't run on your PC" after update** | Gen1 app deployed to Gen2 image (or vice versa) | Verify image definition is Gen2. Ensure all apps are 64-bit and Gen2-compatible. Rebuild image if architecture mismatch. |
| **Rollback fails: "Image version not found"** | Previous version was deleted due to retention policy | Always keep N-1 and N-2 versions. Implement lifecycle policy with minimum 3-version retention. If deleted, rebuild from Git template history. |
| **New version causes 50% slower login times** | Profile load issue, missing optimization, or network change | Compare VDOT settings between versions. Check FSLogix registry keys. Use Azure Monitor to identify bottleneck (disk I/O, CPU, network). |
| **Image version replication stuck at 50%** | Regional outage or storage throttling | Check Azure Service Health. Verify target region has available quota. Increase replica count or reduce target regions. |
| **Deployed hosts show "Agent not ready"** | AVD agent failed to install or register | RDP to session host, check `C:\Windows\Temp\ScriptLog.txt` for agent install errors. Verify host pool registration token is valid (expires after 27 days). |
| **Excluded version still deploying** | Deployment template hardcodes specific version instead of "latest" | Update ARM template to use `"version": "latest"` instead of `"version": "1.0.2"`. Alternatively, manually specify `1.0.1` in template. |
| **Cannot delete old image version** | VMs or scale sets still using version | Query all VMs: `az vm list --query "[?storageProfile.imageReference.exactVersion=='1.0.0']"`. Deallocate or delete dependent VMs first, then retry deletion. |
| **Patch version takes 4 hours to build** | Windows Update customizer downloading too many updates | Split into two builds: monthly cumulative update only (PATCH), feature updates quarterly (MINOR). Use `updateLimit: 20` to cap updates per build. |

## Next Steps

- **[[azure-compute-gallery]]** - Gallery structure and RBAC configuration
- **[[azure-image-builder]]** - Automating version builds with templates
- **[[multi-session-vs-single-session-windows|Multi vs Single Session]]** - Choosing the right image type for your workload