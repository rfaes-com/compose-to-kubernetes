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

### Task 2: Create Team ServiceAccounts

Create separate identities for each team.

**Requirements:**

- ServiceAccount `team-alpha-user` in namespace `team-alpha`
- ServiceAccount `team-beta-user` in namespace `team-beta`
- Disable automatic token mounting on both ServiceAccounts
- Verify the ServiceAccounts exist

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

### Task 4: Test RBAC Boundaries

Use authorization checks to prove each identity is namespace-scoped.

**Requirements:**

- `team-alpha-user` can list Pods in `team-alpha`
- `team-alpha-user` cannot list Pods in `team-beta`
- `team-alpha-user` can create Deployments in `team-alpha`
- `team-alpha-user` cannot delete namespaces
- `team-beta-user` has equivalent access only in `team-beta`

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

### Task 7: Verify Pod Security Blocks Risky Pods

Try to create a Pod that violates the restricted profile.

**Requirements:**

- Pod name: `privileged-test`
- Namespace: `team-alpha`
- Image: `busybox:1.36`
- Set `privileged: true`
- Confirm the API server rejects it
- Explain which namespace labels caused the rejection

### Task 8: Add Network Isolation

Add basic NetworkPolicies for tenant isolation.

**Requirements:**

- Default deny ingress policy in `team-alpha`
- Allow ingress to `secure-web` only from Pods in `team-alpha`
- Include a note that enforcement depends on the cluster CNI supporting NetworkPolicy
- Test connectivity from a Pod in `team-alpha`
- Test connectivity from a Pod in `team-beta`

## Hints

<details markdown="1">
<summary>Hint for Task 1: Pod Security namespace labels</summary>

Pod Security labels live on the Namespace:

```bash
kubectl label namespace team-alpha \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

Repeat for `team-beta`.

</details>

<details markdown="1">
<summary>Hint for Task 2: ServiceAccount token mounting</summary>

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

<details markdown="1">
<summary>Hint for Task 3: Developer Role rules</summary>

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

<details markdown="1">
<summary>Hint for Task 4: Testing permissions</summary>

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

<details markdown="1">
<summary>Hint for Task 5: ResourceQuota and LimitRange</summary>

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

<details markdown="1">
<summary>Hint for Task 6: Restricted security context</summary>

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

</details>

<details markdown="1">
<summary>Hint for Task 7: Testing Pod Security admission</summary>

Try a privileged Pod:

```bash
kubectl run privileged-test -n team-alpha --image=busybox:1.36 \
  --overrides='{"spec":{"containers":[{"name":"privileged-test","image":"busybox:1.36","command":["sleep","3600"],"securityContext":{"privileged":true}}]}}'
```

The restricted profile should reject it.

</details>

<details markdown="1">
<summary>Hint for Task 8: NetworkPolicy selectors</summary>

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

NetworkPolicy enforcement requires a CNI that supports NetworkPolicy.

Because both team namespaces enforce the restricted Pod Security profile, connectivity test Pods also need a restricted security context. A temporary curl Pod can be created with an override:

```bash
kubectl run curl-alpha -n team-alpha --image=curlimages/curl:8.5.0 \
  --restart=Never --rm -it \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"curl-alpha","image":"curlimages/curl:8.5.0","args":["-m","5","http://secure-web"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
```

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

## Check Your Understanding

1. Why use RoleBindings instead of ClusterRoleBindings for team access?
2. Why should ServiceAccounts avoid unnecessary token mounting?
3. What does the restricted Pod Security profile prevent?
4. Why do NetworkPolicies require CNI support?
5. How do ResourceQuota and LimitRange complement each other?

## Bonus Challenges

- Add a read-only ClusterRole that allows viewing Nodes.
- Create a separate `team-alpha-viewer` ServiceAccount.
- Add an egress policy that allows DNS and blocks everything else.
- Try deploying a root nginx image and adjust it until it passes restricted Pod Security.
