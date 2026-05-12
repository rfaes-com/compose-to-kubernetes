# Lab: Namespaces and Resource Quotas

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

**Hints:**

```bash
# Create namespaces
kubectl create namespace dev --dry-run=client -o yaml | \
  kubectl label --local -f - environment=development -o yaml | \
  kubectl apply -f -

kubectl create namespace staging
kubectl create namespace prod

# Add labels
kubectl label namespace staging environment=staging
kubectl label namespace prod environment=production

# View namespaces
kubectl get namespaces --show-labels
```

### Task 2: Create ResourceQuota

Create a ResourceQuota for the dev namespace.

**Requirements:**

- Namespace: `dev`
- Quota name: `dev-quota`
- Limits:
  - Max 3 pods
  - Max 2 CPU requests
  - Max 4Gi memory requests

**Hints:**

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

### Task 3: Create LimitRange

Create a LimitRange in the dev namespace to provide default resource values.

**Requirements:**

- Namespace: `dev`
- LimitRange name: `dev-limits`
- Default CPU request: 100m
- Default memory request: 128Mi
- Default CPU limit: 200m
- Default memory limit: 256Mi

**Hints:**

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
```

### Task 4: Deploy Without Resource Specifications

Create a Pod in the dev namespace without specifying resources. Verify defaults are applied.

**Requirements:**

- Namespace: `dev`
- Pod name: `test-pod-1`
- Image: nginx:1.25-alpine
- Don't specify resources

**Hints:**

```bash
# Create pod without resources
kubectl run test-pod-1 --image=nginx:1.25-alpine -n dev

# Check applied resources
kubectl get pod test-pod-1 -n dev -o jsonpath='{.spec.containers[0].resources}'
```

### Task 5: Test Quota Enforcement

Try to create more pods than the quota allows.

**Requirements:**

- Try to create 4 pods in dev namespace (quota is 3)
- Observe the error when quota is exceeded

**Hints:**

```bash
# Create 2 more pods (total 3)
kubectl run test-pod-2 --image=nginx:1.25-alpine -n dev
kubectl run test-pod-3 --image=nginx:1.25-alpine -n dev

# Check quota usage
kubectl get resourcequota -n dev
kubectl describe resourcequota dev-quota -n dev

# Try to create 4th pod (should fail)
kubectl run test-pod-4 --image=nginx:1.25-alpine -n dev
```

### Task 6: Cross-Namespace Communication

Create services in different namespaces and test DNS resolution.

**Requirements:**

- Create a service in `prod` namespace
- Create a pod in `dev` namespace
- Test accessing the service from dev using FQDN

**Hints:**

```bash
# Create deployment and service in prod
kubectl create deployment api --image=nginx:1.25-alpine -n prod
kubectl expose deployment api --port=80 -n prod

# Create test pod in dev
kubectl run curl-pod --image=curlimages/curl:8.5.0 -n dev --rm -it --restart=Never -- sh

# Inside the pod:
# Try short name (will fail - different namespace)
curl http://api

# Try FQDN (will succeed)
curl http://api.prod.svc.cluster.local

# Exit
exit
```

## Verification

Check your work:

```bash
# List all namespaces
kubectl get namespaces

# List resource quotas across all namespaces
kubectl get resourcequota --all-namespaces

# List limit ranges
kubectl get limitrange --all-namespaces

# View quota usage
kubectl describe quota dev-quota -n dev

# View all pods with namespace
kubectl get pods --all-namespaces

# View pods in specific namespace
kubectl get pods -n dev
kubectl get pods -n prod
```

## Cleanup

```bash
# Delete all test resources
kubectl delete namespace dev
kubectl delete namespace staging
kubectl delete namespace prod

# Verify deletion
kubectl get namespaces
```

Note: Deleting a namespace deletes ALL resources within it!

## Check Your Understanding

1. What happens to pods when you delete a namespace?
2. What error occurs when you exceed a ResourceQuota?
3. How do LimitRanges differ from ResourceQuotas?
4. What is the FQDN format for accessing services across namespaces?
5. Why are pods rejected when a ResourceQuota for CPU/memory exists but pods don't specify resources?

## Next Steps

Proceed to [kubectl and k9s Essentials](../../09-tools/README.md) to learn essential command-line tools.
