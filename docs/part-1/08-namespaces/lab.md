# Lab

**Duration:** 25 minutes

## Objectives

- Create and manage multiple namespaces
- Apply ResourceQuotas and observe enforcement
- Set LimitRanges and see default values applied
- Test cross-namespace communication
- Understand quota violations

## Prerequisites

- Kind cluster running
- kubectl configured

## Tasks

### Task 1: Create Namespaces

Create three namespaces for different environments.

**Requirements:**

- Namespaces: `dev`, `staging`, `prod`
- Add labels to identify environment

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod

kubectl label namespace dev environment=development
kubectl label namespace staging environment=staging
kubectl label namespace prod environment=production
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create namespaces
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod

# Add labels
kubectl label namespace dev environment=development
kubectl label namespace staging environment=staging
kubectl label namespace prod environment=production

# View namespaces with labels
kubectl get namespaces --show-labels

# Output:
# NAME              STATUS   AGE   LABELS
# default           Active   1h    kubernetes.io/metadata.name=default
# dev               Active   5s    environment=development,...
# prod              Active   5s    environment=production,...
# staging           Active   5s    environment=staging,...
```

</details>

### Task 2: Create ResourceQuota

Create a ResourceQuota for the dev namespace.

**Requirements:**

- Namespace: `dev`
- Quota name: `dev-quota`
- Limits:
  - Max 3 pods
  - Max 2 CPU requests
  - Max 4Gi memory requests
  - Max 4 CPU limits
  - Max 8Gi memory limits

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
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

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
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

# Output:
# resourcequota/dev-quota created

# Verify
kubectl get resourcequota -n dev
kubectl describe resourcequota dev-quota -n dev

# Output:
# Name:            dev-quota
# Namespace:       dev
# Resource         Used  Hard
# --------         ----  ----
# limits.cpu       0     4
# limits.memory    0     8Gi
# pods             0     3
# requests.cpu     0     2
# requests.memory  0     4Gi
```

</details>

### Task 3: Create LimitRange

Create a LimitRange in the dev namespace to provide default resource values.

**Requirements:**

- Namespace: `dev`
- LimitRange name: `dev-limits`
- Default CPU request: 100m, limit: 200m
- Default memory request: 128Mi, limit: 256Mi
- Max: 1 CPU, 1Gi memory
- Min: 50m CPU, 64Mi memory

<details class="hint" markdown="1">
<summary>Hint</summary>

```yaml
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
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
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

# Output:
# limitrange/dev-limits created

kubectl describe limitrange dev-limits -n dev
```

</details>

### Task 4: Deploy Without Resource Specifications

Create a Pod in the dev namespace without specifying resources. Verify defaults are applied from the LimitRange.

**Requirements:**

- Namespace: `dev`
- Pod name: `test-pod-1`
- Image: `nginx:1.25-alpine`
- Do NOT specify any resources

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Create pod without resources
kubectl run test-pod-1 --image=nginx:1.25-alpine -n dev

# Check applied resources
kubectl get pod test-pod-1 -n dev -o jsonpath='{.spec.containers[0].resources}'
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create pod without resources in dev namespace
kubectl run test-pod-1 --image=nginx:1.25-alpine -n dev

# Wait for pod to start
kubectl wait --for=condition=Ready pod/test-pod-1 -n dev --timeout=30s

# Check applied resources - LimitRange defaults should be applied
kubectl get pod test-pod-1 -n dev -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool

# Output (defaults applied by LimitRange):
# {
#   "limits": {
#     "cpu": "200m",
#     "memory": "256Mi"
#   },
#   "requests": {
#     "cpu": "100m",
#     "memory": "128Mi"
#   }
# }

# Also check quota usage (1 pod now counted)
kubectl describe resourcequota dev-quota -n dev
```

</details>

### Task 5: Test Quota Enforcement

Try to create more Pods than the quota allows.

**Requirements:**

- Create Pods 2 and 3 in dev namespace (to reach the limit of 3)
- Check quota usage
- Try to create a 4th Pod and observe the error

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Create 2 more pods
kubectl run test-pod-2 --image=nginx:1.25-alpine -n dev
kubectl run test-pod-3 --image=nginx:1.25-alpine -n dev

# Check quota usage
kubectl describe resourcequota dev-quota -n dev

# Try to create 4th pod (should fail)
kubectl run test-pod-4 --image=nginx:1.25-alpine -n dev
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create 2 more pods to reach quota
kubectl run test-pod-2 --image=nginx:1.25-alpine -n dev
kubectl run test-pod-3 --image=nginx:1.25-alpine -n dev

# Check quota usage (should show 3 pods used)
kubectl describe resourcequota dev-quota -n dev

# Output:
# Name:            dev-quota
# Namespace:       dev
# Resource         Used   Hard
# --------         ----   ----
# limits.cpu       600m   4
# limits.memory    768Mi  8Gi
# pods             3      3    <-- quota at maximum
# requests.cpu     300m   2
# requests.memory  384Mi  4Gi

# Try to create a 4th pod (will be rejected)
kubectl run test-pod-4 --image=nginx:1.25-alpine -n dev

# Output:
# Error from server (Forbidden): pods "test-pod-4" is forbidden:
# exceeded quota: dev-quota, requested: pods=1, used: pods=3, limited: pods=3
```

</details>

### Task 6: Cross-Namespace Communication

Create services in different namespaces and test DNS resolution.

**Requirements:**

- Create an nginx deployment and Service in `prod` namespace
- Create a temporary curl Pod in `dev` namespace
- Test DNS resolution using the short name and FQDN

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Create deployment and service in prod
kubectl create deployment api --image=nginx:1.25-alpine -n prod
kubectl expose deployment api --port=80 -n prod

# Test from dev namespace
kubectl run curl-pod --image=curlimages/curl:8.5.0 -n dev --rm -it --restart=Never -- sh

# Inside the pod:
# Short name fails (different namespace)
curl http://api

# FQDN works
curl http://api.prod.svc.cluster.local
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create deployment and service in prod
kubectl create deployment api --image=nginx:1.25-alpine -n prod
kubectl expose deployment api --port=80 -n prod

# Wait for deployment to be ready
kubectl wait --for=condition=Available deployment/api -n prod --timeout=60s

# Create curl pod in dev and test DNS
kubectl run curl-pod --image=curlimages/curl:8.5.0 -n dev --rm -it --restart=Never -- sh

# Inside the pod:
# Try short name (will fail with "Could not resolve host")
curl --max-time 3 http://api || echo "Short name failed"

# Try with namespace (will work)
curl http://api.prod.svc.cluster.local

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>

# Exit
exit
```

</details>

## Verification

```bash
# Check all three namespaces
kubectl get namespaces --show-labels

# Check ResourceQuota usage
kubectl describe resourcequota dev-quota -n dev

# Check LimitRange
kubectl describe limitrange dev-limits -n dev

# List all pods in dev namespace
kubectl get pods -n dev

# List resources in prod
kubectl get all -n prod
```

## Cleanup

```bash
# Delete namespaces (removes all resources within)
kubectl delete namespace dev staging prod

# Verify cleanup
kubectl get namespaces
```

## Bonus Challenges

**1. Switch Default Namespace:** Change your kubectl default namespace and verify it works

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Recreate dev namespace
kubectl create namespace dev

# Run nginx in dev
kubectl run web --image=nginx:1.25-alpine -n dev

# Set default namespace to dev
kubectl config set-context --current --namespace=dev

# Now this works without -n dev
kubectl get pods

# Output:
# NAME   READY   STATUS    RESTARTS   AGE
# web    1/1     Running   0          30s

# Reset to default
kubectl config set-context --current --namespace=default

# Cleanup
kubectl delete namespace dev
```

</details>

**2. Memory Limit Violation:** Try to create a container that exceeds the LimitRange max

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Recreate namespace with LimitRange
kubectl create namespace dev
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - type: Container
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
    default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
EOF

# Try to create a Pod exceeding memory limit
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: big-pod
  namespace: dev
spec:
  containers:
  - name: app
    image: nginx:1.25-alpine
    resources:
      limits:
        memory: 2Gi    # Exceeds max of 1Gi
EOF

# Output:
# Error from server (Forbidden): error when creating "STDIN":
# pods "big-pod" is forbidden: maximum memory limit is 1Gi, but limit is 2Gi

# Cleanup
kubectl delete namespace dev
```

</details>

## Common issues

| Issue                                   | Solution                                                                                        |
| --------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Pods rejected with "exceeded quota"     | ResourceQuota is at limit; delete unneeded resources or increase quota                          |
| Pod rejected with "must specify limits" | ResourceQuota is active but pod has no resources; add LimitRange defaults or explicit resources |
| Cross-namespace DNS not resolving       | Use the FQDN: `service.namespace.svc.cluster.local`                                             |
| Can't create resources in namespace     | Check RBAC — you may not have permission in that namespace                                      |

## Key takeaways

1. **Namespaces** isolate resources logically within a cluster
2. **ResourceQuotas** enforce limits on total resource consumption per namespace
3. **LimitRanges** provide defaults and constraints for individual containers
4. **When ResourceQuota enforces CPU/memory, all Pods must specify requests/limits**
5. **Cross-namespace DNS** uses the FQDN pattern: `service.namespace.svc.cluster.local`
6. **Deleting a namespace deletes everything inside it**


## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../09-tools/README.md)
