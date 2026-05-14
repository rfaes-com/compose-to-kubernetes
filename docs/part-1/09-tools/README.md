# kubectl and k9s essentials

**Duration:** 65 minutes (35 minutes theory + 30 minutes lab)

## Learning Objectives

- Master essential kubectl commands
- Learn kubectl productivity tips and shortcuts
- Understand kubectl context and configuration
- Use k9s for visual cluster management
- Debug and troubleshoot common issues
- Efficiently navigate and inspect cluster resources

## kubectl Basics

kubectl is the primary CLI tool for interacting with Kubernetes clusters.

### Command Structure

```bash
kubectl [command] [TYPE] [NAME] [flags]
```

**Examples:**

```bash
kubectl get pods
kubectl describe pod nginx
kubectl delete deployment web
kubectl logs nginx -f
```

### Essential Commands

| Command        | Purpose             | Example                                         |
| -------------- | ------------------- | ----------------------------------------------- |
| `get`          | List resources      | `kubectl get pods`                              |
| `describe`     | Show detailed info  | `kubectl describe pod nginx`                    |
| `create`       | Create resource     | `kubectl create deployment nginx --image=nginx` |
| `apply`        | Apply configuration | `kubectl apply -f app.yaml`                     |
| `delete`       | Delete resources    | `kubectl delete pod nginx`                      |
| `logs`         | View logs           | `kubectl logs nginx -f`                         |
| `exec`         | Execute command     | `kubectl exec -it nginx -- bash`                |
| `port-forward` | Forward ports       | `kubectl port-forward pod/nginx 8080:80`        |
| `edit`         | Edit resource       | `kubectl edit deployment nginx`                 |
| `scale`        | Scale replicas      | `kubectl scale deployment nginx --replicas=3`   |

## kubectl get - List Resources

```bash
# List pods
kubectl get pods
kubectl get pods -o wide          # More details (node, IP)
kubectl get pods -o yaml          # Full YAML output
kubectl get pods -o json          # JSON output
kubectl get pods -o name          # Just names

# List all resource types
kubectl get all

# List across all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A               # Short form

# Filter by labels
kubectl get pods -l app=nginx
kubectl get pods -l 'app=nginx,env=prod'
kubectl get pods -l 'env!=dev'

# Watch for changes
kubectl get pods -w
kubectl get pods --watch

# Sort by field
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
```

## kubectl describe - Detailed Information

```bash
# Describe pod (shows events!)
kubectl describe pod nginx

# Describe multiple resources
kubectl describe pods -l app=nginx

# Describe node
kubectl describe node kind-worker

# Describe service
kubectl describe service nginx
```

**Key information shown:**

- Configuration details
- Status and conditions
- Events (creations, errors, scheduling)
- Resource usage
- Volume mounts

## kubectl logs - View Logs

```bash
# View pod logs
kubectl logs nginx

# Follow logs (like tail -f)
kubectl logs -f nginx

# Logs from previous container instance
kubectl logs nginx --previous

# Logs from specific container in multi-container pod
kubectl logs nginx -c sidecar

# Last N lines
kubectl logs nginx --tail=50

# Logs since timestamp
kubectl logs nginx --since=1h
kubectl logs nginx --since-time=2024-01-01T00:00:00Z

# All pods matching label
kubectl logs -l app=nginx --all-containers=true

# Combine with grep
kubectl logs nginx | grep ERROR
```

## kubectl exec - Execute Commands

```bash
# Interactive shell
kubectl exec -it nginx -- /bin/sh
kubectl exec -it nginx -- /bin/bash

# Run single command
kubectl exec nginx -- ls /etc
kubectl exec nginx -- cat /etc/nginx/nginx.conf

# Specific container in multi-container pod
kubectl exec -it nginx -c sidecar -- sh

# Run command in all pods
kubectl get pods -l app=nginx -o name | \
  xargs -I {} kubectl exec {} -- date
```

## kubectl apply vs create

```bash
# create: Create new resource (fails if exists)
kubectl create -f app.yaml

# apply: Create or update resource (idempotent)
kubectl apply -f app.yaml

# apply is declarative and tracks changes
kubectl apply -f app.yaml --dry-run=client  # Preview
kubectl apply -f app.yaml --dry-run=server  # Validate with server

# Apply directory of manifests
kubectl apply -f ./manifests/

# Apply with kustomize
kubectl apply -k ./overlays/production/
```

## kubectl port-forward - Access Services Locally

```bash
# Forward pod port to localhost
kubectl port-forward pod/nginx 8080:80

# Forward service port
kubectl port-forward service/nginx 8080:80

# Forward deployment port
kubectl port-forward deployment/nginx 8080:80

# Bind to specific address
kubectl port-forward --address 0.0.0.0 pod/nginx 8080:80

# Multiple ports
kubectl port-forward pod/nginx 8080:80 8443:443
```

Access at: http://localhost:8080

## kubectl Context and Configuration

```bash
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context kind-kind

# View full config
kubectl config view

# Set default namespace for context
kubectl config set-context --current --namespace=development

# Create new context
kubectl config set-context dev --cluster=kind-kind --user=kind-kind --namespace=development
```

## kubectl Shortcuts and Aliases

```bash
# Enable kubectl autocompletion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Alias k=kubectl
alias k=kubectl
complete -F __start_kubectl k

# Common aliases
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
```

## kubectl Productivity Tips

### 1. Resource Type Shortcuts

```bash
# Full name        Short name
pods            ->  po
services        ->  svc
deployments     ->  deploy
replicasets     ->  rs
namespaces      ->  ns
nodes           ->  no
persistentvolumeclaims -> pvc
configmaps      ->  cm
secrets         ->  secret
```

Example:

```bash
kubectl get po
kubectl get svc
kubectl get deploy
```

### 2. Watch Mode

```bash
# Watch resources update in real-time
kubectl get pods -w

# Watch with timestamp
watch kubectl get pods
```

### 3. JSONPath Queries

```bash
# Extract specific fields
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Complex queries
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Get resource limits
kubectl get pod nginx -o jsonpath='{.spec.containers[*].resources}'
```

### 4. Dry Run

```bash
# Generate YAML without creating
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml

# Validate manifest
kubectl apply -f app.yaml --dry-run=server

# Generate manifest and apply
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml | kubectl apply -f -
```

### 5. Explain Resources

```bash
# View resource documentation
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy

# Show all fields
kubectl explain deployment --recursive
```

## k9s - Terminal UI for Kubernetes

k9s provides a terminal-based UI for managing and monitoring Kubernetes clusters.

### Launch k9s

```bash
k9s

# Specific namespace
k9s -n development

# Read-only mode
k9s --readonly

# Different context
k9s --context kind-kind
```

### k9s Navigation

| Key              | Action              |
| ---------------- | ------------------- |
| `:pods` or `:po` | View pods           |
| `:svc`           | View services       |
| `:deploy`        | View deployments    |
| `:ns`            | View namespaces     |
| `0`              | Show all namespaces |
| `1-9`            | Switch namespace    |
| `/`              | Filter resources    |
| `esc`            | Clear filter/back   |
| `?`              | Help                |
| `:q` or `Ctrl+C` | Quit                |

### k9s Pod Operations

When viewing pods:

| Key      | Action           |
| -------- | ---------------- |
| `enter`  | View pod details |
| `l`      | View pod logs    |
| `s`      | Shell into pod   |
| `d`      | Describe pod     |
| `e`      | Edit pod         |
| `y`      | View YAML        |
| `ctrl+k` | Delete pod       |
| `p`      | Previous logs    |
| `f`      | Port forward     |

### k9s Log Viewing

While viewing logs:

| Key   | Action              |
| ----- | ------------------- |
| `f`   | Toggle follow mode  |
| `t`   | Toggle timestamp    |
| `w`   | Toggle wrap         |
| `s`   | Save logs to file   |
| `/`   | Search in logs      |
| `n/N` | Next/previous match |
| `c`   | Clear logs          |

### k9s Pulses (Live View)

k9s shows live metrics:

- CPU and memory usage (if metrics-server installed)
- Pod status changes
- Events in real-time

### k9s Skins and Themes

```bash
# Change skin (in k9s)
:skin <skin-name>

# Available skins: default, dracula, monokai, nord, etc.
```

### k9s Configuration

Config file: `~/.k9s/config.yml`

```yaml
k9s:
  refreshRate: 2
  maxConnRetry: 5
  readOnly: false
  noExitOnCtrlC: false
  ui:
    enableMouse: false
    headless: false
    logoless: false
    crumbsless: false
    skin: dracula
  skipLatestRevCheck: false
  logger:
    sinceSeconds: 300
```

## Troubleshooting Common Issues

### Pod Not Starting

```bash
# Check pod status
kubectl get pods

# Describe for events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous container

# Common issues:
# - ImagePullBackOff: Wrong image name, no credentials
# - CrashLoopBackOff: Application crashes on startup
# - Pending: Insufficient resources, PVC not bound
# - CreateContainerConfigError: ConfigMap/Secret missing
```

### Service Not Accessible

```bash
# Verify service exists
kubectl get svc

# Check endpoints (pod IPs)
kubectl get endpoints <service-name>

# Verify labels match
kubectl get pods --show-labels
kubectl describe service <service-name>  # Check selector

# Test from another pod
kubectl run test --image=busybox --rm -it -- wget -O- http://<service>

# Check service type
kubectl get svc <service-name> -o jsonpath='{.spec.type}'
```

### Debugging Network Issues

```bash
# DNS resolution
kubectl run test --image=busybox --rm -it -- nslookup kubernetes.default

# Connectivity test
kubectl run test --image=nicolaka/netshoot --rm -it -- bash
# Then: ping, curl, dig, traceroute, etc.

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Resource Constraints

```bash
# Check node resources
kubectl top nodes
kubectl describe node <node-name>

# Check pod resources
kubectl top pods
kubectl top pods --containers

# Check resource quotas
kubectl describe resourcequota -n <namespace>

# Check limit ranges
kubectl describe limitrange -n <namespace>
```

### Deployment Rollout Issues

```bash
# Check rollout status
kubectl rollout status deployment/<name>

# View rollout history
kubectl rollout history deployment/<name>

# View specific revision
kubectl rollout history deployment/<name> --revision=2

# Rollback
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2

# Pause/resume rollout
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
```

## kubectl Debug Commands

```bash
# Debug pod (creates ephemeral container)
kubectl debug pod/nginx -it --image=busybox

# Debug node (creates privileged pod on node)
kubectl debug node/kind-worker -it --image=ubuntu

# Copy pod for debugging
kubectl debug pod/nginx --copy-to=nginx-debug --container=nginx -- sh

# Attach to running container
kubectl attach nginx -it
```

## Best Practices

1. **Use namespaces**: Organize resources logically
2. **Label everything**: Makes filtering and selection easier
3. **Use `describe`**: First stop for troubleshooting
4. **Check events**: Often reveals the root cause
5. **Use `--dry-run`**: Preview changes before applying
6. **Use contexts**: Switch between clusters easily
7. **Enable completion**: Save typing and prevent errors
8. **Learn JSONPath**: Extract exactly what you need
9. **Use k9s**: Faster for exploration and monitoring
10. **Keep cheatsheets handy**: Reference common commands

## Quick Reference

See the [kubectl Cheatsheet](../../resources/kubectl-cheatsheet.md) and [k9s Shortcuts](../../resources/k9s-shortcuts.md) for complete references.

## Key takeaways

- kubectl is your primary interface to Kubernetes
- `describe` and `logs` are essential for troubleshooting
- k9s provides a faster, more visual way to manage clusters
- Labels and selectors make resource management easier
- Contexts let you work with multiple clusters
- Dry-run and explain help understand and validate configurations

## Check your understanding

1. What's the difference between `kubectl create` and `kubectl apply`?
2. How do you view logs from a previous container instance?
3. What's the purpose of `kubectl describe` vs `kubectl get`?
4. How do you access a ClusterIP service from your local machine?
5. What are the advantages of using k9s over kubectl?

<details class="solution" markdown="1">
<summary>Solution</summary>

1. **`create` fails if the resource already exists; `apply` creates or updates and is idempotent**
2. **`kubectl logs <pod> --previous`**
3. **`get` gives a brief overview; `describe` gives full details, conditions, and crucially the Events section for debugging**
4. **Use `kubectl port-forward service/<name> <local-port>:<service-port>`**
5. **k9s is faster for browsing resources, real-time log tailing, and navigating multiple resource types without typing commands**

</details>

## Hands-on

Apply the concepts from this section in the [lab](lab.md) exercises.

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../10-manifests/README.md)
