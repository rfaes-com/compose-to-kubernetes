# Flux CLI Reference

Quick reference for Flux CLI commands used in GitOps workflows.

## Installation

```bash
# Install Flux CLI (Linux)
curl -s https://fluxcd.io/install.sh | sudo bash

# Verify installation
flux version

# Enable shell completions
. <(flux completion bash)
```

## Bootstrap

Bootstrap installs Flux on a cluster and configures it to sync from a Git repository.

```bash
# Bootstrap with GitHub
flux bootstrap github \
  --owner=<org-or-user> \
  --repository=<repo-name> \
  --branch=main \
  --path=clusters/my-cluster \
  --personal

# Bootstrap with GitLab
flux bootstrap gitlab \
  --owner=<group> \
  --repository=<repo-name> \
  --branch=main \
  --path=clusters/my-cluster

# Bootstrap with a generic Git server
flux bootstrap git \
  --url=ssh://git@github.com/<org>/<repo> \
  --branch=main \
  --path=clusters/my-cluster

# Check prerequisites before bootstrapping
flux check --pre
```

## Checking Status

```bash
# Check Flux system health
flux check

# Get all Flux resources
flux get all
flux get all -A                  # All namespaces

# Get specific resource types
flux get sources git
flux get sources helm
flux get sources oci
flux get kustomizations
flux get helmreleases
flux get helmcharts
flux get receivers
flux get alerts
flux get providers
flux get imageupdateautomations
flux get imagerepositories
flux get imagepolicies
```

## Sources

Sources define where Flux fetches configuration from.

```bash
# Create a GitRepository source
flux create source git my-app \
  --url=https://github.com/org/my-app \
  --branch=main \
  --interval=1m

# Create a GitRepository with SSH key
flux create source git my-app \
  --url=ssh://git@github.com/org/my-app \
  --branch=main \
  --secret-ref=my-ssh-key

# Create a HelmRepository source
flux create source helm bitnami \
  --url=https://charts.bitnami.com/bitnami \
  --interval=10m

# Create an OCI source
flux create source oci my-app \
  --url=oci://ghcr.io/org/manifests/my-app \
  --tag=latest \
  --interval=5m

# Export source as YAML
flux export source git my-app
```

## Kustomizations

Kustomizations define how to apply manifests from a source.

```bash
# Create a Kustomization
flux create kustomization my-app \
  --source=GitRepository/my-app \
  --path=./deploy \
  --prune=true \
  --interval=10m

# Create a Kustomization with health checks
flux create kustomization my-app \
  --source=GitRepository/my-app \
  --path=./deploy \
  --prune=true \
  --health-check="Deployment/my-app.default" \
  --health-check-timeout=5m

# Create a Kustomization targeting a specific namespace
flux create kustomization my-app \
  --source=GitRepository/my-app \
  --path=./deploy \
  --prune=true \
  --target-namespace=production

# Export Kustomization as YAML
flux export kustomization my-app
```

## Helm Releases

```bash
# Create a HelmRelease
flux create helmrelease my-app \
  --chart=my-app \
  --source=HelmRepository/bitnami \
  --chart-version=">=1.0.0" \
  --interval=10m

# Create a HelmRelease with values
flux create helmrelease my-app \
  --chart=nginx \
  --source=HelmRepository/bitnami \
  --values=./values.yaml

# Create a HelmRelease targeting a namespace
flux create helmrelease my-app \
  --chart=nginx \
  --source=HelmRepository/bitnami \
  --target-namespace=web \
  --create-target-namespace=true

# Export HelmRelease as YAML
flux export helmrelease my-app
```

## Reconciliation

Force Flux to immediately sync resources without waiting for the interval.

```bash
# Reconcile all Flux resources
flux reconcile source git flux-system
flux reconcile kustomization flux-system

# Reconcile a specific source
flux reconcile source git my-app

# Reconcile a Kustomization
flux reconcile kustomization my-app

# Reconcile a HelmRelease
flux reconcile helmrelease my-app

# Reconcile with reset (forces re-apply)
flux reconcile kustomization my-app --with-source
```

## Suspend and Resume

```bash
# Suspend reconciliation (stop syncing)
flux suspend kustomization my-app
flux suspend helmrelease my-app
flux suspend source git my-app

# Resume reconciliation
flux resume kustomization my-app
flux resume helmrelease my-app
flux resume source git my-app
```

## Logs

```bash
# Stream Flux controller logs
flux logs

# Filter by source
flux logs --source=GitRepository/my-app

# Filter by level
flux logs --level=error

# Follow logs
flux logs --follow

# Show logs from all controllers
flux logs --all-namespaces
```

## Events

```bash
# Watch Flux events
flux events

# Filter by resource
flux events --for=Kustomization/my-app
flux events --for=HelmRelease/my-app

# Watch live events
flux events --watch
```

## Image Automation

Automatically update image tags in Git when new images are published.

```bash
# Create an ImageRepository (scan image tags)
flux create image repository my-app \
  --image=ghcr.io/org/my-app \
  --interval=5m

# Create an ImagePolicy (select which tag to use)
flux create image policy my-app \
  --image-ref=my-app \
  --select-semver=">=1.0.0"

# Create an ImageUpdateAutomation
flux create image update flux-system \
  --git-repo-ref=flux-system \
  --git-repo-path=./clusters/my-cluster \
  --checkout-branch=main \
  --push-branch=main \
  --author-name=fluxbot \
  --author-email=fluxbot@example.com \
  --commit-template="chore: update images"
```

## Notifications

Send alerts when Flux events occur.

```bash
# Create a Slack provider
flux create alert-provider slack \
  --type=slack \
  --channel=general \
  --secret-ref=slack-url

# Create an alert
flux create alert my-alert \
  --event-severity=error \
  --event-source=GitRepository/* \
  --provider-ref=slack

# Create a GitHub commit status provider
flux create alert-provider github-status \
  --type=githubcommitstatuses \
  --secret-ref=github-token
```

## Secrets

```bash
# Create a secret for private Git repository (SSH)
flux create secret git my-ssh-key \
  --url=ssh://git@github.com/org/repo \
  --private-key-file=./id_rsa

# Create a secret for private container registry
flux create secret oci my-registry \
  --url=ghcr.io \
  --username=<user> \
  --password=<token>

# Create a secret for Helm repository with basic auth
flux create secret helm my-helm-auth \
  --username=<user> \
  --password=<password>
```

## Diff and Dry Run

```bash
# Show what would change if Flux reconciled now
flux diff kustomization my-app --path ./deploy

# Diff a HelmRelease
flux diff helmrelease my-app --values ./values.yaml
```

## Uninstalling Flux

```bash
# Remove Flux components from the cluster
flux uninstall

# Remove Flux and delete the GitRepository source
flux uninstall --namespace=flux-system
```

## Typical GitOps Repository Structure

```text
fleet-infra/
├── clusters/
│   ├── production/
│   │   ├── flux-system/        # Flux bootstrap manifests
│   │   │   ├── gotk-components.yaml
│   │   │   ├── gotk-sync.yaml
│   │   │   └── kustomization.yaml
│   │   ├── apps.yaml           # Kustomization pointing to apps/
│   │   └── infrastructure.yaml # Kustomization pointing to infra/
│   └── staging/
│       └── ...
├── apps/
│   ├── base/
│   │   └── my-app/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── kustomization.yaml
│   ├── production/
│   │   └── my-app/
│   │       ├── kustomization.yaml  # Patches for production
│   │       └── values.yaml
│   └── staging/
│       └── my-app/
│           └── kustomization.yaml  # Patches for staging
└── infrastructure/
    ├── controllers/
    │   └── ingress-nginx/
    │       ├── helmrelease.yaml
    │       └── kustomization.yaml
    └── configs/
        └── cluster-issuers.yaml
```

## Common Annotations

```yaml
# Force reconciliation via annotation
metadata:
  annotations:
    reconcile.fluxcd.io/requestedAt: "2024-01-01T00:00:00Z"

# Disable garbage collection for specific resources
metadata:
  labels:
    kustomize.toolkit.fluxcd.io/prune: "disabled"
```
