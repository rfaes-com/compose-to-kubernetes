# Lab Solution: Environment Verification

## Task 1: Cluster Verification

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

## Task 2: Create and Inspect a Pod

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

## Task 3: Explore with k9s

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

## Task 4: Context and Namespaces

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

## Validation Commands

```bash
# Verify cluster is healthy
kubectl get nodes
# Both nodes should be Ready

# Verify test Pod is gone
kubectl get pod test-busybox
# Should return "NotFound" error

# Verify context
kubectl config current-context
# Should output: kind-workshop

# Verify namespaces
kubectl get ns
# Should show 4 default namespaces
```

## Bonus Challenge Solutions

### Bonus 1: Create and Delete Multiple Pods

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

### Bonus 2: Port Forwarding

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

### Bonus 3: Node Resource Inspection

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

### Bonus 4: System Exploration

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

## Key Learnings

1. **kubectl run** is the quickest way to create a single Pod
2. **kubectl describe** shows detailed information and events
3. **kubectl exec** lets you run commands or get a shell inside Pods
4. **k9s** provides a visual way to explore the cluster
5. **Contexts** let you switch between different clusters
6. **Namespaces** provide logical separation of resources

**Great job!** You now have a working Kubernetes cluster and know how to interact with it. Let's move on to understanding Pods in depth.

**Next:** [03-pods - Core Kubernetes Resources: Pods](../../../03-pods/README.md)
