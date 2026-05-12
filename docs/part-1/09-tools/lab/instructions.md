# Lab: kubectl and k9s Essentials

**Duration:** 30 minutes

## Objectives

- Practice essential kubectl commands
- Use kubectl shortcuts and productivity features
- Debug common issues
- Explore cluster using k9s
- Master log viewing and pod inspection

## Prerequisites

- Kind cluster running
- kubectl configured
- k9s installed (available in the workshop container)

## Tasks

### Task 1: kubectl Basics

Practice fundamental kubectl commands.

**Requirements:**

- Create a deployment imperatively
- Scale it
- View logs
- Get resource in different output formats

**Commands:**

```bash
# Create deployment
kubectl create deployment web --image=nginx:1.25-alpine --replicas=2

# View pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods -o yaml | head -30
kubectl get pods -o json | jq '.items[].metadata.name'

# Scale deployment
kubectl scale deployment web --replicas=4

# Watch scaling
kubectl get pods -w
# (Ctrl+C to stop)

# View deployment
kubectl get deployment web
kubectl describe deployment web

# View logs
kubectl logs deployment/web
kubectl logs -l app=web --tail=10
```

### Task 2: Resource Filtering and Selection

Use labels and selectors to filter resources.

**Requirements:**

- Create multiple pods with different labels
- Filter by labels
- Use field selectors

**Commands:**

```bash
# Create pods with labels
kubectl run pod1 --image=nginx:1.25-alpine --labels="app=web,env=dev"
kubectl run pod2 --image=nginx:1.25-alpine --labels="app=web,env=prod"
kubectl run pod3 --image=nginx:1.25-alpine --labels="app=api,env=dev"

# Filter by single label
kubectl get pods -l app=web

# Filter by multiple labels
kubectl get pods -l 'app=web,env=dev'

# Negative filter
kubectl get pods -l 'env!=prod'

# Show labels
kubectl get pods --show-labels

# Filter by field selector
kubectl get pods --field-selector status.phase=Running

# Combine filters
kubectl get pods -l app=web --field-selector status.phase=Running
```

### Task 3: Debugging Failing Pod

Debug a pod that won't start.

**Requirements:**

- Create a pod with an error
- Use `describe` and `logs` to diagnose
- Fix the issue

**Commands:**

```bash
# Create pod with wrong image
kubectl run broken --image=nginx:nonexistent

# Check status
kubectl get pods broken

# See why it failed
kubectl describe pod broken
# Look for "Events" section - shows "ImagePullBackOff"

# Check logs (will be empty since container never started)
kubectl logs broken

# Fix by updating image
kubectl set image pod/broken broken=nginx:1.25-alpine

# Or delete and recreate
kubectl delete pod broken
kubectl run broken --image=nginx:1.25-alpine

# Verify it's running
kubectl get pods broken
```

### Task 4: Executing Commands and Port Forwarding

Access pods and services.

**Requirements:**

- Execute commands in a running pod
- Get an interactive shell
- Forward ports to access locally

**Commands:**

```bash
# Create a service
kubectl expose deployment web --port=80

# Execute single command
kubectl exec deployment/web -- nginx -v

# Interactive shell
kubectl exec -it deployment/web -- sh
# Inside pod:
ls /etc/nginx/
cat /etc/nginx/nginx.conf | head -10
exit

# Port forward pod
kubectl port-forward deployment/web 8080:80 &
PF_PID=$!

# Test locally
curl http://localhost:8080

# Stop port forward
kill $PF_PID
```

### Task 5: JSONPath and Custom Output

Extract specific information using JSONPath.

**Requirements:**

- Get pod names only
- Extract pod IPs
- Use custom columns

**Commands:**

```bash
# Get all pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
echo ""

# Get pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
echo ""

# Formatted output with newlines
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP,NODE:.spec.nodeName

# Get container image for all pods
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
echo ""

# Get resource requests
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources.requests.memory}{"\n"}{end}'
```

### Task 6: Manifest Generation

Use `--dry-run` to generate manifests.

**Requirements:**

- Generate deployment YAML
- Generate service YAML
- Combine and apply

**Commands:**

```bash
# Generate deployment YAML
kubectl create deployment api --image=nginx:1.25-alpine --replicas=3 --dry-run=client -o yaml > api-deployment.yaml

# View generated file
cat api-deployment.yaml

# Generate service YAML
kubectl create service clusterip api --tcp=80:80 --dry-run=client -o yaml > api-service.yaml

# View generated file
cat api-service.yaml

# Apply both
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml

# Verify
kubectl get deployment,service api

# Cleanup generated files
rm api-deployment.yaml api-service.yaml
```

### Task 7: Working with Contexts and Namespaces

Manage contexts and default namespaces.

**Requirements:**

- Create a namespace
- Set it as default
- Switch back to default

**Commands:**

```bash
# Create namespace
kubectl create namespace testing

# View current context
kubectl config current-context

# Set default namespace
kubectl config set-context --current --namespace=testing

# Verify (should show "testing")
kubectl config view --minify | grep namespace:

# Create resource (goes to testing namespace)
kubectl run test-pod --image=nginx:1.25-alpine

# Verify pod is in testing namespace
kubectl get pods
kubectl get pods -n testing
kubectl get pods -n default  # Empty

# Switch back to default namespace
kubectl config set-context --current --namespace=default

# Verify
kubectl get pods  # Empty
kubectl get pods -n testing  # Has test-pod
```

### Task 8: Using k9s

Explore the cluster using k9s.

**Requirements:**

- Launch k9s
- Navigate between resources
- View logs
- Delete a resource

**Steps:**

```bash
# Launch k9s
k9s

# Inside k9s, try these commands:
# :pods     - View all pods
# :svc      - View services
# :deploy   - View deployments
# 0         - Show all namespaces
# /web      - Filter for "web"
# esc       - Clear filter
#
# Navigate to a pod and:
# enter     - View details
# l         - View logs
# s         - Shell into pod
# d         - Describe
# y         - View YAML
# ctrl+k    - Delete
#
# :q or Ctrl+C to quit
```

**k9s Practice:**

1. View all pods across all namespaces (`:pods` then `0`)
2. Filter for nginx pods (`/nginx`)
3. Select a pod and view logs (`enter` then `l`)
4. Shell into a pod (`s`)
5. View pod YAML (`y`)
6. Go back to pods and delete test-pod (`ctrl+k`)

## Verification

```bash
# List all resources
kubectl get all

# Check across all namespaces
kubectl get all -A

# Verify deployments
kubectl get deployments

# Check services
kubectl get services

# View pods with labels
kubectl get pods --show-labels

# Check what namespace is default
kubectl config view --minify | grep namespace:
```

## Cleanup

```bash
# Delete deployments
kubectl delete deployment web api

# Delete services
kubectl delete service web api

# Delete standalone pods
kubectl delete pod pod1 pod2 pod3 broken

# Delete namespace
kubectl delete namespace testing

# Verify cleanup
kubectl get all
```

## Check Your Understanding

1. How do you view logs from all containers in a deployment?
2. What's the difference between `-o yaml` and `-o json` output?
3. How can you filter pods that are NOT in "Running" phase?
4. What does `--dry-run=client` do differently from `--dry-run=server`?
5. How do you access a ClusterIP service from your local machine?
6. What's the advantage of using k9s over kubectl for log viewing?

## Bonus Challenges

### Challenge 1: Custom Resource Queries

Extract specific information using advanced JSONPath.

```bash
# Get all container images used in the cluster
kubectl get pods -A -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u

# Get pods with restart count > 0
kubectl get pods -o json | jq -r '.items[] | select(.status.containerStatuses[]?.restartCount > 0) | .metadata.name'

# Get pods scheduled on specific node
kubectl get pods -A --field-selector spec.nodeName=kind-worker
```

### Challenge 2: Debugging Network Issues

```bash
# Create test deployment
kubectl create deployment nettest --image=nicolaka/netshoot -- sleep 3600

# Shell into pod
kubectl exec -it deployment/nettest -- bash

# Inside pod, test:
# DNS resolution
nslookup kubernetes.default

# Connectivity to service
curl http://web.default.svc.cluster.local

# Network testing
ping google.com
traceroute google.com

# Exit
exit

# Cleanup
kubectl delete deployment nettest
```

### Challenge 3: Batch Operations

```bash
# Create multiple resources
for i in {1..5}; do kubectl run pod-$i --image=nginx:1.25-alpine; done

# Get all pod names
kubectl get pods -o name

# Delete all pods matching pattern
kubectl get pods -o name | grep pod- | xargs kubectl delete

# Or using labels
kubectl delete pods -l run=pod-1,run=pod-2,run=pod-3
```

## Key Takeaways

1. **kubectl get** with various output formats (`-o wide`, `-o yaml`, `-o json`)
2. **kubectl describe** shows events and detailed information
3. **Labels** make filtering and selection powerful
4. **`--dry-run=client -o yaml`** generates manifests without creating resources
5. **JSONPath** extracts specific data from resources
6. **k9s** provides faster navigation and real-time monitoring
7. **`port-forward`** enables local access to cluster services
8. **Contexts** manage multiple clusters and default namespaces

## Next Steps

Proceed to [Manifest Organization and Best Practices](../../10-manifests/README.md).
