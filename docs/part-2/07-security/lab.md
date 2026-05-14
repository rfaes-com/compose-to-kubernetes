# Lab: Security and RBAC

**Duration:** 25 minutes

## Objectives

- Create isolated team namespaces
- Grant limited permissions with RBAC
- Test access with `kubectl auth can-i`
- Apply ResourceQuota and LimitRange defaults
- Enforce Pod Security Standards
- Add basic NetworkPolicies for namespace isolation

## Prerequisites

- Kind cluster running
- kubectl configured and working
- Basic understanding of namespaces and ServiceAccounts

## Tasks

### Task 1: Create Team Namespaces

Create two namespaces that represent separate tenants.

**Requirements:**

- Namespace `team-alpha`
- Namespace `team-beta`
- Add label `tenant=alpha` to `team-alpha`
- Add label `tenant=beta` to `team-beta`
- Add Pod Security labels to both namespaces
- Enforce the `restricted` profile
- Warn and audit with the `restricted` profile

<details class="hint" markdown="1">
<summary>Hint</summary>

Pod Security labels live on the Namespace:

```bash
kubectl label namespace team-alpha \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

Repeat for `team-beta`.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl create namespace team-alpha
kubectl create namespace team-beta

kubectl label namespace team-alpha tenant=alpha \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

kubectl label namespace team-beta tenant=beta \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

</details>

### Task 2: Create Team ServiceAccounts

Create separate identities for each team.

**Requirements:**

- ServiceAccount `team-alpha-user` in namespace `team-alpha`
- ServiceAccount `team-beta-user` in namespace `team-beta`
- Disable automatic token mounting on both ServiceAccounts
- Verify the ServiceAccounts exist

<details class="hint" markdown="1">
<summary>Hint</summary>

Disable automatic token mounting in the ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-alpha-user
  namespace: team-alpha
automountServiceAccountToken: false
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `team-alpha-sa.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-alpha-user
  namespace: team-alpha
automountServiceAccountToken: false
```

Create `team-beta-sa.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-beta-user
  namespace: team-beta
automountServiceAccountToken: false
```

Apply:

```bash
kubectl apply -f team-alpha-sa.yaml
kubectl apply -f team-beta-sa.yaml
kubectl get serviceaccount -n team-alpha
kubectl get serviceaccount -n team-beta
```

</details>

### Task 3: Grant Least-Privilege Access

Grant each team permission to manage common workload resources only in its own namespace.

**Requirements:**

- Role name: `developer`
- Role exists in both namespaces
- Allow read access to Pods, Services, ConfigMaps, and Secrets
- Allow read/write access to Deployments
- Allow read access to Pod logs
- Bind `team-alpha-user` to the `developer` Role only in `team-alpha`
- Bind `team-beta-user` to the `developer` Role only in `team-beta`

<details class="hint" markdown="1">
<summary>Hint</summary>

Deployments are in the `apps` API group. Pods, Services, ConfigMaps, Secrets, and Pod logs are in the core API group:

```yaml
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `team-alpha-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: team-alpha
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Create `team-alpha-binding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: team-alpha
subjects:
- kind: ServiceAccount
  name: team-alpha-user
  namespace: team-alpha
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Create `team-beta-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: team-beta
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Create `team-beta-binding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: team-beta
subjects:
- kind: ServiceAccount
  name: team-beta-user
  namespace: team-beta
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f team-alpha-rbac.yaml
kubectl apply -f team-alpha-binding.yaml
kubectl apply -f team-beta-rbac.yaml
kubectl apply -f team-beta-binding.yaml
```

</details>

### Task 4: Test RBAC Boundaries

Use authorization checks to prove each identity is namespace-scoped.

**Requirements:**

- `team-alpha-user` can list Pods in `team-alpha`
- `team-alpha-user` cannot list Pods in `team-beta`
- `team-alpha-user` can create Deployments in `team-alpha`
- `team-alpha-user` cannot delete namespaces
- `team-beta-user` has equivalent access only in `team-beta`

<details class="hint" markdown="1">
<summary>Hint</summary>

Use `kubectl auth can-i` with `--as`:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:team-alpha:team-alpha-user \
  -n team-alpha
```

Check a denied cross-namespace action:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:team-alpha:team-alpha-user \
  -n team-beta
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:team-alpha:team-alpha-user \
  -n team-alpha

kubectl auth can-i list pods \
  --as=system:serviceaccount:team-alpha:team-alpha-user \
  -n team-beta

kubectl auth can-i create deployments \
  --as=system:serviceaccount:team-alpha:team-alpha-user \
  -n team-alpha

kubectl auth can-i delete namespaces \
  --as=system:serviceaccount:team-alpha:team-alpha-user

kubectl auth can-i list pods \
  --as=system:serviceaccount:team-beta:team-beta-user \
  -n team-beta
```

**Expected output:**

```text
yes
no
yes
no
yes
```

</details>

### Task 5: Apply Resource Controls

Add guardrails to prevent one team from consuming unlimited resources.

**Requirements:**

- ResourceQuota named `team-quota` in both namespaces
- Maximum Pods: 10
- Maximum CPU requests: `2`
- Maximum memory requests: `4Gi`
- LimitRange named `default-limits` in both namespaces
- Default CPU request: `100m`
- Default memory request: `128Mi`
- Default CPU limit: `500m`
- Default memory limit: `512Mi`

<details class="hint" markdown="1">
<summary>Hint</summary>

ResourceQuota example:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 4Gi
```

LimitRange example:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    default:
      cpu: 500m
      memory: 512Mi
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `team-alpha-quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-alpha
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 4Gi
```

Create `team-alpha-limits.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-alpha
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    default:
      cpu: 500m
      memory: 512Mi
```

Create equivalent files for `team-beta`, replacing the namespace with `team-beta`.

Apply:

```bash
kubectl apply -f team-alpha-quota.yaml
kubectl apply -f team-alpha-limits.yaml
kubectl apply -f team-beta-quota.yaml
kubectl apply -f team-beta-limits.yaml
```

</details>

### Task 6: Deploy a Restricted Workload

Deploy a workload that passes the restricted Pod Security profile.

**Requirements:**

- Deployment name: `secure-web`
- Namespace: `team-alpha`
- Image: `nginxinc/nginx-unprivileged:1.25-alpine`
- Replicas: 1
- Run as non-root user `101`
- Drop all Linux capabilities
- Disable privilege escalation
- Use RuntimeDefault seccomp
- Use a read-only root filesystem
- Mount an `emptyDir` volume at `/tmp`
- Expose it with a ClusterIP Service named `secure-web`

<details class="hint" markdown="1">
<summary>Hint</summary>

Use a non-root nginx image and this security context shape:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 101
  seccompProfile:
    type: RuntimeDefault
```

Container security context:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
    - ALL
```

The `nginxinc/nginx-unprivileged` image listens on port 8080, not 80.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `secure-web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-web
  namespace: team-alpha
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secure-web
  template:
    metadata:
      labels:
        app: secure-web
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:1.25-alpine
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
```

Create `secure-web-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: secure-web
  namespace: team-alpha
spec:
  selector:
    app: secure-web
  ports:
  - port: 80
    targetPort: 8080
```

Apply:

```bash
kubectl apply -f secure-web-deployment.yaml
kubectl apply -f secure-web-service.yaml
kubectl wait --for=condition=Available deployment/secure-web -n team-alpha --timeout=90s
```

</details>

### Task 7: Verify Pod Security Blocks Risky Pods

Try to create a Pod that violates the restricted profile.

**Requirements:**

- Pod name: `privileged-test`
- Namespace: `team-alpha`
- Image: `busybox:1.36`
- Set `privileged: true`
- Confirm the API server rejects it
- Explain which namespace labels caused the rejection

<details class="hint" markdown="1">
<summary>Hint</summary>

Try a privileged Pod:

```bash
kubectl run privileged-test -n team-alpha --image=busybox:1.36 \
  --overrides='{"spec":{"containers":[{"name":"privileged-test","image":"busybox:1.36","command":["sleep","3600"],"securityContext":{"privileged":true}}]}}'
```

The `restricted` profile should reject it.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl run privileged-test -n team-alpha --image=busybox:1.36 \
  --overrides='{"spec":{"containers":[{"name":"privileged-test","image":"busybox:1.36","command":["sleep","3600"],"securityContext":{"privileged":true}}]}}'
```

**Expected result:**

```text
Error from server (Forbidden): pods "privileged-test" is forbidden: violates PodSecurity "restricted:latest": ...
```

The rejection is caused by `pod-security.kubernetes.io/enforce=restricted` on `team-alpha`. The restricted profile prohibits privileged containers, running as root, and disallowed capabilities.

</details>

### Task 8: Add Network Isolation

Add basic NetworkPolicies for tenant isolation.

**Requirements:**

- Default deny ingress policy in `team-alpha`
- Allow ingress to `secure-web` only from Pods in `team-alpha`
- Include a note that enforcement depends on the cluster CNI supporting NetworkPolicy
- Test connectivity from a Pod in `team-alpha`
- Test connectivity from a Pod in `team-beta`

> **Note:** NetworkPolicy enforcement requires a CNI plugin that supports NetworkPolicy (e.g., Calico, Cilium). kind uses kindnet by default, which does not enforce NetworkPolicies. Use this task to practice writing NetworkPolicy YAML and observe that the policies are accepted by the API server even if they are not enforced by kindnet.

<details class="hint" markdown="1">
<summary>Hint</summary>

A default deny ingress policy selects all Pods:

```yaml
podSelector: {}
policyTypes:
- Ingress
```

Allow traffic from same-namespace Pods:

```yaml
ingress:
- from:
  - podSelector: {}
```

Because both team namespaces enforce the `restricted` Pod Security profile, connectivity test Pods also need a restricted security context:

```bash
kubectl run curl-alpha -n team-alpha --image=curlimages/curl:8.5.0 \
  --restart=Never --rm -it \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"curl-alpha","image":"curlimages/curl:8.5.0","args":["-m","5","http://secure-web"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `team-alpha-default-deny.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

Create `team-alpha-allow-same-namespace.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace-to-secure-web
  namespace: team-alpha
spec:
  podSelector:
    matchLabels:
      app: secure-web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 8080
```

Apply:

```bash
kubectl apply -f team-alpha-default-deny.yaml
kubectl apply -f team-alpha-allow-same-namespace.yaml
kubectl get networkpolicy -n team-alpha
```

Test from `team-alpha`:

```bash
kubectl run curl-alpha -n team-alpha --image=curlimages/curl:8.5.0 \
  --restart=Never --rm -it \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"curl-alpha","image":"curlimages/curl:8.5.0","args":["-m","5","http://secure-web"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
```

Test from `team-beta`:

```bash
kubectl run curl-beta -n team-beta --image=curlimages/curl:8.5.0 \
  --restart=Never --rm -it \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"curl-beta","image":"curlimages/curl:8.5.0","args":["-m","5","http://secure-web.team-alpha.svc.cluster.local"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
```

**Expected result:** Same-namespace traffic succeeds. Cross-namespace traffic is blocked only when the cluster CNI enforces NetworkPolicy (not the case with default kindnet).

</details>

## Verification

Check your work:

```bash
kubectl get namespaces --show-labels | grep team-
kubectl get role,rolebinding,resourcequota,limitrange -n team-alpha
kubectl get role,rolebinding,resourcequota,limitrange -n team-beta
kubectl auth can-i list pods --as=system:serviceaccount:team-alpha:team-alpha-user -n team-alpha
kubectl auth can-i list pods --as=system:serviceaccount:team-alpha:team-alpha-user -n team-beta
kubectl get networkpolicy -n team-alpha
```

Expected outcomes:

- Each ServiceAccount can operate only in its own namespace
- Risky Pods are rejected by Pod Security admission
- ResourceQuota and LimitRange exist in both namespaces
- NetworkPolicies exist for `team-alpha`

## Cleanup

```bash
kubectl delete namespace team-alpha team-beta
```

## Bonus Challenges

- Add a read-only ClusterRole that allows viewing Nodes.
- Create a separate `team-alpha-viewer` ServiceAccount with a matching RoleBinding.
- Add an egress policy that allows DNS and blocks everything else.
- Try deploying a root nginx image and adjust it until it passes the restricted Pod Security profile.

## Key takeaways

1. **RBAC** uses Roles, ClusterRoles, and Bindings to grant least-privilege access — prefer RoleBindings scoped to a namespace over ClusterRoleBindings
2. **`kubectl auth can-i`** verifies effective permissions without trial and error
3. **Pod Security Standards** (baseline/restricted) block privilege escalation at admission time, before Pods are scheduled
4. **ResourceQuota** caps total namespace consumption; **LimitRange** sets per-container defaults and prevents unbounded Pods
5. **NetworkPolicies** restrict Pod-to-Pod traffic — a default-deny policy forces explicit allow rules for all communication
6. **Disabling automatic token mounting** reduces the attack surface of Pods that do not need API access
7. **NetworkPolicies are enforced by the CNI plugin**, not by kube-proxy — without a CNI that supports NetworkPolicy, rules are applied but silently ignored

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../08-multi-cluster/README.md).
