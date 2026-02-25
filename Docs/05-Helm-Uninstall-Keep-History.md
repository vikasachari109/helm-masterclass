# Helm Uninstall with Keep-History

## Overview
This module covers the Helm uninstall process with specific focus on the `--keep-history` flag. Proper uninstallation practices are crucial for maintaining deployment history, enabling rollback capabilities, and managing release lifecycle effectively.

## Core Concepts

### Uninstall Approaches

**Two Uninstall Methods:**

1. **With `--keep-history`:** Preserves release data for future rollback
2. **Without Flags:** Completely removes release and history

| Aspect | With --keep-history | Without Flags |
|--------|-------------------|---------------|
| Release Status | Uninstalled | Deleted |
| History Preserved | Yes | No |
| Rollback Possible | Yes | No |
| helm status | Works | Error: not found |
| helm history | Shows | Not available |
| Redeployment | Easy via rollback | Must reinstall |

---

## Part 1: Understanding Release Lifecycle

### Release States in Helm

```
INSTALLED → UPGRADED → FAILED → UNINSTALLED (with --keep-history)
   ↓         ↓           ↓          ↓
deployed  deployed    failed    uninstalled
  (active) (active)  (inactive)  (inactive)
```

**State Meanings:**
- **Deployed:** Active, running release
- **Uninstalled:** Deleted with history preserved
- **Uninstalledfall:** Deleted with no history
- **Failed:** Installation/upgrade failed
- **Superseded:** Replaced by newer revision

---

## Part 2: Uninstall with Keep-History

### Prerequisites
Assumes continuation from previous demos where releases were created and managed.

### List Existing Releases

```bash
# List all deployed releases
helm list
# Shows DEPLOYED status releases

# List only deployed releases explicitly
helm list --deployed
# Filters to active releases only

# Check release history before uninstall
helm history myapp101
# Output:
# REVISION   UPDATED              STATUS      CHART          DESCRIPTION
# 1          <timestamp>          superseded  mychart2-0.1.0 Install complete
# 2          <timestamp>          superseded  mychart2-0.2.0 Upgrade complete
# 3          <timestamp>          superseded  mychart2-0.3.0 Upgrade complete
# 4          <timestamp>          superseded  mychart2-0.4.0 Upgrade complete
# 5          <timestamp>          deployed    mychart2-0.3.0 Rollback to 3
```

### Uninstall with History Preservation

```bash
# Uninstall release preserving history
helm uninstall <RELEASE-NAME> --keep-history

# Example: Uninstall myapp101
helm uninstall myapp101 --keep-history

# Short form
helm uninstall myapp101 -k
```

**What Happens:**
1. Kubernetes resources are deleted (pods, services, deployments)
2. Release metadata is preserved in Kubernetes
3. Release history remains accessible
4. Release marked as "uninstalled" status

### Verify Uninstallation with History

```bash
# List uninstalled releases only
helm list --uninstalled
# Shows releases with uninstalled status

# Check all releases including uninstalled
helm list --all
# Shows both deployed and uninstalled

# Get status of uninstalled release
helm status myapp101
# Output:
# STATUS: uninstalled
# REVISION: 5
# NOTES: ...previous notes...
# Shows that release was uninstalled but data preserved

# View release history (still accessible)
helm history myapp101
# All revisions still show in history
# Last revision shows "Uninstall" action

# Get values from uninstalled release
helm get values myapp101
# Shows the values configuration used
```

**Uninstalled Status Details:**
- Release considered "inactive" but not deleted
- All history remains accessible
- Kubernetes resources removed
- Can be restored via rollback

---

## Part 3: Restore Uninstalled Release via Rollback

### Rollback Uninstalled Release

```bash
# View history to identify revision
helm history myapp101

# Rollback to any previous revision
helm rollback myapp101 <REVISION>

# Example: Restore to revision 3
helm rollback myapp101 3

# Rollback to most recent revision (last deployed)
helm rollback myapp101
# Automatically goes to last active revision before uninstall
```

**Rollback Behavior:**
1. Recreates Kubernetes resources from target revision
2. Restores configuration and manifests
3. Creates new pods with restored image
4. Service becomes accessible again
5. Creates new revision recording the rollback

### Verify Restored Release

```bash
# List releases after rollback
helm list
# myapp101 now shows as deployed again

# Check release status
helm status myapp101 --show-resources
# Shows restored resources

# Verify Kubernetes resources
kubectl get pods
kubectl get svc
# Shows pods and services for restored release

# Access restored application
http://localhost:31232
# Application accessible again

# View updated history
helm history myapp101
# Shows uninstall action followed by rollback
```

**Example History After Restore:**
```
REVISION   UPDATED              STATUS      CHART              DESCRIPTION
1          <timestamp>          superseded  mychart2-0.1.0     Install complete
2          <timestamp>          superseded  mychart2-0.2.0     Upgrade complete
3          <timestamp>          deployed    mychart2-0.3.0     Upgrade complete
4          <timestamp>          superseded  mychart2-0.4.0     Upgrade complete
5          <timestamp>          superseded  mychart2-0.3.0     Rollback to 3
6          <timestamp>          superseded  (none)             Uninstall complete
7          <timestamp>          deployed    mychart2-0.3.0     Rollback to 3
```

---

## Part 4: Uninstall Without History

### Permanent Uninstallation

```bash
# Uninstall without preservation (default)
helm uninstall <RELEASE-NAME>

# Example: Completely remove myapp101
helm uninstall myapp101

# This is equivalent to:
helm uninstall myapp101 --no-hooks
# Don't execute hooks during removal
```

**What Happens:**
1. Kubernetes resources deleted
2. Release metadata removed
3. History completely deleted
4. No rollback option available
5. Release cannot be recovered

### Verify Permanent Deletion

```bash
# Check if release appears in uninstalled list
helm list --uninstalled
# myapp101 NOT in output (completely gone)

# Try to get status of deleted release
helm status myapp101
# Error: release: not found
# Release completely removed from system

# Try to view history
helm history myapp101
# Error: release: not found
# History no longer available

# Kubernetes resources gone
kubectl get pods
kubectl get svc
# No resources remain from myapp101
```

---

## Part 5: Comparison and Decision Matrix

### When to Use --keep-history

**Use Cases for --keep-history:**
- Production deployments where rollback may be needed
- Temporary uninstalls that may be reinstated
- Audit/compliance requirements for deployment history
- Testing before final removal
- Maintaining deployment audit trail

**Command:**
```bash
helm uninstall myapp101 --keep-history
```

### When to Uninstall Completely

**Use Cases for Permanent Removal:**
- Development/testing installations
- Confirmed final removal of applications
- Resource cleanup for abandoned projects
- Avoiding confusion with outdated releases
- Complete environment reset

**Command:**
```bash
helm uninstall myapp101
```

---

## Part 6: Best Practices for Release Management

### Safe Uninstall Workflow

```bash
# Step 1: List and review release
helm list
helm status myapp101 --show-resources

# Step 2: Check release history (understand what's running)
helm history myapp101

# Step 3: Decision point
# Option A: Keep history (recommended for production)
helm uninstall myapp101 --keep-history

# Option B: Permanent removal (for dev/test)
helm uninstall myapp101
```

### Recovery Scenarios

**Scenario 1: Accidental Uninstall with History**
```bash
# Release uninstalled with --keep-history
# Check status
helm status myapp101  # Shows uninstalled

# Easy recovery
helm rollback myapp101
# Application restored

# Check history
helm history myapp101
# Shows recovery in history
```

**Scenario 2: Accidental Uninstall Without History**
```bash
# Release completely removed
helm status myapp101
# Error: release: not found

# Must reinstall from scratch
helm install myapp101 stacksimplify/mychart2 --version "0.3.0"
# Start over, history lost

# Lesson: Always use --keep-history in production
```

### Backup and Recovery Strategy

```bash
# Before uninstalling, export release info
helm get all myapp101 > myapp101-backup.yaml
helm history myapp101 > myapp101-history.txt

# Verify backup contains what you need
cat myapp101-backup.yaml

# Only then uninstall
helm uninstall myapp101 --keep-history
# Or even --keep-history provides protection
```

---

## Part 7: Management of Uninstalled Releases

### Listing and Filtering

```bash
# List only uninstalled releases
helm list --uninstalled

# List all releases (deployed + uninstalled)
helm list --all

# Count uninstalled releases
helm list --uninstalled --output json | jq '.[] | .status' | grep -c "uninstalled"

# Get uninstalled releases in namespace
helm list --uninstalled --namespace myapp

# Sort uninstalled by deletion date
helm list --uninstalled --sort date
```

### Cleanup Old Uninstalled Releases

```bash
# If historical burden becomes issue:
# Get uninstalled releases
helm list --uninstalled --output json > uninstalled.json

# Uninstall permanently (remove history)
helm uninstall myapp101  # Without --keep-history

# Only do this after confirming:
# 1. You have backed up configuration
# 2. You never need to restore
# 3. Your audit requirements are satisfied
```

---

## Reference: Helm Uninstall Flags

| Flag | Purpose | Default |
|------|---------|---------|
| `--keep-history, -k` | Keep release history | false |
| `--no-hooks` | Skip pre/post-delete hooks | false |
| `--wait` | Wait for resources to be deleted | false |
| `--timeout` | Timeout for deletion | 5m0s |

---

## Commands Summary

| Task | Command |
|------|---------|
| List deployed | `helm list --deployed` |
| List uninstalled | `helm list --uninstalled` |
| List all | `helm list --all` |
| View history | `helm history <RELEASE>` |
| Get status | `helm status <RELEASE>` |
| Uninstall with history | `helm uninstall <RELEASE> --keep-history` |
| Uninstall permanently | `helm uninstall <RELEASE>` |
| Restore release | `helm rollback <RELEASE>` |
| Get release data | `helm get all <RELEASE>` |

---

## Best Practice Recommendation

**For all production environments:**
```bash
# Always use --keep-history
helm uninstall myapp101 --keep-history

# Provides safety net for accidental removals
# Maintains complete audit trail
# Enables rapid recovery if needed
# Minimal storage overhead
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| "release not found" | Used direct uninstall, not --keep-history |
| Can't rollback | Release uninstalled without history |
| Old releases cluttering list | Use non-keep-history uninstall for dev/test only |
| Audit requirements failed | Ensure --keep-history on production |

### Debugging Commands

```bash
# Full release information
helm get all myapp101

# Raw release data
kubectl get secret -l owner=helm -o jsonpath='{.items[?(@.metadata.name=="myapp101.v5")].data.release}'

# Search for uninstalled
helm list --all --output json | grep uninstalled
```

---

## Summary

Key points about uninstall with --keep-history:

✅ **Preserves deployment history**
✅ **Enables rollback recovery**
✅ **Maintains audit trail**
✅ **Minimal storage impact**
✅ **Safe for production**
✅ **Recommended best practice**

---

## References

- [Helm Uninstall Documentation](https://helm.sh/docs/helm/helm_uninstall/)
- [Helm Rollback Documentation](https://helm.sh/docs/helm/helm_rollback/)
- [Release Management Best Practices](https://helm.sh/docs/chart_best_practices/versioning/)
