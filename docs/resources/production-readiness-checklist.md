# Production Readiness Checklist

A checklist for reviewing Kubernetes workloads before promoting to production.

---

## Workload Configuration

### Pods & Containers

- [ ] **Image tags** — Use specific, immutable tags (e.g., `v1.2.3` or digest `sha256:...`). Never use `latest` in production.
- [ ] **Resource requests and limits** — Set CPU and memory requests/limits on every container.
- [ ] **Liveness probe** — Configured to restart unhealthy containers automatically.
- [ ] **Readiness probe** — Configured to prevent traffic from reaching unready pods.
- [ ] **Startup probe** — Added for slow-starting applications to avoid premature liveness failures.
- [ ] **Security context** — Run containers as non-root (`runAsNonRoot: true`, `runAsUser: <non-zero>`).
- [ ] **Read-only root filesystem** — Set `readOnlyRootFilesystem: true` where possible.
- [ ] **Privilege escalation disabled** — Set `allowPrivilegeEscalation: false`.
- [ ] **Drop capabilities** — Drop all capabilities and add only what is required.
- [ ] **No privileged containers** — Avoid `privileged: true`.

```yaml
# Example security context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

### Deployments & StatefulSets

- [ ] **Replica count** — Use at least 2 replicas for high availability.
- [ ] **Pod Disruption Budget (PDB)** — Define a PDB to limit disruption during node maintenance.
- [ ] **Update strategy** — Use `RollingUpdate` with `maxUnavailable: 0` for zero-downtime deploys.
- [ ] **Rollback plan** — Verify `revisionHistoryLimit` is set (default 10) for rollback capability.
- [ ] **Pod anti-affinity** — Spread replicas across nodes and/or availability zones.

```yaml
# Example PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 1      # or maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

---

## Resource Management

- [ ] **Namespace resource quotas** — Define `ResourceQuota` to cap namespace resource consumption.
- [ ] **LimitRange** — Set default requests/limits for pods that don't specify them.
- [ ] **Cluster capacity** — Confirm nodes have sufficient headroom for rolling deploys and autoscaling.
- [ ] **QoS class** — Aim for `Guaranteed` or `Burstable` QoS (avoid `BestEffort` for critical workloads).

```yaml
# Example ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: my-app
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
```

---

## Networking

- [ ] **Services** — Expose only what needs to be exposed; prefer `ClusterIP` internally.
- [ ] **Ingress** — Use Ingress with TLS termination for HTTP(S) workloads.
- [ ] **TLS certificates** — Use cert-manager with valid certificates; do not use self-signed certs in production.
- [ ] **Network Policies** — Restrict pod-to-pod communication; apply least-privilege network rules.
- [ ] **DNS** — Validate service discovery via cluster DNS.
- [ ] **External load balancer health checks** — Confirm health check paths are correctly configured.

```yaml
# Example NetworkPolicy — deny all ingress, allow only from same namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}
```

---

## Configuration & Secrets

- [ ] **No secrets in image or source code** — All secrets come from Kubernetes Secrets or an external secrets manager.
- [ ] **Secrets encrypted at rest** — Enable encryption at rest for the etcd datastore.
- [ ] **Environment variables vs volume mounts** — Prefer mounting secrets as volumes over environment variables for sensitive values.
- [ ] **ConfigMaps for non-sensitive config** — Separate configuration from application code.
- [ ] **External Secrets Operator** — Consider using ESO for syncing secrets from Vault, AWS Secrets Manager, etc.
- [ ] **Secret rotation plan** — Define how secrets are rotated and how pods pick up the new values.

---

## Storage

- [ ] **Persistent Volume Claims** — Ensure PVCs are bound and using the appropriate StorageClass.
- [ ] **StorageClass** — Confirm the StorageClass reclaim policy (`Retain` vs `Delete`) is correct.
- [ ] **Volume backups** — Enable backups for stateful data (e.g., Velero).
- [ ] **StatefulSet pod identity** — Verify that StatefulSet pods use stable network identifiers and persistent volumes.
- [ ] **ReadWriteMany volumes** — Only use `RWX` access modes if your storage backend supports it.

---

## Observability

- [ ] **Structured logging** — Application logs are in JSON or another structured format.
- [ ] **Log aggregation** — Logs are collected by a tool such as Loki, Fluentd, or Fluent Bit.
- [ ] **Metrics exposed** — Application exposes a `/metrics` endpoint compatible with Prometheus.
- [ ] **ServiceMonitor or PodMonitor** — Prometheus scrape targets are configured.
- [ ] **Dashboards** — Grafana dashboards exist for key application metrics.
- [ ] **Alerting rules** — Alerts are defined for SLO breaches, error rates, and resource saturation.
- [ ] **Tracing** — Distributed tracing is configured (e.g., OpenTelemetry, Jaeger) for critical paths.
- [ ] **Runbooks** — Each alert links to a runbook explaining how to investigate and resolve.

---

## Reliability & Autoscaling

- [ ] **Horizontal Pod Autoscaler (HPA)** — Configured for stateless workloads with appropriate min/max replicas.
- [ ] **Vertical Pod Autoscaler (VPA)** — Considered for right-sizing resource requests.
- [ ] **Cluster Autoscaler** — Enabled if running on a cloud provider with node pools.
- [ ] **Graceful shutdown** — Application handles `SIGTERM` and finishes in-flight requests within `terminationGracePeriodSeconds`.
- [ ] **PreStop hook** — Add a sleep PreStop hook if the application needs time to drain connections.
- [ ] **Chaos testing** — Run chaos experiments (e.g., kill pods, drain nodes) to validate resilience.

```yaml
# Example graceful shutdown
containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]
    terminationMessagePolicy: FallbackToLogsOnError
```

---

## Security

- [ ] **RBAC** — Least-privilege roles and service accounts for all components.
- [ ] **ServiceAccount token auto-mount** — Set `automountServiceAccountToken: false` unless the pod needs it.
- [ ] **Pod Security Standards** — Apply the appropriate Pod Security Standard (`baseline` or `restricted`) to namespaces.
- [ ] **Image scanning** — Container images are scanned for vulnerabilities before deployment.
- [ ] **Image signing** — Images are signed and verified (e.g., Cosign, Notary).
- [ ] **Admission control** — Policies enforced via OPA/Gatekeeper or Kyverno.
- [ ] **Audit logging** — Kubernetes API server audit logs are enabled and collected.
- [ ] **etcd encryption** — Sensitive Secret data is encrypted at rest in etcd.
- [ ] **Node hardening** — Worker nodes have CIS Kubernetes Benchmark controls applied.

---

## CI/CD & Deployment

- [ ] **GitOps / IaC** — All manifests are stored in version control; no manual `kubectl apply`.
- [ ] **Environment promotion** — Changes flow through dev → staging → production with approvals.
- [ ] **Automated testing** — Integration and smoke tests run against staging before production promotion.
- [ ] **Rollback procedure** — Rollback steps are documented and tested.
- [ ] **Deployment notifications** — Team is notified on successful and failed deployments.
- [ ] **Change freeze windows** — Deployment schedule respects maintenance windows.

---

## Cluster Operations

- [ ] **Kubernetes version** — Running a supported Kubernetes version; upgrade path is planned.
- [ ] **Control plane HA** — Multiple control plane nodes in production clusters.
- [ ] **etcd backup** — Automated, regularly tested etcd backups are in place.
- [ ] **Node OS updates** — Node OS patches are applied regularly via rolling node replacement.
- [ ] **Namespace hygiene** — Unused namespaces and resources are cleaned up.
- [ ] **Cost visibility** — Namespace or team cost allocation is visible (e.g., Kubecost).
- [ ] **Disaster recovery plan** — DR procedure is documented and tested at least annually.

---

## Final Review

| Category                  | Status | Owner | Notes |
| ------------------------- | ------ | ----- | ----- |
| Workload configuration    |        |       |       |
| Resource management       |        |       |       |
| Networking                |        |       |       |
| Configuration & secrets   |        |       |       |
| Storage                   |        |       |       |
| Observability             |        |       |       |
| Reliability & autoscaling |        |       |       |
| Security                  |        |       |       |
| CI/CD & deployment        |        |       |       |
| Cluster operations        |        |       |       |
