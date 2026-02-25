# Helm Upgrade with Chart Versions and Rollback

## Overview
This module covers advanced Helm upgrade scenarios using specific chart versions and the rollback mechanism. You'll learn how to manage multiple chart versions in production, perform targeted upgrades, and recover from failed deployments using Helm's rollback functionality.

## Core Concepts

### Chart Versions vs Application Versions
- **Chart Version:** Version of the Helm chart package itself
- **Application Version:** Version of the application inside the chart
- **Independent Versioning:** Chart and app versions may differ
- **Backwards Compatibility:** Charts may support multiple app versions

### Understanding Versioning

```
Chart Version → App Version Mapping:
0.1.0 → 1.0.0 (Initial chart for app v1)
0.2.0 → 2.0.0 (Updated chart for app v2)
0.3.0 → 3.0.0 (Enhanced chart for app v3)
0.4.0 → 4.0.0 (Latest chart for app v4)
```

---

## Part 1: Chart Repository and Version Discovery

### Reference Resources

**StackSimplify Helm Charts:**
- [GitHub Repository](https://github.com/stacksimplify/helm-charts)
- [mychart2 Documentation](https://artifacthub.io/packages/helm/stacksimplify/mychart2/)

**mychart2 Version Mapping:**

| Chart Version | App Version | Description |
|---------------|-------------|-------------|
| 0.1.0 | 1.0.0 | Initial release |
| 0.2.0 | 2.0.0 | Feature updates |
| 0.3.0 | 3.0.0 | Enhanced functionality |
| 0.4.0 | 4.0.0 | Latest features |

### Search Repository for All Versions

```bash
# Search for latest chart version only
helm search repo mychart2
# Output shows only the latest version available

# Display all available chart versions
helm search repo mychart2 --versions
# Output:
# NAME                   CHART VERSION   APP VERSION   DESCRIPTION
# stacksimplify/mychart2  0.4.0          4.0.0        Kubernetes app
# stacksimplify/mychart2  0.3.0          3.0.0        Kubernetes app
# stacksimplify/mychart2  0.2.0          2.0.0        Kubernetes app
# stacksimplify/mychart2  0.1.0          1.0.0        Kubernetes app

# Search for specific chart version
helm search repo mychart2 --version "0.2.0"
# Shows only version 0.2.0 if available

# Search with detailed information
helm search repo mychart2 --versions --detailed
# Provides extended information for all versions
```

**Search Options:**

| Flag | Purpose |
|------|---------|
| `--versions` | Show all available chart versions |
| `--version "X.Y.Z"` | Show specific chart version |
| `--detailed` | Extended information output |
| `--max-cols N` | Limit output columns |
| `--devel` | Include development versions |

---

## Part 2: Installation with Specific Chart Versions

### Install Exact Chart Version

```bash
# Install chart specifying exact version
helm install <RELEASE-NAME> <REPO>/<CHART> --version "VERSION"

# Example: Install chart version 0.1.0
helm install myapp101 stacksimplify/mychart2 --version "0.1.0"
```

**Installation Details:**
- Specifies exact chart version to deploy
- Useful for reproducible deployments
- Required for regression testing
- Avoids accidental latest version deployment

### Verify Installation

```bash
# List installed releases
helm list
# Shows: myapp101, with deployed status

# View release status with resources
helm status myapp101 --show-resources
# Displays:
# - Release NAME and STATUS
# - Deployed resources (pods, services, deployments)
# - Release notes from chart

# Detailed helm list output
helm list --detailed
# More information about release

# Get installed chart version
helm get all myapp101 | grep -i version
# Shows which chart version was used
```

**Status Output Breakdown:**
```yaml
STATUS: deployed           # Current release state
REVISION: 1               # Number of updates (1 = fresh install)
CHART: mychart2-0.1.0     # Chart name and version
APP VERSION: 1.0.0        # Application version
```

### Access Deployed Application

```bash
# List pods and services
kubectl get pods
kubectl get svc

# Access application on NodePort
http://localhost:31232
# Port comes from Kubernetes service definition

# View pod logs to verify version
kubectl get pods
# Output: myapp101-xxx-yyy

kubectl logs <POD-NAME>
# Logs from application version 1.0.0

# Check running application version
kubectl exec <POD-NAME> -- app --version
# Or check via web interface
```

---

## Part 3: Upgrade with Specific Chart Versions

### Targeted Version Upgrade

```bash
# Upgrade to specific chart version (not latest)
helm upgrade <RELEASE-NAME> <REPO>/<CHART> --version "VERSION"

# Example: Upgrade from 0.1.0 to 0.2.0
helm upgrade myapp101 stacksimplify/mychart2 --version "0.2.0"
```

**What Happens:**
1. Helm fetches specified chart version
2. Renders templates with current values
3. Updates Kubernetes resources
4. New pods created with app version 2.0.0
5. Release revision incremented to 2

### Verify Targeted Upgrade

```bash
# Check release status after upgrade
helm list
# Observation: Revision increases to 2

# View detailed status with resources
helm status myapp101 --show-resources
# Shows updated resources with new chart version

# Access application
http://localhost:31232
# Should display application version 2.0.0

# View complete upgrade history
helm history myapp101
# Output:
# REVISION   UPDATED                    STATUS      CHART              DESCRIPTION
# 1          <timestamp>                superseded  mychart2-0.1.0     Install complete
# 2          <timestamp>                deployed    mychart2-0.2.0     Upgrade complete
```

### Multiple Version Upgrades

```bash
# Upgrade 0.1.0 → 0.2.0
helm upgrade myapp101 stacksimplify/mychart2 --version "0.2.0"

# Upgrade 0.2.0 → 0.3.0
helm upgrade myapp101 stacksimplify/mychart2 --version "0.3.0"

# Upgrade 0.3.0 → 0.4.0
helm upgrade myapp101 stacksimplify/mychart2 --version "0.4.0"

# Check complete history
helm history myapp101
# Shows all 4 revisions
```

---

## Part 4: Upgrade to Latest Chart Version

### Upgrade Without Version Specification

```bash
# Upgrade to latest chart version (default behavior)
helm upgrade myapp101 stacksimplify/mychart2
# No --version flag specified

# This will deploy:
# - Chart: 0.4.0 (latest)
# - App Version: 4.0.0 (latest)
```

**Behavior:**
- Uses latest available chart version
- Latest may include breaking changes
- No explicit version control
- Useful for keeping current but risky for production

### Verify Latest Version Upgrade

```bash
# List release status
helm list
# Shows highest revision number

# Check what version was deployed
helm status myapp101 --show-resources
# Display shows Chart: mychart2-0.4.0

# Full history shows progression
helm history myapp101
# Output:
# REVISION   CHART                STATUS
# 1          mychart2-0.1.0      superseded
# 2          mychart2-0.2.0      superseded
# 3          mychart2-0.3.0      superseded
# 4          mychart2-0.4.0      deployed

# Access latest version
http://localhost:31232
# Shows app version 4.0.0

# Verify in pod logs
kubectl logs <POD-NAME>
# Logs from version 4.0.0
```

---

## Part 5: Helm Rollback Mechanism

### Understanding Rollback

**Rollback Purpose:**
- Recover from failed deployments
- Quick recovery from breaking changes
- Version testing and validation
- Production incident response

**Rollback Features:**
- Non-destructive (preserves data)
- Instant version reversion
- Complete history preservation
- Conditional rollback scenarios

### Rollback to Previous Revision

```bash
# Rollback to immediately previous revision
helm rollback <RELEASE-NAME>

# Example: Rollback from revision 4 to revision 3
helm rollback myapp101
# Automatically goes to previous working version
```

**Rollback Behavior:**
1. Identifies previous successful revision
2. Recreates resources from that revision
3. Pods are updated with old image
4. Service continues running
5. New revision created recording the rollback

### Rollback to Specific Revision

```bash
# Rollback to any specific revision number
helm rollback <RELEASE-NAME> <REVISION-NUMBER>

# Example: Rollback all the way to initial version
helm rollback myapp101 1
# Reverts to Revision 1 (chart 0.1.0, app 1.0.0)

# Return to Revision 2
helm rollback myapp101 2
# Reverts from current to Revision 2 (chart 0.2.0, app 2.0.0)
```

**Revision Numbers:**
- Start at 1 for initial installation
- Increment with each upgrade
- Persist after rollback
- Traceable in history

### Verify Rollback

```bash
# Check current release state
helm list
# Revision increases again (rollback creates new revision)

# View complete history including rollback
helm history myapp101
# Output shows original revisions plus rollback entry:
# REVISION   UPDATED                    STATUS      CHART              DESCRIPTION
# 1          <timestamp>                superseded  mychart2-0.1.0     Install complete
# 2          <timestamp>                superseded  mychart2-0.2.0     Upgrade complete
# 3          <timestamp>                superseded  mychart2-0.3.0     Upgrade complete
# 4          <timestamp>                superseded  mychart2-0.4.0     Upgrade complete
# 5          <timestamp>                deployed    mychart2-0.3.0     Rollback to 3

# Get rolled-back version info
helm get values myapp101
# Shows values from previous revision

# Access application (should be previous version)
http://localhost:31232
# Should display original version

# Check pod recreation
kubectl get pods
# Shows new pods with old image
```

---

## Part 6: Rollback Use Cases

### Scenario 1: Failed Deployment

```bash
# Upgrade causes issues
helm upgrade myapp101 stacksimplify/mychart2 --version "0.4.0"
# App crashes or shows errors

# Detect failure
http://localhost:31232  # Error page or timeout

# Immediate rollback
helm rollback myapp101
# Returns to working Revision 3

# Verify recovery
http://localhost:31232  # Back to working state
```

### Scenario 2: Testing New Version

```bash
# Upgrade to test new version
helm upgrade myapp101 stacksimplify/mychart2 --version "0.4.0"

# Run tests
kubectl exec <POD-NAME> -- test-suite

# If tests fail, rollback
helm rollback myapp101

# Maintain history for analysis
helm history myapp101
# Shows test attempts and outcomes
```

### Scenario 3: Rolling Back Multiple Revisions

```bash
# Go back two versions at once
helm rollback myapp101 2
# Jumps from revision 4 directly to revision 2

# Verify target reached
helm list
# Shows current revision as 5 (rollback from 4)

# Check history
helm history myapp101
# Linear progression visible
```

---

## Part 7: Best Practices for Version Management

### Safe Upgrade Practices

```bash
# 1. Always preview before upgrading
helm status myapp101 --show-resources
# Review current deployment

# 2. Test in development first
helm install test-myapp101 stacksimplify/mychart2 \
  --version "0.4.0" \
  --namespace dev

# 3. Use specific versions in production
helm upgrade myapp101 stacksimplify/mychart2 \
  --version "0.4.0"
# Never upgrade without specifying version

# 4. Keep release history
helm list  # Shows revision history
helm history myapp101  # Complete version history

# 5. Document version changes
# Use release notes during deployments
helm get all myapp101 | grep -A 5 "NOTES:"
```

### Version Strategy

| Strategy | Best For | Command |
|----------|----------|---------|
| Specific Version | Production | `--version "0.4.0"` |
| Latest | Development | (default, no --version) |
| Pinned | Testing | Document version used |
| Range | Patch updates | Use compatible versions |

### Monitoring Version Changes

```bash
# Track all upgrades
helm history myapp101 --max 20
# Shows last 20 changes

# Export history for audit
helm history myapp101 --output json > version-history.json
# Audit trail for compliance

# Compare revisions
helm get values myapp101 --revision 1 > v1.yaml
helm get values myapp101 --revision 2 > v2.yaml
diff v1.yaml v2.yaml
# See what changed between versions
```

---

## Helm Commands Reference

| Command | Purpose |
|---------|---------|
| `helm search repo <CHART> --versions` | List all chart versions |
| `helm search repo <CHART> --version "X.Y.Z"` | Search specific version |
| `helm install ... --version "X.Y.Z"` | Install with exact version |
| `helm upgrade ... --version "X.Y.Z"` | Upgrade to specific version |
| `helm upgrade ... ` (no --version) | Upgrade to latest version |
| `helm rollback <RELEASE>` | Rollback to previous revision |
| `helm rollback <RELEASE> <REVISION>` | Rollback to specific revision |
| `helm history <RELEASE>` | View all revisions |
| `helm status <RELEASE>` | Check current release |
| `helm get all <RELEASE>` | View release configuration |

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Chart version not found | Verify version exists: `helm search repo <CHART> --versions` |
| Rollback shows same revision | Use explicit revision number: `helm rollback myapp101 1` |
| History shows only current | Use `--max 100` to see more: `helm history myapp101 --max 100` |
| Rollback didn't work | Check rollback command succeeded, verify pods updated |

### Debugging Commands

```bash
# Detailed rollback information
helm rollback myapp101 --dry-run --debug

# Complete history with timestamps
helm history myapp101 --output yaml

# Monitor rollback progress
kubectl get pods --watch

# Check pod image used
kubectl get pods -o wide
```

---

## Summary

This module covers:
- 🔍 **Version Discovery:** Finding available chart versions
- 📦 **Targeted Installation:** Install specific chart versions
- ⬆️ **Controlled Upgrades:** Upgrade to specific versions
- 🔄 **Latest Updates:** Upgrade to newest version
- ⏮️ **Rollback Capability:** Revert to any previous version
- 📊 **Version Tracking:** Complete deployment history
- ✅ **Best Practices:** Safe production deployment strategies

---

## References

- [Helm Upgrade Documentation](https://helm.sh/docs/helm/helm_upgrade/)
- [Helm Rollback Documentation](https://helm.sh/docs/helm/helm_rollback/)
- [Helm History Documentation](https://helm.sh/docs/helm/helm_history/)
- [Semantic Versioning](https://semver.org/)
