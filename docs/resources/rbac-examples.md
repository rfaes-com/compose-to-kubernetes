# RBAC Examples

Practical examples of Kubernetes Role-Based Access Control (RBAC) configurations.

## Core Concepts

| Resource             | Scope        | Description                                                              |
| -------------------- | ------------ | ------------------------------------------------------------------------ |
| `Role`               | Namespace    | Grants permissions within a single namespace                             |
| `ClusterRole`        | Cluster-wide | Grants permissions across all namespaces or for cluster-scoped resources |
| `RoleBinding`        | Namespace    | Binds a Role or ClusterRole to subjects within a namespace               |
| `ClusterRoleBinding` | Cluster-wide | Binds a ClusterRole to subjects across the entire cluster                |

## Roles

### Read-Only Role (Namespace)

Allows viewing all common resources in a namespace without modification.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-only
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints", "configmaps", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch"]
```

### Developer Role (Namespace)

Allows developers to manage application workloads but not cluster-level resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: my-app
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec", "services", "endpoints", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]           # Read secrets but not create/modify
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "create", "delete"]
```

### CI/CD Deployment Role (Namespace)

Allows a pipeline to deploy and update applications.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```

## ClusterRoles

### Cluster Read-Only

Allows viewing all resources across all namespaces and cluster-scoped resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-read-only
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["*"]
    verbs: ["get"]
```

### Namespace Admin

Full control over a namespace's resources (used as a ClusterRole scoped via RoleBinding).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-admin
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
  # Excludes cluster-scoped resources — these are only accessible
  # when bound with a RoleBinding to a specific namespace
```

### Metrics Reader

Allows reading metrics from pods and nodes (used by HPA, monitoring tools).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
rules:
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list"]
```

### Custom Resource Definition Manager

Allows managing CRDs and their instances.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: crd-manager
rules:
  - apiGroups: ["apiextensions.k8s.io"]
    resources: ["customresourcedefinitions"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

## RoleBindings

### Bind a User to a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: my-app
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### Bind a Group to a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-read
  namespace: frontend
subjects:
  - kind: Group
    name: frontend-engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: read-only
  apiGroup: rbac.authorization.k8s.io
```

### Bind a ServiceAccount to a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: cicd-bot
    namespace: cicd
roleRef:
  kind: Role
  name: cicd-deployer
  apiGroup: rbac.authorization.k8s.io
```

### Bind a ClusterRole to a Namespace (Scoped)

```yaml
# Grants cluster-read-only permissions scoped only to the 'staging' namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ops-read-staging
  namespace: staging
subjects:
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole           # ClusterRole scoped to this namespace
  name: cluster-read-only
  apiGroup: rbac.authorization.k8s.io
```

## ClusterRoleBindings

### Bind a User to Cluster-Wide Read

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bob-cluster-viewer
subjects:
  - kind: User
    name: bob@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-read-only
  apiGroup: rbac.authorization.k8s.io
```

### Bind a ServiceAccount to Cluster Admin (use sparingly)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: flux-cluster-admin
subjects:
  - kind: ServiceAccount
    name: flux
    namespace: flux-system
roleRef:
  kind: ClusterRole
  name: cluster-admin             # Built-in full admin role
  apiGroup: rbac.authorization.k8s.io
```

## ServiceAccounts

### Create a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
automountServiceAccountToken: false   # Opt-in rather than auto-mount
```

### Reference in a Pod/Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app    # Use the named ServiceAccount
      automountServiceAccountToken: false
      containers:
        - name: app
          image: my-app:latest
```

## Multi-Tenant Namespace Isolation

Full example of isolating two teams in separate namespaces.

```yaml
# --- Namespaces ---
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-beta

---
# --- ServiceAccounts ---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alpha-deployer
  namespace: team-alpha
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: beta-deployer
  namespace: team-beta

---
# --- Roles (one per namespace) ---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-deploy
  namespace: team-alpha
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources:
      - pods
      - services
      - deployments
      - statefulsets
      - ingresses
      - configmaps
      - jobs
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-deploy
  namespace: team-beta
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources:
      - pods
      - services
      - deployments
      - statefulsets
      - ingresses
      - configmaps
      - jobs
    verbs: ["*"]

---
# --- RoleBindings ---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alpha-deployer-binding
  namespace: team-alpha
subjects:
  - kind: ServiceAccount
    name: alpha-deployer
    namespace: team-alpha
  - kind: Group
    name: team-alpha-engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-deploy
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: beta-deployer-binding
  namespace: team-beta
subjects:
  - kind: ServiceAccount
    name: beta-deployer
    namespace: team-beta
  - kind: Group
    name: team-beta-engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-deploy
  apiGroup: rbac.authorization.k8s.io
```

## Debugging RBAC

```bash
# Check if a user can perform an action
kubectl auth can-i get pods --namespace default --as alice@example.com
kubectl auth can-i delete deployments --namespace production --as bob@example.com

# Check all permissions for a user in a namespace
kubectl auth can-i --list --namespace default --as alice@example.com

# Check permissions for a ServiceAccount
kubectl auth can-i get pods --as=system:serviceaccount:default:my-app

# Describe a Role or ClusterRole
kubectl describe role developer -n my-app
kubectl describe clusterrole cluster-read-only

# List RoleBindings for a namespace
kubectl get rolebindings -n my-app
kubectl describe rolebinding alice-developer -n my-app

# List ClusterRoleBindings
kubectl get clusterrolebindings
```

## Built-In ClusterRoles

| ClusterRole                      | Description                                                |
| -------------------------------- | ---------------------------------------------------------- |
| `cluster-admin`                  | Full cluster control (use sparingly)                       |
| `admin`                          | Namespace admin with read/write, but no node/quota control |
| `edit`                           | Read/write to most namespace resources, no role management |
| `view`                           | Read-only access to most namespace resources               |
| `system:node`                    | Used by kubelets                                           |
| `system:kube-scheduler`          | Used by the scheduler                                      |
| `system:kube-controller-manager` | Used by controller manager                                 |

## Common Verbs

| Verb               | HTTP Method     | Description                              |
| ------------------ | --------------- | ---------------------------------------- |
| `get`              | GET             | Retrieve a single resource               |
| `list`             | GET             | List resources                           |
| `watch`            | GET (streaming) | Watch for changes                        |
| `create`           | POST            | Create a resource                        |
| `update`           | PUT             | Replace a resource                       |
| `patch`            | PATCH           | Partially modify a resource              |
| `delete`           | DELETE          | Delete a resource                        |
| `deletecollection` | DELETE          | Delete a collection of resources         |
| `impersonate`      | POST            | Act as another user                      |
| `bind`             | POST            | Bind roles (for creating RoleBindings)   |
| `escalate`         | POST            | Modify roles to grant higher permissions |

Use `"*"` as a wildcard to allow all verbs or all resources in a rule.
