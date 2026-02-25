# Helm Upgrade with --set Option

## Overview
This module covers the Helm upgrade process, specifically using the `helm upgrade` command combined with the `--set` flag to modify release configurations. Helm upgrades allow you to update deployed applications without uninstalling and reinstalling them, preserving release history and enabling rollback capabilities.

## Core Upgrade Concepts

### Why Use Helm Upgrade?
- **Non-destructive Updates:** Preserves persistent data and release history
- **Rollback Capability:** Can revert to previous versions if upgrade fails
- **Configuration Changes:** Update deployment without full reinstallation
- **Image Updates:** Deploy new application versions seamlessly
- **Release Tracking:** Maintains complete version history

### Upgrade vs Reinstall
| Aspect | Upgrade | Reinstall |
|--------|---------|-----------|
| Data Preservation | Preserves PV data | May lose data |
| History | Tracked with revision | Lost |
| Rollback | Easy rollback available | Not possible |
| Service Continuity | Maintains service | Service interruption |
| Use Case | Production apps | Development/testing |

---

## Part 1: Understanding Chart Repositories

### Custom Helm Repository

For this demonstration, we use StackSimplify's custom Helm repository hosting pre-built charts.

**Repository Details:**
- **Repository URL:** [https://stacksimplify.github.io/helm-charts/](https://stacksimplify.github.io/helm-charts/)
- **GitHub Source:** [stacksimplify/helm-charts](https://github.com/stacksimplify/helm-charts)
- **ArtifactHub:** Search for "stacksimplify" on [artifacthub.io](https://artifacthub.io)
- **Sample Chart:** [mychart1 on ArtifactHub](https://artifacthub.io/packages/helm/stacksimplify/mychart1)

### Add Custom Repository

```bash
# List existing repositories
helm repo list
# Output shows all currently configured repositories

# Add custom Helm repository with local alias
helm repo add <LOCAL-NAME> <REPOSITORY-URL>

# Example: Add StackSimplify repository
helm repo add stacksimplify https://stacksimplify.github.io/helm-charts/

# Verify addition
helm repo list
# Shows new "stacksimplify" entry in list

# Update repositories to get latest charts
helm repo update
```

### Search Custom Repository

```bash
# Search for specific chart
helm search repo mychart1
# Output shows chart versions available

# Detailed chart search
helm search repo mychart1 --detailed

# Search in specific repository only
helm search repo stacksimplify/
# Lists all charts in stacksimplify repository

# View all chart versions
helm search repo mychart1 --versions
```

---

## Part 2: Initial Installation

### Install Chart from Custom Repository

```bash
# Install chart with release name
helm install <RELEASE-NAME> <REPO-NAME>/<CHART-NAME>

# Example: Install mychart1 from stacksimplify repo
helm install myapp1 stacksimplify/mychart1

# Install with specific namespace
helm install myapp1 stacksimplify/mychart1 --namespace myapp --create-namespace

# Install with dry run (preview before actual installation)
helm install myapp1 stacksimplify/mychart1 --dry-run --debug
```

**Installation Steps:**
1. Helm downloads chart from repository
2. Renders Kubernetes manifests from templates
3. Creates resources in Kubernetes cluster
4. Records release information for tracking

### Verify Initial Installation

```bash
# List installed releases
helm list
# Output shows: NAME, NAMESPACE, REVISION, STATUS, CHART, APP VERSION, UPDATED, STATUS

# Check pods created
kubectl get pods
# Shows pods deployed by the chart

# Check services created
kubectl get svc
# Shows service with NodePort (e.g., 31231)

# Get release information
helm get values myapp1
# Shows values used during installation

# View deployed manifests
helm get manifest myapp1
# Shows actual Kubernetes resources created

# Access application via NodePort
http://localhost:31231
# Or use IP:NodePort format
http://127.0.0.1:31231
```

---

## Part 3: Understanding Application Versions

### Docker Image Versions

The sample application (kubenginx) has multiple versions available:

**Available Image Tags:**
- **Tag 1.0.0:** Initial version
- **Tag 2.0.0:** Updated version with improvements
- **Tag 3.0.0:** Advanced features
- **Tag 4.0.0:** Latest features

**Docker Image Reference:**
[StackSimplify kubenginx Container Images](https://github.com/users/stacksimplify/packages/container/package/kubenginx)

**Full Image URLs:**
```
ghcr.io/stacksimplify/kubenginx:1.0.0
ghcr.io/stacksimplify/kubenginx:2.0.0
ghcr.io/stacksimplify/kubenginx:3.0.0
ghcr.io/stacksimplify/kubenginx:4.0.0
```

### View Chart Values

```bash
# Check current image configuration
helm get values myapp1

# Check what values can be overridden
helm show values stacksimplify/mychart1

# Look for image section in values
# Typically looks like:
# image:
#   repository: ghcr.io/stacksimplify/kubenginx
#   tag: 1.0.0
#   pullPolicy: IfNotPresent
```

---

## Part 4: Helm Upgrade Process

### Basic Upgrade Command

```bash
# Upgrade chart with new configuration
helm upgrade <RELEASE-NAME> <REPO-NAME>/<CHART-NAME> --set <KEY=VALUE>

# Example: Upgrade to version 2.0.0
helm upgrade myapp1 stacksimplify/mychart1 --set "image.tag=2.0.0"
```

**Key Points:**
- Release name (myapp1) must match existing release
- Repository path must be valid (can be different from install)
- --set overrides specific values from values.yaml
- Previous configuration is preserved unless explicitly overridden

### Multiple Value Overrides

```bash
# Upgrade with multiple --set options
helm upgrade myapp1 stacksimplify/mychart1 \
  --set "image.tag=2.0.0" \
  --set "replicaCount=3" \
  --set "service.type=LoadBalancer"

# Upgrade with values file
helm upgrade myapp1 stacksimplify/mychart1 -f custom-values.yaml

# Combine file and --set options
helm upgrade myapp1 stacksimplify/mychart1 \
  -f custom-values.yaml \
  --set "image.tag=2.0.0"
```

### Upgrade Options

```bash
# Dry run to preview upgrade
helm upgrade myapp1 stacksimplify/mychart1 \
  --set "image.tag=2.0.0" \
  --dry-run --debug

# Upgrade with history tracking
helm upgrade myapp1 stacksimplify/mychart1 --set "image.tag=2.0.0"
# Automatically records as Revision 2

# Force upgrade (replace existing resources)
helm upgrade myapp1 stacksimplify/mychart1 \
  --set "image.tag=2.0.0" \
  --force

# Upgrade with cleanup on failure
helm upgrade myapp1 stacksimplify/mychart1 \
  --set "image.tag=2.0.0" \
  --cleanup-on-fail
```

### Upgrade Behavior

```bash
# Key facts during upgrade:
# 1. Kubernetes performs rolling update (gradual replacement)
# 2. Service remains accessible during update
# 3. Previous resources are updated in-place
# 4. No data loss occurs
# 5. New pods are created with new image
# 6. Old pods are terminated
```

---

## Part 5: Post-Upgrade Verification

### Check Release Status

```bash
# List all releases
helm list
# Observation: Revision number increases (e.g., 2, 3, 4)

# List deployed releases only
helm list --deployed
# Shows only successfully deployed releases

# List with detailed information
helm list --detailed
# Shows more columns including description

# Show superseded releases (old versions kept with --keep-history)
helm list --superseded
# Displays releases that have been replaced by newer versions
```

### Verify Release History

```bash
# View complete release history
helm history myapp1
# Output shows all revisions:
# REVISION   UPDATED TIME              STATUS      CHART          DESCRIPTION
# 1          <timestamp>               superseded  mychart1-1.0   Install complete
# 2          <timestamp>               deployed    mychart1-1.0   Upgrade complete

# View specific revision status
helm status myapp1
# Shows current deployment status

# Get values from specific revision
helm get values myapp1 --revision=1
# Shows values used during revision 1
```

### Verify Kubernetes Resources

```bash
# Check updated pods
kubectl get pods
# Shows new pod(s) with updated image
# Old pods are being terminated

# Detailed pod information
kubectl get pods --wide
# Shows pod names, images, nodes

# Describe specific pod to see update details
kubectl describe pod <POD-NAME>
# Look for "Events" section showing:
# - Image pull events: "ghcr.io/stacksimplify/kubenginx:2.0.0 pulled"
# - Container creation events
# - Pod startup events

# View pod logs for new version
kubectl logs <POD-NAME>
# New application logs from updated version

# Check deployment status
kubectl get deploy
# Shows deployment with updated image
```

### Verify Application Update

```bash
# Access application in browser
http://localhost:31231
# Should display Version 2 (or new version number)
# Application changes may be visible in UI

# Alternative access methods
curl http://localhost:31231
# Returns HTML from new application version

kubectl port-forward svc/myapp1 9000:8080
# Port forward to new version on localhost:9000

# Check service endpoints
kubectl get endpoints myapp1
# Shows backend pods serving the service
```

---

## Part 6: Sequential Upgrades

### Performing Multiple Upgrades

```bash
# Upgrade to Version 3.0.0
helm upgrade myapp1 stacksimplify/mychart1 --set "image.tag=3.0.0"

# Verify upgrade
helm list
# Revision should be 3

# Access application
http://localhost:31231
# Should display Version 3

# Upgrade to Version 4.0.0
helm upgrade myapp1 stacksimplify/mychart1 --set "image.tag=4.0.0"

# Verify
helm list
# Revision should be 4

# Check history of all upgrades
helm history myapp1
# Shows complete upgrade trail
```

### Upgrade Scenarios

```bash
# Scenario 1: Bug fix in version
helm upgrade myapp1 stacksimplify/mychart1 --set "image.tag=2.0.1"

# Scenario 2: Major feature release
helm upgrade myapp1 stacksimplify/mychart1 --set "image.tag=3.0.0"

# Scenario 3: Configuration change with same image
helm upgrade myapp1 stacksimplify/mychart1 \
  --set "image.tag=2.0.0" \
  --set "replicaCount=5"

# Scenario 4: Rollback to previous version
helm rollback myapp1 1
# Reverts to Revision 1
```

---

## Part 7: Upgrade Patterns and Best Practices

### Safe Upgrade Practices

```bash
# 1. Always dry run first
helm upgrade myapp1 stacksimplify/mychart1 \
  --set "image.tag=2.0.0" \
  --dry-run --debug

# 2. Test in development first
helm upgrade myapp1 stacksimplify/mychart1 \
  --set "image.tag=2.0.0" \
  --namespace dev

# 3. Upgrade with cleanup on failure
helm upgrade myapp1 stacksimplify/mychart1 \
  --set "image.tag=2.0.0" \
  --cleanup-on-fail

# 4. Keep upgrade history for rollback
helm upgrade myapp1 stacksimplify/mychart1 --set "image.tag=2.0.0"
# Default behavior keeps history
```

### Monitoring Upgrades

```bash
# Watch pod update in real-time
kubectl get pods --watch
# Shows pod replacement as they happen

# Monitor deployment rollout
kubectl rollout status deployment/myapp1
# Shows progress of update

# Check events for troubleshooting
kubectl get events --sort-by='.lastTimestamp'
# Recent cluster events

# View pod creation timeline
kubectl get pods --sort-by=.metadata.creationTimestamp
# Shows pods by creation time
```

---

## Helm Upgrade Commands Reference

| Command | Purpose |
|---------|---------|
| `helm upgrade <RELEASE> <CHART>` | Basic upgrade |
| `helm upgrade <RELEASE> <CHART> --set KEY=VALUE` | Upgrade with value override |
| `helm upgrade --dry-run` | Preview upgrade without applying |
| `helm upgrade --debug` | Detailed debug output |
| `helm upgrade --force` | Force recreate resources |
| `helm history <RELEASE>` | View upgrade history |
| `helm status <RELEASE>` | Current release status |
| `helm rollback <RELEASE>` | Revert to previous version |
| `helm get values <RELEASE>` | View current values |
| `helm get manifest <RELEASE>` | View deployed manifests |

---

## Troubleshooting Upgrades

### Common Issues

| Issue | Solution |
|-------|----------|
| Image pull fails | Check image exists, verify registry credentials |
| Pods don't update | Check selector labels match, watch pod events |
| Upgrade hangs | Check readiness probes, view pod logs |
| Service unavailable | Verify load balancer, check service selector |
| Resource conflicts | Use --force flag or delete old resources |

### Debugging Commands

```bash
# View upgrade attempts
helm history myapp1 --max=10

# Check what changed
helm diff upgrade myapp1 stacksimplify/mychart1 --set "image.tag=2.0.0"
# (requires helm-diff plugin)

# Detailed pod description
kubectl describe pod <POD-NAME>

# Pod events for troubleshooting
kubectl get events --field-selector involvedObject.name=<POD-NAME>
```

---

## Summary

The Helm upgrade process enables:
- 🔄 **Non-destructive Updates:** Change configurations without data loss
- 📊 **Version Tracking:** Complete history of all changes
- ⏮️ **Rollback Capability:** Revert to any previous version
- 🔐 **Data Preservation:** Persistent data remains intact
- 🚀 **Production-Ready:** Designed for stable, running applications

---

## Next Steps

- Master configuration value management (Part 2: Override Values)
- Learn chart versioning strategies
- Explore automatic upgrade triggers
- Implement update notifications
- Study rollback procedures

---

## References

- [Helm Upgrade Documentation](https://helm.sh/docs/helm/helm_upgrade/)
- [Helm Values Management](https://helm.sh/docs/chart_template_engine/values/)
- [Kubernetes Rolling Updates](https://kubernetes.io/docs/tutorials/k8s201/rolling-update/)
- [Release Management Best Practices](https://helm.sh/docs/intro/best_practices/)
