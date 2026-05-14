# Helm - The Package Manager for Kubernetes

Duration: 50 minutes (25 minutes theory + 25 minutes lab)

## Introduction

Managing Kubernetes applications with raw YAML manifests can become complex:

- Multiple manifests per application (Deployment, Service, ConfigMap, Ingress, etc.)
- Duplicate configuration across environments (dev, staging, prod)
- No versioning or rollback mechanism
- Difficult to share and distribute applications

**Helm** solves these problems by providing package management for Kubernetes.

## What is Helm?

Helm is a package manager for Kubernetes that:

- Packages Kubernetes applications as **Charts**
- Provides templating for manifest customization
- Manages application lifecycle (install, upgrade, rollback)
- Handles release versioning
- Enables sharing via chart repositories

Think of Helm as:

- **apt/yum** for Kubernetes
- **npm/pip** for K8s applications
- A deployment tool with version control

## Core

 Concepts

### Chart
A Helm package containing all resources needed to run an application:

```text
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration
├── charts/             # Dependencies
└── templates/          # Kubernetes manifest templates
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

### Release
An instance of a chart running in a Kubernetes cluster.
One chart can have multiple releases (e.g., `myapp-dev`, `myapp-prod`).

### Repository
A collection of charts that can be shared and downloaded.
Public repos: **ArtifactHub**, Bitnami, etc.

## Helm Architecture

Helm 3 architecture (simplified from Helm 2):

```mermaid
flowchart TB
    Helm["Helm CLI<br/>(your machine)"]

    subgraph Cluster["Kubernetes Cluster"]
        direction TB

        Releases["Releases<br/>(stored as Secrets)"]
    end

    Helm -->|kubectl-like| Releases
```

**Note:** Helm 3 removed Tiller (server-side component) for better security.

## Installing Helm

### Workshop Container (already installed)

```bash
helm version
```

### Manual Installation

```bash
# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# MacOS
brew install helm

# Windows
choco install kubernetes-helm
```

## Basic Helm Commands

```bash
# Search for charts
helm search hub wordpress
helm search repo bitnami

# Install a chart
helm install myrelease chartname

# List releases
helm list

# Upgrade a release
helm upgrade myrelease chartname

# Rollback to previous version
helm rollback myrelease

# Uninstall a release
helm uninstall myrelease

# Show release history
helm history myrelease
```

## Chart Structure

A typical Helm chart:

```text
mychart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default values
├── charts/                 # Chart dependencies
├── templates/              # Kubernetes manifests
│   ├── NOTES.txt          # Post-install instructions
│   ├── _helpers.tpl       # Template helpers
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── tests/
│       └── test-connection.yaml
└── .helmignore            # Patterns to ignore
```

### Chart.yaml

Defines chart metadata:

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for my application
type: application
version: 1.0.0        # Chart version
appVersion: "1.16.0"  # Application version

maintainers:
  - name: Your Name
    email: you@example.com

dependencies:
  - name: postgresql
    version: 12.x.x
    repository: https://charts.bitnami.com/bitnami
```

### values.yaml

Default configuration values:

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25-alpine"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  hosts:
    - host: example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### templates/deployment.yaml

Templated Kubernetes manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

## Templating

Helm uses Go templates with additional functions:

### Variables

```yaml
# Access values from values.yaml
{{ .Values.replicaCount }}
{{ .Values.image.repository }}

# Access chart metadata
{{ .Chart.Name }}
{{ .Chart.Version }}

# Access release information
{{ .Release.Name }}
{{ .Release.Namespace }}
```

### Control Structures

```yaml
# If/else
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# Range (loop)
{{- range .Values.environments }}
- name: {{ . }}
{{- end }}

# With (scope)
{{- with .Values.service }}
type: {{ .type }}
port: {{ .port }}
{{- end }}
```

### Functions

```yaml
# String functions
{{ .Values.name | upper }}
{{ .Values.name | lower }}
{{ .Values.name | quote }}

# Default values
{{ .Values.optional | default "default-value" }}

# Include templates
{{- include "myapp.labels" . | nindent 4 }}

# toYaml (preserve formatting)
{{- toYaml .Values.resources | nindent 10 }}
```

### Pipelines

```yaml
# Chain functions
{{ .Values.name | upper | quote }}
{{ .Values.value | default "none" | quote }}
```

## Named Templates (_helpers.tpl)

Reusable template snippets:

```yaml
{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create a default fully qualified app name
*/}}
{{- define "myapp.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
```

Usage in templates:

```yaml
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
```

## Creating a Chart

### Using `helm create`

```bash
# Generate chart scaffold
helm create myapp

# Chart structure created
myapp/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── ...
```

### Directory Structure

```bash
# Navigate to your chart
cd myapp

# Edit values.yaml for defaults
vi values.yaml

# Edit templates
vi templates/deployment.yaml
```

## Installing a Chart

### From local directory

```bash
helm install myrelease ./myapp
```

### With custom values

```bash
# Using --set flag
helm install myrelease ./myapp \
  --set replicaCount=5 \
  --set image.tag=1.26-alpine

# Using custom values file
helm install myrelease ./myapp \
  -f custom-values.yaml
```

### Dry run (preview)

```bash
helm install myrelease ./myapp --dry-run --debug
```

### Different namespace

```bash
helm install myrelease ./myapp --namespace production --create-namespace
```

## Upgrading Releases

```bash
# Upgrade with new values
helm upgrade myrelease ./myapp \
  --set replicaCount=10

# Upgrade and install if not exists
helm upgrade --install myrelease ./myapp

# Wait for resources to be ready
helm upgrade myrelease ./myapp --wait --timeout=5m
```

## Values Precedence

Values are merged in this order (lowest to highest priority):

1. Default `values.yaml` in chart
2. Parent chart `values.yaml` (if subchart)
3. User-supplied values file (`-f custom.yaml`)
4. Individual parameters (`--set key=value`)

Example:

```bash
# Final values = defaults + custom-values.yaml + --set overrides
helm install myrelease ./myapp \
  -f custom-values.yaml \
  --set replicas=5
```

## Rolling Back

```bash
# View revision history
helm history myrelease

REVISION  UPDATED                   STATUS      DESCRIPTION
1         Mon Jan 1 10:00:00 2024   superseded  Install complete
2         Mon Jan 1 11:00:00 2024   superseded  Upgrade complete
3         Mon Jan 1 12:00:00 2024   deployed    Upgrade complete

# Rollback to previous version
helm rollback myrelease

# Rollback to specific revision
helm rollback myrelease 2
```

## Chart Repositories

### Adding repositories

```bash
# Add Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repo index
helm repo update

# Search repo
helm search repo bitnami/nginx

# List repos
helm repo list
```

### Installing from repository:

```bash
helm install my-nginx bitnami/nginx
```

### ArtifactHub

Browse public charts at [artifacthub.io](https://artifacthub.io/)

## Chart Dependencies

Define dependencies in `Chart.yaml`:

```yaml
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled

  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

Install dependencies:

```bash
helm dependency update ./myapp
```

This downloads charts to `charts/` directory.

## Testing Charts

### Lint (validate)

```bash
helm lint ./myapp
```

### Test (from templates/tests/)

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "myapp.fullname" . }}-test"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "myapp.fullname" . }}:80']
  restartPolicy: Never
```

Run tests:

```bash
helm test myrelease
```

## Packaging and Distributing

### Package chart

```bash
helm package ./myapp

# Creates: myapp-1.0.0.tgz
```

### Install from package

```bash
helm install myrelease myapp-1.0.0.tgz
```

### Create repository index

```bash
helm repo index ./charts --url https://example.com/charts
```

## Helm vs Raw Manifests

### Raw Manifests (kubectl)

```bash
# Directory structure
myapp/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
└── configmap.yaml

# Deploy
kubectl apply -f myapp/

# Update (manual editing)
vi myapp/deployment.yaml
kubectl apply -f myapp/deployment.yaml

# No versioning or rollback
```

### Helm Charts

```bash
# Chart structure with templates
myapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── configmap.yaml

# Deploy with custom values
helm install prod myapp -f prod-values.yaml

# Upgrade
helm upgrade prod myapp --set replicas=10

# Automatic versioning and rollback
helm rollback prod
```

## Best Practices

1. **Use semantic versioning**: For both chart and app versions
2. **Document values.yaml**: Add comments explaining each value
3. **Provide sensible defaults**: Chart should work with default values
4. **Use named templates**: For reusable snippets (_helpers.tpl)
5. **Validate input**: Use `required` function for mandatory values
6. **Test thoroughly**: Use `helm lint` and `helm test`
7. **Don't hardcode**: Use values and templates
8. **Resource limits**: Always set CPU/memory limits
9. **Security**: Don't include secrets in values.yaml
10. **README**: Document chart usage and configuration

## Common Patterns

### Required Values

```yaml
{{- if not .Values.database.host }}
{{- fail "database.host is required" }}
{{- end }}

# Or inline
host: {{ required "database.host is required" .Values.database.host }}
```

### Conditional Resources

```yaml
# Only create Ingress if enabled
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}
```

### Image Pull Secrets

```yaml
{{- if .Values.imagePullSecrets }}
imagePullSecrets:
{{- range .Values.imagePullSecrets }}
  - name: {{ . }}
{{- end }}
{{- end }}
```

### Environment Variables from ConfigMap

```yaml
env:
{{- range $key, $value := .Values.env }}
  - name: {{ $key }}
    value: {{ $value | quote }}
{{- end }}
```

## Debugging

```bash
# Preview rendered templates
helm template myrelease ./myapp

# Dry run with debug output
helm install myrelease ./myapp --dry-run --debug

# Get manifest of deployed release
helm get manifest myrelease

# Get values of deployed release
helm get values myrelease

# Get all release information
helm get all myrelease
```

## Helm Hooks

Execute actions at specific points in release lifecycle:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        command: ["./migrate-db.sh"]
      restartPolicy: Never
```

Hook types:

- `pre-install`, `post-install`
- `pre-upgrade`, `post-upgrade`
- `pre-delete`, `post-delete`
- `pre-rollback`, `post-rollback`

## Key takeaways

- **Helm is the package manager for Kubernetes** — it bundles manifests into reusable Charts
- **Charts use Go templates** to generate Kubernetes manifests from configurable values
- **Releases are versioned** — you can upgrade, roll back, and inspect history with simple commands
- **`values.yaml` is the primary customisation point** — override at install or upgrade time
- **Chart repositories** (ArtifactHub, Bitnami) provide ready-made Charts for common software

## Check your understanding

1. What is the difference between a Chart, a Release, and a Repository in Helm?
2. How do you override default values when installing a Chart?
3. What command would you run to roll back a release to the previous version?
4. What file contains the default configuration for a Chart?
5. What is the purpose of `_helpers.tpl`?

<details class="solution" markdown="1">
<summary>Solution</summary>

1. **A Chart is the package (templates + defaults); a Release is a deployed instance of a Chart; a Repository is a collection of Charts hosted remotely**
2. **With `--set key=value` flags or by supplying a custom values file with `-f myvalues.yaml`**
3. **`helm rollback <release-name>` (optionally specify a revision number)**
4. **`values.yaml` in the Chart's root directory**
5. **It defines named templates (helper snippets) that can be reused across multiple template files in the Chart**

</details>

## Hands-on

Apply the concepts from this section in the [lab](lab.md) exercises.

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../03-gitops/README.md).

## Further reading

- [Helm Documentation](https://helm.sh/docs/)
- [Best Practices Guide](https://helm.sh/docs/chart_best_practices/)
- [ArtifactHub](https://artifacthub.io/)
- [Helm Chart Template Guide](https://helm.sh/docs/chart_template_guide/)
