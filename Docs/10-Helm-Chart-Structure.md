# Helm Chart Structure and Organization

## Overview
This module covers the anatomy of a Helm chart - the standardized directory structure that defines a Kubernetes application package. Understanding chart structure is fundamental to both using and creating Helm charts.

## Core Concepts

### What is a Helm Chart?

**Definition:** A collection of files that describe a set of Kubernetes resources

**Key Characteristics:**
- Organized directory format
- Contains templates and configuration
- Reusable and shareable
- Can be versioned
- Includes metadata and documentation

### Chart as Package

A Helm chart is like:
- **RPM/DEB:** Package format for applications
- **Docker Image:** Containerized application
- **Template Library:** Reusable infrastructure code

---

## Part 1: Creating a Chart

### Generate Chart Using helm create

```bash
# Command to create new chart
helm create <CHART-NAME>

# Example: Create a chart named basechart
helm create basechart
# Creates complete chart structure with default templates

# Verify creation
ls -la basechart/
# Shows chart directory created
```

**What `helm create` Provides:**
- Complete directory structure
- Default configuration files
- Sample templates
- Best practice examples
- Ready-to-use starting point

---

## Part 2: Chart Directory Structure

### Complete Chart Layout

```
basechart/                          # Chart root directory
├── .helmignore                    # Files to ignore (like .gitignore)
├── Chart.yaml                     # Chart metadata
├── LICENSE                        # License file
├── README.md                      # Chart documentation
├── values.yaml                    # Default configuration values
├── charts/                        # Subchart dependencies
│   └── (Subcharts go here)
├── templates/                     # Kubernetes manifest templates
│   ├── NOTES.txt                 # Installation notes
│   ├── _helpers.tpl              # Template helper functions
│   ├── deployment.yaml           # Deployment template
│   ├── hpa.yaml                  # Horizontal Pod Autoscaling
│   ├── ingress.yaml              # Ingress configuration
│   ├── service.yaml              # Service template
│   ├── serviceaccount.yaml       # ServiceAccount template
│   └── tests/                    # Test templates
│       └── test-connection.yaml  # Test pod definition
└── (Optional directories)
    ├── .gitignore                # Git ignore patterns
    └── charts.lock               # Dependency lock file
```

---

## Part 3: Root-Level Files

### Chart.yaml - Chart Metadata

**Purpose:** Defines chart identity and version information

```yaml
apiVersion: v2
name: basechart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
keywords:
  - word1
  - word2
home: https://example.com
sources:
  - https://github.com/example/basechart
maintainers:
  - name: Name
    email: name@example.com
```

**Key Fields:**
| Field | Purpose | Example |
|-------|---------|---------|
| apiVersion | Helm API version | v2 (API v2) or v1 (API v1) |
| name | Chart name | basechart |
| description | Chart description | A Helm chart for Kubernetes |
| type | Chart type | application or library |
| version | Chart version (semver) | 0.1.0 |
| appVersion | Application version | 1.16.0 |

### values.yaml - Default Configuration

**Purpose:** Provides default values for chart templates

```yaml
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
```

### .helmignore - File Exclusion

**Purpose:** Specify files/patterns to exclude from chart package

```
# Comments start with #
OWNERS
.git
.gitignore
.bzr
.bzrignore
.hg
.hgignore
.svn
*.swp
*.swo
*~
.DS_Store
.idea/
*.iml
*.pyc
*.pyo
.env
```

**Similar to:** .gitignore for Helm packages

### README.md - Documentation

**Purpose:** Chart documentation for users

Typical contents:
- Chart description
- Installation instructions
- Configuration options
- Usage examples
- Requirements
- Support information

### LICENSE - Legal

**Purpose:** License terms for the chart

Common options:
- Apache 2.0
- MIT
- Commercial license

---

## Part 4: Templates Directory

### Purpose of templates/

**Overview:** Contains Kubernetes manifest templates

**Key Concept:** Templated YAML files that Helm renders with values

```
templates/
├── NOTES.txt              # Post-install notes
├── deployment.yaml        # Deployment resource
├── service.yaml          # Service resource
├── serviceaccount.yaml   # ServiceAccount resource
├── ingress.yaml          # Ingress resource
├── hpa.yaml             # Horizontal Pod Autoscaler
├── _helpers.tpl         # Helper template functions
└── tests/
    └── test-connection.yaml  # Helm test
```

### deployment.yaml Template

**Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "basechart.fullname" . }}
  labels:
    {{- include "basechart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "basechart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "basechart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
```

**Template Features:**
- `{{ }}` - Helm template expressions
- `.Values` - Access to values.yaml
- `.Chart` - Access to chart metadata
- Conditionals and loops
- Functions and filters

### service.yaml Template

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "basechart.fullname" . }}
  labels:
    {{- include "basechart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "basechart.selectorLabels" . | nindent 4 }}
```

### _helpers.tpl - Template Helpers

**Purpose:** Define reusable template functions

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "basechart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "basechart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "basechart.labels" -}}
helm.sh/chart: {{ include "basechart.chart" . }}
{{ include "basechart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "basechart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "basechart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### NOTES.txt - Post-Install Notes

**Purpose:** Display information after successful installation

```
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "basechart.name" . }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
```

---

## Part 5: Charts Directory

### Purpose

**Charts Directory:** Contains Helm chart dependencies (subcharts)

```
charts/
├── subchart1/   # Dependency chart 1
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── subchart2/   # Dependency chart 2
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
└── Chart.lock   # Dependency lock file
```

### Dependency Management

```yaml
# In main chart's Chart.yaml
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: "https://charts.bitnami.com/bitnami"
  - name: redis
    version: "16.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

---

## Part 6: Tests Directory

### Purpose

**Tests Directory:** Contains test pod definitions

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "basechart.fullname" . }}-test-connection"
  labels:
    {{- include "basechart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "basechart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

**Usage:**
```bash
# Run Helm tests
helm test <RELEASE-NAME>
# Tests verify deployment works correctly
```

---

## Part 7: File Organization Best Practices

### Template Files

```
templates/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── configmap.yaml      # Configuration data
├── secret.yaml         # Sensitive data
├── statefulset.yaml    # Stateful applications
├── job.yaml            # One-time tasks
├── cronjob.yaml        # Scheduled tasks
├── _helpers.tpl        # Helpers at top
├── NOTES.txt
└── tests/
    └── test-connection.yaml
```

### Organization Patterns

```
# Pattern 1: Resource-type based (shown above)
# Clear separation by Kubernetes resource type

# Pattern 2: Feature-based
templates/
├── authentication/
├── monitoring/
├── logging/
└── core/

# Pattern 3: Namespaced templates
templates/
├── namespace/
├── rbac/
└── application/
```

---

## Part 8: Creating Chart from Scratch

### Manual Chart Creation

```bash
# Create directory
mkdir mychart
cd mychart

# Create Chart.yaml
cat > Chart.yaml <<EOF
apiVersion: v2
name: mychart
version: 0.1.0
appVersion: "1.0"
EOF

# Create values.yaml
cat > values.yaml <<EOF
replicaCount: 1
image:
  repository: nginx
  tag: latest
EOF

# Create templates directory
mkdir -p templates

# Create simple deployment template
cat > templates/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: app
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
EOF

# Test the chart
helm template . --debug
helm lint .
```

---

## Part 9: Chart Validation

### Lint Chart

```bash
# Validate chart structure and syntax
helm lint basechart

# Output:
# ==> Linting basechart
# [INFO] Chart.yaml: icon is recommended
# [WARNING] Chart.yaml: chart type is often 'application' or 'library'
# [OK] values.yaml is a valid JSON Schema
# [OK] basechart appears to be a valid chart
```

### Render Templates

```bash
# Render templates without installing
helm template my-release basechart

# Render with specific values
helm template my-release basechart -f values-custom.yaml

# Debug rendering
helm template my-release basechart --debug
```

### Dry-Run Install

```bash
# Preview what would be installed
helm install my-release basechart --dry-run --debug

# Shows all resources that would be created
```

---

## Part 10: Chart Versioning

### Version Scheme

```yaml
# Chart.yaml
version: 1.2.3        # Chart version (Semantic Versioning)
appVersion: "2.5.0"   # Application version inside chart

# Semantic Versioning (MAJOR.MINOR.PATCH):
# 1.0.0 - Initial release
# 1.1.0 - New features, backward compatible
# 1.1.1 - Bug fixes
# 2.0.0 - Breaking changes
```

### Version Commands

```bash
# Check chart version
helm show chart basechart | grep version

# Update chart version after changes
# Edit Chart.yaml and update version field

# List chart history
helm history my-release
# Shows version deployments
```

---

## Summary

Chart Structure Essentials:

| Component | Purpose | Editable |
|-----------|---------|----------|
| Chart.yaml | Metadata | Yes |
| values.yaml | Defaults | Yes |
| templates/ | K8s manifests | Yes |
| _helpers.tpl | Functions | Yes |
| NOTES.txt | Instructions | Yes |
| charts/ | Dependencies | Yes |
| .helmignore | Exclusions | Yes |
| LICENSE | Legal | No |
| README.md | Documentation | Yes |

---

## References

- [Helm Chart Structure](https://helm.sh/docs/topics/charts/)
- [Helm Charts Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Chart Metadata](https://helm.sh/docs/topics/charts/#the-chartyaml-file)
