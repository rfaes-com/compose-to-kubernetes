# Lab: Services and Networking

**Duration:** 20 minutes

## Objectives

- Create different types of Services (ClusterIP, NodePort)
- Understand DNS-based service discovery
- Test connectivity between Pods using Services
- Expose applications externally

## Prerequisites

- Kind cluster running (simple.yaml or multi-node.yaml)
- kubectl configured and working

## Tasks

### Task 1: Create a Backend Service (ClusterIP)

Create a Deployment with 3 replicas of nginx and expose it with a ClusterIP Service.

**Requirements:**

- Deployment name: `backend`
- Label: `app=backend`
- Image: `nginx:1.25-alpine`
- 3 replicas
- Service name: `backend-service`
- Service type: `ClusterIP`
- Service port: 80

**Hints:**

```bash
# Create deployment
kubectl create deployment backend --image=nginx:1.25-alpine --replicas=3

# Expose with ClusterIP service
kubectl expose deployment backend --name=backend-service --port=80 --type=ClusterIP
```

### Task 2: Test Service Discovery

Create a temporary Pod to test DNS resolution and connectivity to the backend service.

**Requirements:**

- Use a busybox or curl container
- Test DNS resolution of `backend-service`
- Make HTTP requests to the service
- Verify load balancing across replicas

**Hints:**

```bash
# Run temporary Pod with curl
kubectl run test-pod --image=curlimages/curl:8.5.0 --rm -it --restart=Never -- sh

# Inside the pod:
# Test DNS
nslookup backend-service

# Test HTTP connectivity
curl http://backend-service
```

### Task 3: Create a NodePort Service

Expose the backend application externally using a NodePort Service.

**Requirements:**

- Service name: `backend-nodeport`
- Service type: `NodePort`
- Service port: 80
- NodePort: 30100 (explicit)

**Hints:**

```bash
# Create NodePort service
kubectl expose deployment backend --name=backend-nodeport --port=80 --type=NodePort

# Edit to set specific NodePort
kubectl edit service backend-nodeport
# Change nodePort to 30100
```

### Task 4: Access the NodePort Service

Access the service from outside the cluster.

**Requirements:**

- Get the node IP address
- Access the service via NodePort
- Verify you can reach the nginx welcome page

**Hints:**

```bash
# Get node IP (in kind, use localhost)
kubectl get nodes -o wide

# For kind clusters, access via localhost
curl http://localhost:30100

# Or from your browser
# http://localhost:30100
```

### Task 5: Multi-Port Service

Create a Redis deployment and expose both Redis (6379) and a metrics port.

**Requirements:**

- Deployment name: `redis`
- Image: `redis:7.2-alpine`
- Service name: `redis-service`
- Expose port 6379 (redis) and 9090 (metrics placeholder)
- Service type: ClusterIP

**Hints:**

```bash
# Create deployment
kubectl create deployment redis --image=redis:7.2-alpine

# Create multi-port service (requires YAML)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  - name: metrics
    port: 9090
    targetPort: 6379
EOF
```

### Task 6: Service Endpoints

Investigate how Services track Pod IPs using Endpoints.

**Requirements:**

- View the Endpoints for `backend-service`
- Scale the backend deployment
- Observe how Endpoints update automatically

**Hints:**

```bash
# View endpoints
kubectl get endpoints backend-service

# Describe for details
kubectl describe endpoints backend-service

# Scale deployment
kubectl scale deployment backend --replicas=5

# Watch endpoints update
kubectl get endpoints backend-service --watch
```

## Verification

Check your work:

```bash
# List all services
kubectl get svc

# List all endpoints
kubectl get endpoints

# Test connectivity from a test pod
kubectl run test --image=busybox:1.36 --rm -it --restart=Never -- wget -O- http://backend-service

# Access NodePort from outside
curl http://localhost:30100
```

## Cleanup

```bash
# Delete deployments
kubectl delete deployment backend redis

# Delete services
kubectl delete service backend-service backend-nodeport redis-service

# Delete any test pods (if not using --rm flag)
kubectl delete pod test-pod --ignore-not-found
```

## Check Your Understanding

1. What is the difference between ClusterIP and NodePort services?
2. How does Kubernetes DNS work for service discovery?
3. What happens to a Service's Endpoints when you scale a Deployment?
4. When would you use a LoadBalancer service type? (Hint: Cloud environments)
5. How can you access a ClusterIP service from outside the cluster? (Hint: kubectl port-forward)

## Next Steps

Proceed to [ConfigMaps and Secrets](../../06-config/README.md) to learn about configuration management.
