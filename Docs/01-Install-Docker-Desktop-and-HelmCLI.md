# Docker Desktop and Helm CLI Installation Guide

## Overview
This document covers the complete setup process for installing Docker Desktop and Helm CLI, which are essential prerequisites for working with Kubernetes and Helm package management. This section establishes the foundational environment needed for all subsequent Helm operations.

## Prerequisites
- macOS or Windows operating system
- System administrator access for installation
- Internet connection for downloading installers
- Docker Hub account

---

## Part 1: Docker Desktop Installation

### Introduction to Docker Desktop
Docker Desktop is a containerization platform that includes:
- Docker Engine (containerization runtime)
- Docker CLI for managing containers
- Kubernetes cluster support (embedded)
- Docker Compose for multi-container orchestration

### Docker Desktop Setup

#### Pricing and Account Creation
- **Pricing Information:** [Docker Desktop Pricing](https://www.docker.com/pricing/)
- **Create Account:** [Docker Hub SignUp](https://hub.docker.com/)
- **Download:** [Docker Desktop Downloads](https://www.docker.com/products/docker-desktop/)

#### Installation on macOS
```bash
# Method: Direct Installation
# 1. Download Docker Desktop for Mac from docker.com
# 2. Copy Docker.dmg to Applications folder
# 3. Double-click Docker.app in Applications
# 4. Enter system password when prompted
# 5. Wait for Docker to start (look for Docker icon in menu bar)

# Create Docker Hub Account
# Navigate to: https://hub.docker.com
# Complete registration with email and password

# Sign in to Docker Desktop
# 1. Open Docker Desktop application
# 2. Click menu bar icon -> Sign in
# 3. Enter Docker Hub credentials
# 4. Verify sign-in success
```

#### Installation on Windows
```bash
# Method: Installer
# 1. Download Docker Desktop Installer (.exe) from docker.com
# 2. Double-click Docker Desktop Installer.exe
# 3. Follow installation wizard
# 4. Enable required features (Hyper-V or WSL2)
# 5. Restart computer when prompted
# 6. Launch Docker Desktop from Start menu

# Create Docker Hub Account
# Navigate to: https://hub.docker.com
# Complete registration with email and password

# Sign in to Docker Desktop
# 1. Open Docker Desktop application
# 2. Click system tray icon -> Sign in
# 3. Enter Docker Hub credentials
# 4. Verify sign-in success

# Configure kubectl CLI on Windows PATH
# Add C:\Program Files\Docker\Docker\Resources\bin to system PATH
# Verify: Open new Command Prompt/PowerShell and type: kubectl version
```

---

## Part 2: Enable Kubernetes Cluster in Docker Desktop

### Kubernetes Integration
Docker Desktop includes an embedded Kubernetes cluster that provides a lightweight local development environment.

**Reference:** [Docker Desktop - Kubernetes Cluster Documentation](https://docs.docker.com/desktop/kubernetes/)

### Enable Kubernetes Cluster
```bash
# Step-by-step enablement:
# 1. Open Docker Desktop application
# 2. Navigate to: Settings (or Preferences on macOS)
# 3. Go to: Kubernetes tab/section
# 4. Check: "Enable Kubernetes" checkbox
# 5. Click: "Apply & Restart" button
# 6. When prompted: "Install required images"? -> Click "Install"
# 7. Wait for Kubernetes cluster initialization (5-10 minutes)
# 8. Verify Docker Desktop status in menu bar shows green indicator

# The cluster is ready when:
# - Docker icon shows green/running state
# - No error messages in application
# - kubectl commands respond successfully
```

---

## Part 3: Configure and Verify kubectl

### kubectl Configuration
kubectl is the command-line tool for managing Kubernetes clusters. Docker Desktop automatically installs kubectl.

```bash
# Verify kubectl is installed
which kubectl
# Output: /usr/local/bin/kubectl (or similar path)

# Check kubectl version
kubectl version
# Shows both client and server version information

# Display short version format
kubectl version --short
# Output: Client Version: v1.x.x, Server Version: v1.x.x

# Get detailed YAML format output
kubectl version --client --output=yaml
# Shows comprehensive client version details

# List all configured contexts
kubectl config get-contexts
# Shows all Kubernetes cluster contexts available locally
# Example output:
# CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
# *         docker-desktop   docker-desktop   docker-desktop   default
# (asterisk indicates current context)

# Display currently active context
kubectl config current-context
# Output: docker-desktop (for Docker Desktop k8s)

# Switch to Docker Desktop context (if not currently active)
# Run only if multiple contexts exist
kubectl config use-context docker-desktop

# Verify Kubernetes cluster is running
kubectl get nodes
# Output shows:
# NAME             STATUS   ROLES            AGE   VERSION
# docker-desktop   Ready    control-plane    XXh   vX.XX.X
```

### Understanding kubectl Context
- **Context:** A configuration that specifies which Kubernetes cluster to connect to
- **Default Context:** docker-desktop when using Docker Desktop
- **Multiple Contexts:** Useful when managing multiple clusters (local, dev, prod)

---

## Part 4: Verify Kubernetes Cluster with Sample Application

### Reference Resources
- **StackSimplify Docker Images:** [Docker Packages](https://github.com/stacksimplify?tab=packages)
- **Sample Application Image:** [kubenginxhelm Container Image](https://github.com/users/stacksimplify/packages/container/package/kubenginxhelm)

### Kubernetes Manifests Structure
The sample application uses two manifest files:

**deployment.yaml:**
```yaml
# Defines Kubernetes Deployment resource
# - Container image specification
# - Number of replicas (pod instances)
# - Resource requests and limits
# - Container port definitions
```

**service.yaml:**
```yaml
# Defines Kubernetes Service resource
# - Service type (ClusterIP, NodePort, LoadBalancer)
# - Port mapping configuration
# - Label selectors for pod targeting
```

### Deploy Sample Application

```bash
# Navigate to manifest directory
cd kube-manifests

# Review manifest files before deployment
cat deployment.yaml
cat service.yaml

# Deploy all resources (same directory)
kubectl apply -f kube-manifests/

# Alternative: Deploy individual files
kubectl apply -f kube-manifests/deployment.yaml
kubectl apply -f kube-manifests/service.yaml

# Verify Deployment creation
kubectl get deploy
# Output shows: deployment name, desired/ready replicas, image used

# Verify Pods are running
kubectl get pods
# Output shows pod names, ready state, status, restarts, age
# Example: NAME                READY   STATUS    RESTARTS   AGE
#          myapp-xxx-yyy   1/1     Running   0          2m

# Verify Service creation
kubectl get svc
# Output shows service name, type, cluster-ip, external-ip, ports
# Example: myapp-service    NodePort   10.x.x.x   <none>   8080:31300/TCP

# Get detailed service information
kubectl get svc -o wide
# Shows additional columns including selector labels and endpoints
```

### Access the Application

```bash
# Once Service is running, access via NodePort
# For Docker Desktop (local development):
http://localhost:31300
# or
http://127.0.0.1:31300

# Check application status
# Should see customized nginx page or application homepage

# View pod logs
kubectl logs <pod-name>
# Example: kubectl logs myapp-deployment-6d8b5f9c8d-7xd9f

# Describe pod for troubleshooting
kubectl describe pod <pod-name>
# Shows detailed pod information, events, and error messages

# Port forward for local access (alternative to NodePort)
kubectl port-forward svc/<service-name> 8080:80
# Access via: http://localhost:8080
```

---

## Part 5: Helm CLI Installation

### Helm Overview
Helm is the package manager for Kubernetes, simplifying application deployment and management.

**Key Helm Concepts:**
- **Charts:** Helm packages containing Kubernetes manifests
- **Releases:** Deployed instances of charts
- **Repositories:** Collections of charts
- **Values:** Configuration parameters for charts

### Helm Installation

#### On macOS
```bash
# Using Homebrew (recommended)
brew install helm

# Verify Helm installation
helm version
# Output: version.BuildInfo{Version:"v3.x.x", ... }

# Check Helm CLI location
which helm
# Output: /usr/local/bin/helm
```

#### On Windows
```bash
# Using Chocolatey
choco install kubernetes-helm

# Or download from: https://github.com/helm/helm/releases
# Extract to a directory in your PATH

# Verify Helm installation
helm version
# Output: version.BuildInfo{Version:"v3.x.x", ... }
```

### Verify Helm Configuration

```bash
# Check Helm version and release information
helm version --long
# Shows detailed version information

# Display directory structure for Helm
helm env
# Shows HELM_HOME and other environment variables

# List Helm repositories (initially empty)
helm repo list
# Output: (no repositories installed)

# Add official Helm repository
helm repo add stable https://charts.helm.sh/stable

# Update repository index
helm repo update

# Search for available charts
helm search repo stable
# Lists available charts in stable repository
```

---

## Troubleshooting Common Issues

### Docker Desktop Issues
| Issue | Resolution |
|-------|-----------|
| Docker won't start | Restart computer, check system resources, reinstall Docker |
| Kubernetes cluster won't enable | Ensure virtualization enabled in BIOS, update Docker Desktop |
| kubectl command not found | Add Docker bin directory to system PATH |

### Kubernetes Issues
| Issue | Resolution |
|-------|-----------|
| Pods stuck in Pending | Check node resources, verify image availability |
| Service not accessible | Verify service type, check NodePort assignment |
| Connection refused | Ensure cluster is running, check port availability |

### Helm Issues
| Issue | Resolution |
|-------|-----------|
| Helm command not found | Verify installation, check PATH configuration |
| Repository connection failed | Check internet connection, verify repository URL |
| Chart not found | Update repositories, verify chart name spelling |

---

## Summary

This foundational setup section covers:
1. ✅ Docker Desktop installation and configuration
2. ✅ Kubernetes cluster enablement in Docker Desktop
3. ✅ kubectl configuration and verification
4. ✅ Kubernetes cluster validation with sample application
5. ✅ Helm CLI installation
6. ✅ Initial Helm configuration

**Next Steps:** Once this environment is set up, you're ready to proceed with Helm installation, chart management, and Kubernetes deployments using Helm.

---

## References
- [Docker Desktop Documentation](https://docs.docker.com/desktop/)
- [Kubernetes Installation](https://kubernetes.io/docs/setup/)
- [Helm Official Documentation](https://helm.sh/docs/)
- [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
