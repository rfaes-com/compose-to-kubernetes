# Lab Solutions: Services and Networking

Complete solutions for the Services lab exercises.

## Task 1: Create a Backend Service (ClusterIP)

```bash
# Create deployment with labels
kubectl create deployment backend --image=nginx:1.25-alpine --replicas=3

# Expose with ClusterIP service
kubectl expose deployment backend --name=backend-service --port=80 --type=ClusterIP

# Verify deployment
kubectl get deployment backend
```

**Expected output:**

```text
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
backend   3/3     3            3           10s
```

```bash
# Verify service
kubectl get service backend-service
```

**Expected output:**

```text
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
backend-service   ClusterIP   10.96.123.45    <none>        80/TCP    5s
```

```bash
# Describe service to see selector and endpoints
kubectl describe service backend-service
```

**Expected output:**

```text
Name:              backend-service
Namespace:         default
Labels:            app=backend
Selector:          app=backend
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.123.45
IPs:               10.96.123.45
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.10:80,10.244.1.11:80,10.244.2.10:80
```

## Task 2: Test Service Discovery

```bash
# Run temporary pod with curl
kubectl run test-pod --image=curlimages/curl:8.5.0 --rm -it --restart=Never -- sh

# Inside the pod, test DNS resolution
nslookup backend-service
```

**Expected output:**

```text
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   backend-service.default.svc.cluster.local
Address: 10.96.123.45
```

```bash
# Test HTTP connectivity
curl http://backend-service
```

**Expected output:**

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

```bash
# Test multiple times to verify load balancing
for i in 1 2 3 4 5; do
  echo "Request $i:"
  curl -s http://backend-service | grep -i "welcome"
done
```

**Expected output:**

```text
Request 1:
<title>Welcome to nginx!</title>
Request 2:
<title>Welcome to nginx!</title>
...
```

```bash
# Exit the test pod
exit
```

## Task 3: Create a NodePort Service

```bash
# Create NodePort service
kubectl expose deployment backend --name=backend-nodeport --port=80 --type=NodePort

# Get the assigned NodePort
kubectl get service backend-nodeport
```

**Expected output:**

```text
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
backend-nodeport   NodePort   10.96.234.56    <none>        80:31234/TCP   5s
```

```bash
# Edit to set specific NodePort to 30100
kubectl patch service backend-nodeport --type='json' -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value":30100}]'

# Verify the change
kubectl get service backend-nodeport
```

**Expected output:**

```text
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
backend-nodeport   NodePort   10.96.234.56    <none>        80:30100/TCP   15s
```

## Task 4: Access the NodePort Service

```bash
# Get node information
kubectl get nodes -o wide
```

**Expected output:**

```text
NAME                    STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP
kind-control-plane      Ready    control-plane   10m   v1.28.0   172.18.0.2    <none>
kind-worker             Ready    <none>          10m   v1.28.0   172.18.0.3    <none>
```

```bash
# For kind clusters, access via localhost
curl http://localhost:30100
```

**Expected output:**

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
...
</html>
```

```bash
# Check which pod handled each request (check nginx logs)
for i in 1 2 3; do curl -s http://localhost:30100 > /dev/null; done
kubectl logs -l app=backend --tail=3
```

## Task 5: Multi-Port Service

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
```

**Expected output:**

```text
service/redis-service created
```

```bash
# Verify the service
kubectl get service redis-service
kubectl describe service redis-service
```

**Expected output:**

```text
Name:              redis-service
Namespace:         default
Selector:          app=redis
Type:              ClusterIP
IP:                10.96.145.67
Port:              redis  6379/TCP
TargetPort:        6379/TCP
Port:              metrics  9090/TCP
TargetPort:        6379/TCP
Endpoints:         10.244.1.20:6379
```

```bash
# Test Redis connectivity
kubectl run redis-client --image=redis:7.2-alpine --rm -it --restart=Never -- redis-cli -h redis-service -p 6379 PING
```

**Expected output:**

```text
PONG
```

## Task 6: Service Endpoints

```bash
# View endpoints for backend-service
kubectl get endpoints backend-service
```

**Expected output:**

```text
NAME              ENDPOINTS                                            AGE
backend-service   10.244.1.10:80,10.244.1.11:80,10.244.2.10:80         5m
```

```bash
# Describe endpoints for details
kubectl describe endpoints backend-service
```

**Expected output:**

```text
Name:         backend-service
Namespace:    default
Labels:       app=backend
Subsets:
  Addresses:          10.244.1.10,10.244.1.11,10.244.2.10
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    <unset>  80    TCP
```

```bash
# Scale the deployment up
kubectl scale deployment backend --replicas=5

# Watch endpoints update (Ctrl+C to stop)
kubectl get endpoints backend-service --watch
```

**Expected output:**

```text
NAME              ENDPOINTS                                            AGE
backend-service   10.244.1.10:80,10.244.1.11:80,10.244.2.10:80         5m
backend-service   10.244.1.10:80,10.244.1.11:80,10.244.2.10:80 + 2 more...   5m
```

```bash
# View all endpoint IPs
kubectl get endpoints backend-service -o jsonpath='{.subsets[*].addresses[*].ip}' | tr ' ' '\n'
```

**Expected output:**

```text
10.244.1.10
10.244.1.11
10.244.1.12
10.244.2.10
10.244.2.11
```

```bash
# Scale back down
kubectl scale deployment backend --replicas=2

# Verify endpoints updated
kubectl get endpoints backend-service
```

**Expected output:**

```text
NAME              ENDPOINTS                      AGE
backend-service   10.244.1.10:80,10.244.2.10:80  7m
```

## Cleanup

```bash
# Delete all resources
kubectl delete deployment backend redis
kubectl delete service backend-service backend-nodeport redis-service

# Verify cleanup
kubectl get deployments,services,endpoints
```

**Expected output:**

```text
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   30m
```

## Bonus Challenges

### Challenge 1: Headless Service

Create a headless service (ClusterIP: None) to get direct Pod IPs instead of load balancing.

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

# Create backend deployment if not exists
kubectl create deployment backend --image=nginx:1.25-alpine --replicas=3 || true

# Test DNS - returns all Pod IPs
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup backend-headless
```

**Expected output:**

```text
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   backend-headless.default.svc.cluster.local
Address: 10.244.1.10
Name:   backend-headless.default.svc.cluster.local
Address: 10.244.1.11
Name:   backend-headless.default.svc.cluster.local
Address: 10.244.2.10
```

### Challenge 2: Service Without Selector

Create a Service and manually manage its Endpoints to point to an external resource.

```bash
# Create service without selector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
- addresses:
  - ip: 93.184.216.34  # example.com IP
  ports:
  - port: 80
EOF

# Test connectivity
kubectl run curl-test --image=curlimages/curl:8.5.0 --rm -it --restart=Never -- curl -I http://external-service
```

### Challenge 3: Port Forward to ClusterIP

Access a ClusterIP service from your local machine without NodePort.

```bash
# Start port-forward (runs in foreground)
kubectl port-forward service/backend-service 8080:80

# In another terminal, test locally
curl http://localhost:8080
```

## Key Takeaways

1. **ClusterIP** services are for internal cluster communication only
2. **NodePort** services expose applications on a port on every node
3. Services use **label selectors** to find Pods
4. Kubernetes DNS provides automatic service discovery
5. **Endpoints** track the actual Pod IPs behind a Service
6. Services provide load balancing across multiple Pod replicas
7. Each service port must have a unique name when defining multiple ports

## Common Issues and Solutions

**Issue:** Service has no endpoints

- **Cause:** Selector doesn't match any Pods
- **Solution:** Check `kubectl describe service <name>` and verify Pod labels match the selector

**Issue:** Cannot access NodePort service

- **Cause:** In kind, need to use localhost instead of node IP
- **Solution:** Use `http://localhost:<nodePort>`

**Issue:** DNS resolution fails

- **Cause:** CoreDNS pods not running
- **Solution:** Check `kubectl get pods -n kube-system | grep coredns`

**Issue:** Service load balancing not working

- **Cause:** Only one Pod is running
- **Solution:** Scale deployment to multiple replicas and test again
