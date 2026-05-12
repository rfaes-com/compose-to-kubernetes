# Ingress Lab

Duration: 25 minutes

## Objectives

- Install NGINX Ingress Controller in kind
- Create services and deployments
- Configure path-based routing
- Configure host-based routing
- Test Ingress routing
- Implement TLS termination

## Prerequisites

- kind cluster running (from Part 1)
- kubectl configured
- Basic understanding of Services and Deployments

## Lab Tasks

### Task 1: Install NGINX Ingress Controller

Install the NGINX Ingress Controller for kind:

```bash
# Apply the NGINX Ingress Controller manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for the controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

**Expected Output:**
You should see the ingress-nginx-controller pod running and a LoadBalancer service.

---

### Task 2: Deploy Backend Applications

Create two simple applications that we'll route to using Ingress:

```bash
# Create a namespace for this lab
kubectl create namespace ingress-lab

# Set default namespace
kubectl config set-context --current --namespace=ingress-lab
```

Create `backend-apps.yaml`:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ingress-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo '<h1>Frontend Application</h1><p>Path: /frontend</p>' > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: ingress-lab
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: ingress-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo '<h1>Backend API</h1><p>Path: /api</p>' > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: ingress-lab
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

Apply the manifests:

```bash
kubectl apply -f backend-apps.yaml

# Verify pods are running
kubectl get pods
kubectl get svc
```

---

### Task 3: Create Path-Based Ingress

Create an Ingress that routes traffic based on URL paths.

Create `path-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  namespace: ingress-lab
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /frontend
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

Apply and test:

```bash
# Apply the Ingress
kubectl apply -f path-ingress.yaml

# Check Ingress status
kubectl get ingress
kubectl describe ingress path-ingress

# Test from outside the cluster
curl http://localhost/frontend
curl http://localhost/api
```

**Expected Output:**

- `curl http://localhost/frontend` should show "Frontend Application"
- `curl http://localhost/api` should show "Backend API"

---

### Task 4: Create Host-Based Ingress

Now create an Ingress that routes based on hostname.

First, create additional services:

Create `host-based-apps.yaml`:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: ingress-lab
spec:
  replicas: 1
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
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo '<h1>Web Application</h1><p>Host: web.local</p>' > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: ingress-lab
spec:
  selector:
    app: web
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
  namespace: ingress-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo '<h1>API Application</h1><p>Host: api.local</p>' > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
---
apiVersion: v1
kind: Service
metadata:
  name: api-backend-service
  namespace: ingress-lab
spec:
  selector:
    app: api
  ports:
  - port: 80
```

Create `host-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
  namespace: ingress-lab
spec:
  ingressClassName: nginx
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: api.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-backend-service
            port:
              number: 80
```

Apply and test:

```bash
# Apply the apps and Ingress
kubectl apply -f host-based-apps.yaml
kubectl apply -f host-ingress.yaml

# Test with Host header
curl -H "Host: web.local" http://localhost
curl -H "Host: api.local" http://localhost
```

**Expected Output:**

- `web.local` request shows "Web Application"
- `api.local` request shows "API Application"

---

### Task 5: Combine Path and Host Routing

Create an Ingress that uses both path and host-based routing.

Create `combined-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: combined-ingress
  namespace: ingress-lab
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: app.local
    http:
      paths:
      - path: /web(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-backend-service
            port:
              number: 80
```

Test:

```bash
kubectl apply -f combined-ingress.yaml

curl -H "Host: app.local" http://localhost/web
curl -H "Host: app.local" http://localhost/api
```

---

### Task 6: Add TLS/HTTPS Support

Create a self-signed certificate and configure HTTPS.

```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=secure.local/O=workshop"

# Create TLS secret
kubectl create secret tls secure-tls \
  --cert=tls.crt --key=tls.key \
  -n ingress-lab

# Verify secret
kubectl get secret secure-tls -n ingress-lab
```

Create `tls-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: ingress-lab
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.local
    secretName: secure-tls
  rules:
  - host: secure.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

Apply and test:

```bash
kubectl apply -f tls-ingress.yaml

# Test HTTPS (with -k to ignore self-signed cert warning)
curl -k -H "Host: secure.local" https://localhost

# Test HTTP redirect to HTTPS
curl -v -H "Host: secure.local" http://localhost 2>&1 | grep -i location
```

**Expected Output:**

- HTTPS request succeeds
- HTTP request redirects to HTTPS (301)

---

### Task 7: Add Custom Annotations

Enhance your Ingress with custom annotations for rate limiting and CORS.

Create `annotated-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  namespace: ingress-lab
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /protected
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

Test:

```bash
kubectl apply -f annotated-ingress.yaml

# Test rate limiting
for i in {1..15}; do 
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost/protected
  sleep 0.1
done
```

**Expected Output:**
First 10 requests return 200, subsequent requests may return 429 (Too Many Requests) or 503.

---

## Verification

Check all your Ingress resources:

```bash
# List all Ingresses
kubectl get ingress -n ingress-lab

# Describe to see rules and backends
kubectl describe ingress -n ingress-lab

# Check Ingress Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=50
```

## Cleanup

```bash
# Delete the namespace (removes all resources)
kubectl delete namespace ingress-lab

# Optional: Uninstall NGINX Ingress Controller
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

## Bonus Challenges

### Challenge 1: Default Backend

Create a custom 404 page as the default backend for unmatched routes.

**Hint:** Use `defaultBackend` in the Ingress spec.

### Challenge 2: Path Type Comparison

Create three Ingresses with different pathType values (Prefix, Exact, ImplementationSpecific) and test the differences.

### Challenge 3: Basic Authentication

Implement HTTP Basic Auth on one of your Ingress paths.

**Hint:**

```bash
htpasswd -c auth myuser
kubectl create secret generic basic-auth --from-file=auth
```

Use annotation: `nginx.ingress.kubernetes.io/auth-type: basic`

### Challenge 4: Canary Deployments

Use Ingress annotations to implement a canary deployment that routes 10% of traffic to a new version.

**Hint:** Look up `nginx.ingress.kubernetes.io/canary` annotations.

## Troubleshooting Tips

**Ingress not working:**

- Check Ingress Controller is running: `kubectl get pods -n ingress-nginx`
- Verify Ingress: `kubectl describe ingress <name>`
- Check service endpoints exist: `kubectl get endpoints`
- Review controller logs: `kubectl logs -n ingress-nginx <pod-name>`

**404 errors:**

- Verify service name and port match in Ingress
- Check path and pathType configuration
- Ensure ingressClassName is set correctly

**Host-based routing not working:**

- Use `-H "Host: hostname"` with curl
- Check DNS or /etc/hosts configuration
- Verify host field in Ingress rules

**TLS issues:**

- Verify secret exists: `kubectl get secret <secret-name>`
- Check secret has tls.crt and tls.key
- Ensure hostname in TLS matches rules

## Summary

You've learned to:

- Install and configure NGINX Ingress Controller
- Create path-based routing (single domain, multiple paths)
- Implement host-based routing (multiple domains)
- Combine path and host routing
- Configure TLS/HTTPS termination
- Use annotations for advanced features
- Debug Ingress issues

In production, you would typically:

- Use real TLS certificates (Let's Encrypt, cert-manager)
- Configure proper DNS records
- Set resource limits on Ingress Controller
- Monitor Ingress metrics
- Implement rate limiting and security policies
