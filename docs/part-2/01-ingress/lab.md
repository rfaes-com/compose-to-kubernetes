# Ingress Lab

**Duration:** 25 minutes

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
$ kubectl get pods -n ingress-lab
NAME                        READY   STATUS    RESTARTS   AGE
frontend-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
frontend-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
backend-xxxxxxxxxx-xxxxx    1/1     Running   0          1m
backend-xxxxxxxxxx-xxxxx    1/1     Running   0          1m

$ kubectl get svc -n ingress-lab
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
frontend-service   ClusterIP   10.96.xxx.xxx   <none>        80/TCP    1m
backend-service    ClusterIP   10.96.xxx.xxx   <none>        80/TCP    1m
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
$ kubectl apply -f path-ingress.yaml
ingress.networking.k8s.io/path-ingress created

$ kubectl get ingress -n ingress-lab
NAME           CLASS   HOSTS   ADDRESS     PORTS   AGE
path-ingress   nginx   *       localhost   80      30s

$ curl http://localhost/frontend
<h1>Frontend Application</h1><p>Path: /frontend</p>

$ curl http://localhost/api
<h1>Backend API</h1><p>Path: /api</p>
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
$ kubectl apply -f host-based-apps.yaml
$ kubectl apply -f host-ingress.yaml

$ curl -H "Host: web.local" http://localhost
<h1>Web Application</h1><p>Host: web.local</p>

$ curl -H "Host: api.local" http://localhost
<h1>API Application</h1><p>Host: api.local</p>
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
$ kubectl apply -f combined-ingress.yaml

$ curl -H "Host: app.local" http://localhost/web
<h1>Web Application</h1><p>Host: web.local</p>

$ curl -H "Host: app.local" http://localhost/api
<h1>API Application</h1><p>Host: api.local</p>
```

The `rewrite-target: /$2` annotation removes the `/web` and `/api` prefix before forwarding to backends.

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=secure.local/O=workshop"
Generating a RSA private key
...
writing new private key to 'tls.key'

$ kubectl create secret tls secure-tls --cert=tls.crt --key=tls.key -n ingress-lab
secret/secure-tls created

$ kubectl apply -f tls-ingress.yaml
ingress.networking.k8s.io/tls-ingress created

$ curl -k -H "Host: secure.local" https://localhost
<h1>Web Application</h1><p>Host: web.local</p>

$ curl -v -H "Host: secure.local" http://localhost 2>&1 | grep -i location
< location: https://secure.local/
```

The HTTP request is automatically redirected to HTTPS (301) due to the `force-ssl-redirect` annotation.

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
$ kubectl apply -f annotated-ingress.yaml

$ for i in {1..15}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost/protected
  sleep 0.1
done
200
200
200
200
200
200
200
200
200
200
503
503
503
503
503
```

After 10 requests (the rate limit), NGINX returns 503 Service Temporarily Unavailable.

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-backend
  namespace: ingress-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default-backend
  template:
    metadata:
      labels:
        app: default-backend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          cat > /usr/share/nginx/html/index.html <<EOF
          <!DOCTYPE html>
          <html>
          <head><title>404 Not Found</title></head>
          <body>
            <h1>404 - Page Not Found</h1>
            <p>The requested resource was not found on this server.</p>
          </body>
          </html>
          EOF
          nginx -g 'daemon off;'
---
apiVersion: v1
kind: Service
metadata:
  name: default-backend-service
  namespace: ingress-lab
spec:
  selector:
    app: default-backend
  ports:
  - port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
  namespace: ingress-lab
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-backend-service
      port:
        number: 80
  rules:
  - host: app.local
    http:
      paths:
      - path: /exists
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

```bash
$ kubectl apply -f default-backend.yaml

$ curl -H "Host: app.local" http://localhost/nonexistent
<h1>404 - Page Not Found</h1>
<p>The requested resource was not found on this server.</p>

$ curl -H "Host: app.local" http://localhost/exists
<h1>Web Application</h1><p>Host: web.local</p>
```

</details>

### Challenge 2: Path Type Comparison

Create three Ingresses with different pathType values (Prefix, Exact, ImplementationSpecific) and test the differences.

<details class="solution" markdown="1">
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pathtype-comparison
  namespace: ingress-lab
spec:
  ingressClassName: nginx
  rules:
  - host: pathtype.local
    http:
      paths:
      # Prefix: Matches /prefix, /prefix/, /prefix/anything
      - path: /prefix
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80

      # Exact: Only matches /exact (no trailing slash)
      - path: /exact
        pathType: Exact
        backend:
          service:
            name: api-backend-service
            port:
              number: 80
```

```bash
$ kubectl apply -f pathtype-comparison.yaml

# Prefix tests
$ curl -H "Host: pathtype.local" http://localhost/prefix
<h1>Web Application</h1>

$ curl -H "Host: pathtype.local" http://localhost/prefix/
<h1>Web Application</h1>

$ curl -H "Host: pathtype.local" http://localhost/prefix/sub/path
<h1>Web Application</h1>

# Exact tests
$ curl -H "Host: pathtype.local" http://localhost/exact
<h1>API Application</h1>

$ curl -H "Host: pathtype.local" http://localhost/exact/
<html><head><title>404 Not Found</title></head>...

$ curl -H "Host: pathtype.local" http://localhost/exact/anything
<html><head><title>404 Not Found</title></head>...
```

**Summary:**

- **Prefix**: Matches the path and any sub-paths
- **Exact**: Only matches the exact path, no trailing slash or sub-paths

</details>

### Challenge 3: Basic Authentication

Implement HTTP Basic Auth on one of your Ingress paths.

**Hint:**

```bash
htpasswd -c auth myuser
kubectl create secret generic basic-auth --from-file=auth
```

Use annotation: `nginx.ingress.kubernetes.io/auth-type: basic`

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Install htpasswd (if not available)
# On Fedora: dnf install httpd-tools
# On Ubuntu: apt-get install apache2-utils

# Create auth file with user 'admin' and password 'secret'
$ htpasswd -c auth admin
New password: secret
Re-type new password: secret
Adding password for user admin

# Create secret
$ kubectl create secret generic basic-auth --from-file=auth -n ingress-lab
secret/basic-auth created
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-ingress
  namespace: ingress-lab
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required - Workshop"
spec:
  ingressClassName: nginx
  rules:
  - host: auth.local
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

```bash
$ kubectl apply -f auth-ingress.yaml

# Without credentials - returns 401
$ curl -H "Host: auth.local" http://localhost
<html>
<head><title>401 Authorization Required</title></head>
...
</html>

# With correct credentials - works
$ curl -u admin:secret -H "Host: auth.local" http://localhost
<h1>Web Application</h1><p>Host: web.local</p>

# With incorrect credentials - returns 401
$ curl -u admin:wrongpass -H "Host: auth.local" http://localhost
<html>
<head><title>401 Authorization Required</title></head>
...
</html>
```

</details>

### Challenge 4: Canary Deployments

Use Ingress annotations to implement a canary deployment that routes 10% of traffic to a new version.

**Hint:** Look up `nginx.ingress.kubernetes.io/canary` annotations.

<details class="solution" markdown="1">
<summary>Solution</summary>

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-v2
  namespace: ingress-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      version: v2
  template:
    metadata:
      labels:
        app: web
        version: v2
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo '<h1>Web Application V2</h1><p>New version with new features!</p>' > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
---
apiVersion: v1
kind: Service
metadata:
  name: web-service-v2
  namespace: ingress-lab
spec:
  selector:
    app: web
    version: v2
  ports:
  - port: 80
---
# Main Ingress (90% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-main
  namespace: ingress-lab
spec:
  ingressClassName: nginx
  rules:
  - host: canary.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
---
# Canary Ingress (10% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-canary
  namespace: ingress-lab
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  rules:
  - host: canary.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service-v2
            port:
              number: 80
```

```bash
$ kubectl apply -f web-v2.yaml
$ kubectl apply -f canary-ingress.yaml

# Make 20 requests and count responses
$ for i in {1..20}; do
  curl -s -H "Host: canary.local" http://localhost | grep -o "V2" || echo "V1"
done

# Expected output: ~18 "V1" and ~2 "V2" (approximately 10% to v2)
```

You can also use header-based canary:

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
  nginx.ingress.kubernetes.io/canary-by-header-value: "true"
```

```bash
# Normal users get v1
$ curl -H "Host: canary.local" http://localhost
<h1>Web Application</h1>

# Users with header get v2
$ curl -H "Host: canary.local" -H "X-Canary: true" http://localhost
<h1>Web Application V2</h1>
```

</details>

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

## Key takeaways

1. **Ingress controllers** act as the single entry point for external traffic into the cluster
2. **Path-based routing** serves multiple services from the same domain via URL paths
3. **Host-based routing** routes traffic based on the HTTP Host header to different backend services
4. **TLS termination** offloads HTTPS handling to the Ingress controller, keeping backend services simple
5. **Annotations** customize Ingress controller behavior without modifying the controller itself
6. **Ingress resources** require a running controller — the resource alone does nothing without one

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../02-helm/README.md).
