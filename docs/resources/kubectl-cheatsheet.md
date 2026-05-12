# kubectl Cheatsheet

Quick reference for essential kubectl commands.

## Cluster Information

```bash
# Cluster info
kubectl cluster-info
kubectl version

# Node information
kubectl get nodes
kubectl describe node <node-name>
kubectl top nodes
```

## Working with Resources

### Get Resources

```bash
# List resources
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all

# All namespaces
kubectl get pods -A
kubectl get pods --all-namespaces

# Output formats
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o name

# Watch changes
kubectl get pods -w

# Sort
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime

# Filter by labels
kubectl get pods -l app=nginx
kubectl get pods -l 'app=nginx,env=prod'
kubectl get pods -l 'env!=dev'

# Show labels
kubectl get pods --show-labels

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase
```

### Describe Resources

```bash
kubectl describe pod <pod-name>
kubectl describe service <service-name>
kubectl describe node <node-name>

# Describe all pods matching label
kubectl describe pods -l app=nginx
```

### Create Resources

```bash
# Imperative
kubectl create deployment nginx --image=nginx:1.25-alpine
kubectl create service clusterip nginx --tcp=80:80
kubectl create configmap app-config --from-literal=key=value
kubectl create secret generic db-pass --from-literal=password=secret

# Declarative
kubectl apply -f manifest.yaml
kubectl apply -f ./directory/
kubectl apply -k ./kustomize-dir/

# Create and output manifest
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml
```

### Update Resources

```bash
# Apply changes
kubectl apply -f manifest.yaml

# Edit resource
kubectl edit deployment nginx

# Scale
kubectl scale deployment nginx --replicas=3

# Set image
kubectl set image deployment/nginx nginx=nginx:1.26-alpine

# Set resources
kubectl set resources deployment nginx --limits=cpu=200m,memory=512Mi --requests=cpu=100m,memory=256Mi

# Patch
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'
```

### Delete Resources

```bash
# Delete specific resource
kubectl delete pod nginx
kubectl delete deployment nginx
kubectl delete service nginx

# Delete by label
kubectl delete pods -l app=nginx

# Delete from file
kubectl delete -f manifest.yaml

# Delete all in namespace
kubectl delete all --all -n my-namespace

# Force delete
kubectl delete pod nginx --force --grace-period=0
```

## Pods

```bash
# Create pod
kubectl run nginx --image=nginx:1.25-alpine

# Execute command
kubectl exec nginx -- ls /etc
kubectl exec -it nginx -- /bin/sh

# View logs
kubectl logs nginx
kubectl logs -f nginx  # Follow
kubectl logs nginx --previous  # Previous container
kubectl logs -l app=nginx --all-containers=true

# Port forward
kubectl port-forward pod/nginx 8080:80

# Copy files
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf
kubectl cp ./file.txt nginx:/tmp/file.txt

# Attach to running container
kubectl attach nginx -it
```

## Deployments

```bash
# Create deployment
kubectl create deployment web --image=nginx:1.25-alpine --replicas=3

# Scale
kubectl scale deployment web --replicas=5

# Autoscale
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=80

# Update image
kubectl set image deployment/web nginx=nginx:1.26-alpine

# Rollout commands
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl rollout undo deployment/web --to-revision=2
kubectl rollout pause deployment/web
kubectl rollout resume deployment/web
kubectl rollout restart deployment/web
```

## Services

```bash
# Expose deployment
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Create service
kubectl create service clusterip nginx --tcp=80:80
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080

# Get endpoints
kubectl get endpoints nginx
```

## ConfigMaps and Secrets

```bash
# ConfigMap from literal
kubectl create configmap app-config --from-literal=key1=value1 --from-literal=key2=value2

# ConfigMap from file
kubectl create configmap app-config --from-file=config.yaml

# ConfigMap from directory
kubectl create configmap app-config --from-file=./config-dir/

# Secret from literal
kubectl create secret generic db-secret --from-literal=password=secret123

# Secret from file
kubectl create secret generic db-secret --from-file=./password.txt

# TLS secret
kubectl create secret tls my-tls --cert=cert.pem --key=key.pem

# Docker registry secret
kubectl create secret docker-registry regcred --docker-server=registry.example.com --docker-username=user --docker-password=pass
```

## Namespaces

```bash
# Create namespace
kubectl create namespace development

# Set default namespace
kubectl config set-context --current --namespace=development

# View resources in namespace
kubectl get pods -n development

# All namespaces
kubectl get pods -A
```

## Contexts

```bash
# View contexts
kubectl config get-contexts
kubectl config current-context

# Switch context
kubectl config use-context kind-kind

# Set namespace for context
kubectl config set-context --current --namespace=production

# Create context
kubectl config set-context prod --cluster=kind-kind --user=kind-kind --namespace=production
```

## Labels and Annotations

```bash
# Add label
kubectl label pod nginx env=prod

# Update label
kubectl label pod nginx env=staging --overwrite

# Remove label
kubectl label pod nginx env-

# Add annotation
kubectl annotate pod nginx description="Web server"

# Remove annotation
kubectl annotate pod nginx description-
```

## JSONPath Queries

```bash
# Get pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Complex query
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Get specific field
kubectl get pod nginx -o jsonpath='{.spec.containers[0].image}'

# All images in use
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u
```

## Resource Quotas and Limits

```bash
# View quotas
kubectl get resourcequota
kubectl describe resourcequota <quota-name>

# View limit ranges
kubectl get limitrange
kubectl describe limitrange <limitrange-name>
```

## Debugging

```bash
# Get events
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -w

# Describe for details and events
kubectl describe pod <pod-name>

# Check pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>  # Specific container
kubectl logs <pod-name> --previous  # Previous instance

# Execute commands
kubectl exec -it <pod-name> -- sh

# Debug pod
kubectl debug pod/<pod-name> -it --image=busybox

# Debug node
kubectl debug node/<node-name> -it --image=ubuntu

# Resource usage
kubectl top nodes
kubectl top pods
kubectl top pods --containers
```

## Validation and Testing

```bash
# Dry run
kubectl apply -f manifest.yaml --dry-run=client
kubectl apply -f manifest.yaml --dry-run=server

# Diff
kubectl diff -f manifest.yaml

# Explain resource
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment --recursive
```

## Useful Aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
alias kex='kubectl exec -it'
alias kcx='kubectl config get-contexts'
alias kns='kubectl config set-context --current --namespace'

# Enable completion
source <(kubectl completion bash)
complete -F __start_kubectl k
```

## Productivity Tips

```bash
# Use short names
po    = pods
svc   = services
deploy = deployments
rs    = replicasets
cm    = configmaps
ns    = namespaces
pv    = persistentvolumes
pvc   = persistentvolumeclaims

# Generate manifest template
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml

# Apply and watch
kubectl apply -f manifest.yaml && kubectl get pods -w

# Delete and wait
kubectl delete pod nginx --wait=true

# Get pod on specific node
kubectl get pods --field-selector spec.nodeName=kind-worker

# Get all resources in namespace
kubectl get all -n my-namespace

# Get resource usage
kubectl top pods --sort-by=memory
kubectl top pods --sort-by=cpu
```

## Completion

```bash
# Bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Zsh
source <(kubectl completion zsh)
echo "source <(kubectl completion zsh)" >> ~/.zshrc

# Fish
kubectl completion fish | source

# PowerShell
kubectl completion powershell | Out-String | Invoke-Expression
```

## Help

```bash
# General help
kubectl help
kubectl <command> --help

# Explain resources
kubectl explain pod
kubectl explain deployment.spec
```
