# Helm Installation and Repository Management

## Overview
This module covers the fundamental Helm commands for managing repositories and installing Helm charts. It introduces the concept of Helm repositories (collections of packaged applications) and demonstrates how to discover, manage, and deploy applications using Helm.

## Core Concepts

### What is Helm?
Helm is the package manager for Kubernetes that provides:
- **Package Management:** Install, upgrade, and manage applications
- **Templating Engine:** Dynamic Kubernetes manifest generation
- **Chart Distribution:** Standardized application packaging
- **Release Management:** Version control for deployments

### Helm Terminology
- **Repository:** Remote or local collection of Helm charts
- **Chart:** A packaged Helm application (like .deb or .rpm)
- **Release:** An instance of a chart deployed in Kubernetes
- **Values:** Configuration parameters that customize chart behavior

---

## Part 1: Helm Repository Management

### Understanding Helm Repositories

Helm repositories are servers hosting Helm charts. Popular public repositories include:
- **Bitnami:** [https://charts.bitnami.com/bitnami](https://bitnami.com/stacks/helm) - Pre-packaged applications
- **Official Helm Stable:** Legacy repository (deprecated but still available)
- **ArtifactHub:** [https://artifacthub.io/](https://artifacthub.io/) - Central index for Helm charts
- **Jetstack:** Certificates and TLS management
- **Prometheus Community:** Monitoring and alerting

### List Helm Repositories

```bash
# View all configured repositories on your system
helm repo list
# Output:
# NAME           URL
# bitnami        https://charts.bitnami.com/bitnami
# stable         https://charts.helm.sh/stable

# Alternative (shorter syntax)
helm ls --repo
```

**Understanding the Output:**
- **NAME:** Local alias for the repository
- **URL:** Remote location of the chart repository

### Add Helm Repository

```bash
# Add a Helm repository with custom local name
helm repo add <LOCAL-NAME> <REPOSITORY-URL>

# Example: Add Bitnami repository
helm repo add mybitnami https://charts.bitnami.com/bitnami

# Example: Add official stable charts (deprecated but available)
helm repo add stable https://charts.helm.sh/stable

# Example: Add other popular repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add jetstack https://charts.jetstack.io

# List repositories after adding
helm repo list
```

**Key Points:**
- Local name (mybitnami) can be any alias you choose
- URL must be a valid Helm chart repository
- Same URL can be added multiple times with different local names

### Remove a Repository

```bash
# Remove a repository by its local name
helm repo remove <LOCAL-NAME>

# Example: Remove bitnami repository
helm repo remove mybitnami

# Verify removal
helm repo list
```

### Update Repository Index

```bash
# Update all repository indexes from remote sources
# Must run before searching to get latest charts
helm repo update

# Update specific repository only
helm repo update <REPO-NAME>

# Example: Update only Bitnami repository
helm repo update mybitnami

# What it does:
# - Fetches the latest chart index from repositories
# - Updates local cache (~/.helm/cache)
# - Discovers new chart versions
# - Typically completes within seconds
```

**Update Behavior:**
- Doesn't update installed charts (only chart index)
- Required before `helm search repo` to see latest charts
- Safe to run frequently
- Downloads minimal data (index files only)

---

## Part 2: Search and Discovery

### Search Helm Repositories

```bash
# Search all added repositories for keyword
helm search repo <KEYWORD>

# Example: Search for Nginx charts
helm search repo nginx
# Output:
# NAME                 CHART VERSION    APP VERSION    DESCRIPTION
# mybitnami/nginx      15.3.0          1.25.0        NGINX Open Source is a web server and reverse proxy

# Search for Apache
helm search repo apache

# Search for Wildfly (Java application server)
helm search repo wildfly

# Search for database charts
helm search repo mysql
helm search repo postgresql

# Search specific repository only
helm search repo mybitnami/mysql
```

**Output Columns:**
| Column | Meaning |
|--------|---------|
| NAME | Repository/ChartName format |
| CHART VERSION | Version of the chart package |
| APP VERSION | Version of the application |
| DESCRIPTION | Short description of the chart |

### Advanced Search Options

```bash
# Display full chart details
helm search repo nginx --detailed

# Search with version constraints
helm search repo nginx --version "15.3.0"

# Search with output formatting
helm search repo nginx --output=table
helm search repo nginx --output=json
helm search repo nginx --output=yaml

# Show maximum results (default is unlimited)
helm search repo nginx --max-cols=5

# Development versions (includes pre-releases)
helm search repo nginx --devel
```

### Find Available Chart Versions

```bash
# View all versions of a specific chart
helm search repo mybitnami/nginx --versions
# Output shows all available versions for upgrade scenarios

# View latest version only
helm search repo mybitnami/nginx | head -1
```

---

## Part 3: Installing Helm Charts

### Basic Installation

```bash
# Install a chart with default configuration
helm install <RELEASE-NAME> <REPO-NAME>/<CHART-NAME>

# Example: Install Nginx from Bitnami
helm install mynginx mybitnami/nginx
```

**What Happens During Installation:**
1. Helm fetches chart from repository
2. Renders templates with default values
3. Creates Kubernetes resources (deployments, services, etc.)
4. Tracks release state in Kubernetes

### Installation with Namespace

```bash
# Install in specific namespace (creates namespace if needed)
helm install mynginx mybitnami/nginx --namespace myapp --create-namespace

# Install in default namespace
helm install mynginx mybitnami/nginx --namespace default

# Short form
helm install mynginx mybitnami/nginx -n myapp --create-namespace
```

### Installation with Custom Values

```bash
# Install with inline value overrides
helm install mynginx mybitnami/nginx \
  --set replicaCount=3 \
  --set service.type=LoadBalancer

# Install with values file
helm install mynginx mybitnami/nginx -f values.yaml

# Combine methods
helm install mynginx mybitnami/nginx \
  -f values.yaml \
  --set replicaCount=5
```

### Pre-Installation Validation

```bash
# Dry run to see what would be installed (no actual deployment)
helm install mynginx mybitnami/nginx --dry-run

# Dry run with debug output to see rendered templates
helm install mynginx mybitnami/nginx --dry-run --debug

# Validate chart structure
helm lint mybitnami/nginx
# Checks for common issues and best practices
```

---

## Part 4: Managing Installed Releases

### List Helm Releases

```bash
# List releases in default namespace
helm list

# Alternative
helm ls

# List releases with detailed information
helm list --detailed

# List releases as YAML
helm list --output=yaml

# List releases as JSON
helm list --output=json

# List releases in all namespaces
helm list --all-namespaces
helm list -A

# List releases in specific namespace
helm list --namespace=default
helm list -n default

# Filter by status (deployed, failed, pending-install, etc.)
helm list --deployed
helm list --all

# Sort by release date
helm list --sort=date

# Show only last 3 releases
helm list --max=3
```

**Helm Release Status Values:**
- **deployed:** Successfully installed and running
- **failed:** Installation failed or was rolled back
- **pending-install:** Installation in progress
- **pending-upgrade:** Upgrade in progress
- **superseded:** Replaced by newer release (with history)
- **uninstalled:** Removed with history preserved

### Get Release Details

```bash
# View installation values used for a release
helm get values mynginx
# Shows only overridden values

helm get values mynginx --all
# Shows default + overridden values

# View rendered manifests
helm get manifest mynginx
# Shows actual Kubernetes resources created

# Get release history
helm history mynginx
# Shows all revisions and their status

# View release metadata (manifest, values, notes)
helm get all mynginx
# Combines manifest, values, and notes
```

---

## Part 5: Verify Kubernetes Resources

### Check Pods and Services

```bash
# List all pods created by Helm release
kubectl get pods

# Detailed pod info
kubectl get pods --wide

# Watch pods in real-time
kubectl get pods --watch

# List services
kubectl get svc

# Details about specific service
kubectl get svc mynginx
# Shows EXTERNAL-IP (for LoadBalancer type)
# Shows port mapping and selectors

# Get service endpoints
kubectl get endpoints mynginx
# Shows backend pods serving the service
```

**Service Type Reference:**
| Type | EXTERNAL-IP | Usage |
|------|-------------|-------|
| ClusterIP | None | Internal cluster communication |
| NodePort | <nodes> | Access via node IP + port |
| LoadBalancer | <external-ip> | Cloud load balancer (if supported) |
| ExternalName | N/A | DNS alias to external service |

### Access Deployed Application

```bash
# For LoadBalancer service with External-IP
http://localhost:80
http://127.0.0.1:80

# For NodePort service
http://<node-ip>:<nodeport>

# Using curl command
curl http://localhost:80
curl http://127.0.0.1:80

# Port forwarding (alternative access method)
kubectl port-forward svc/mynginx 8080:80
# Then access: http://localhost:8080

# Get service details
kubectl describe svc mynginx
```

---

## Part 6: Uninstall Helm Release

### Remove a Release

```bash
# Uninstall (delete) a Helm release
helm uninstall <RELEASE-NAME>

# Example: Remove Nginx release
helm uninstall mynginx

# Uninstall with options
helm uninstall mynginx --namespace default
helm uninstall mynginx -n default

# Keep release history (for future rollbacks)
helm uninstall mynginx --keep-history

# Verify uninstall
helm list
# mynginx should no longer appear
```

**Uninstall Behavior:**
- **Default:** Deletes release history
- **With --keep-history:** Preserves release for rollback
- **Resources:** Removes all Kubernetes resources
- **PersistentVolumes:** Typically NOT deleted (data preserved)

### Verify Removal

```bash
# Confirm release is removed
helm list
# Should not show mynginx

# Check if Kubernetes resources are deleted
kubectl get pods
# No pods from mynginx should appear

kubectl get svc
# Services should be removed

# View uninstalled releases (with history)
helm list --uninstalled
# Only shows if --keep-history was used during uninstall
```

---

## Complete Workflow Example

```bash
# 1. Update repositories
helm repo update

# 2. Search for available application
helm search repo nginx

# 3. Dry run installation
helm install mynginx mybitnami/nginx --dry-run --debug

# 4. Install actual release
helm install mynginx mybitnami/nginx

# 5. Verify installation
helm list
helm get values mynginx
helm get manifest mynginx

# 6. Check Kubernetes resources
kubectl get pods
kubectl get svc

# 7. Access application
curl http://localhost:80

# 8. Manage release
helm get history mynginx
helm list --detailed

# 9. Uninstall when done
helm uninstall mynginx
```

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "chart not found" | Repository not added or outdated | Run `helm repo add` and `helm repo update` |
| Service has no EXTERNAL-IP | Docker Desktop limitation | Use `kubectl port-forward` instead |
| Release already exists | Release name conflict | Use `helm upgrade` or different name |
| Access application fails | Service not ready | Check pod status, review logs |
| Repository unreachable | Network or URL issue | Verify URL, check internet connectivity |

### Debugging Commands

```bash
# Check Helm configuration
helm config view

# View helm directories
helm env

# Check chart syntax
helm lint <chart-path>

# Get pod logs
kubectl logs <pod-name>

# Describe pod for errors
kubectl describe pod <pod-name>

# Check service connectivity
kubectl exec <pod-name> -- curl http://localhost:80
```

---

## Key Helm Commands Summary

| Command | Purpose |
|---------|---------|
| `helm repo list` | List configured repositories |
| `helm repo add` | Add new repository |
| `helm repo update` | Update repository index |
| `helm search repo` | Search for charts |
| `helm install` | Deploy a chart |
| `helm list` | Show installed releases |
| `helm get` | Retrieve release information |
| `helm uninstall` | Remove a release |
| `helm history` | View release revisions |
| `helm rollback` | Revert to previous version |

---

## Next Steps

After mastering basic Helm installation, explore:
1. **Value Overrides:** Advanced configuration options
2. **Chart Upgrades:** Update deployed applications
3. **Helm Templates:** Understanding and creating charts
4. **Dependencies:** Managing chart relationships
5. **Advanced Release Management:** Hooks, testing, policies

---

## References

- [Bitnami Applications](https://bitnami.com/stacks/helm)
- [ArtifactHub Chart Discovery](https://artifacthub.io/)
- [Helm Official Documentation](https://helm.sh/docs/)
- [Helm CLI Reference](https://helm.sh/docs/helm/)
