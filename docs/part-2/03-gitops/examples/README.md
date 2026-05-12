# GitOps Example Repository Structure

This directory demonstrates a GitOps repository structure for Flux CD.

## Repository Layout

```text
gitops-example/
├── clusters/
│   └── dev-cluster/
│       └── flux-system/
│           ├── gotk-components.yaml
│           ├── gotk-sync.yaml
│           └── kustomization.yaml
├── infrastructure/
│   ├── base/
│   │   └── namespace.yaml
│   └── controllers/
│       └── ingress-nginx.yaml
└── apps/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── production/
        ├── deployment-patch.yaml
        └── kustomization.yaml
```

## GitRepository Source

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: https://github.com/your-org/your-repo
```

## Kustomization for Applications

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 5m0s
  path: ./apps/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: myapp
    namespace: production
  timeout: 2m0s
```

## HelmRelease Example

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: nginx-ingress
  namespace: ingress-nginx
spec:
  interval: 5m
  chart:
    spec:
      chart: ingress-nginx
      version: '4.8.x'
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
      interval: 1m
  values:
    controller:
      replicaCount: 2
      service:
        type: LoadBalancer
      metrics:
        enabled: true
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
```

## HelmRepository Source

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 5m0s
  url: https://kubernetes.github.io/ingress-nginx
```

## Using in Lab

The lab will guide you through:

1. Installing Flux CD
2. Bootstrapping a Git repository
3. Creating these manifests
4. Watching Flux reconcile changes
5. Implementing image automation
