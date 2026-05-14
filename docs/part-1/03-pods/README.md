# Pods: The Atomic Unit of Kubernetes

**Duration:** 35 minutes (20 minutes theory + 15 minutes lab)

## Learning Objectives

- Understand what a Pod is and why it exists
- Learn Pod lifecycle and phases
- Create Pods using kubectl and YAML manifests
- Work with multi-container Pods
- Map Docker Compose services to Kubernetes Pods

## What is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes. It represents a single instance of a running process in your cluster.

### Key Characteristics

- **One or more containers** that share resources
- **Shared network namespace** - containers in a Pod share an IP address
- **Shared storage** - can mount same volumes
- **Atomic unit** - scaled, scheduled, and managed together
- **Ephemeral** - Pods are mortal, they can die and be replaced

### Pod vs Container

```mermaid
flowchart TB
    subgraph Pod["Pod<br/>IP: 10.244.0.5"]
        direction TB

        subgraph Containers[" "]
            direction LR

            C1["Container 1<br/>(nginx)<br/>Port: 80"]
            C2["Container 2<br/>(sidecar)<br/>Port: 9090"]
        end

        Shared["Shared: Network, IPC, Volumes"]
    end
```

## Pod Lifecycle

### Pod Phases

| Phase         | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| **Pending**   | Accepted but not yet running (downloading image, scheduling) |
| **Running**   | All containers are created, at least one is running          |
| **Succeeded** | All containers terminated successfully (won't restart)       |
| **Failed**    | All containers terminated, at least one failed               |
| **Unknown**   | State cannot be determined (communication issue)             |

### Pod Lifecycle Flow

Create → Pending → Running → Succeeded/Failed → Terminated

### Container States

Within a Pod, each container has its own state:

- **Waiting**: Container is waiting to start (pulling image, waiting for init containers)
- **Running**: Container is executing
- **Terminated**: Container has finished executing

## Creating Pods

### Method 1: Imperative (kubectl run)

```bash
# Quickest way - creates Pod directly
kubectl run my-nginx --image=nginx:latest

# With port specification
kubectl run my-nginx --image=nginx:latest --port=80

# With custom command
kubectl run busybox --image=busybox -- sleep 3600

# With environment variables
kubectl run my-app --image=myapp:v1 --env="ENV=production"
```

**Pros:** Fast, great for testing
**Cons:** Not reproducible, hard to manage

### Method 2: Declarative (YAML manifests)

See [examples/simple-pod.yaml](examples/simple-pod.yaml):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: web
    environment: development
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
          name: http
          protocol: TCP
```

Apply it:

```bash
kubectl apply -f examples/simple-pod.yaml

# View it
kubectl get pod nginx-pod

# Details
kubectl describe pod nginx-pod
```

**Pros:** Reproducible, version-controlled, manageable
**Cons:** More verbose

## Multi-Container Pods

Pods can contain multiple containers that work together.

### Common Patterns

1. **Sidecar**: Helper container (logging, monitoring, proxies)
2. **Ambassador**: Proxy to external services
3. **Adapter**: Normalize output (log formatting, metrics conversion)

### Example: Web Server + Log Collector

See [examples/multi-container-pod.yaml](examples/multi-container-pod.yaml):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  containers:
    # Main application container
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
      volumeMounts:
        - name: logs
          mountPath: /var/log/nginx

    # Sidecar: Log processor
    - name: log-collector
      image: busybox:latest
      command: ['sh', '-c', 'tail -f /logs/access.log']
      volumeMounts:
        - name: logs
          mountPath: /logs

  # Shared volume
  volumes:
    - name: logs
      emptyDir: {}
```

**Key Points:**

- Containers share network (both can use `localhost`)
- Containers share volumes (both can read/write logs)
- Containers share Pod lifecycle (die together)

## Container Ports

### Declaring Ports

```yaml
spec:
  containers:
    - name: app
      ports:
        - containerPort: 8080    # Port the container listens on
          name: http             # Optional: Name for reference
          protocol: TCP          # TCP (default) or UDP
        - containerPort: 9090
          name: metrics
```

**Note:** Declaring ports is **documentation** - containers will listen on ports whether you declare them or not. But declaring helps with:

- Service auto-discovery
- Documentation
- Network policies

## Labels and Annotations

### Labels

Key-value pairs for identification and selection:

```yaml
metadata:
  labels:
    app: web
    tier: frontend
    environment: production
    version: "1.0"
```

**Used for:**

- Selecting Pods (Services, Deployments use labels)
- Organizing resources
- Filtering in kubectl

```bash
# Get Pods with specific label
kubectl get pods -l app=web

# Get Pods with multiple labels
kubectl get pods -l app=web,environment=production

# Label an existing Pod
kubectl label pod my-pod environment=staging
```

### Annotations

Key-value pairs for non-identifying metadata:

```yaml
metadata:
  annotations:
    description: "Web frontend for user dashboard"
    maintainer: "team@example.com"
    build-date: "2024-03-11"
    git-commit: "abc123"
```

**Used for:**

- Documentation
- Tool configuration
- Build information
- Not used for selection

## Inspecting Pods

### Get Pod Information

```bash
# List Pods
kubectl get pods

# Wide output (more columns)
kubectl get pods -o wide

# YAML output (full spec)
kubectl get pod my-pod -o yaml

# JSON output
kubectl get pod my-pod -o json

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP
```

### Describe Pod

```bash
# Detailed information including events
kubectl describe pod my-pod

# Look for:
# - Status and conditions
# - Container statuses
# - Events (image pulls, failures, restarts)
# - Resource usage
# - Volumes
```

### Pod Logs

```bash
# View logs
kubectl logs my-pod

# Follow logs (like tail -f)
kubectl logs my-pod -f

# Previous container logs (if crashed)
kubectl logs my-pod --previous

# Specific container in multi-container Pod
kubectl logs multi-pod -c nginx
kubectl logs multi-pod -c log-collector

# Last 20 lines
kubectl logs my-pod --tail=20

# Since timestamp
kubectl logs my-pod --since=1h
```

### Execute Commands

```bash
# Execute command
kubectl exec my-pod -- ls /app

# Interactive shell
kubectl exec -it my-pod -- /bin/bash

# Specific container
kubectl exec -it multi-pod -c nginx -- /bin/bash
```

## Docker Compose → Kubernetes Pods

### Simple Service

**Docker Compose:**

```yaml
version: '3'
services:
  web:
    image: nginx:latest
    container_name: my-web
    ports:
      - "8080:80"
    environment:
      - ENV=production
```

**Kubernetes Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-web
spec:
  containers:
    - name: web
      image: nginx:latest
      ports:
        - containerPort: 80
      env:
        - name: ENV
          value: "production"
```

**Key Differences:**

- No direct port mapping (handled by Services)
- Environment variables use structured format
- More explicit about API version and resource kind

### gandalf Service (From Workshop Example)

**Docker Compose:**

```yaml
gandalf:
  mem_limit: 512M
  image: nginx:latest  # Using public image
  container_name: gandalf
  volumes:
    - ./config:/app/config:ro
    - ./data:/app/data
  environment:
    - MAGIC_WAND=staff
```

**Kubernetes Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gandalf
  labels:
    app: gandalf
spec:
  containers:
    - name: gandalf
      image: nginx:latest
      resources:
        limits:
          memory: "512Mi"
      env:
        - name: MAGIC_WAND
          value: "staff"
      volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: data
          mountPath: /app/data
  volumes:
    - name: config
      hostPath:
        path: /path/on/host/config
    - name: data
      hostPath:
        path: /path/on/host/data
```

**Note:** Volumes are handled differently (we'll cover this in detail in the Storage section).

## Pod Limitations

### Why Not Use Pods Directly?

Pods are ephemeral and have limitations:

- **No self-healing**: If a Pod dies, it's gone
- **No scaling**: Can't automatically create replicas
- **No rolling updates**: Can't update without downtime
- **No load balancing**: Single IP, no distribution

**Solution:** Use higher-level abstractions like **Deployments** (next section!).

### When to Use Pods Directly

- Quick testing and debugging
- One-off jobs (use Jobs/CronJobs instead)
- Understanding Kubernetes fundamentals
- **Not recommended for production applications**

## Best Practices

1. **Always use labels** - for organization and selection
2. **Declare resource limits** - prevent resource starvation
3. **Use meaningful names** - lowercase, hyphens, descriptive
4. **One process per container** - follow microservices pattern
5. **Use liveness/readiness probes** - for production (covered later)
6. **Don't use `latest` tag** - pin specific versions for reproducibility

## Examples

Check the [examples/](examples/index.md) directory for:

- `simple-pod.yaml` - Basic single-container Pod
- `multi-container-pod.yaml` - Pod with sidecar
- `pod-with-resources.yaml` - Pod with resource limits
- `compose.yaml` - Docker Compose comparison

## Key takeaways

- **Pods are the smallest unit** in Kubernetes
- **Pods can have multiple containers** that share network and storage
- **Pods are ephemeral** - they can be created and destroyed
- **Labels** are used for selection and organization
- **Use Deployments, not Pods directly**, for production apps

## Check your understanding

1. What's the difference between a Pod and a container?
2. Can containers in a Pod communicate via localhost?
3. What happens if a Pod's main container crashes?
4. How do you view logs from a specific container in a multi-container Pod?
5. Why shouldn't you use Pods directly in production?

<details class="solution" markdown="1">
<summary>Solution</summary>

1. **A Pod can contain one or more containers; it's the unit of deployment in Kubernetes**
2. **Yes, containers in a Pod share the network namespace**
3. **The Pod might restart (depends on restart policy), but without a controller like Deployment, it won't be recreated if deleted**
4. **`kubectl logs <pod-name> -c <container-name>`**
5. **Pods don't have self-healing, scaling, or rolling update capabilities - use Deployments instead**

</details>

## Hands-on

Apply the concepts from this section in the [lab](lab.md) exercises.

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../04-deployments/README.md)
