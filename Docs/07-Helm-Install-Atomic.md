# Helm Install with Atomic Flag for Safe Deployments

## Overview
This module covers the `--atomic` flag for Helm's install and upgrade commands. The atomic flag provides a critical safety mechanism by automatically rolling back a deployment if it fails, ensuring that failed installations don't leave your cluster in a broken state with orphaned resources.

## Core Concepts

### What is Atomic Installation?

**Atomic Behavior:**
- Installation proceeds only if all Kubernetes resources reach ready state
- If any part fails during installation, entire release is automatically removed
- Failed releases are not left hanging in error state
- Prevents partial/broken deployments
- Automatically enables `--wait` mechanism

### Why Use Atomic Flag?

**Problems Without Atomic:**
- Failed installation leaves broken resources
- Release stuck in failed state
- Manual cleanup required
- Service port conflicts cause port allocation errors
- Orphaned pods consume resources

**Solutions With Atomic:**
- Failed installation completely rolled back
- No broken state left behind
- Automatic cleanup of failed deployment
- Cluster remains clean
- Safe for automated pipelines

---

## Part 1: Installation Failure Scenarios

### Scenario: Port Conflict Without Atomic

```bash
# First release: dev101
helm install dev101 stacksimplify/mychart1
# SUCCESS: Installs successfully
# Service uses NodePort 31231

# Second release: qa101 (without --atomic)
helm install qa101 stacksimplify/mychart1
# FAILURE: Port 31231 already allocated to dev101

# Error Message:
# Error: INSTALLATION FAILED: 1 error occurred:
# * Service "qa101-mychart1" is invalid: spec.ports[0].nodePort: Invalid value: 31231: 
#   provided port is already allocated

# At this point WITHOUT --atomic:
# - Installation failed but partial resources created
# - qa101 release stuck in FAILED state
# - Left-over Kubernetes resources remain
# - Must manually investigate and cleanup
```

### Checking Failed Installation State

```bash
# List releases shows failed status
helm list
# Output:
# NAME    NAMESPACE  REVISION  STATUS   CHART
# dev101  default    1         deployed mychart1-1.0
# qa101   default    1         failed   mychart1-1.0

# Get status of failed release
helm status qa101
# Shows installation failed details

# Kubernetes resources may exist in broken state
kubectl get all -l release=qa101
# Shows partial resources created before failure

# Service might be partially created
kubectl get svc -l release=qa101
# May show service with unallocated port or error state
```

### Manual Cleanup of Failed Release

```bash
# Must manually uninstall failed release
helm uninstall qa101
# Removes failed release

# Verify cleanup
helm list
# qa101 no longer appears

# Check Kubernetes resources cleaned
kubectl get all -l release=qa101
# Should show nothing (or minimal residual)
```

---

## Part 2: Solution with Atomic Flag

### Install with Atomic Flag

```bash
# Install with safety net enabled
helm install <RELEASE> <CHART> --atomic

# Example: Install qa101 with --atomic
helm install qa101 stacksimplify/mychart1 --atomic
```

**Atomic Behavior:**
1. Begins resource creation
2. Waits for all resources to be ready (automatic --wait)
3. If any resource fails, stops immediately
4. Automatically deletes ALL resources created
5. Removes release completely
6. Cluster returns to clean pre-installation state

### Automatic Rollback on Failure

```bash
# Install with --atomic enabled
helm install qa101 stacksimplify/mychart1 --atomic

# Port conflict error occurs
# Error: INSTALLATION FAILED: 1 error occurred:
# * Service "qa101-mychart1" is invalid: spec.ports[0].nodePort: Invalid value: 31231: 
#   provided port is already allocated

# Key difference WITH --atomic:
# - Immediate failure detected
# - All partially created resources deleted
# - Release NOT created in system
# - Cluster returns to clean state

# Verify no failed release exists
helm list
# qa101 NOT in list (no failed release to cleanup)

# Check Kubernetes state
kubectl get all -l release=qa101
# Nothing appears (completely cleaned up)
```

---

## Part 3: Atomic vs Non-Atomic Comparison

### Comparison Matrix

| Aspect | Without --atomic | With --atomic |
|--------|-----------------|---------------|
| Installation failure | Stops with resources remaining | Completes rollback, clean state |
| Release state | FAILED status in list | Not created if fails |
| Cleanup | Manual cleanup required | Automatic cleanup |
| helm list | Shows failed releases | Failed releases don't appear |
| Action needed | Must uninstall and investigate | Can retry immediately |
| Cluster state | Dirty with partial resources | Clean, ready to retry |
| Time to recover | Manual + investigation time | Automatic, instant |

### Hands-On Example

**Scenario: Same chart to same port conflict**

**Without --atomic:**
```bash
# Install first release
helm install dev101 stacksimplify/mychart1

# Try second release (fails)
helm install qa101 stacksimplify/mychart1
# Error: port conflict

# Check status
helm list
# Shows: dev101 deployed, qa101 failed

# Must cleanup manually
helm uninstall qa101

# Then can retry, and would need different port
```

**With --atomic:**
```bash
# Install first release
helm install dev101 stacksimplify/mychart1

# Try second release with --atomic (fails)
helm install qa101 stacksimplify/mychart1 --atomic
# Error: port conflict
# Automatic cleanup happens instantly

# Check status
helm list
# Shows: dev101 deployed, qa101 NOT IN LIST

# Cluster is clean, can retry immediately
# Fix issue (like port config) and retry
```

---

## Part 4: Atomic Flag with Related Options

### --atomic Automatically Enables --wait

```bash
# These are equivalent:
helm install dev101 stacksimplify/mychart1 --atomic

# Is essentially:
helm install dev101 stacksimplify/mychart1 --atomic --wait

# --wait behavior with --atomic:
# 1. Waits for Pods, PVCs, Services to be ready
# 2. Waits for Deployment/StatefulSet minimum pods
# 3. Continues waiting until timeout or ready
# 4. If timeout or error, triggers rollback
```

### Timeout Configuration

```bash
# --atomic with default timeout (5 minutes)
helm install dev101 stacksimplify/mychart1 --atomic

# --atomic with custom timeout
helm install dev101 stacksimplify/mychart1 \
  --atomic \
  --timeout 10m0s

# --atomic with short timeout for quick failure
helm install dev101 stacksimplify/mychart1 \
  --atomic \
  --timeout 1m0s

# Useful for different scenarios:
# - 5m default: good for normal apps
# - 10m+: for apps with initialization time
# - <1m: for quick validation testing
```

### Combined Flags for robust Installation

```bash
# Comprehensive safe installation
helm install qa101 stacksimplify/mychart1 \
  --atomic \
  --wait \
  --timeout 10m0s \
  --create-namespace \
  --namespace myapp

# This ensures:
# - Automatic rollback on failure (--atomic)
# - Wait for resource readiness (--wait, implicit with --atomic)
# - 10 minute window for initialization
# - Create namespace if doesn't exist
# - Deploy to specific namespace
```

---

## Part 5: Atomic with Upgrade Operations

### Upgrade with Atomic Flag

```bash
# Upgrade release with atomic safety
helm upgrade <RELEASE> <CHART> --atomic

# Example: Upgrade existing release
helm upgrade myapp stacksimplify/mychart1 --atomic

# If upgrade fails partway through:
# - Detects failure
# - Automatically rollback to previous version
# - Release returns to prior working state
# - Cluster remains stable
```

### Upgrade Atomic Behavior

```bash
# Before upgrade
helm list
# Shows: myapp  deployed  Revision 1

# Upgrade fails partway
helm upgrade myapp stacksimplify/mychart1 --atomic

# After failure with --atomic:
helm list
# Shows: myapp  deployed  Revision 1 (reverted)
# NOT: Revision 2 (failed upgrade is not recorded)

# Without --atomic during upgrade failure:
# Would show: Revision 2 failed
# Manual rollback needed: helm rollback myapp 1
```

---

## Part 6: When to Use Atomic

### Recommended for Production

```bash
# Production installation - ALWAYS use --atomic
helm install myapp-prod stacksimplify/mychart1 --atomic \
  --namespace production \
  --timeout 10m0s

# Why:
# - Prevents broken production state
# - Automatic recovery from failures
# - Clean cluster, easy to diagnose
# - No orphaned resources
```

### Recommended for CI/CD Pipelines

```bash
#!/bin/bash
# Automated deployment script
helm install $RELEASE_NAME $CHART \
  --atomic \
  --namespace $NAMESPACE \
  --create-namespace \
  --timeout 5m0s

# Why:
# - No manual intervention needed
# - Automatic cleanup on failure
# - Pipeline doesn't need cleanup step
# - Reliable automation
```

### Optional for Development

```bash
# Development installation - atomic useful
helm install dev-env stacksimplify/mychart1 --atomic

# Why:
# - Clean cluster for next attempt
# - No debug residual resources
# - Quick iterate-and-retry cycle

# Could also install without for investigation:
helm install dev-env stacksimplify/mychart1
# To keep failed state for debugging
```

---

## Part 7: Practical Examples

### Example 1: Safe Production Deployment

```bash
#!/bin/bash
# Safe production deployment script

ENVIRONMENT="production"
RELEASE="myapp-prod"
CHART="mycompany/myapp"
IMAGE_TAG="${1:-latest}"

echo "Deploying $CHART to $ENVIRONMENT..."

# Deploy with atomic safety
helm install $RELEASE $CHART \
  --namespace $ENVIRONMENT \
  --create-namespace \
  --atomic \
  --timeout 10m \
  --set image.tag=$IMAGE_TAG \
  --values values-prod.yaml

if [ $? -eq 0 ]; then
    echo "✅ Deployment successful"
    helm status $RELEASE --show-resources
    exit 0
else
    echo "❌ Deployment failed - rolled back automatically"
    echo "Review error and retry"
    exit 1
fi
```

### Example 2: Canary Deployment with Atomic

```bash
#!/bin/bash
# Canary deployment testing

RELEASE_CANARY="myapp-canary"
CHART="mycompany/myapp"
NEW_VERSION="2.0.0"

echo "Testing canary version $NEW_VERSION..."

# Deploy canary with atomic (safe to fail)
helm install $RELEASE_CANARY $CHART \
  --namespace staging \
  --atomic \
  --timeout 5m \
  --set image.tag=$NEW_VERSION \
  --set canary.enabled=true

if [ $? -eq 0 ]; then
    echo "✅ Canary deployed, running tests..."
    sleep 10
    
    # Run tests
    kubectl run test -n staging \
      --image=mycompany/test-suite:latest \
      --restart=Never \
      -- test-canary
    
    TEST_RESULT=$?
    
    if [ $TEST_RESULT -eq 0 ]; then
        echo "✅ Canary tests passed - promote to production"
        helm install myapp-prod $CHART \
          --namespace production \
          --create-namespace \
          --atomic \
          --set image.tag=$NEW_VERSION
    else
        echo "⚠️ Canary tests failed - automatic rollback completed"
        echo "Investigate and retry"
    fi
    
    # Clean canary
    helm uninstall $RELEASE_CANARY -n staging
else
    echo "❌ Canary deployment failed - automatically cleaned up"
fi
```

### Example 3: Multiple Environment Deployment

```bash
#!/bin/bash
# Deploy to dev, staging, prod with atomic safety

ENVIRONMENTS=("dev" "staging" "production")
CHART="mycompany/myapp"

for ENV in "${ENVIRONMENTS[@]}"; do
    echo "Deploying to $ENV..."
    
    helm install myapp-$ENV $CHART \
      --namespace $ENV \
      --create-namespace \
      --atomic \
      --timeout 5m \
      -f values-$ENV.yaml
    
    if [ $? -eq 0 ]; then
        echo "✅ $ENV deployment successful"
    else
        echo "❌ $ENV deployment failed - rolled back automatically"
        # Continue to next environment
        continue
    fi
done
```

---

## Part 8: Troubleshooting Atomic Failures

### Debug Atomic Installation Failure

```bash
# Get more details about why atomic installation failed
helm install myapp stacksimplify/mychart1 \
  --atomic \
  --debug \
  --timeout 20s

# Shows:
# Rendered manifests being applied
# Waiting for resources
# When timeout/failure occurs
# What rolled back

# Check Kubernetes events for failure reason
kubectl get events --sort-by='.lastTimestamp' | head -20

# Check pod status when atomic fails
kubectl get pods --show-all

# Check service status
kubectl get svc
```

### Common Atomic Failures

| Failure | Cause | Solution |
|---------|-------|----------|
| Pod CrashLoopBackOff | Image not found or env config | Fix image tag, check env vars |
| Port already allocated | NodePort in use | Use different NodePort or configure |
| Resource quota exceeded | Namespace resource limits | Increase quota or use different namespace |
| Timeout waiting for ready | Pod takes long to init | Increase --timeout value |
| PVC not binding | Storage not available | Check storage class, create PVC first |

### Increase Timeout for Slow Startup

```bash
# If application takes long to startup
helm install myapp stacksimplify/mychart1 \
  --atomic \
  --timeout 15m0s
  # Allow 15 minutes for app to be ready
  # Prevents atomic rollback on slow startup

# Check pod logs to understand startup time
kubectl logs <pod-name> --follow
# Monitor initialization process
```

---

## Commands Summary

| Task | Command |
|------|---------|
| Install with atomic | `helm install <RELEASE> <CHART> --atomic` |
| Install with timeout | `helm install <RELEASE> <CHART> --atomic --timeout 10m` |
| Upgrade with atomic | `helm upgrade <RELEASE> <CHART> --atomic` |
| Install with wait (atomic includes this) | `helm install <RELEASE> <CHART> --atomic` |
| Debug atomic failure | `helm install ... --atomic --debug` |
| View events | `kubectl get events --sort-by='.lastTimestamp'` |

---

## Best Practices

✅ **Always use --atomic for production deployments**
✅ **Use appropriate timeout for your application startup time**
✅ **Combine with --debug for troubleshooting**
✅ **Document why deployment might fail**
✅ **Monitor events for root cause analysis**
✅ **Use in CI/CD pipelines for automation safety**
✅ **Test timeout values in staging before production**

---

## Summary

The `--atomic` flag provides:

🛡️ **Automatic Rollback** - Failed installations cleaned up automatically
🔒 **Safe Deployments** - No broken partial state left behind
⚡ **Quick Recovery** - Instant failure detection and cleanup
📊 **Pipeline Ready** - Perfect for automated deployments
🏗️ **Infrastructure Clean** - Cluster stays in known good state
✅ **Production Ready** - Enterprise-grade safety mechanism

---

## References

- [Helm Install Atomic Documentation](https://helm.sh/docs/helm/helm_install/)
- [Helm Upgrade Atomic Documentation](https://helm.sh/docs/helm/helm_upgrade/)
- [Kubernetes Resource Readiness](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/#creating-and-exploring-an-nginx-deployment)
- [Helm Deployment Best Practices](https://helm.sh/docs/chart_best_practices/)
