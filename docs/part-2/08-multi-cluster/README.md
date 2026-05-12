# Multi-Cluster Management

Duration: 45 minutes (20 min theory + 25 min lab)

## Introduction

Multi-cluster strategies distribute workloads across multiple Kubernetes clusters for high availability, disaster recovery, geographic distribution, and environment isolation.

**Use Cases:**

- High availability across regions
- Disaster recovery and backup
- Dev/staging/production separation
- Geographic data residency
- Load distribution
- Blue/green at cluster level

## Multi-Cluster Patterns

### 1. Federation Pattern

Single control plane manages multiple clusters:

- Central policy management
- Cross-cluster scheduling
- Unified configuration

### 2. Independent Clusters

Separate clusters with service mesh connectivity:

- Cluster independence
- Service-to-service communication
- Traffic routing across clusters

### 3. Hub and Spoke

Central hub cluster manages spoke clusters:

- Central monitoring
- Policy distribution
- Workload orchestration

## Kubernetes Context Management

### View and Switch Contexts

```bash
# List contexts
kubectl config get-contexts

# Current context
kubectl config current-context

# Switch context
kubectl config use-context cluster-prod

# View full config
kubectl config view
```

### Configure Multiple Clusters

```yaml
# ~/.kube/config
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://cluster1.example.com
    certificate-authority-data: <ca-cert-1>
  name: cluster-1
- cluster:
    server: https://cluster2.example.com
    certificate-authority-data: <ca-cert-2>
  name: cluster-2
contexts:
- context:
    cluster: cluster-1
    user: admin-1
    namespace: default
  name: context-1
- context:
    cluster: cluster-2
    user: admin-2
    namespace: default
  name: context-2
current-context: context-1
users:
- name: admin-1
  user:
    client-certificate-data: <cert-1>
    client-key-data: <key-1>
- name: admin-2
  user:
    client-certificate-data: <cert-2>
    client-key-data: <key-2>
```

### Quick Context Switching

```bash
# Create aliases
alias k1='kubectl --context=context-1'
alias k2='kubectl --context=context-2'

# Use kubectx tool
kubectx context-1
kubectx context-2

# List contexts
kubectx
```

## Cross-Cluster Service Discovery

### DNS-Based Discovery

```yaml
# Service in cluster-1
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: production
spec:
  selector:
    app: api
  ports:
  - port: 8080
  type: ClusterIP
```

Access from another cluster:

```yaml
# ExternalName Service in cluster-2
apiVersion: v1
kind: Service
metadata:
  name: api-cluster1
  namespace: production
spec:
  type: ExternalName
  externalName: api.production.cluster1.example.com
```

### Multi-Cluster Services with Cilium

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-cross-cluster
spec:
  endpointSelector:
    matchLabels:
      app: frontend
  egress:
  - toEndpoints:
    - matchLabels:
        io.cilium.k8s.policy.cluster: cluster-2
        app: api
```

Enable Cluster Mesh:

```bash
# Install Cilium
cilium install

# Enable cluster mesh on both clusters
cilium clustermesh enable --context cluster-1
cilium clustermesh enable --context cluster-2

# Connect clusters
cilium clustermesh connect --context cluster-1 --destination-context cluster-2

# Verify
cilium clustermesh status --context cluster-1
```

### Service Export/Import

```yaml
# In cluster-1: Export service
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: api
  namespace: production
---
# In cluster-2: Import service
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: api
  namespace: production
spec:
  type: ClusterSetIP
  ports:
  - port: 8080
    protocol: TCP
```

## GitOps for Multi-Cluster

### Flux Multi-Cluster Setup

```yaml
# clusters/cluster-1/flux-system/kustomization.yaml
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
  path: ./apps/cluster-1
  prune: true
---
# clusters/cluster-2/flux-system/kustomization.yaml
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
  path: ./apps/cluster-2
  prune: true
```

Repository structure:

```text
fleet-infra/
├── apps/
│   ├── base/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── cluster-1/
│   │   ├── kustomization.yaml
│   │   └── values.yaml
│   └── cluster-2/
│       ├── kustomization.yaml
│       └── values.yaml
└── clusters/
    ├── cluster-1/
    │   └── flux-system/
    └── cluster-2/
        └── flux-system/
```

## Disaster Recovery

### Backup with Velero

```bash
# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket k8s-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero

# Create backup
velero backup create production-backup \
  --include-namespaces production \
  --default-volumes-to-fs-backup

# Schedule regular backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production

# Restore to another cluster
velero restore create --from-backup production-backup \
  --namespace-mappings production:production
```

### Backup Strategy

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: full-cluster-backup
  namespace: velero
spec:
  includedNamespaces:
  - '*'
  excludedNamespaces:
  - kube-system
  - kube-public
  - velero
  includedResources:
  - '*'
  excludedResources:
  - events
  - events.events.k8s.io
  labelSelector:
    matchLabels:
      backup: "true"
  ttl: 720h0m0s  # 30 days
  storageLocation: default
  volumeSnapshotLocations:
  - default
```

## Traffic Management

### Global Load Balancing

Using external DNS and geo-routing:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app.example.com
    external-dns.alpha.kubernetes.io/geo-region: us-east-1
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
```

### Istio Multi-Cluster

```bash
# Install Istio with multi-cluster support
istioctl install --set profile=default \
  --set values.global.meshID=mesh1 \
  --set values.global.multiCluster.clusterName=cluster-1 \
  --set values.global.network=network1

# Create remote secret for cluster-2
istioctl create-remote-secret \
  --context=cluster-2 \
  --name=cluster-2 | \
  kubectl apply -f - --context=cluster-1

# Deploy sample app
kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

## Centralized Monitoring

### Prometheus Federation

```yaml
# Central Prometheus in hub cluster
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'federate-cluster-1'
      honor_labels: true
      metrics_path: '/federate'
      params:
        'match[]':
          - '{job="kubernetes-pods"}'
      static_configs:
      - targets:
        - 'prometheus.cluster-1.example.com:9090'
        labels:
          cluster: 'cluster-1'
    - job_name: 'federate-cluster-2'
      honor_labels: true
      metrics_path: '/federate'
      params:
        'match[]':
          - '{job="kubernetes-pods"}'
      static_configs:
      - targets:
        - 'prometheus.cluster-2.example.com:9090'
        labels:
          cluster: 'cluster-2'
```

### Grafana Multi-Cluster Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - name: Cluster-1
      type: prometheus
      url: http://prometheus.cluster-1.svc:9090
      isDefault: false
      editable: true
    - name: Cluster-2
      type: prometheus
      url: http://prometheus.cluster-2.svc:9090
      isDefault: false
      editable: true
    - name: All-Clusters
      type: prometheus
      url: http://prometheus-federated.svc:9090
      isDefault: true
      editable: true
```

## Multi-Cluster Deployment Tools

### Argo CD ApplicationSet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-app
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: cluster-1
        url: https://cluster1.example.com
        region: us-east-1
      - cluster: cluster-2
        url: https://cluster2.example.com
        region: us-west-2
  template:
    metadata:
      name: 'app-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/manifests
        targetRevision: HEAD
        path: apps/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Rancher Multi-Cluster Management

Rancher provides UI and API for multi-cluster management:

```bash
# Deploy Rancher
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.example.com

# Import existing cluster
# UI: Add Cluster > Import Existing > Run generated command on target cluster
```

## Best Practices

1. **Consistent Configuration** - Use GitOps for all clusters
2. **Namespace Naming** - Use same namespace names across clusters
3. **Resource Labels** - Label resources with cluster name
4. **Secrets Management** - Use external secrets operator
5. **Monitoring** - Centralized monitoring with federation
6. **Backup Strategy** - Regular automated backups
7. **Network Connectivity** - Ensure cross-cluster networking
8. **Cost Management** - Monitor costs per cluster
9. **Access Control** - Separate RBAC per cluster
10. **Documentation** - Document cluster purposes and configs

## Failure Scenarios

### Cluster Failover

```bash
# Detect failure (health check)
kubectl --context=cluster-1 get nodes

# Update DNS to point to cluster-2
# This is typically done by load balancer or DNS provider

# Restore services in cluster-2 from backup
velero restore create --from-backup production-backup
```

### Data Replication

Use external data replication:

- **Databases**: Native replication (PostgreSQL streaming, MySQL replication)
- **Object Storage**: S3 cross-region replication
- **Volumes**: Cloud provider volume replication

## Common Pitfalls

1. **Network Latency** - Cross-cluster calls are slower
2. **Data Consistency** - No distributed transactions
3. **Cost** - Multiple clusters = higher costs
4. **Complexity** - More moving parts
5. **Secret Synchronization** - Keep secrets in sync
6. **Version Skew** - Keep Kubernetes versions aligned

## Multi-Cluster Decision Matrix

| Factor            | Single Cluster          | Multi-Cluster        |
| ----------------- | ----------------------- | -------------------- |
| Complexity        | Low                     | High                 |
| Availability      | Single point of failure | High availability    |
| Cost              | Lower                   | Higher               |
| Latency           | Low                     | Higher cross-cluster |
| Isolation         | Namespace-based         | Cluster-based        |
| Management        | Simple                  | Complex              |
| Disaster Recovery | Backup/restore          | Active-active        |

## Next Steps

- Complete hands-on lab in `lab/instructions.md`
- Set up multiple test clusters
- Configure context switching
- Implement backup strategy
- Practice disaster recovery

## Additional Resources

- [Multi-Cluster Services](https://github.com/kubernetes-sigs/mcs-api)
- [Cilium Cluster Mesh](https://docs.cilium.io/en/stable/network/clustermesh/)
- [Velero Documentation](https://velero.io/docs/)
- [Istio Multi-Cluster](https://istio.io/latest/docs/setup/install/multicluster/)
