# Helm Deployments with Kubernetes Namespaces

## Overview
This module covers how Helm interacts with Kubernetes namespaces. Namespaces provide logical resource isolation within a Kubernetes cluster, allowing multiple teams or applications to coexist with separate configurations. Understanding namespace management is crucial for multi-tenant deployments and production environments.

## Core Concepts

### What are Kubernetes Namespaces?

**Namespace Purpose:**
- Logical resource isolation
- Multiple environments in one cluster (dev, staging, prod)
- Team-based resource separation
- Resource quota and limits per namespace
- Role-based access control (RBAC) per namespace

### Default Behavior

By default, Helm deploys resources to the **default** namespace:
```bash
helm install myapp stacksimplify/mychart1
# Resources deployed to: default namespace
```

### Multi-Namespace Strategy

```
Kubernetes Cluster:
├── default (system resources, general use)
├── kube-system (Kubernetes system)
├── dev (development environment)
├── staging (staging environment)
└── production (production environment)
```

---

## Part 1: Viewing Kubernetes Namespaces

### List Namespaces

```bash
# List all namespaces in cluster
kubectl get ns
kubectl get namespaces

# Output:
# NAME              STATUS   AGE
# default           Active   XXm
# kube-node-lease   Active   XXm
# kube-public       Active   XXm
# kube-system       Active   XXm
# dev               Active   XXm (if created)

# Get detailed namespace information
kubectl get ns -o wide

# Describe namespace
kubectl describe ns dev
# Shows resource quotas, limits, and status
```

### Built-in Namespaces

| Namespace | Purpose |
|-----------|---------|
| default | Default location for user resources |
| kube-system | Kubernetes system components |
| kube-public | Publicly readable data |
| kube-node-lease | Node lease information |

---

## Part 2: Install with Explicit Namespace

### Install to Specific Namespace

**Syntax:**
```bash
helm install <RELEASE> <CHART> -n <NAMESPACE>
# or
helm install <RELEASE> <CHART> --namespace <NAMESPACE>
```

**Example: Install to dev namespace**
```bash
# Install to dev namespace (namespace must exist)
helm install dev101 stacksimplify/mychart2 \
  --version "0.1.0" \
  --namespace dev

# Long form
helm install dev101 stacksimplify/mychart2 \
  --version "0.1.0" \
  --namespace dev
```

### Verify Namespace Deployment

```bash
# List releases in specific namespace only
helm list -n dev
helm list --namespace dev
# Shows: dev101 release deployed in dev namespace

# Helm releases in default namespace
helm list
# Shows releases in default namespace (dev101 not here)

# Helm releases in ALL namespaces
helm list -A
helm list --all-namespaces
# Shows releases across all namespaces
```

---

## Part 3: Create Namespace During Install

### Auto-Create Namespace

**Key Feature:** Helm can create namespace automatically if it doesn't exist

```bash
# Install with namespace creation
helm install dev101 stacksimplify/mychart2 \
  --version "0.1.0" \
  --namespace dev \
  --create-namespace
# Creates 'dev' namespace if doesn't exist
```

**Benefits:**
- No need to pre-create namespaces
- Automated deployment pipeline friendly
- Single command handles both namespace and release

### Default Behavior (Without --create-namespace)

```bash
# Install without --create-namespace (namespace must exist)
helm install dev101 stacksimplify/mychart2 \
  --namespace dev
# Error if 'dev' namespace doesn't exist:
# Error: release dev101 failed: 
# namespaces "dev" not found

# Solution: Create namespace first or use --create-namespace
kubectl create namespace dev
# Then install

# OR use --create-namespace flag
helm install dev101 stacksimplify/mychart2 \
  --namespace dev \
  --create-namespace
# Creates namespace automatically
```

---

## Part 4: Verify Namespace Deployment

### Check Kubernetes Namespaces

```bash
# After installation with --create-namespace
kubectl get ns

# Output shows new namespace:
# NAME              STATUS   AGE
# default           Active   XXm
# dev               Active   XXm (newly created)
# kube-public       Active   XXm
# kube-system       Active   XXm

# Verify namespace created by Helm
kubectl describe ns dev
# Shows namespace details and resource policies
```

### View Release in Namespace

```bash
# List Helm releases in dev namespace
helm list -n dev
# Output:
# NAME    NAMESPACE  REVISION  STATUS    CHART         APP VERSION  UPDATED
# dev101  dev        1         deployed  mychart2-0.1  1.0.0        <time>

# Get release status with resources
helm status dev101 -n dev --show-resources
# Shows:
# - Release status
# - All Kubernetes resources (pods, services, deployments)
# - Namespace: dev
```

### Verify Kubernetes Resources in Namespace

```bash
# List pods in dev namespace
kubectl get pods -n dev
# Shows pods created by dev101 release

# List services in dev namespace
kubectl get svc -n dev
# Shows service (NodePort with port 31232)

# List deployments
kubectl get deploy -n dev
# Shows deployment created by mychart2

# Get all resources in namespace
kubectl get all -n dev
# Shows all resource types

# Detailed resource view
kubectl get pods -n dev -o wide
# Shows pod names, images, nodes, IPs
```

---

## Part 5: Helm Operations in Namespaces

### Upgrade Release in Namespace

```bash
# Upgrade release in dev namespace
helm upgrade dev101 stacksimplify/mychart2 \
  --version "0.2.0" \
  --namespace dev
# or
helm upgrade dev101 stacksimplify/mychart2 \
  --version "0.2.0" \
  -n dev

# Verify upgrade
helm list -n dev
# Revision increases from 1 to 2

# Check status with resources
helm status dev101 -n dev --show-resources
# Shows updated resources with new chart version
```

### Access Application in Namespace

```bash
# Application service is accessible regardless of namespace
# NodePort service is accessible from cluster externally

# Access via browser (NodePort is cluster-wide)
http://localhost:31232
# Works because service uses NodePort

# Direct pod access within namespace
kubectl exec -it <POD-NAME> -n dev -- /bin/bash
# Execute commands inside pod in dev namespace

# Port forward from namespace
kubectl port-forward svc/<SERVICE-NAME> 8080:80 -n dev
# Forwards from service in dev namespace
# Access via: http://localhost:8080
```

---

## Part 6: Uninstall from Namespace

### Uninstall Release from Namespace

```bash
# Uninstall release from dev namespace
helm uninstall dev101 --namespace dev
# or
helm uninstall dev101 -n dev

# Verify uninstallation
helm list -n dev
# dev101 no longer appears

# Kubernetes resources deleted
kubectl get all -n dev
# Shows only default service (empty namespace)
```

### Namespace Persistence After Uninstall

**Important:** Uninstalling Helm release does NOT delete the namespace

```bash
# After uninstalling dev101
helm list -n dev
# No releases in dev namespace

# But namespace still exists
kubectl get ns
# dev namespace still present

# Helm release removed, but Kubernetes namespace remains
kubectl get pods -n dev
# No pods (release uninstalled)
```

### Delete Namespace Manually

```bash
# Delete dev namespace (required separately)
kubectl delete ns dev
# Namespace and all its resources deleted

# Verify deletion
kubectl get ns
# dev namespace no longer appears

# Confirm all resources removed
kubectl get all -n dev
# Error: namespace "dev" not found (expected)
```

---

## Part 7: Multi-Environment Management

### Setup Multiple Environments

```bash
# Create and setup dev environment
helm install dev101 stacksimplify/mychart2 \
  --version "0.1.0" \
  --namespace dev \
  --create-namespace

# Create and setup staging environment
helm install staging101 stacksimplify/mychart2 \
  --version "0.2.0" \
  --namespace staging \
  --create-namespace

# Create and setup production environment
helm install prod101 stacksimplify/mychart2 \
  --version "0.4.0" \
  --namespace production \
  --create-namespace

# View all environments
helm list --all-namespaces
# Shows all releases across all namespaces with their versions
```

### Environment-Specific Management

```bash
# Different values for different environments
helm install dev101 stacksimplify/mychart2 \
  -n dev \
  -f values-dev.yaml

helm install prod101 stacksimplify/mychart2 \
  -n production \
  -f values-prod.yaml

# List by environment
helm list -n dev
helm list -n production

# Upgrade specific environment
helm upgrade dev101 stacksimplify/mychart2 --version "0.3.0" -n dev
# Only dev affected, production unchanged
```

---

## Part 8: Namespace Best Practices

### Naming Convention

```bash
# Consistent namespace naming
helm install myapp stacksimplify/mychart2 \
  --namespace dev \
  --create-namespace
# Namespace: dev

helm install myapp stacksimplify/mychart2 \
  --namespace staging \
  --create-namespace
# Namespace: staging

helm install myapp stacksimplify/mychart2 \
  --namespace production \
  --create-namespace
# Namespace: production
```

### Production Recommendations

```bash
# Production deployment best practices
helm install myapp-prod stacksimplify/mychart2 \
  --namespace production \
  --create-namespace \
  --atomic \
  --timeout 10m \
  -f values-prod.yaml

# Key points:
# - Explicit release name (myapp-prod)
# - Dedicated namespace (production)
# - Auto-create namespace with --create-namespace
# - Atomic flag for safety
# - Production-specific values file
```

### Development Flexibility

```bash
# Dev allows more flexibility
helm install interactive-test stacksimplify/mychart2 \
  --namespace dev \
  --dry-run --debug
# Preview before install

# Easy cleanup
helm uninstall interactive-test -n dev
```

---

## Part 9: Namespace Isolation Benefits

### Resource Separation

```
Default Namespace          Dev Namespace
├── myapp1-release        └── dev101-release
├── myapp2-release           (isolated resources)
└── myapp3-release

Each namespace has separate:
- Deployments
- Services
- ConfigMaps
- Secrets
- Persistent Volumes
```

### RBAC Per Namespace

```bash
# Different teams can have access to different namespaces
# Team A: access to dev namespace only
# Team B: access to staging namespace only
# Team C: access to production with restrictions

kubectl create rolebinding dev-admin \
  --clusterrole=admin \
  --serviceaccount=dev:default \
  --namespace=dev
# Binds admin role to dev namespace
```

### Resource Quotas Per Namespace

```bash
# Limit resources per namespace
kubectl set quota dev-quota \
  --hard=pods=10,cpu=5,memory=10Gi \
  --namespace=dev

# Dev environment limited to 10 pods max
# Cannot consume more than 5 CPUs
# Cannot use more than 10 GB memory
```

---

## Commands Summary

| Task | Command |
|------|---------|
| List namespaces | `kubectl get ns` |
| Install to namespace | `helm install <REL> <CHART> -n <NS>` |
| Auto-create namespace | `helm install <REL> <CHART> -n <NS> --create-namespace` |
| List releases in NS | `helm list -n <NS>` |
| List all releases | `helm list -A` |
| Upgrade in namespace | `helm upgrade <REL> <CHART> -n <NS>` |
| Uninstall from NS | `helm uninstall <REL> -n <NS>` |
| Delete namespace | `kubectl delete ns <NS>` |
| Show resources in NS | `kubectl get all -n <NS>` |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Namespace not found | Use `--create-namespace` flag when installing |
| Release in wrong namespace | Specify correct namespace with `-n` flag |
| Can't find release | Check namespace: `helm list -n <NS>` |
| Confused with multiple releases | Use distinct names and track by namespace |

---

## Summary

Namespace management enables:

✅ **Multi-environment separation** (dev, staging, prod)
✅ **Resource isolation** (multiple teams sharing cluster)
✅ **Independent management** (upgrade one without affecting others)
✅ **Resource quotas** (limit resources per team)
✅ **RBAC control** (access control per namespace)
✅ **Clean organization** (logical grouping of applications)

---

## References

- [Kubernetes Namespaces Documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Helm Namespace Usage](https://helm.sh/docs/intro/using_helm/#helm-namespaces)
- [kubectl namespace guide](https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/)
