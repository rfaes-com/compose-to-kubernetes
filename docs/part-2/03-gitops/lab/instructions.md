# GitOps Lab

Duration: 25 minutes

## Objectives

By the end of this lab, you will:

- Install and configure Flux CD
- Bootstrap a Git repository for GitOps
- Deploy applications using Kustomization
- Deploy Helm charts via HelmRelease
- Implement automated image updates
- Configure notifications

## Prerequisites

- Kubernetes cluster (Docker Desktop, kind, minikube, or cloud provider)
- kubectl configured
- Git repository (GitHub, GitLab, or Bitbucket)
- Personal access token for your Git provider

## Task 1: Install Flux CLI

### Workshop Container (already installed)

```bash
flux version
```

### Manual Installation

Install the Flux command-line tool:

```bash
# macOS/Linux
curl -s https://fluxcd.io/install.sh | sudo bash

# Or using Homebrew
brew install fluxcd/tap/flux

# Windows (using Chocolatey)
choco install flux

# Verify installation
flux --version
```

Check if your cluster is ready for Flux:

```bash
flux check --pre
```

## Task 2: Bootstrap Flux

Bootstrap Flux to your cluster and connect it to your Git repository:

```bash
# Set variables
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=fleet-infra

# Bootstrap Flux
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=./clusters/dev-cluster \
  --personal
```

This command:

- Creates the repository if it doesn't exist
- Adds Flux components to the cluster
- Configures Flux to sync from the repository
- Commits manifests to the repo

**Verify Flux installation:**

```bash
# Check Flux pods
kubectl get pods -n flux-system

# Should see:
# - source-controller
# - kustomize-controller
# - helm-controller
# - notification-controller

# Check GitRepository source
flux get sources git

# Check Kustomizations
flux get kustomizations
```

## Task 3: Deploy Application with Kustomization

Create a simple application using Kustomize:

**1. Clone your repository:**

```bash
git clone https://github.com/$GITHUB_USER/$GITHUB_REPO
cd $GITHUB_REPO
```

**2. Create application structure:**

```bash
mkdir -p apps/base
```

**3. Create `apps/base/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfo
        image: ghcr.io/stefanprodan/podinfo:6.5.3
        ports:
        - containerPort: 9898
          name: http
        Resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
```

**4. Create `apps/base/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  namespace: default
spec:
  selector:
    app: podinfo
  ports:
  - port: 80
    targetPort: 9898
```

**5. Create `apps/base/kustomization.yaml`:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

**6. Create Flux Kustomization in `clusters/dev-cluster/apps.yaml`:**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 5m0s
  path: ./apps/base
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: podinfo
    namespace: default
  timeout: 2m
```

**7. Commit and push:**

```bash
git add -A
git commit -m "Add podinfo application"
git push
```

**8. Watch Flux reconcile:**

```bash
# Watch reconciliation
flux reconcile kustomization flux-system --with-source

# Check application
kubectl get pods -l app=podinfo

# Check service
kubectl get svc podinfo

# Test the application
kubectl port-forward svc/podinfo 9898:80
# Visit http://localhost:9898
```

## Task 4: Deploy Helm Chart with HelmRelease

**1. Create HelmRepository source `clusters/dev-cluster/sources.yaml`:**

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 30m
  url: https://charts.bitnami.com/bitnami
```

**2. Create HelmRelease `apps/base/redis-release.yaml`:**

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: redis
  namespace: default
spec:
  interval: 5m
  chart:
    spec:
      chart: redis
      version: '18.x.x'
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    auth:
      enabled: false
    master:
      persistence:
        enabled: false
    replica:
      replicaCount: 1
      persistence:
        enabled: false
```

**3. Update `apps/base/kustomization.yaml`:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - redis-release.yaml
```

**4. Commit and observe:**

```bash
git add -A
git commit -m "Add Redis via Helm"
git push

# Watch Helm releases
flux get helmreleases

# Check Redis pods
kubectl get pods -l app.kubernetes.io/name=redis
```

## Task 5: Image Automation

Configure Flux to automatically update container images:

**1. Create ImageRepository `clusters/dev-cluster/image-automation.yaml`:**

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  image: ghcr.io/stefanprodan/podinfo
  interval: 1m0s
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: podinfo
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: podinfo
  policy:
    semver:
      range: 6.x.x
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: |
        Automated image update

        Automation name: {{ .AutomationObject }}

        Files:
        {{ range $filename, $_ := .Updated.Files -}}
        - {{ $filename }}
        {{ end -}}

        Objects:
        {{ range $resource, $_ := .Updated.Objects -}}
        - {{ $resource.Kind }} {{ $resource.Name }}
        {{ end -}}
    push:
      branch: main
  update:
    path: ./apps/base
    strategy: Setters
```

**2. Add image policy marker in `apps/base/deployment.yaml`:**

```yaml
spec:
  template:
    spec:
      containers:
      - name: podinfo
        image: ghcr.io/stefanprodan/podinfo:6.5.3 # {"$imagepolicy": "flux-system:podinfo"}
```

**3. Commit and push:**

```bash
git add -A
git commit -m "Add image automation"
git push

# Watch image updates
flux get image repository podinfo
flux get image policy podinfo

# Monitor for automated commits
watch flux get image update
```

Flux will now automatically update the image tag when new versions are available!

## Task 6: Suspend and Resume

Learn to control reconciliation:

```bash
# Suspend the apps Kustomization
flux suspend kustomization apps

# Make manual changes to cluster
kubectl scale deployment podinfo --replicas=5

# Resume reconciliation (will revert to Git state)
flux resume kustomization apps

# Verify reverted to 2 replicas
kubectl get deployment podinfo
```

## Task 7: Notifications

Set up Slack notifications (or use a generic webhook):

**1. Create notification provider `clusters/dev-cluster/notifications.yaml`:**

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta2
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: kubernetes-alerts
  secretRef:
    name: slack-webhook
---
apiVersion: notification.toolkit.fluxcd.io/v1beta2
kind: Alert
metadata:
  name: on-deploy
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: info
  eventSources:
  - kind: Kustomization
    name: apps
  - kind: HelmRelease
    name: '*'
```

**2. Create secret with webhook URL:**

```bash
kubectl create secret generic slack-webhook \
  -n flux-system \
  --from-literal=address=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

**3. Commit and push:**

```bash
git add clusters/dev-cluster/notifications.yaml
git commit -m "Add Slack notifications"
git push
```

You'll now receive notifications in Slack for deployments!

## Bonus Challenge 1: Multi-Environment Setup

Create separate overlays for staging and production:

```bash
mkdir -p apps/staging apps/production

# apps/staging/kustomization.yaml
cat > apps/staging/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
resources:
  - ../base
patches:
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 1
  target:
    kind: Deployment
    name: podinfo
EOF

# apps/production/kustomization.yaml
cat > apps/production/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../base
patches:
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 3
  target:
    kind: Deployment
    name: podinfo
EOF
```

Create separate Flux Kustomizations for each environment.

## Bonus Challenge 2: Sealed Secrets Integration

Encrypt secrets before committing to Git:

```bash
# Install sealed-secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Install kubeseal CLI
brew install kubeseal

# Create and seal a secret
kubectl create secret generic mysecret \
  --from-literal=password=supersecret \
  --dry-run=client \
  -o yaml | \
  kubeseal -o yaml > apps/base/sealed-secret.yaml

# Add to kustomization and commit
git add apps/base/sealed-secret.yaml
git commit -m "Add sealed secret"
git push
```

## Bonus Challenge 3: Flux Health Checks

Add detailed health checks to your Kustomization:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  # ... existing spec ...
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: podinfo
    namespace: default
  - apiVersion: v1
    kind: Service
    name: podinfo
    namespace: default
  - apiVersion: helm.toolkit.fluxcd.io/v2beta1
    kind: HelmRelease
    name: redis
    namespace: default
  timeout: 5m
  wait: true
```

## Bonus Challenge 4: Dependency Ordering

Ensure infrastructure is deployed before applications:

```yaml
# clusters/dev-cluster/infrastructure.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m
  path: ./infrastructure
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
---
# clusters/dev-cluster/apps.yaml (updated)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
  - name: infrastructure  # Wait for infrastructure first
  interval: 5m
  path: ./apps/base
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

## Verification

Ensure everything is working:

```bash
# Check all Flux components
flux check

# View all sources
flux get sources all

# View all Kustomizations
flux get kustomizations

# View all Helm releases
flux get helmreleases

# View all image policies
flux get image policy

# Check for errors
flux logs --all-namespaces
```

## Cleanup

```bash
# Suspend all reconciliation
flux suspend kustomization flux-system

# Delete applications
kubectl delete -f apps/base/

# Uninstall Flux
flux uninstall --silent

# Delete repository (optional)
# Delete from GitHub/GitLab manually or via API
```

## Key Takeaways

1. **GitOps provides** declarative, version-controlled infrastructure
2. **Flux automates** synchronization between Git and cluster
3. **Kustomization** enables environment-specific configurations
4. **HelmRelease** manages Helm charts declaratively
5. **Image automation** keeps containers up-to-date automatically
6. **Notifications** provide visibility into deployments
7. **Sealed Secrets** enable safe secret storage in Git

## Next Steps

- Explore multi-tenancy with Flux
- Implement progressive delivery with Flagger
- Set up automated testing before deployment
- Integrate with CI/CD pipelines
- Use Flux with Terraform for complete IaC
