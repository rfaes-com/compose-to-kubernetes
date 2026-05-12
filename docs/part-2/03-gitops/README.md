# GitOps with Flux

Duration: 50 minutes (25 min theory + 25 min lab)

## Introduction

**GitOps** is a way of implementing Continuous Deployment for cloud-native applications. You describe your entire system declaratively in Git, and updates are automatically deployed.

**Key Principles:**

1. Declarative - entire system described declaratively in Git
2. Versioned - Git as single source of truth
3. Automatically pulled - agents pull changes from Git
4. Continuously reconciled - actual state matches desired state

**Flux** is a CNCF project that implements GitOps for Kubernetes.

## Why GitOps?

**Traditional CI/CD:**

```text
Code Push → CI Build → CI Pushes to Cluster
```

- CI has cluster credentials (security risk)
- Push-based (manual or triggered)
- No drift detection

**GitOps with Flux:**

```text
Code Push → Git Repo → Flux Pulls Changes → Cluster Updated
```

- Cluster pulls from Git (no external access needed)
- Continuous reconciliation
- Automatic drift detection and correction
- Git history = deployment audit log

## Flux Architecture

**Core Controllers:**

1. **Source Controller** - Fetches artifacts (Git repos, Helm repos, S3 buckets)
2. **Kustomize Controller** - Applies Kustomize overlays
3. **Helm Controller** - Manages Helm releases
4. **Notification Controller** - Sends alerts (Slack, Discord, etc.)
5. **Image Automation Controllers** - Automates image updates

## Installing Flux

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Verify installation
flux --version

# Bootstrap Flux on your cluster
flux bootstrap github \
  --owner=your-username \
  --repository=fleet-infra \
  --path=clusters/my-cluster \
  --personal
```

## GitOps Repository Structure

Typical GitOps repo layout:

```text
fleet-infra/
├── clusters/
│   ├── production/
│   │   ├── infrastructure.yaml
│   │   └── apps.yaml
│   └── staging/
│       ├── infrastructure.yaml
│       └── apps.yaml
├── infrastructure/
│   ├── nginx-ingress/
│   ├── cert-manager/
│   └── monitoring/
└── apps/
    ├── backend/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── frontend/
        ├── deployment.yaml
        ├── service.yaml
        └── kustomization.yaml
```

## GitOps Source

Define where Flux should pull manifests from:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: fleet-infra
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/user/fleet-infra
  ref:
    branch: main
```

## Kustomization

Tell Flux what to apply from the repository:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./apps
  prune: true
  wait: true
  timeout: 2m
```

## Helm Releases with Flux

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: nginx
  namespace: default
spec:
  interval: 5m
  chart:
    spec:
      chart: nginx
      version: '13.x.x'
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    replicaCount: 3
    service:
      type: LoadBalancer
```

## Image Automation

Automatically update images when new versions are pushed:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: docker.io/myorg/myapp
  interval: 1m
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: '>=1.0.0 <2.0.0'
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: flux
        email: flux@example.com
      messageTemplate: 'Update image to {{range .Updated.Images}}{{println .}}{{end}}'
    push:
      branch: main
  update:
    path: ./apps
    strategy: Setters
```

## Notifications

Alert on deployments and failures:

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Alert
metadata:
  name: slack-alert
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: info
  eventSources:
  - kind: Kustomization
    name: '*'
  - kind: HelmRelease
    name: '*'
---
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: deployments
  secretRef:
    name: slack-webhook
```

## Multi-Tenancy

Isolate teams with separate namespaces and permissions:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-a-apps
  namespace: flux-system
spec:
  serviceAccountName: team-a-reconciler
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./teams/team-a
  prune: true
  targetNamespace: team-a
```

## Best Practices

1. **Separate repos** - Infrastructure vs applications
2. **Environment branches** - dev/staging/prod branches or directories
3. **Encrypted secrets** - Use Sealed Secrets or SOPS
4. **Structured commits** - Meaningful commit messages
5. **Health checks** - Configure proper health assessment
6. **Notifications** - Alert on failures
7. **Progressive delivery** - Use Flagger for canary deployments
8. **RBAC** - Least privilege for Flux service accounts

## Flux CLI Commands

```bash
# Check Flux status
flux check

# Get resources
flux get sources git
flux get kustomizations
flux get helmreleases

# Suspend/resume reconciliation
flux suspend kustomization apps
flux resume kustomization apps

# Trigger immediate reconciliation
flux reconcile source git fleet-infra
flux reconcile kustomization apps

# Export resources
flux export source git fleet-infra
flux export kustomization apps

# View logs
flux logs --all-namespaces
flux logs --kind=Kustomization --name=apps
```

## Handling Secrets

### Option 1: Sealed Secrets

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Seal a secret
kubectl create secret generic mysecret --from-literal=password=secret123 --dry-run=client -o yaml | \
  kubeseal -o yaml > mysealedsecret.yaml

# Commit sealed secret to Git
git add mysealedsecret.yaml
git commit -m "Add sealed secret"
```

### Option 2: Mozilla SOPS

```yaml
# .sops.yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1...publickey...
```

```bash
# Encrypt secret
sops -e secret.yaml > secret.enc.yaml

# Flux decrypts automatically with decryption key
```

## Complete GitOps Workflow

1. **Developer pushes code** to application repo
2. **CI builds** and pushes container image
3. **Image automation** detects new tag
4. **Flux updates** manifest in GitOps repo
5. **Flux reconciles** and deploys to cluster
6. **Notification sent** to team
7. **Monitoring detects** issues (if any)
8. **Automatic rollback** if health checks fail

## Next Steps

- Complete hands-on lab in `lab/instructions.md`
- Explore Flagger for progressive delivery
- Set up image automation
- Implement secret management
- Configure notifications

## Additional Resources

- [Flux Documentation](https://fluxcd.io/docs/)
- [GitOps Toolkit](https://toolkit.fluxcd.io/)
- [Flux GitHub](https://github.com/fluxcd/flux2)
