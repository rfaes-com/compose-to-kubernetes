# Lab

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

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Create deployment
kubectl create deployment backend --image=nginx:1.25-alpine --replicas=3

# Expose with ClusterIP service
kubectl expose deployment backend --name=backend-service --port=80 --type=ClusterIP
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create deployment with labels
kubectl create deployment backend --image=nginx:1.25-alpine --replicas=3

# Expose with ClusterIP service
kubectl expose deployment backend --name=backend-service --port=80 --type=ClusterIP

# Verify deployment
kubectl get deployment backend

# Output:
# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# backend   3/3     3            3           10s

# Verify service
kubectl get service backend-service

# Output:
# NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# backend-service   ClusterIP   10.96.123.45    <none>        80/TCP    5s

# Describe service to see selector and endpoints
kubectl describe service backend-service
```

</details>

### Task 2: Test Service Discovery

Create a temporary Pod to test DNS resolution and connectivity to the backend service.

**Requirements:**

- Use a busybox or curl container
- Test DNS resolution of `backend-service`
- Make HTTP requests to the service
- Verify load balancing across replicas

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Run temporary Pod with curl
kubectl run test-pod --image=curlimages/curl:8.5.0 --rm -it --restart=Never -- sh

# Inside the pod:
# Test DNS
nslookup backend-service

# Test HTTP connectivity
curl http://backend-service
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Run temporary pod with curl
kubectl run test-pod --image=curlimages/curl:8.5.0 --rm -it --restart=Never -- sh

# Inside the pod, test DNS resolution:
nslookup backend-service

# Output:
# Server:         10.96.0.10
# Address:        10.96.0.10:53
# Name:   backend-service.default.svc.cluster.local
# Address: 10.96.123.45

# Test HTTP connectivity
curl http://backend-service

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...

# Exit the test pod
exit
```

</details>

### Task 3: Create a NodePort Service

Expose the backend application externally using a NodePort Service.

**Requirements:**

- Service name: `backend-nodeport`
- Service type: `NodePort`
- Service port: 80
- NodePort: 30100 (explicit)

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Create NodePort service
kubectl expose deployment backend --name=backend-nodeport --port=80 --type=NodePort

# Edit to set specific NodePort
kubectl patch service backend-nodeport --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value":30100}]'
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create NodePort service
kubectl expose deployment backend --name=backend-nodeport --port=80 --type=NodePort

# Set specific NodePort to 30100
kubectl patch service backend-nodeport --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value":30100}]'

# Verify the change
kubectl get service backend-nodeport

# Output:
# NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# backend-nodeport   NodePort   10.96.234.56    <none>        80:30100/TCP   15s
```

</details>

### Task 4: Access the NodePort Service

Access the service from outside the cluster.

**Requirements:**

- Get the node IP address
- Access the service via NodePort
- Verify you can reach the nginx welcome page

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Get node IP (in kind, use localhost)
kubectl get nodes -o wide

# For kind clusters, access via localhost
curl http://localhost:30100
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Get node information
kubectl get nodes -o wide

# For kind clusters, access via localhost
curl http://localhost:30100

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...
```

</details>

### Task 5: Multi-Port Service

Create a Redis deployment and expose both Redis (6379) and a metrics port.

**Requirements:**

- Deployment name: `redis`
- Image: `redis:7.2-alpine`
- Service name: `redis-service`
- Expose port 6379 (redis) and 9090 (metrics placeholder)
- Service type: ClusterIP

<details class="hint" markdown="1">
<summary>Hint</summary>

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

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create Redis deployment
kubectl create deployment redis --image=redis:7.2-alpine

# Create multi-port service
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

# Output:
# service/redis-service created

# Verify the service
kubectl describe service redis-service

# Test Redis connectivity
kubectl run redis-client --image=redis:7.2-alpine --rm -it --restart=Never \
  -- redis-cli -h redis-service -p 6379 PING

# Output:
# PONG
```

</details>

### Task 6: Service Endpoints

Investigate how Services track Pod IPs using Endpoints.

**Requirements:**

- View the Endpoints for `backend-service`
- Scale the backend deployment
- Observe how Endpoints update automatically

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# View endpoints
kubectl get endpoints backend-service

# Scale deployment
kubectl scale deployment backend --replicas=5

# Watch endpoints update
kubectl get endpoints backend-service --watch
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# View endpoints for backend-service
kubectl get endpoints backend-service

# Output:
# NAME              ENDPOINTS                                            AGE
# backend-service   10.244.1.10:80,10.244.1.11:80,10.244.2.10:80         5m

# Describe endpoints for details
kubectl describe endpoints backend-service

# Scale the deployment up
kubectl scale deployment backend --replicas=5

# Watch endpoints update (Ctrl+C to stop)
kubectl get endpoints backend-service --watch

# Output shows additional endpoints as new Pods become ready

# View all endpoint IPs
kubectl get endpoints backend-service -o jsonpath='{.subsets[*].addresses[*].ip}' | tr ' ' '\n'

# Scale back down
kubectl scale deployment backend --replicas=2
```

</details>

## Verification

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

## Bonus Challenges

**1. Headless Service:** Create a headless service (`clusterIP: None`) to get direct Pod IPs instead of load balancing

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create headless service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
spec:
  clusterIP: None
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
EOF

# Create backend deployment if not present
kubectl create deployment backend --image=nginx:1.25-alpine --replicas=3 2>/dev/null || true

# Test DNS - returns all Pod IPs instead of a single virtual IP
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never \
  -- nslookup backend-headless

# Cleanup
kubectl delete service backend-headless
```

</details>

**2. Port Forward to ClusterIP:** Access a ClusterIP service from your local machine without NodePort

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Start port-forward in background
kubectl port-forward service/backend-service 8080:80 &
PF_PID=$!

# Test locally
curl http://localhost:8080

# Stop port forward
kill $PF_PID
```

</details>

## Common issues

| Issue                         | Solution                                                                               |
| ----------------------------- | -------------------------------------------------------------------------------------- |
| Service has no Endpoints      | Check selector matches Pod labels: `kubectl describe service <name>`                   |
| Can't access NodePort         | In kind clusters, ports must be mapped in the cluster config; use `localhost`          |
| DNS not resolving             | Verify CoreDNS is running: `kubectl get pods -n kube-system -l k8s-app=kube-dns`       |
| Connection refused to service | Verify target Pods are Running and port matches `targetPort` in the Service definition |

## Key takeaways

1. **Services provide stable endpoints** for ephemeral Pods with changing IPs
2. **ClusterIP** for internal communication, **NodePort** for testing, **LoadBalancer** for production
3. **DNS-based service discovery** is automatic — use the service name as hostname
4. **Label selectors** connect Services to Pods; mismatches cause empty Endpoints
5. **Endpoints** are updated automatically as Pods scale up or down

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../06-config/README.md)
