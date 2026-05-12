# Environment Setup & First Steps

**Duration:** 25 minutes (including 10-minute lab)
**Format:** Hands-on walkthrough

## Learning Objectives

- Create your first Kubernetes cluster using kind
- Verify kubectl connectivity
- Explore the cluster with k9s
- Understand kubeconfig and contexts
- Run your first Pod

## Creating Your First Cluster

### Step 1: Verify Workshop Environment

Make sure you're inside the workshop container:

```bash
# You should see this prompt:
vscode ➜ /workspaces/compose-to-kubernetes (main) $

# Verify tools are available
kubectl version --client
kind version
k9s version
```

### Step 2: Create a Cluster

```bash
# Create a simple cluster (1 control-plane, 1 worker)
kind create cluster --config /workspaces/compose-to-kubernetes/setup/kind/simple.yaml

# This takes 1-2 minutes
# You'll see output like:
# Creating cluster "workshop" ...
# ✓ Ensuring node image (kindest/node:vX.XX.X)
# ✓ Preparing nodes
# ✓ Writing configuration
# ✓ Starting control-plane
# ✓ Installing CNI
# ✓ Installing StorageClass
# ✓ Joining worker nodes
# ✓ Ready after Xs
```

### Step 3: Verify the Cluster

```bash
# Check cluster info
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://127.0.0.1:XXXXX
# CoreDNS is running at https://...

# List nodes
kubectl get nodes

# Expected output:
# NAME                     STATUS   ROLES           AGE   VERSION
# workshop-control-plane   Ready    control-plane   1m    v1.XX.X
# workshop-worker          Ready    <none>          1m    v1.XX.X
```

**What you see:**

- `workshop-control-plane`: The node running control plane components
- `workshop-worker`: The node where your Pods will run
- `STATUS: Ready`: Both nodes are healthy
- `ROLES`: Identifies the node's role in the cluster

## Understanding kubeconfig

### What is kubeconfig?

`kubeconfig` is a configuration file that tells kubectl:

- **Where** the cluster is (API server address)
- **Who** you are (authentication)
- **What** you can do (authorization/context)

### View Your kubeconfig

```bash
# Default location: ~/.kube/config
kubectl config view

# You'll see (simplified):
# clusters:
#   - cluster:
#       server: https://127.0.0.1:XXXXX
#     name: kind-workshop
# contexts:
#   - context:
#       cluster: kind-workshop
#       user: kind-workshop
#     name: kind-workshop
# current-context: kind-workshop
# users:
#   - name: kind-workshop
```

### Contexts

A **context** combines:

- A **cluster** (where to connect)
- A **user** (how to authenticate)
- A **namespace** (optional default)

```bash
# List contexts
kubectl config get-contexts

# Current context (marked with *)
# * kind-workshop   kind-workshop   kind-workshop   default

# Switch contexts (if you have multiple clusters)
kubectl config use-context kind-workshop

# Set default namespace for context
kubectl config set-context --current --namespace=kube-system
```

## Introduction to kubectl

### Basic kubectl Structure

```bash
kubectl [command] [resource type] [resource name] [flags]
```

### Essential Commands

```bash
# Get resources
kubectl get nodes
kubectl get pods
kubectl get services
kubectl get all

# Describe (detailed info)
kubectl describe node workshop-worker
kubectl describe pod <pod-name>

# Create resources
kubectl create deployment nginx --image=nginx
kubectl apply -f myfile.yaml

# Delete resources
kubectl delete pod <pod-name>
kubectl delete deployment nginx

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> -f  # Follow logs

# Execute commands in Pod
kubectl exec <pod-name> -- ls /
kubectl exec -it <pod-name> -- /bin/bash  # Interactive shell

# Port forwarding (access Pod locally)
kubectl port-forward pod/<pod-name> 8080:80
```

## Exploring with k9s

### What is k9s?

k9s is a terminal UI for Kubernetes that provides:

- Visual cluster navigation
- Real-time updates
- Quick access to logs, events, and descriptions
- No need to remember kubectl commands

### Launch k9s

```bash
# Start k9s
k9s

# You'll see a colorful interface!
```

### k9s Navigation

| Key              | Action                     |
| ---------------- | -------------------------- |
| `:pods`          | View Pods                  |
| `:deployments`   | View Deployments           |
| `:services`      | View Services              |
| `:nodes`         | View Nodes                 |
| `/`              | Filter/search              |
| `d`              | Describe selected resource |
| `l`              | View logs                  |
| `s`              | Shell into Pod             |
| `Ctrl+d`         | Delete                     |
| `?`              | Help                       |
| `:q` or `Ctrl+c` | Quit                       |

### Try It Out

1. Launch k9s: `k9s`
2. Type `:nodes` and press Enter
3. Use arrow keys to select a node
4. Press `d` to describe it
5. Press `Esc` to go back
6. Type `:q` to quit

## Your First Pod

Let's create and interact with a Pod:

```bash
# Create a Pod running nginx
kubectl run my-nginx --image=nginx:latest

# Wait for it to be ready
kubectl get pods -w
# Press Ctrl+C to stop watching

# Check its status
kubectl get pod my-nginx

# Get detailed information
kubectl describe pod my-nginx

# View logs
kubectl logs my-nginx

# Execute a command in the Pod
kubectl exec my-nginx -- nginx -v

# Get an interactive shell
kubectl exec -it my-nginx -- /bin/bash
# Try: ls, cat /etc/nginx/nginx.conf, exit

# Expose it locally
kubectl port-forward pod/my-nginx 8080:80 &

# Test it (in another terminal or stop port-forward first)
curl http://localhost:8080

# Clean up
kill %1  # Stop port-forward
kubectl delete pod my-nginx
```

## Exploring the Cluster

### System Pods

Kubernetes runs its own Pods for cluster functionality:

```bash
# View system Pods
kubectl get pods -n kube-system

# You'll see:
# - coredns-*: DNS server for the cluster
# - etcd-*: Key-value store
# - kube-apiserver-*: API server
# - kube-controller-manager-*: Controller manager# - kube-proxy-*: Network proxy
# - kube-scheduler-*: Scheduler
```

### Namespaces

Namespaces provide logical separation:

```bash
# List namespaces
kubectl get namespaces

# Default namespaces:
# - default: Default namespace for resources
# - kube-system: System components
# - kube-public: Public resources
# - kube-node-lease: Node heartbeat information

# Get resources from specific namespace
kubectl get pods -n kube-system

# Get resources from ALL namespaces
kubectl get pods --all-namespaces
# or
kubectl get pods -A
```

## Cluster Information

### Comprehensive Cluster View

```bash
# Cluster info
kubectl cluster-info
kubectl cluster-info dump  # Very detailed (lots of output!)

# Node resources
kubectl top nodes  # Requires metrics-server (not in kind by default)

# API resources available
kubectl api-resources

# API versions
kubectl api-versions

# Check component status
kubectl get componentstatuses
# or
kubectl get cs
```

## Cleanup (Optional)

If you want to start fresh:

```bash
# Delete the cluster
kind delete cluster --name workshop

# Recreate it
kind create cluster --config /workspaces/compose-to-kubernetes/setup/kind/simple.yaml
```

## Lab: Environment Verification

**Time:** 10 minutes

### Tasks

1. **Verify your cluster**
   - Create the cluster (if not done)
   - Ensure both nodes are `Ready`
   - View the kubeconfig

2. **Create and inspect a test Pod**
   - Create a Pod running `busybox:latest` that sleeps
   - Check its status
   - Execute a command inside it
   - View its logs
   - Delete it

3. **Explore with k9s**
   - Launch k9s
   - Navigate to Pods
   - Navigate to Nodes
   - View system Pods in `kube-system` namespace

4. **Test kubectl contexts**
   - List all contexts
   - View the current context

### Lab Instructions

See [lab/instructions.md](lab/instructions.md) for detailed steps.

### Solution

After attempting the lab, check [lab/solutions/](lab/solutions/lab-solution.md) for the complete solution.

## Tips

**Alias kubectl to k:**

```bash
alias k=kubectl
k get nodes  # Much faster!
```

**Watch resources in real-time:**

```bash
kubectl get pods -w
# or
watch kubectl get pods
```

**Use bash completion:**

```bash
# Already configured in the workshop container
kubectl get <Tab><Tab>  # Auto-complete resource types
```

**Quick describe:**

```bash
# Shorthand for describe
kubectl describe node/<node-name>
kubectl describe pod/<pod-name>
```

## Troubleshooting

### "Connection refused" error

```bash
# Check cluster is running
kind get clusters

# If not listed, recreate
kind create cluster --config /workspaces/compose-to-kubernetes/setup/kind/simple.yaml
```

### "No resources found"

```bash
# Make sure you're in the right namespace
kubectl get pods -A  # Check ALL namespaces

# Check current context
kubectl config current-context
```

### k9s shows errors

```bash
# Make sure cluster is healthy
kubectl get nodes

# Check k9s config
k9s info
```

## Key Takeaways

- **kind** creates local Kubernetes clusters using containers as nodes
- **kubectl** is the command-line tool to interact with the cluster
- **kubeconfig** tells kubectl where and how to connect
- **contexts** allow switching between clusters
- **k9s** provides a visual interface for cluster management
- **Namespaces** provide logical separation of resources

## Next Section

Now that your environment is ready, let's dive into Kubernetes resources!

 **Next:** [03-pods - Core Kubernetes Resources: Pods](../03-pods/README.md)

## Check Your Understanding

1. What command lists all nodes in your cluster?
2. How do you view logs from a Pod?
3. What's the difference between `kubectl get` and `kubectl describe`?
4. What does a context contain?
5. How do you view Pods in all namespaces?

<details markdown="1">
<summary>Click for answers</summary>

1. **`kubectl get nodes`**
2. **`kubectl logs <pod-name>`**
3. **`get` shows a list/table view; `describe` shows detailed information about a specific resource**
4. **A cluster, a user, and optionally a default namespace**
5. **`kubectl get pods --all-namespaces` or `kubectl get pods -A`**

</details>
