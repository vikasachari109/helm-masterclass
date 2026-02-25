# Helm Override Default Values and Value Hierarchy

## Overview
This module covers how to override Helm chart values, which is fundamental to customizing deployments. Values provide configuration parameters that customize Helm charts without modifying the chart itself. Understanding value hierarchy and override mechanisms is crucial for managing multiple environments and configurations.

## Core Concepts

### What Are Helm Values?

**Values Definition:**
- Configuration parameters for Helm charts
- Stored in `values.yaml` file in chart
- Can be overridden at install/upgrade time
- Support various data types (strings, numbers, lists, objects)
- Enable chart reusability across environments

### Value Hierarchy (Priority Order)

```
Priority (High to Low):
1. Command-line --set flags (highest)
2. Values from -f file
3. Parent chart values
4. Default values.yaml (lowest)
```

**Example:**
```bash
# If default is port: 8080
# And values-custom.yaml has port: 9000
# And --set port=8888

# Final result: port=8888 (--set wins)
```

---

## Part 1: Understanding Values Files

### Default values.yaml in Chart

**Reference:** [mychart1 values.yaml](https://github.com/stacksimplify/helm-charts/blob/main/mychart1/values.yaml)

**Typical Structure:**
```yaml
# Number of replicas
replicaCount: 1

# Image configuration
image:
  repository: ghcr.io/stacksimplify/kubenginx
  pullPolicy: IfNotPresent
  tag: ""  # Overrides the chart appVersion

# Service configuration
service:
  type: NodePort
  port: 80
  nodePort: 31231
```

### Viewing Default Values

```bash
# View default values from chart (before install)
helm show values stacksimplify/mychart1

# Output shows all configurable parameters:
# image:
#   repository: ghcr.io/stacksimplify/kubenginx
#   pullPolicy: IfNotPresent
#   tag: ""
# service:
#   nodePort: 31231
#   port: 80
#   type: NodePort
```

---

## Part 2: Override with --set Flag

### Basic --set Override

**Syntax:**
```bash
helm install <RELEASE> <CHART> --set KEY=VALUE
```

**Example: Override NodePort**
```bash
# Install with custom NodePort
helm install myapp901 stacksimplify/mychart1 \
  --set service.nodePort=31240

# Default: 31231
# Overridden to: 31240
```

### Multiple --set Overrides

```bash
# Override multiple values
helm install myapp901 stacksimplify/mychart1 \
  --set service.nodePort=31240 \
  --set replicaCount=3 \
  --set image.tag=2.0.0

# Results:
# - nodePort: 31240 (custom)
# - replicaCount: 3 (custom)
# - image.tag: 2.0.0 (custom)
# - image.repository: ghcr.io/stacksimplify/kubenginx (default)
```

### Nested Value Override

```bash
# Override nested values using dot notation
helm install myapp901 stacksimplify/mychart1 \
  --set service.nodePort=31240 \
  --set image.pullPolicy=Always \
  --set image.tag=3.0.0

# Format: parent.child=value
# Works through any nesting level
```

---

## Part 3: Dry-Run and Debug Flags

### Dry-Run Installation

**Purpose:** Preview what would be installed without actually deploying

```bash
# Preview installation without applying
helm install myapp901 stacksimplify/mychart1 \
  --set service.nodePort=31240 \
  --dry-run

# Shows:
# - Release name: myapp901
# - Status: pending-install
# - Rendered manifests (not applied to cluster)
# - Does NOT create actual resources
```

### Dry-Run with Debug

**Purpose:** Detailed preview with full debugging information

```bash
# Dry run with detailed output
helm install myapp901 stacksimplify/mychart1 \
  --set service.nodePort=31240 \
  --dry-run --debug

# Output includes:
# NAME: myapp901
# NAMESPACE: default
# STATUS: pending-install
# REVISION: 1
# USER-SUPPLIED VALUES:
# service:
#   nodePort: 31240
# COMPUTED VALUES: (all values after merging)
# fullnameOverride: ""
# image:
#   pullPolicy: IfNotPresent
#   repository: ghcr.io/stacksimplify/kubenginx
#   tag: ""
# service:
#   nodePort: 31240
#   port: 80
#   type: NodePort

# MANIFESTS:
# ---
# # Full Kubernetes manifests that would be created
```

### Workflow: Preview Before Install

```bash
# Step 1: Preview with --dry-run --debug
helm install myapp901 stacksimplify/mychart1 \
  --set service.nodePort=31240 \
  --dry-run --debug | less

# Review output carefully:
# - Check rendered service with port 31240
# - Check image configuration
# - Check deployment: strategy, replicas

# Step 2: Looks good? Actually install
helm install myapp901 stacksimplify/mychart1 \
  --set service.nodePort=31240

# Step 3: Verify
helm status myapp901 --show-resources
```

---

## Part 4: Install with Values File

### Create Custom Values File

**File: myvalues.yaml**
```yaml
# Custom configuration overrides

# Change-1: increase replicas from 1 to 2
replicaCount: 2

# Change-2: set image tag to 2.0.0
image:
  repository: ghcr.io/stacksimplify/kubenginx
  pullPolicy: IfNotPresent
  tag: "2.0.0"

# Change-3: change nodePort to 31250
service:
  type: NodePort
  port: 80
  nodePort: 31250
```

### Install with Values File

```bash
# Install using values file
helm install myapp902 stacksimplify/mychart1 -f myvalues.yaml

# File path options:
helm install myapp902 stacksimplify/mychart1 -f ./myvalues.yaml
helm install myapp902 stacksimplify/mychart1 --values myvalues.yaml

# Multiple values files (merge order, later overrides earlier):
helm install myapp902 stacksimplify/mychart1 \
  -f values-base.yaml \
  -f values-prod.yaml
```

### Verify File-Based Installation

```bash
# Check deployed values
helm get values myapp902
# Shows only overridden values from myvalues.yaml

# Get all values (defaults + overrides)
helm get values myapp902 --all
# Shows complete configuration

# View manifests
helm get manifest myapp902
# Shows actual Kubernetes resources created with values applied

# Check pod details
kubectl get pods -l app=mychart1
# Shows 2 replicas (from myvalues.yaml)

# Check service
kubectl get svc myapp902-mychart1
# Shows NodePort: 31250 (from myvalues.yaml)

# Access application
http://localhost:31250
# Uses custom port from values file
```

---

## Part 5: Value Hierarchy and Precedence

### Combining --set and -f Options

**Hierarchy (High to Low Priority):**
1. `--set` command-line flags (highest priority)
2. `-f` values files (later files override earlier)
3. Chart default values.yaml (lowest priority)

```bash
# Example with hierarchy:
helm install myapp902 stacksimplify/mychart1 \
  -f myvalues.yaml \
  --set service.nodePort=31999

# Result:
# From myvalues.yaml:
#   replicaCount: 2
#   image.tag: 2.0.0
#   service.nodePort: 31250
# Overridden by --set:
#   service.nodePort: 31999 (--set wins)
#
# Final configuration:
#   replicaCount: 2
#   image.tag: 2.0.0
#   service.nodePort: 31999
```

### Multiple Value Files

```bash
# Multiple files merge in order (later overrides earlier)
helm install myapp902 stacksimplify/mychart1 \
  -f values-base.yaml \
  -f values-prod.yaml \
  -f values-secrets.yaml

# values-base.yaml applied first
# values-prod.yaml overrides base
# values-secrets.yaml overrides prod
# All overridden by --set if provided
```

---

## Part 6: Retrieve and Compare Values

### Get Installed Values

```bash
# View values used for current release
helm get values myapp902
# Shows ONLY overridden values from install

# Get all values (defaults + overrides)
helm get values myapp902 --all
# Shows complete configuration as Helm sees it

# Get values from specific revision
helm get values myapp902 --revision 1
# Useful when tracking changes across upgrades

# Output as JSON
helm get values myapp902 --all --output json
# Machine-readable format for scripting
```

### Understand What Got Deployed

```bash
# Full release information
helm get all myapp902
# Combines:
# - Manifest (actual Kubernetes resources)
# - Values (configuration used)
# - NOTES (chart notes/instructions)

# Just the manifests
helm get manifest myapp902
# Shows rendered Kubernetes YAML

# Compare values between revisions
helm get values myapp902 --revision 1 > values-rev1.yaml
helm get values myapp902 --revision 2 > values-rev2.yaml
diff values-rev1.yaml values-rev2.yaml
# See what changed between upgrades
```

---

## Part 7: Upgrade with Value Changes

### Upgrade Using Values File

```bash
# Upgrade with new values file
helm upgrade myapp902 stacksimplify/mychart1 \
  -f myvalues-v2.yaml

# myvalues-v2.yaml content:
# replicaCount: 3  # Changed from 2
# image.tag: 3.0.0  # Changed from 2.0.0
# service.nodePort: 31260  # Changed from 31250

# Result: Release updated with new values
```

### Upgrade with --set

```bash
# Upgrade with inline value changes
helm upgrade myapp902 stacksimplify/mychart1 \
  --set image.tag=4.0.0 \
  --set replicaCount=4

# Other values from previous install preserved
# Only specified values updated
```

### Track Value Changes

```bash
# View history with revision info
helm history myapp902

# Compare values across revisions
helm get values myapp902 --revision 1  # Initial install
helm get values myapp902 --revision 2  # After first upgrade
helm get values myapp902 --revision 3  # After second upgrade
# Track configuration evolution
```

---

## Part 8: Best Practices

### For Development

```bash
# Quick testing with --set
helm install test-app stacksimplify/mychart1 \
  --set service.nodePort=31333 \
  --set replicaCount=1 \
  --dry-run --debug

# Actually install
helm install test-app stacksimplify/mychart1 \
  --set service.nodePort=31333 \
  --set replicaCount=1
```

### For Production

```bash
# Use values file (versioned in Git)
helm install myapp-prod stacksimplify/mychart1 \
  -f values-prod.yaml \
  --atomic

# Git tracked values-prod.yaml enables:
# - Audit trail of changes
# - Easy rollback
# - Configuration as code
# - Team collaboration
```

### For Multiple Environments

```bash
# values-dev.yaml
replicaCount: 1
image.tag: dev-latest
service.nodePort: 31240

# values-staging.yaml
replicaCount: 2
image.tag: rc-v1.0
service.nodePort: 31241

# values-prod.yaml
replicaCount: 3
image.tag: v1.0
service.nodePort: 31242

# Install each environment
helm install myapp-dev stacksimplify/mychart1 -f values-dev.yaml
helm install myapp-staging stacksimplify/mychart1 -f values-staging.yaml
helm install myapp-prod stacksimplify/mychart1 -f values-prod.yaml
```

---

## Part 9: Advanced Value Syntax

### Nested Value Paths

```bash
# Access nested values with dot notation
--set image.repository=myecr.my-repo
--set image.tag=custom-tag
--set service.type=LoadBalancer
--set service.port=8080

# Works through multiple nesting levels
--set parent.child.grandchild=value
```

### List Values

```bash
# Using --set with lists
--set env[0].name=ENV_VAR
--set env[0].value=myvalue
--set env[1].name=ANOTHER_VAR
--set env[1].value=anothervalue

# Or using values file:
env:
  - name: ENV_VAR
    value: myvalue
  - name: ANOTHER_VAR
    value: anothervalue
```

### Map/Object Values

```bash
# Complex nested objects
--set labels.team=devops
--set labels.environment=prod
--set labels.owner=john

# Or in values file:
labels:
  team: devops
  environment: prod
  owner: john
```

---

## Commands Summary

| Task | Command |
|------|---------|
| View default values | `helm show values <CHART>` |
| Install with --set | `helm install <REL> <CHART> --set KEY=VALUE` |
| Install with file | `helm install <REL> <CHART> -f values.yaml` |
| Dry run preview | `helm install <REL> <CHART> --dry-run` |
| Debug preview | `helm install <REL> <CHART> --dry-run --debug` |
| Get installed values | `helm get values <RELEASE>` |
| Get all values | `helm get values <RELEASE> --all` |
| Upgrade with file | `helm upgrade <REL> <CHART> -f values.yaml` |
| Upgrade with --set | `helm upgrade <REL> <CHART> --set KEY=VALUE` |
| Compare revisions | `helm diff upgrade ... (requires plugin)` |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Value not applied | Check hierarchy: --set overrides -f, -f overrides defaults |
| Syntax error in --set | Verify dot notation: --set parent.child=value |
| Complex values won't work with --set | Use -f values file instead |
| Can't find what values to use | Run: helm show values <CHART> |
| Upgrade reverted old values | Use --set or -f to re-apply custom values |

---

## Summary

Key Capabilities:

✅ **Override Defaults** - Customize chart behavior without modifying chart
✅ **Multiple Methods** - Use --set for quick changes, -f for complex configs
✅ **Value Hierarchy** - Understand priority: --set > -f > defaults
✅ **Environment Strategy** - Different values files for dev/staging/prod
✅ **Preview Changes** - Use --dry-run --debug before installing
✅ **Track Changes** - helm history and helm get values track evolution
✅ **Validate Configuration** - helm show values reveals all options

---

## References

- [Helm Values Documentation](https://helm.sh/docs/chart_template_engine/values/)
- [Helm Install Command](https://helm.sh/docs/helm/helm_install/)
- [Helm Upgrade Command](https://helm.sh/docs/helm/helm_upgrade/)
- [Values Best Practices](https://helm.sh/docs/chart_best_practices/values/)
