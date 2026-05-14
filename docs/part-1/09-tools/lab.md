# Lab

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

Practice fundamental kubectl commands: create, scale, observe, and inspect.

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Create deployment
kubectl create deployment web --image=nginx:1.25-alpine --replicas=2

# View pods in different formats
kubectl get pods
kubectl get pods -o wide
kubectl get pods -o yaml | head -30

# Scale, watch, describe
kubectl scale deployment web --replicas=4
kubectl get pods -w
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create deployment
kubectl create deployment web --image=nginx:1.25-alpine --replicas=2

# View pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods -o yaml | head -30

# Scale deployment
kubectl scale deployment web --replicas=4

# Watch scaling (Ctrl+C to stop)
kubectl get pods -w

# View deployment
kubectl get deployment web
kubectl describe deployment web

# View logs
kubectl logs deployment/web
kubectl logs -l app=web --tail=10
```

</details>

### Task 2: Resource Filtering and Selection

Create multiple pods with different labels and practice filtering.

**Requirements:**

- Create 3 pods with different `app` and `env` labels
- Filter by single label, multiple labels, and negation
- Use `--show-labels` and `--field-selector`

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
kubectl run pod1 --image=nginx:1.25-alpine --labels="app=web,env=dev"
kubectl run pod2 --image=nginx:1.25-alpine --labels="app=web,env=prod"
kubectl run pod3 --image=nginx:1.25-alpine --labels="app=api,env=dev"

kubectl get pods -l app=web
kubectl get pods -l 'app=web,env=dev'
kubectl get pods -l 'env!=prod'
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create pods with labels
kubectl run pod1 --image=nginx:1.25-alpine --labels="app=web,env=dev"
kubectl run pod2 --image=nginx:1.25-alpine --labels="app=web,env=prod"
kubectl run pod3 --image=nginx:1.25-alpine --labels="app=api,env=dev"

# Wait for pods
kubectl wait --for=condition=Ready pods pod1 pod2 pod3 --timeout=30s

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

</details>

### Task 3: Debugging Failing Pods

Debug a Pod that won't start using `describe` and `logs`.

**Requirements:**

- Create a Pod with an intentionally wrong image name
- Use `kubectl describe` to find the error
- Fix the issue

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Create pod with wrong image
kubectl run broken --image=nginx:nonexistent

# Check status
kubectl get pods broken

# See events showing the error
kubectl describe pod broken
# Look at the "Events" section
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create pod with wrong image
kubectl run broken --image=nginx:nonexistent

# Check status
kubectl get pods broken

# Output:
# NAME     READY   STATUS             RESTARTS   AGE
# broken   0/1     ImagePullBackOff   0          10s

# Describe to see what's wrong
kubectl describe pod broken

# Output (excerpt from Events):
# Events:
#   Type     Reason     Age    From               Message
#   ----     ------     ----   ----               -------
#   Normal   Scheduled  30s    default-scheduler  Successfully assigned default/broken to kind-worker
#   Normal   Pulling    30s    kubelet            Pulling image "nginx:nonexistent"
#   Warning  Failed     25s    kubelet            Failed to pull image "nginx:nonexistent":
#            rpc error: ... manifest for nginx:nonexistent not found

# Fix by deleting and recreating with correct image
kubectl delete pod broken
kubectl run broken --image=nginx:1.25-alpine

kubectl wait --for=condition=Ready pod/broken --timeout=30s
kubectl get pods broken

# Output:
# NAME     READY   STATUS    RESTARTS   AGE
# broken   1/1     Running   0          15s

# Cleanup
kubectl delete pod broken
```

</details>

### Task 4: Executing Commands and Port Forwarding

Access Pods interactively and expose services locally.

**Requirements:**

- Expose the `web` deployment as a Service
- Execute commands inside the Pod
- Get an interactive shell
- Use port-forward to test locally

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Expose service
kubectl expose deployment web --port=80

# Execute commands
kubectl exec deployment/web -- nginx -v
kubectl exec -it deployment/web -- sh

# Port forward
kubectl port-forward deployment/web 8080:80 &
curl http://localhost:8080
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create a service for web deployment
kubectl expose deployment web --port=80

# Execute single command
kubectl exec deployment/web -- nginx -v

# Output:
# nginx version: nginx/1.25.x

# Interactive shell
kubectl exec -it deployment/web -- sh
# Inside pod:
ls /etc/nginx/
cat /etc/nginx/nginx.conf | head -10
exit

# Port forward in background
kubectl port-forward deployment/web 8080:80 &
PF_PID=$!

# Wait briefly for port-forward to start
sleep 1

# Test locally
curl http://localhost:8080 | head -5

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>

# Stop port forward
kill $PF_PID
```

</details>

### Task 5: JSONPath and Custom Output

Extract specific information from kubectl output.

**Requirements:**

- Get all pod names as a list
- Extract pod IPs
- Use custom columns to show name, status, IP, and node

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# All pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# With formatting
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Get all pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
echo ""

# Get pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
echo ""

# Formatted output with newlines
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Custom columns showing useful fields
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP,NODE:.spec.nodeName

# Get container images for all pods
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
echo ""

# Check resource requests on a specific pod
kubectl get pod pod1 -o jsonpath='{.spec.containers[*].resources}' 2>/dev/null || echo "pod1 may not exist; use any running pod name"
```

</details>

### Task 6: Dry Run and Manifest Generation

Use dry-run to generate manifests and validate configs.

**Requirements:**

- Generate a Deployment YAML without creating it
- Validate a manifest with server-side dry-run
- Use `kubectl explain` to explore a resource field

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Generate YAML
kubectl create deployment test --image=nginx:1.25-alpine --dry-run=client -o yaml

# Validate
kubectl apply -f <file> --dry-run=server

# Explain resource
kubectl explain deployment.spec.strategy
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Generate deployment YAML without creating it
kubectl create deployment test --image=nginx:1.25-alpine --dry-run=client -o yaml

# Generate and save to file
kubectl create deployment test --image=nginx:1.25-alpine --dry-run=client -o yaml > /tmp/test-deploy.yaml

# Validate with server-side dry-run
kubectl apply -f /tmp/test-deploy.yaml --dry-run=server

# Output:
# deployment.apps/test configured (server dry run)

# Explore resource documentation
kubectl explain deployment
kubectl explain deployment.spec.strategy
kubectl explain deployment.spec.strategy.rollingUpdate

# Generate service YAML
kubectl expose deployment web --port=80 --dry-run=client -o yaml

# Cleanup
rm /tmp/test-deploy.yaml
```

</details>

### Task 7: Explore with k9s

Use k9s to explore and manage the cluster visually.

**Requirements:**

- Launch k9s
- Navigate to pods, services, deployments
- View logs for a pod
- Shell into a pod
- Filter resources

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Launch k9s
k9s

# In k9s:
# :po    - view pods
# :svc   - view services
# :deploy - view deployments
# l      - view logs (when pod selected)
# s      - shell into pod
# /      - filter
# ?      - help
# :q     - quit
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Launch k9s
k9s

# Inside k9s, try these actions:
# 1. Press :po <Enter> to view pods
# 2. Select a pod with arrow keys, press l to view logs
# 3. Press Esc to go back, select pod again, press s to shell in
# 4. Type 'exit' to leave the shell
# 5. Press / then type 'web' to filter pods by name
# 6. Press Esc to clear the filter
# 7. Press :svc <Enter> to view services
# 8. Press :deploy <Enter> to view deployments
# 9. Press 0 to show all namespaces
# 10. Press :q to quit k9s
```

</details>

## Verification

```bash
# Check all resources you've created
kubectl get all

# Verify label filtering works
kubectl get pods --show-labels

# Confirm port-forward is stopped
lsof -i :8080 2>/dev/null | grep kubectl || echo "No port-forward running"
```

## Cleanup

```bash
# Delete deployment and service
kubectl delete deployment web
kubectl delete service web

# Delete pods
kubectl delete pods pod1 pod2 pod3 --ignore-not-found
kubectl delete pod broken --ignore-not-found
```

## Bonus Challenges

**1. Advanced JSONPath:** Extract a table showing pod name, restarts, and age

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create deployment to have some pods
kubectl create deployment web --image=nginx:1.25-alpine --replicas=2

# Wait for pods
kubectl wait --for=condition=Ready pods -l app=web --timeout=30s

# Table with pod name, restarts, and phase
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}Restarts: {.status.containerStatuses[0].restartCount}{"\t"}{.status.phase}{"\n"}{end}'

# Get specific annotations
kubectl get deployment web -o jsonpath='{.metadata.annotations}'

# Cleanup
kubectl delete deployment web
```

</details>

**2. Events Monitoring:** Watch all events in the cluster in real-time

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Watch all events (newest first)
kubectl get events --sort-by='.lastTimestamp'

# Watch events in real-time
kubectl get events -w

# Filter events by type (Warning)
kubectl get events --field-selector type=Warning

# Filter events for a specific object
kubectl get events --field-selector involvedObject.name=web

# Events across all namespaces
kubectl get events -A --sort-by='.lastTimestamp'
```

</details>

## Common issues

| Issue                           | Solution                                                                  |
| ------------------------------- | ------------------------------------------------------------------------- |
| `command not found: k9s`        | Check it's installed: `which k9s` or install via: `brew install k9s`      |
| kubectl context is wrong        | Run `kubectl config get-contexts` and `kubectl config use-context <name>` |
| JSONPath returns nothing        | Check the path is correct with `kubectl get pod <name> -o yaml` first     |
| Port-forward connection refused | Ensure pod is Running and the port number is correct                      |

## Key takeaways

1. **`kubectl describe`** is your first stop for debugging — always check the Events section
2. **Labels** and selectors power resource filtering — label everything
3. **JSONPath** lets you extract exactly the field you need from any resource
4. **`--dry-run=client`** is perfect for generating manifests; **`--dry-run=server`** validates them
5. **k9s** provides fast, visual navigation for exploring clusters

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../10-manifests/README.md)
