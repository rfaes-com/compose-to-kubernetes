# Lab Solutions: Namespaces and Resource Quotas

Complete solutions for the Namespaces lab exercises.

## Task 1: Create Namespaces

```bash
# Create dev namespace with label
kubectl create namespace dev
kubectl label namespace dev environment=development
```

**Expected output:**

```text
namespace/dev created
namespace/dev labeled
```

```bash
# Create staging namespace with label
kubectl create namespace staging
kubectl label namespace staging environment=staging
```

**Expected output:**

```text
namespace/staging created
namespace/staging labeled
```

```bash
# Create prod namespace with label
kubectl create namespace prod
kubectl label namespace prod environment=production
```

**Expected output:**

```text
namespace/prod created
namespace/prod labeled
```

```bash
# View all namespaces with labels
kubectl get namespaces --show-labels
```

**Expected output:**

```text
NAME              STATUS   AGE   LABELS
default           Active   10m   kubernetes.io/metadata.name=default
dev               Active   30s   environment=development,kubernetes.io/metadata.name=dev
kube-node-lease   Active   10m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   10m   kubernetes.io/metadata.name=kube-public
kube-system       Active   10m   kubernetes.io/metadata.name=kube-system
prod              Active   15s   environment=production,kubernetes.io/metadata.name=prod
staging           Active   20s   environment=staging,kubernetes.io/metadata.name=staging
```

```bash
# View specific namespaces
kubectl get namespaces dev staging prod
```

**Expected output:**

```text
NAME      STATUS   AGE
dev       Active   1m
staging   Active   50s
prod      Active   40s
```

## Task 2: Create ResourceQuota

```bash
# Create ResourceQuota for dev namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "3"
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
EOF
```

**Expected output:**

```text
resourcequota/dev-quota created
```

```bash
# View resource quota
kubectl get resourcequota -n dev
```

**Expected output:**

```text
NAME        AGE   REQUEST                                           LIMIT
dev-quota   10s   pods: 0/3, requests.cpu: 0/2, requests.memory: 0/4Gi   limits.cpu: 0/4, limits.memory: 0/8Gi
```

```bash
# Describe for detailed information
kubectl describe resourcequota dev-quota -n dev
```

**Expected output:**

```text
Name:            dev-quota
Namespace:       dev
Resource         Used  Hard
--------         ----  ----
limits.cpu       0     4
limits.memory    0     8Gi
pods             0     3
requests.cpu     0     2
requests.memory  0     4Gi
```

## Task 3: Create LimitRange

```bash
# Create LimitRange for dev namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
EOF
```

**Expected output:**

```text
limitrange/dev-limits created
```

```bash
# View LimitRange
kubectl get limitrange -n dev
```

**Expected output:**

```text
NAME         CREATED AT
dev-limits   2024-01-02T12:30:00Z
```

```bash
# Describe LimitRange
kubectl describe limitrange dev-limits -n dev
```

**Expected output:**

```text
Name:       dev-limits
Namespace:  dev
Type        Resource  Min   Max  Default Request  Default Limit
----        --------  ---   ---  ---------------  -------------
Container   cpu       50m   1    100m             200m
Container   memory    64Mi  1Gi  128Mi            256Mi
```

## Task 4: Deploy Without Resource Specifications

```bash
# Create pod without specifying resources
kubectl run test-pod-1 --image=nginx:1.25-alpine -n dev
```

**Expected output:**

```text
pod/test-pod-1 created
```

```bash
# Wait for pod to be ready
kubectl wait --for=condition=Ready pod/test-pod-1 -n dev --timeout=30s
```

**Expected output:**

```text
pod/test-pod-1 condition met
```

```bash
# Check applied resources (defaults from LimitRange)
kubectl get pod test-pod-1 -n dev -o jsonpath='{.spec.containers[0].resources}' | jq
```

**Expected output:**

```json
{
  "limits": {
    "cpu": "200m",
    "memory": "256Mi"
  },
  "requests": {
    "cpu": "100m",
    "memory": "128Mi"
  }
}
```

The LimitRange automatically applied default values!

```bash
# View full pod spec
kubectl get pod test-pod-1 -n dev -o yaml | grep -A 10 resources:
```

**Expected output:**

```yaml
    resources:
      limits:
        cpu: 200m
        memory: 256Mi
      requests:
        cpu: 100m
        memory: 128Mi
```

## Task 5: Test Quota Enforcement

```bash
# Create second pod
kubectl run test-pod-2 --image=nginx:1.25-alpine -n dev
```

**Expected output:**

```text
pod/test-pod-2 created
```

```bash
# Create third pod
kubectl run test-pod-3 --image=nginx:1.25-alpine -n dev
```

**Expected output:**

```text
pod/test-pod-3 created
```

```bash
# Check quota usage (3/3 pods used)
kubectl get resourcequota -n dev
```

**Expected output:**

```text
NAME        AGE   REQUEST                                                        LIMIT
dev-quota   5m    pods: 3/3, requests.cpu: 300m/2, requests.memory: 384Mi/4Gi   limits.cpu: 600m/4, limits.memory: 768Mi/8Gi
```

```bash
# Describe quota for detailed view
kubectl describe resourcequota dev-quota -n dev
```

**Expected output:**

```text
Name:            dev-quota
Namespace:       dev
Resource         Used   Hard
--------         ----   ----
limits.cpu       600m   4
limits.memory    768Mi  8Gi
pods             3      3
requests.cpu     300m   2
requests.memory  384Mi  4Gi
```

```bash
# Try to create 4th pod (should fail - exceeds quota)
kubectl run test-pod-4 --image=nginx:1.25-alpine -n dev
```

**Expected output:**

```text
Error from server (Forbidden): pods "test-pod-4" is forbidden: exceeded quota: dev-quota, requested: pods=1, used: pods=3, limited: pods=3
```

The quota prevents creating the 4th pod!

```bash
# Verify only 3 pods exist
kubectl get pods -n dev
```

**Expected output:**

```text
NAME         READY   STATUS    RESTARTS   AGE
test-pod-1   1/1     Running   0          3m
test-pod-2   1/1     Running   0          2m
test-pod-3   1/1     Running   0          1m
```

```bash
# Delete one pod to free quota
kubectl delete pod test-pod-3 -n dev
```

**Expected output:**

```text
pod "test-pod-3" deleted
```

```bash
# Check quota again (2/3 pods)
kubectl get resourcequota -n dev
```

**Expected output:**

```text
NAME        AGE   REQUEST                                                        LIMIT
dev-quota   7m    pods: 2/3, requests.cpu: 200m/2, requests.memory: 256Mi/4Gi   limits.cpu: 400m/4, limits.memory: 512Mi/8Gi
```

```bash
# Now creating pod-4 should succeed
kubectl run test-pod-4 --image=nginx:1.25-alpine -n dev
```

**Expected output:**

```text
pod/test-pod-4 created
```

## Task 6: Cross-Namespace Communication

```bash
# Create deployment in prod namespace
kubectl create deployment api --image=nginx:1.25-alpine -n prod
```

**Expected output:**

```text
deployment.apps/api created
```

```bash
# Expose as service
kubectl expose deployment api --port=80 -n prod
```

**Expected output:**

```text
service/api exposed
```

```bash
# Verify service exists
kubectl get service api -n prod
```

**Expected output:**

```text
NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
api    ClusterIP   10.96.123.45    <none>        80/TCP    10s
```

```bash
# Create test pod in dev namespace with interactive shell
kubectl run curl-pod --image=curlimages/curl:8.5.0 -n dev --rm -it --restart=Never -- sh
```

Inside the pod:

```bash
# Try short name (will fail - different namespace)
curl -v http://api 2>&1 | head -20
```

**Expected output:**

```text
* Could not resolve host: api
* Closing connection 0
curl: (6) Could not resolve host: api
```

```bash
# Try with namespace in DNS (will succeed)
curl -v http://api.prod.svc.cluster.local 2>&1 | head -20
```

**Expected output:**

```text
* Connected to api.prod.svc.cluster.local (10.96.123.45) port 80
> GET / HTTP/1.1
> Host: api.prod.svc.cluster.local
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.25.3
...
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

```bash
# Exit the pod
exit
```

**Expected output:**

```text
pod "curl-pod" deleted
```

Alternative non-interactive test:

```bash
# Run curl command directly
kubectl run curl-test --image=curlimages/curl:8.5.0 -n dev --rm --restart=Never -- \
  curl -s http://api.prod.svc.cluster.local | head -5
```

**Expected output:**

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
pod "curl-test" deleted
```

## Verification

```bash
# List all namespaces
kubectl get namespaces
```

**Expected output:**

```text
NAME              STATUS   AGE
default           Active   20m
dev               Active   10m
kube-node-lease   Active   20m
kube-public       Active   20m
kube-system       Active   20m
prod              Active   8m
staging           Active   9m
```

```bash
# List resource quotas across all namespaces
kubectl get resourcequota --all-namespaces
```

**Expected output:**

```text
NAMESPACE   NAME        AGE   REQUEST                                                        LIMIT
dev         dev-quota   8m    pods: 3/3, requests.cpu: 300m/2, requests.memory: 384Mi/4Gi   limits.cpu: 600m/4, limits.memory: 768Mi/8Gi
```

```bash
# List limit ranges
kubectl get limitrange --all-namespaces
```

**Expected output:**

```text
NAMESPACE   NAME         CREATED AT
dev         dev-limits   2024-01-02T12:30:00Z
```

```bash
# View quota usage
kubectl describe quota dev-quota -n dev
```

**Expected output:**

```text
Name:            dev-quota
Namespace:       dev
Resource         Used   Hard
--------         ----   ----
limits.cpu       600m   4
limits.memory    768Mi  8Gi
pods             3      3
requests.cpu     300m   2
requests.memory  384Mi  4Gi
```

```bash
# View all pods with namespace
kubectl get pods --all-namespaces -l app!=kube-system
```

**Expected output:**

```text
NAMESPACE   NAME                   READY   STATUS    RESTARTS   AGE
dev         test-pod-1             1/1     Running   0          8m
dev         test-pod-2             1/1     Running   0          7m
dev         test-pod-4             1/1     Running   0          3m
prod        api-5c8b9d7f8d-abcde   1/1     Running   0          5m
```

```bash
# View pods by namespace
kubectl get pods -n dev
kubectl get pods -n prod
kubectl get pods -n staging
```

## Cleanup

```bash
# Delete custom namespaces (this deletes ALL resources within them)
kubectl delete namespace dev
kubectl delete namespace staging
kubectl delete namespace prod
```

**Expected output:**

```text
namespace "dev" deleted
namespace "staging" deleted
namespace "prod" deleted
```

```bash
# Verify deletion
kubectl get namespaces
```

**Expected output:**

```text
NAME              STATUS   AGE
default           Active   25m
kube-node-lease   Active   25m
kube-public       Active   25m
kube-system       Active   25m
```

```bash
# Verify no resources remain
kubectl get pods --all-namespaces | grep -E "dev|staging|prod"
```

**Expected output:**
(No output - all resources deleted)

## Bonus Challenges

### Challenge 1: Quota for Specific Resource Types

Create a quota that limits specific service types.

```bash
# Create namespace
kubectl create namespace limited

# Create quota for specific resources
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: service-quota
  namespace: limited
spec:
  hard:
    services.nodeports: "1"
    services.loadbalancers: "0"
EOF

# Try to create NodePort services
kubectl create deployment web --image=nginx:1.25-alpine -n limited
kubectl expose deployment web --type=NodePort --port=80 -n limited --name=web-1

# This should succeed (1/1)
kubectl expose deployment web --type=NodePort --port=80 -n limited --name=web-2
```

**Expected:** Second NodePort service should fail due to quota.

```bash
# Cleanup
kubectl delete namespace limited
```

### Challenge 2: Per-Priority-Class Quotas

Create quotas that differ based on priority class.

```bash
# Create namespace
kubectl create namespace priority-test

# Create priority classes
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
EOF

# Create quota for high priority
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: priority-test
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority"]
EOF

# Test with high priority pod
kubectl run high-pod --image=nginx:1.25-alpine -n priority-test \
  --overrides='{"spec":{"priorityClassName":"high-priority"}}'
```

```bash
# Cleanup
kubectl delete priorityclass high-priority low-priority
kubectl delete namespace priority-test
```

## Key Takeaways

1. **Namespaces** provide resource isolation within a cluster
2. **ResourceQuotas** limit total resource consumption per namespace
3. **LimitRanges** provide defaults and constraints for individual pods/containers
4. Quotas are enforced at resource creation time
5. Deleting a namespace deletes ALL resources within it
6. Cross-namespace DNS: `<service>.<namespace>.svc.cluster.local`
7. When quotas are active for CPU/memory, pods must specify requests/limits or have LimitRange defaults

## Common Issues and Solutions

**Issue:** Pod creation fails with "exceeded quota"

- **Cause:** Namespace has reached its resource limits
- **Solution:** Delete unused resources or increase quota

**Issue:** Pod rejected: "must specify requests/limits"

- **Cause:** ResourceQuota active but pod has no resource specifications and no LimitRange
- **Solution:** Add LimitRange to namespace or specify resources in pod spec

**Issue:** Cannot access service from another namespace

- **Cause:** Using short service name instead of FQDN
- **Solution:** Use `<service>.<namespace>.svc.cluster.local`

**Issue:** Namespace stuck in Terminating state

- **Cause:** Finalizers preventing deletion or resources with owner references
- **Solution:** Check for stuck resources: `kubectl get all -n <namespace>`, force delete if needed
