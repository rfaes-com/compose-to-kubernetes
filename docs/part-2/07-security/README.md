# Security and RBAC

Duration: 50 minutes (25 min theory + 25 min lab)

## Introduction

Securing Kubernetes clusters is critical for production deployments. RBAC (Role-Based Access Control) provides fine-grained access control to cluster resources.

**Security Layers:**

1. **Authentication** - Who are you?
2. **Authorization** - What can you do? (RBAC)
3. **Admission Control** - Should this be allowed?
4. **Network Policies** - Who can talk to whom?
5. **Pod Security** - How secure are the pods?

## RBAC Core Concepts

**Four Resource Types:**

1. **Role** - Permissions in a namespace
2. **ClusterRole** - Permissions across cluster
3. **RoleBinding** - Assigns Role to users/groups/service accounts
4. **ClusterRoleBinding** - Assigns ClusterRole cluster-wide

## Roles and RoleBindings

### Role Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

**API Groups:**

- `""` - core API group (pods, services, configmaps)
- `"apps"` - deployments, statefulsets
- `"batch"` - jobs, cronjobs
- `"networking.k8s.io"` - ingresses, networkpolicies

**Verbs (permissions):**

- `get`, `list`, `watch` - Read operations
- `create`, `update`, `patch` - Write operations
- `delete`, `deletecollection` - Delete operations
- `*` - All verbs (use carefully!)

### RoleBinding Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: myapp
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## ClusterRoles and ClusterRoleBindings

### ClusterRole Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

### ClusterRoleBinding Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-readers
subjects:
- kind: Group
  name: node-admins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

## ServiceAccounts

Pods use ServiceAccounts for API access:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: app
        image: myapp:latest
```

**Default Behavior:**

- Every namespace has a `default` ServiceAccount
- Pods use `default` if not specified
- Token automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`

### Disable Auto-Mount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
automountServiceAccountToken: false
```

Or per pod:

```yaml
spec:
  automountServiceAccountToken: false
  serviceAccountName: myapp-sa
```

## Common RBAC Patterns

### Read-Only Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: viewer
  namespace: production
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### Deployment Manager

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

### Full Namespace Admin

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: team-a
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

## Multi-Tenant Isolation

### Per-Team Namespace Setup

```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
---
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-alpha-user
  namespace: team-alpha
---
# Role (full access in namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-alpha-admin
  namespace: team-alpha
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-admin-binding
  namespace: team-alpha
subjects:
- kind: ServiceAccount
  name: team-alpha-user
  namespace: team-alpha
- kind: Group
  name: team-alpha
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-alpha-admin
  apiGroup: rbac.authorization.k8s.io
---
# ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    pods: "20"
---
# NetworkPolicy (isolate namespace)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
```

## Pod Security

### Pod Security Standards

Three levels:

1. **Privileged** - Unrestricted (default)
2. **Baseline** - Minimally restrictive
3. **Restricted** - Heavily restricted (best practice)

### Enforcing Pod Security

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

**Key Settings:**

- `runAsNonRoot: true` - Don't run as root
- `readOnlyRootFilesystem: true` - Immutable filesystem
- `allowPrivilegeEscalation: false` - Can't gain privileges
- `capabilities.drop: ALL` - Drop all Linux capabilities

## Network Policies

Control traffic between pods:

### Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow from Specific Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-from-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Allow Egress to Specific Services

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-egress-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:  # Allow DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

## Secrets Management

### External Secrets Operator

Sync secrets from external systems:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: prod/db/credentials
      property: username
  - secretKey: password
    remoteRef:
      key: prod/db/credentials
      property: password
```

### Sealed Secrets

Encrypt secrets for Git storage:

```bash
# Install controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Create sealed secret
echo -n mypassword | kubectl create secret generic mysecret \
  --dry-run=client \
  --from-file=password=/dev/stdin \
  -o yaml | \
  kubeseal -o yaml > mysealedsecret.yaml

# Apply (only controller can decrypt)
kubectl apply -f mysealedsecret.yaml
```

## RBAC Debugging

```bash
# Check if user can perform action
kubectl auth can-i create deployments --as=jane

# Check all permissions for user
kubectl auth can-i --list --as=jane

# Check ServiceAccount permissions
kubectl auth can-i list pods --as=system:serviceaccount:default:myapp-sa

# View Role details
kubectl describe role pod-reader

# View RoleBinding
kubectl describe rolebinding read-pods

# View what ServiceAccount a pod uses
kubectl get pod mypod -o jsonpath='{.spec.serviceAccountName}'
```

## Audit Logging

Track API access:

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
- level: RequestResponse
  omitStages:
  - RequestReceived
  resources:
  - group: ""
    resources: ["pods"]
  verbs: ["delete"]
```

## Image Security

### Image Pull Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
---
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
  - name: app
    image: private-registry.com/myapp:latest
  imagePullSecrets:
  - name: regcred
```

### Image Scanning

Use admission controllers to scan images:

```yaml
# Example with OPA Gatekeeper
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequirescannedimages
spec:
  crd:
    spec:
      names:
        kind: K8sRequireScannedImages
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequirescannedimages
      violation[{"msg": msg}] {
        image := input.review.object.spec.containers[_].image
        not scanned_image(image)
        msg := sprintf("Image '%v' has not been scanned", [image])
      }
      scanned_image(image) {
        # Check if image has scan-passed label
        startswith(image, "myregistry.com/scanned/")
      }
```

## Best Practices

1. **Least Privilege** - Give minimum permissions needed
2. **Separate ServiceAccounts** - Per application
3. **No default ServiceAccount** - Disable auto-mount
4. **NetworkPolicies** - Default deny, explicit allow
5. **PodSecurityStandards** - Enforce restricted
6. **ReadOnlyRootFilesystem** - Make containers immutable
7. **Non-root users** - Always run as non-root
8. **Secrets rotation** - Rotate credentials regularly
9. **Audit logging** - Enable and monitor
10. **Regular reviews** - Audit RBAC permissions

## Security Checklist

- [ ] RBAC enabled and configured
- [ ] ServiceAccounts per application
- [ ] NetworkPolicies in place
- [ ] Pod Security Standards enforced
- [ ] Secrets encrypted at rest
- [ ] External secrets management
- [ ] Image scanning enabled
- [ ] Audit logging configured
- [ ] Resource quotas set
- [ ] No privileged containers
- [ ] Read-only root filesystems
- [ ] Drop unnecessary capabilities
- [ ] TLS for all services
- [ ] Regular security updates

## Next Steps

- Complete hands-on lab in `lab/instructions.md`
- Implement RBAC for your applications
- Set up NetworkPolicies
- Configure Pod Security Standards
- Enable audit logging

## Additional Resources

- [RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Security Best Practices](https://kubernetes.io/docs/concepts/security/security-checklist/)
