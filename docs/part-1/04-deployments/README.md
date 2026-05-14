# Deployments & ReplicaSets

**Duration:** 40 minutes (20 minutes theory + 20 minutes lab)

## Learning Objectives

- Understand the Deployment abstraction and its benefits
- Learn how ReplicaSets manage Pod replicas
- Scale applications horizontally
- Perform rolling updates and rollbacks
- Map Docker Compose scaling to Kubernetes Deployments

## The Problem with Naked Pods

Pods created directly have serious limitations:

- **No self-healing:** Pod dies, it's gone forever
- **No scaling:** Can't automatically create multiple instances
- **No rolling updates:** Can't update without downtime
- **Manual management:** You're responsible for everything

**Solution:** Use Deployments, which manage Pods for you.

## What is a Deployment?

A **Deployment** provides declarative updates for Pods and ReplicaSets. It's the recommended way to deploy stateless applications.

### Deployment Hierarchy

```text
Deployment
    └── ReplicaSet
            ├── Pod 1
            ├── Pod 2
            └── Pod 3
```

**Responsibilities:**

- **Deployment:** Manages updates, rollback, scaling strategy
- **ReplicaSet:** Ensures desired number of Pod replicas are running
- **Pod:** Runs your application

## What is a ReplicaSet?

A **ReplicaSet** ensures a specified number of Pod replicas are running at any given time.

### Key Features

- **Self-healing:** If a Pod dies, ReplicaSet creates a new one
- **Scaling:** Maintains desired replica count
- **Pod selector:** Uses labels to identify which Pods it manages

**Note:** You rarely create ReplicaSets directly. Deployments manage them for you.

## Creating a Deployment

### Imperative Method

```bash
# Create a Deployment
kubectl create deployment nginx --image=nginx:latest

# Scale it
kubectl scale deployment nginx --replicas=3

# View it
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

### Declarative Method (Recommended)

See [examples/simple-deployment.yaml](examples/simple-deployment.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f /workspaces/compose-to-kubernetes/part-1/04-deployments/examples/simple-deployment.yaml

# Watch Pods being created
kubectl get pods -w
```

## Understanding the Deployment Spec

### Key Fields

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app              # Deployment name
  labels:                   # Deployment labels
    app: my-app

spec:
  replicas: 3               # How many Pods to run

  selector:                 # How to find Pods to manage
    matchLabels:
      app: my-app

  template:                 # Pod template
    metadata:
      labels:
        app: my-app         # Pod labels (must match selector!)
    spec:
      containers:
        - name: app
          image: myapp:v1
```

**Critical:** The `selector.matchLabels` must match the `template.metadata.labels`!

## Scaling Applications

### Manual Scaling

```bash
# Scale up
kubectl scale deployment nginx-deployment --replicas=5

# Verify
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx
```

### Declarative Scaling

Edit the YAML file and change `replicas: 5`, then apply it again.

### Autoscaling (Preview)

```bash
# Create Horizontal Pod Autoscaler (covered in Part 2)
kubectl autoscale deployment nginx-deployment --min=2 --max=10 --cpu-percent=80
```

## Self-Healing in Action

Watch Kubernetes automatically replace failed Pods:

```bash
# Create a deployment
kubectl create deployment self-heal --image=nginx --replicas=3

# Watch Pods
kubectl get pods -l app=self-heal -w

# In another terminal, delete a Pod
kubectl delete pod <pod-name>

# Back in the first terminal, you'll see:
# - Pod terminating
# - New Pod immediately created
# - ReplicaSet maintaining desired state (3 replicas)
```

## Rolling Updates

Deployments allow zero-downtime updates using rolling update strategy.

### Update Image

```bash
# Update the image
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Watch the rollout
kubectl rollout status deployment/nginx-deployment

# See the rollout history
kubectl rollout history deployment/nginx-deployment
```

### What Happens During Rolling Update

```text
Initial state: 3 Pods running v1

Step 1: Create 1 new Pod with v2
        Old: [v1] [v1] [v1]
        New: [v2]

Step 2: Terminate 1 old Pod
        Old: [v1] [v1]
        New: [v2]

Step 3: Create 1 new Pod with v2
        Old: [v1] [v1]
        New: [v2] [v2]

Step 4: Terminate 1 old Pod
        Old: [v1]
        New: [v2] [v2]

Step 5: Create final Pod with v2
        Old: [v1]
        New: [v2] [v2] [v2]

Step 6: Terminate last old Pod
        Old: []
        New: [v2] [v2] [v2]

Complete: 3 Pods running v2
```

### Update Strategy Configuration

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Max Pods above desired during update
      maxUnavailable: 1   # Max Pods unavailable during update
```

**Example:**

- `replicas: 10`
- `maxSurge: 2` → Can have up to 12 Pods during update
- `maxUnavailable: 1` → Always have at least 9 Pods running

## Rollback

Made a mistake? Rollback to previous version:

```bash
# Undo the last rollout
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout history deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pause a rollout (if you catch a problem)
kubectl rollout pause deployment/nginx-deployment

# Resume when ready
kubectl rollout resume deployment/nginx-deployment
```

## Deployment Status

### Check Deployment

```bash
# List deployments
kubectl get deployments

# Output:
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           5m

# Detailed view
kubectl describe deployment nginx-deployment

# YAML output
kubectl get deployment nginx-deployment -o yaml
```

### Status Fields

- **READY:** Current replicas / Desired replicas
- **UP-TO-DATE:** Replicas updated to latest spec
- **AVAILABLE:** Replicas available to users
- **AGE:** Time since creation

### Check ReplicaSet

```bash
# List ReplicaSets
kubectl get replicasets

# Output shows current and old ReplicaSets
# NAME                          DESIRED   CURRENT   READY   AGE
# nginx-deployment-5d59d67564   3         3         3       5m
# nginx-deployment-7848d4b86f   0         0         0       10m

# Old ReplicaSets are kept for rollback
```

## Docker Compose to Kubernetes

### Docker Compose Scaling

```yaml
version: '3'
services:
  web:
    image: nginx:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
    ports:
      - "8080:80"
```

```bash
# Scale with docker-compose
docker-compose up --scale web=5
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

```bash
# Scale with kubectl
kubectl scale deployment web --replicas=5
```

**Key Differences:**

- Kubernetes separates Deployment from Service (ports/networking)
- More granular control over update strategy
- Self-healing is automatic
- Rollback capability built-in

## Deployment Strategies

### RollingUpdate (Default)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

**Pros:** Zero downtime, gradual rollout
**Cons:** Both versions running simultaneously

### Recreate

```yaml
strategy:
  type: Recreate
```

**Behavior:** Terminate all old Pods, then create new ones

**Pros:** Clean state, only one version at a time
**Cons:** Downtime during update

**Use case:** Databases, stateful apps that can't run multiple versions

## Best Practices

1. **Always use Deployments** (not naked Pods) for stateless apps
2. **Specify image tags** (not `latest`) for predictability
3. **Set resource requests/limits** (covered later)
4. **Use meaningful labels** for organization
5. **Test rollouts gradually** with small `maxSurge`/`maxUnavailable`
6. **Keep rollout history** for easy rollback
7. **Use readiness probes** (covered later) to control traffic during rollout

## Common Issues

### Deployment Not Progressing

```bash
# Check events
kubectl describe deployment <name>

# Check Pod events
kubectl describe pod <pod-name>

# Common causes:
# - Image pull errors
# - Insufficient resources
# - Pod crashes immediately
```

### Wrong Selector

```yaml
# This will fail!
spec:
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: api     # Doesn't match selector!
```

**Error:** `The Deployment spec is invalid: spec.template.metadata.labels: Invalid value: ... selector does not match template labels`

### Too Many Replicas

```bash
# Check node capacity
kubectl describe nodes

# If not enough resources:
# - Pods stay in Pending
# - Scale down or add nodes
```

## Examples

Check the [examples/](examples/index.md) directory for:

- `simple-deployment.yaml` - Basic Deployment
- `deployment-with-strategy.yaml` - Custom update strategy
- `compose.yaml` - Docker Compose comparison

## Key takeaways

- **Deployments manage ReplicaSets, ReplicaSets manage Pods**
- **Self-healing:** Failed Pods are automatically replaced
- **Scaling:** Easy horizontal scaling with `replicas` field
- **Rolling updates:** Zero-downtime deployments
- **Rollback:** Easy revert to previous versions
- **Always use Deployments** for production stateless apps

## Check your understanding

1. What's the relationship between Deployment, ReplicaSet, and Pod?
2. What happens if you manually delete a Pod managed by a Deployment?
3. How do you scale a Deployment to 5 replicas?
4. What's the difference between `maxSurge` and `maxUnavailable`?
5. How do you rollback a failed deployment?

<details class="solution" markdown="1">
<summary>Solution</summary>

1. **Deployment manages ReplicaSets, ReplicaSets manage Pods. Deployment provides declarative updates and rollback.**
2. **The ReplicaSet immediately creates a new Pod to maintain the desired replica count.**
3. **`kubectl scale deployment <name> --replicas=5` or update the YAML and `kubectl apply`**
4. **`maxSurge`: max extra Pods during update. `maxUnavailable`: max Pods that can be down during update.**
5. **`kubectl rollout undo deployment/<name>`**

</details>

## Hands-on

Apply the concepts from this section in the [lab](lab.md) exercises.

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../05-services/README.md)
