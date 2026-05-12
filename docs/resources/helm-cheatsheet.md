# Helm Cheatsheet

Quick reference for essential Helm commands and concepts.

## Installation

```bash
# Install Helm (Linux)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

## Repositories

```bash
# Add a repository
helm repo add <name> <url>
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami

# List repositories
helm repo list

# Update repository index
helm repo update

# Remove a repository
helm repo remove <name>

# Search for charts
helm search repo <keyword>
helm search repo nginx
helm search hub <keyword>        # Search Artifact Hub
```

## Installing Charts

```bash
# Install a chart
helm install <release-name> <chart>
helm install my-nginx bitnami/nginx

# Install in a specific namespace
helm install my-nginx bitnami/nginx --namespace web --create-namespace

# Install with custom values
helm install my-nginx bitnami/nginx --values values.yaml
helm install my-nginx bitnami/nginx --set replicaCount=3
helm install my-nginx bitnami/nginx --set image.tag=1.25,replicaCount=2

# Install a specific chart version
helm install my-nginx bitnami/nginx --version 15.0.0

# Dry run (preview without installing)
helm install my-nginx bitnami/nginx --dry-run
helm install my-nginx bitnami/nginx --dry-run --debug

# Generate manifests without installing
helm template my-nginx bitnami/nginx
helm template my-nginx bitnami/nginx --values values.yaml
```

## Managing Releases

```bash
# List releases
helm list
helm list -A                     # All namespaces
helm list --namespace web

# Get release status
helm status <release-name>

# Get release history
helm history <release-name>

# Get release values
helm get values <release-name>
helm get values <release-name> --all   # Include default values

# Get rendered manifests for a release
helm get manifest <release-name>
```

## Upgrading Releases

```bash
# Upgrade a release
helm upgrade <release-name> <chart>
helm upgrade my-nginx bitnami/nginx

# Upgrade with new values
helm upgrade my-nginx bitnami/nginx --values values.yaml
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# Install if not exists, upgrade if exists
helm upgrade --install my-nginx bitnami/nginx

# Reuse existing values during upgrade
helm upgrade my-nginx bitnami/nginx --reuse-values

# Rollback on failure
helm upgrade my-nginx bitnami/nginx --atomic
helm upgrade my-nginx bitnami/nginx --atomic --timeout 5m
```

## Rolling Back

```bash
# Rollback to previous revision
helm rollback <release-name>

# Rollback to specific revision
helm rollback <release-name> <revision>
helm rollback my-nginx 2

# Dry run rollback
helm rollback my-nginx 2 --dry-run
```

## Uninstalling Releases

```bash
# Uninstall a release
helm uninstall <release-name>
helm uninstall my-nginx

# Uninstall but keep history
helm uninstall my-nginx --keep-history
```

## Chart Information

```bash
# Show chart information
helm show chart bitnami/nginx
helm show values bitnami/nginx      # Default values
helm show readme bitnami/nginx
helm show all bitnami/nginx

# Download chart locally
helm pull bitnami/nginx
helm pull bitnami/nginx --untar     # Extract after download
helm pull bitnami/nginx --version 15.0.0
```

## Creating Charts

```bash
# Create a new chart scaffold
helm create <chart-name>
helm create my-app

# Validate chart syntax
helm lint <chart-directory>
helm lint ./my-app

# Package chart into a .tgz archive
helm package <chart-directory>
helm package ./my-app
```

## Chart Structure

```text
my-app/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── charts/             # Chart dependencies
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # Template helpers / partials
│   ├── NOTES.txt       # Post-install notes
│   └── tests/
│       └── test-connection.yaml
└── .helmignore         # Files to ignore when packaging
```

## Chart.yaml Fields

```yaml
apiVersion: v2
name: my-app
description: A Helm chart for my application
type: application        # application | library
version: 0.1.0           # Chart version (SemVer)
appVersion: "1.0.0"      # App version (informational)
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

## Templating Basics

```yaml
# Reference values
{{ .Values.replicaCount }}
{{ .Values.image.repository }}

# Release metadata
{{ .Release.Name }}
{{ .Release.Namespace }}
{{ .Release.IsInstall }}
{{ .Release.IsUpgrade }}

# Chart metadata
{{ .Chart.Name }}
{{ .Chart.Version }}
{{ .Chart.AppVersion }}

# Conditionals
{{- if .Values.ingress.enabled }}
# ... ingress resource
{{- end }}

# Loops
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}

# Default values
{{ .Values.replicas | default 1 }}

# Quote strings
{{ .Values.image.tag | quote }}

# Include helper template
{{ include "my-app.fullname" . }}
```

## Managing Dependencies

```bash
# Update chart dependencies
helm dependency update <chart-directory>
helm dependency update ./my-app

# List dependencies
helm dependency list ./my-app

# Build dependencies (from Chart.lock)
helm dependency build ./my-app
```

## Testing

```bash
# Run chart tests
helm test <release-name>

# Run tests with logs
helm test my-nginx --logs
```

## Plugins

```bash
# List installed plugins
helm plugin list

# Install a plugin
helm plugin install <url>
helm plugin install https://github.com/databus23/helm-diff

# Popular plugins
# helm-diff    - Show diff between releases
# helm-secrets - Manage encrypted secrets
# helm-push    - Push charts to OCI registries
```

## OCI Registry Support

```bash
# Login to OCI registry
helm registry login <registry>

# Push chart to OCI registry
helm push my-app-0.1.0.tgz oci://registry.example.com/charts

# Install from OCI registry
helm install my-app oci://registry.example.com/charts/my-app --version 0.1.0

# Pull from OCI registry
helm pull oci://registry.example.com/charts/my-app --version 0.1.0
```

## Useful Flags

| Flag                 | Description                              |
| -------------------- | ---------------------------------------- |
| `--namespace` / `-n` | Target namespace                         |
| `--create-namespace` | Create namespace if it doesn't exist     |
| `--values` / `-f`    | Specify values file                      |
| `--set`              | Override individual values               |
| `--dry-run`          | Simulate without applying                |
| `--debug`            | Enable verbose output                    |
| `--atomic`           | Roll back on failure                     |
| `--timeout`          | Time to wait for operations (default 5m) |
| `--wait`             | Wait for pods to be ready                |
| `--version`          | Specify chart version                    |
| `--output` / `-o`    | Output format (table, json, yaml)        |
