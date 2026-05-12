# Lab Solutions: Security and RBAC

Complete solutions for the Security and RBAC lab exercises.

## Task 1: Create Team Namespaces

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

## Task 2: Create Team ServiceAccounts

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

## Task 3: Grant Least-Privilege Access

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

Create equivalent files for `team-beta`, replacing namespace and ServiceAccount names with `team-beta`.

Apply:

```bash
kubectl apply -f team-alpha-rbac.yaml
kubectl apply -f team-alpha-binding.yaml
kubectl apply -f team-beta-rbac.yaml
kubectl apply -f team-beta-binding.yaml
```

## Task 4: Test RBAC Boundaries

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

## Task 5: Apply Resource Controls

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

Create equivalent files for `team-beta`, replacing the namespace.

Apply:

```bash
kubectl apply -f team-alpha-quota.yaml
kubectl apply -f team-alpha-limits.yaml
kubectl apply -f team-beta-quota.yaml
kubectl apply -f team-beta-limits.yaml
```

## Task 6: Deploy a Restricted Workload

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

## Task 7: Verify Pod Security Blocks Risky Pods

```bash
kubectl run privileged-test -n team-alpha --image=busybox:1.36 \
  --overrides='{"spec":{"containers":[{"name":"privileged-test","image":"busybox:1.36","command":["sleep","3600"],"securityContext":{"privileged":true}}]}}'
```

**Expected result:**

```text
The Pod is rejected because team-alpha enforces pod-security.kubernetes.io/enforce=restricted.
```

## Task 8: Add Network Isolation

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

**Expected result:**

```text
Same-namespace traffic succeeds. Cross-namespace traffic is blocked only when the cluster CNI enforces NetworkPolicy.
```

## Verification

```bash
kubectl get namespaces --show-labels | grep team-
kubectl get role,rolebinding,resourcequota,limitrange -n team-alpha
kubectl get role,rolebinding,resourcequota,limitrange -n team-beta
kubectl get networkpolicy -n team-alpha
```

## Cleanup

```bash
kubectl delete namespace team-alpha team-beta
```
