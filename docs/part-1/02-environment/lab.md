# Lab

**Duration:** 10 minutes

## Objectives

- Verify cluster is running correctly
- Practice basic kubectl commands
- Create and interact with a Pod
- Explore using k9s

## Prerequisites

- Workshop container running
- Inside the `/workspaces/compose-to-kubernetes` directory

## Tasks

### Task 1: Cluster Verification

1. List all nodes in the cluster
2. Verify both nodes show `STATUS: Ready`
3. Display cluster information
4. View your kubeconfig (redacted for security)

**Expected Output:**

- 2 nodes: `workshop-control-plane` and `workshop-worker`; both with `Ready` status
- Cluster API server address displayed

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# List all nodes
kubectl get nodes

# Output:
# NAME                     STATUS   ROLES           AGE   VERSION
# workshop-control-plane   Ready    control-plane   5m    v1.28.0
# workshop-worker          Ready    <none>          5m    v1.28.0

# Display cluster information
kubectl cluster-info

# Output:
# Kubernetes control plane is running at https://127.0.0.1:45697
# CoreDNS is running at https://127.0.0.1:45697/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# View kubeconfig
kubectl config view

# Output shows cluster, user, and context information
```

</details>

### Task 2: Create and Inspect a Pod

1. Create a Pod named `test-busybox` using the `busybox:latest` image that runs the command `sleep 3600`

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    Use `kubectl run` with `--` to specify the command:

    ```bash
    kubectl run <pod-name> --image=<image> -- <command> <args>
    ```

    For sleep, the command is `sleep` and the arg is `3600`.

    </details>

2. Wait for the Pod to be in `Running` state
3. Get detailed information about the Pod (describe)
4. Execute the command `echo "Hello from inside the Pod!"` in the Pod

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    Use `kubectl exec`:

    ```bash
    kubectl exec <pod-name> -- <command>
    ```

    For echo, the full command is `echo "Hello from inside the Pod!"` but you may need to handle quotes.

    </details>

5. Check the Pod's logs (even though it's just sleeping, there shouldn't be much)
6. Get an interactive shell into the Pod and run:
   - `hostname` (should show Pod name)
   - `ls /`
   - `exit`

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    Use `kubectl exec` with `-it` flags:

    ```bash
    kubectl exec -it <pod-name> -- /bin/sh
    ```

    Note: busybox uses `/bin/sh`, not `/bin/bash`.

    </details>

7. Delete the Pod

**Expected Outcomes:**

- Pod successfully created and running
- Commands executed successfully
- Pod cleanly deleted

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. Create a Pod running busybox with sleep command
kubectl run test-busybox --image=busybox:latest -- sleep 3600

# Output:
# pod/test-busybox created

# 2. Wait for Pod to be Running
kubectl get pods -w
# Press Ctrl+C when you see:
# NAME           READY   STATUS    RESTARTS   AGE
# test-busybox   1/1     Running   0          5s

# Alternative: Wait until Running
kubectl wait --for=condition=Ready pod/test-busybox --timeout=60s

# 3. Get detailed information about the Pod
kubectl describe pod test-busybox

# Output shows:
# - Pod IP address
# - Node it's running on
# - Container details
# - Events (image pull, container started, etc.)

# 4. Execute a command in the Pod
kubectl exec test-busybox -- echo "Hello from inside the Pod!"

# Output:
# Hello from inside the Pod!

# 5. Check the Pod's logs
kubectl logs test-busybox

# Output:
# (empty or minimal, since sleep doesn't produce output)

# 6. Get an interactive shell
kubectl exec -it test-busybox -- /bin/sh

# Inside the Pod:
/ # hostname
test-busybox

/ # ls /
bin   dev   etc   home  proc  root  sys   tmp   usr   var

/ # exit

# 7. Delete the Pod
kubectl delete pod test-busybox

# Output:
# pod "test-busybox" deleted

# Verify deletion
kubectl get pod test-busybox
# Output:
# Error from server (NotFound): pods "test-busybox" not found
```

</details>

### Task 3: Explore with k9s

1. Launch k9s
2. Navigate to view Pods (`:pods`)
3. Navigate to view Nodes (`:nodes`)
4. Select the worker node and describe it (press `d`)
5. Navigate to `kube-system` namespace and view system Pods:
   - Type `:pods` then press Enter
   - Type `/kube-system` to filter
   - Browse the system Pods
6. Exit k9s

<details class="hint" markdown="1">
<summary>Hint for k9s namespace filtering</summary>

In k9s, you can filter by namespace by typing `/` followed by search text, or use `0-9` keys to select namespace, or type `:` followed by resource type and namespace like `:pods default`.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. Launch k9s
k9s

# 2. Navigate to Pods
# Type: :pods
# Press: Enter

# 3. Navigate to Nodes
# Type: :nodes
# Press: Enter

# 4. Describe a node
# Use arrow keys to select workshop-worker
# Press: d (for describe)
# Press: Esc (to go back)

# 5. View system Pods in kube-system
# Type: :pods
# Press: Enter
# Use namespace selector (press 0-9 or type namespace)
# Or type: :pods kube-system

# You should see Pods like:
# - coredns-XXXXX
# - etcd-workshop-control-plane
# - kube-apiserver-workshop-control-plane
# - kube-controller-manager-workshop-control-plane
# - kube-proxy-XXXXX
# - kube-scheduler-workshop-control-plane

# 6. Exit k9s
# Type: :q
# Or press: Ctrl+C
```

</details>

### Task 4: Context and Namespaces

1. List all available contexts

2. Display the current context (should show `kind-workshop`)

3. List all namespaces in the cluster

**Expected Output:**

- Context: `kind-workshop`
- Namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. List all contexts
kubectl config get-contexts

# Output:
# CURRENT   NAME           CLUSTER        AUTHINFO       NAMESPACE
# *         kind-workshop  kind-workshop  kind-workshop

# 2. Display current context
kubectl config current-context

# Output:
# kind-workshop

# 3. List all namespaces
kubectl get namespaces
# or
kubectl get ns

# Output:
# NAME              STATUS   AGE
# default           Active   10m
# kube-node-lease   Active   10m
# kube-public       Active   10m
# kube-system       Active   10m
```

</details>

## Bonus Challenges

**1. Create multiple Pods:** Create 3 Pods with different names but same image. Delete them all at once.

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create 3 Pods
kubectl run pod1 --image=busybox:latest -- sleep 3600
kubectl run pod2 --image=busybox:latest -- sleep 3600
kubectl run pod3 --image=busybox:latest -- sleep 3600

# Verify they're running
kubectl get pods

# Delete all at once (by label would be better, but these don't have common labels)
kubectl delete pod pod1 pod2 pod3

# Or delete by pattern (if you have many)
kubectl get pods -o name | grep pod | xargs kubectl delete
```

</details>

**2. Port forwarding:** Create an nginx Pod, forward port 8080 locally to port 80 in the Pod, and curl it.

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create nginx Pod
kubectl run nginx --image=nginx:latest

# Wait for it to be ready
kubectl wait --for=condition=Ready pod/nginx --timeout=60s

# Forward local port 8080 to Pod port 80
kubectl port-forward pod/nginx 8080:80 &

# Test it (in same or different terminal)
curl http://localhost:8080

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...

# Stop port forwarding
kill %1  # If running in background

# Clean up
kubectl delete pod nginx
```

</details>

**3. Resource inspection:** Find out how much CPU and memory the `workshop-worker` node has allocated.

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Describe the worker node and look for capacity
kubectl describe node workshop-worker | grep -A 5 "Capacity:"

# Output:
# Capacity:
#   cpu:                X
#   memory:             XXXXKi
#   pods:               110
# Allocatable:
#   cpu:                X
#   memory:             XXXXKi

# Or get all node resources in table format
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory
```

</details>

**4. System exploration:** Use `kubectl get all -n kube-system` to see all resources in the system namespace. What do you notice?

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Get all resources in kube-system
kubectl get all -n kube-system

# Output shows:
# - Pods (system components)
# - Services (kube-dns, etc.)
# - DaemonSets (kube-proxy)
# - Deployments (coredns)
# - ReplicaSets (coredns-XXXXX)

# What you notice:
# 1. System Pods run on control-plane node (most of them)
# 2. DaemonSets ensure Pods run on every node (like kube-proxy)
# 3. CoreDNS uses Deployments (like regular apps)
# 4. Everything is a Pod at some level!
```

</details>

## Common Mistakes

### Using /bin/bash instead of /bin/sh

```bash
# WRONG (busybox doesn't have bash)
kubectl exec -it test-busybox -- /bin/bash

# CORRECT
kubectl exec -it test-busybox -- /bin/sh
```

### Forgetting the -- separator

```bash
# WRONG
kubectl run test --image=busybox sleep 3600

# CORRECT
kubectl run test --image=busybox -- sleep 3600
```

### Not waiting for Pod to be ready

```bash
# Might fail if Pod isn't ready yet
kubectl exec test-busybox -- echo hello

# Wait first
kubectl wait --for=condition=Ready pod/test-busybox
kubectl exec test-busybox -- echo hello
```

## Troubleshooting

| Issue                     | Solution                                                  |
| ------------------------- | --------------------------------------------------------- |
| Pod stays in `Pending`    | Check: `kubectl describe pod <name>` for events           |
| "command not found" error | Verify you're using `/bin/sh` not `/bin/bash` for busybox |
| Can't delete Pod          | Use `kubectl delete pod <name> --force --grace-period=0`  |
| k9s won't start           | Verify cluster is running: `kubectl get nodes`            |

## Key takeaways

1. **kubectl run** is the quickest way to create a single Pod
2. **kubectl describe** shows detailed information and events
3. **kubectl exec** lets you run commands or get a shell inside Pods
4. **k9s** provides a visual way to explore the cluster
5. **Contexts** let you switch between different clusters
6. **Namespaces** provide logical separation of resources

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../03-pods/README.md).
