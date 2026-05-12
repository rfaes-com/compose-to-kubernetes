# Services & Networking

**Duration:** 40 minutes (25 min theory + 15 min lab)
**Format:** Presentation + Hands-on Lab

## Learning Objectives

- Understand why Services are needed
- Learn the different Service types
- Configure Service discovery and DNS
- Map Docker Compose networking to Kubernetes Services

## The Problem: Pods Are Ephemeral

Pods have dynamic IP addresses that change when they're recreated:

```text
Initial state:
Pod A: 10.244.1.5 ──→ Pod B: 10.244.1.6

Pod B crashes and is recreated:
Pod A: 10.244.1.5 ──→ Pod B: 10.244.1.7  (new IP!)
```

**Problem:** How do clients find Pods when IPs change?

**Solution:** Services provide a stable endpoint.

## What is a Service?

A **Service** is an abstraction that defines a logical set of Pods and a policy to access them.

### Key Benefits

- **Stable IP address:** Services get a ClusterIP that doesn't change
- **DNS name:** Services are accessible via DNS
- **Load balancing:** Automatically distributes traffic across Pods
- **Service discovery:** Pods can find each other by name

## Service Types

### 1. ClusterIP (Default)

**Purpose:** Internal cluster communication only

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80          # Service port
      targetPort: 8080  # Pod port
```

**Access:** `http://backend-service:80` (from within cluster)

**Use case:** Internal microservices communication

### 2. NodePort

**Purpose:** Expose Service on each Node's IP at a static port

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080    # Optional: 30000-32767 range
```

**Access:**

- Internal: `http://web-service:80`
- External: `http://<NodeIP>:30080`

**Use case:** Development, testing, or when no LoadBalancer available

### 3. LoadBalancer

**Purpose:** Expose Service externally using cloud provider's load balancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

**Access:**

- Internal: `http://frontend-service:80`
- External: Provided by cloud (AWS ELB, GCP LB, etc.)

**Note:** In kind/local clusters, LoadBalancer behaves like NodePort unless MetalLB is installed

**Use case:** Production external access

### 4. ExternalName

**Purpose:** Map Service to external DNS name

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

**Access:** `external-db` resolves to `database.example.com`

**Use case:** Accessing external services with internal names

## Service Discovery

### DNS-Based Discovery

Kubernetes automatically creates DNS records for Services:

```text
<service-name>.<namespace>.svc.cluster.local
```

**Examples:**

```bash
# Same namespace (short form)
curl http://api-service

# Different namespace
curl http://api-service.production

# Fully qualified domain name (FQDN)
curl http://api-service.production.svc.cluster.local
```

### Environment Variables

Kubernetes injects environment variables for Services:

```bash
# Format: <SERVICE_NAME>_SERVICE_HOST and <SERVICE_NAME>_SERVICE_PORT

# Example for service named "database"
DATABASE_SERVICE_HOST=10.96.0.50
DATABASE_SERVICE_PORT=5432
```

**Note:** Environment variables are set only for Services that exist when the Pod starts.

## Selectors and Labels

Services use label selectors to find Pods:

```yaml
# Service
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web         # Matches Pods with this label
    tier: frontend
  ports:
    - port: 80
      targetPort: 8080
```

```yaml
# Deployment creates Pods with matching labels
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      tier: frontend
  template:
    metadata:
      labels:
        app: web       # Must match Service selector
        tier: frontend
    spec:
      containers:
        - name: web
          image: nginx:latest
```

## Endpoints

When a Service selects Pods, Kubernetes creates an **Endpoints** object:

```bash
# View Service endpoints
kubectl get endpoints web-service

# Output:
# NAME          ENDPOINTS                                    AGE
# web-service   10.244.1.5:8080,10.244.1.6:8080,10.244.1.7:8080   5m
```

**Endpoints** track the IP addresses of Pods backing the Service.

## Port Configuration

### Understanding Ports

```yaml
spec:
  ports:
    - name: http              # Port name (optional but recommended)
      protocol: TCP           # TCP or UDP
      port: 80                # Port exposed by the Service
      targetPort: 8080        # Port on the Pod
      nodePort: 30080         # Node port (for NodePort/LoadBalancer types)
```

**Port mapping:**

```text
Client → Service (port: 80) → Pod (targetPort: 8080)
```

### Named Ports

You can reference container ports by name:

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: web
          ports:
            - name: web-port     # Named port
              containerPort: 8080
---
# Service
apiVersion: v1
kind: Service
spec:
  ports:
    - port: 80
      targetPort: web-port      # Reference by name
```

## Load Balancing

Services automatically load balance across healthy Pods:

```mermaid
flowchart TB
    Client["Client Request"]
        --> Service["Service"]

    Service --> Pod1["Pod1"]
    Service --> Pod2["Pod2"]
    Service --> Pod3["Pod3"]
```

**Algorithm:** Round-robin (default) or session affinity

### Session Affinity

```yaml
spec:
  sessionAffinity: ClientIP    # Stick to same Pod based on client IP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800    # 3 hours
```

## Headless Services

A Service without ClusterIP for direct Pod access:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  clusterIP: None              # Makes it headless
  selector:
    app: database
  ports:
    - port: 5432
```

**DNS returns Pod IPs directly** instead of Service IP.

**Use case:** StatefulSets, databases where you need direct Pod access

## Docker Compose to Kubernetes

### Docker Compose Networking

```yaml
version: '3'
services:
  frontend:
    image: nginx:latest
    ports:
      - "8080:80"             # Host:Container
    networks:
      - webnet

  backend:
    image: api-server:latest
    networks:
      - webnet
      - dbnet
    # frontend can reach this via: http://backend:8080

  database:
    image: postgres:latest
    networks:
      - dbnet

networks:
  webnet:
  dbnet:
```

### Kubernetes Services

```yaml
# Frontend Service (external access)
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer          # or NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
---
# Backend Service (internal only)
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP             # default, internal only
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
---
# Database Service (internal only)
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ClusterIP
  selector:
    app: database
  ports:
    - port: 5432
      targetPort: 5432
```

**Key Differences:**

- No explicit network creation needed in Kubernetes
- Service types control accessibility (internal vs external)
- DNS-based service discovery is automatic
- All Pods can communicate by default (use NetworkPolicies to restrict)

## Creating Services

### Imperative

```bash
# Expose a Deployment
kubectl expose deployment web --port=80 --target-port=8080 --type=LoadBalancer

# Expose a Pod
kubectl expose pod my-pod --port=80 --name=my-service
```

### Declarative (Recommended)

See [examples/clusterip-service.yaml](examples/clusterip-service.yaml)

## Testing Services

```bash
# Create a test Pod
kubectl run test-pod --image=busybox:latest -it --rm -- sh

# Inside the Pod, test DNS resolution
nslookup my-service

# Test HTTP connectivity
wget -O- http://my-service:80

# Exit removes the Pod
exit
```

## Best Practices

1. **Use ClusterIP for internal services** - most common type
2. **Name your ports** - helps with multi-port services
3. **Match selectors carefully** - Service won't route if labels don't match
4. **Use meaningful Service names** - they become DNS names
5. **Avoid NodePort in production** - use LoadBalancer or Ingress instead
6. **Check Endpoints** - if traffic doesn't reach Pods, check endpoints

## Common Issues

### No Endpoints

```bash
kubectl get endpoints my-service
# Shows: <none>

# Causes:
# 1. Selector doesn't match Pod labels
# 2. Pods aren't ready
# 3. Pods don't exist

# Fix: Check labels
kubectl get pods --show-labels
kubectl describe service my-service
```

### Service Not Resolving

```bash
# Test DNS from within a Pod
kubectl run test --image=busybox -it --rm -- nslookup my-service

# If it fails:
# 1. Check CoreDNS is running: kubectl get pods -n kubesystem -l k8s-app=kube-dns
# 2. Verify Service exists: kubectl get svc my-service
```

### Can't Access NodePort Externally

```bash
# In kind clusters, you need to map ports in cluster config
# See setup/kind/simple.yaml for port mapping examples
```

## Lab: Working with Services

**Time:** 15 minutes

Practice creating Services and understanding service discovery.

See [lab/instructions.md](lab/instructions.md)

## Examples

Check the [examples/](examples/) directory for:

- `clusterip-service.yaml` - Internal Service
- `nodeport-service.yaml` - External access via NodePort
- `loadbalancer-service.yaml` - Cloud load balancer
- `compose.yaml` - Docker Compose comparison

## Key Takeaways

- **Services provide stable endpoints** for ephemeral Pods
- **ClusterIP** (default) for internal, **NodePort** for testing, **LoadBalancer** for production external access
- **DNS-based service discovery** is automatic
- **Label selectors** connect Services to Pods
- **Endpoints** track backing Pod IPs

## Next Section

Now we can deploy and expose applications. Let's configure them!

**Next:** [06-config - Configuration Management](../06-config/README.md)

## Check Your Understanding

1. What problem do Services solve?
2. What's the difference between ClusterIP, NodePort, and LoadBalancer?
3. How do Pods discover Services?
4. What happens if a Service's selector doesn't match any Pods?
5. How can you check which Pods a Service is routing to?

<details markdown="1">
<summary>Click for answers</summary>

1. **Services provide stable IP addresses and DNS names for ephemeral Pods with changing IPs**
2. **ClusterIP: internal only; NodePort: accessible via Node IP:port; LoadBalancer: external cloud load balancer**
3. **Via DNS (service-name.namespace) or environment variables**
4. **The Service exists but has no Endpoints - traffic won't be routed anywhere**
5. **`kubectl get endpoints <service-name>` or `kubectl describe service <service-name>`**

</details>
