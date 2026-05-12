# Ingress Lab Solutions

Complete solutions for all tasks and bonus challenges.

## Task 1: Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

**Verification:**

```bash
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

---

## Task 2: Deploy Backend Applications

All manifests applied successfully. Verify with:

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

---

## Task 3: Create Path-Based Ingress

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

---

## Task 4: Create Host-Based Ingress

```bash
$ kubectl apply -f host-based-apps.yaml
$ kubectl apply -f host-ingress.yaml

$ curl -H "Host: web.local" http://localhost
<h1>Web Application</h1><p>Host: web.local</p>

$ curl -H "Host: api.local" http://localhost
<h1>API Application</h1><p>Host: api.local</p>
```

---

## Task 5: Combine Path and Host Routing

```bash
$ kubectl apply -f combined-ingress.yaml

$ curl -H "Host: app.local" http://localhost/web
<h1>Web Application</h1><p>Host: web.local</p>

$ curl -H "Host: app.local" http://localhost/api
<h1>API Application</h1><p>Host: api.local</p>
```

The `rewrite-target: /$2` annotation removes the `/web` and `/api` prefix before forwarding to backends.

---

## Task 6: Add TLS/HTTPS Support

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

---

## Task 7: Add Custom Annotations

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

---

## Bonus Challenge 1: Default Backend

Create a custom 404 page:

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

Test:

```bash
$ kubectl apply -f default-backend.yaml

$ curl -H "Host: app.local" http://localhost/nonexistent
<h1>404 - Page Not Found</h1>
<p>The requested resource was not found on this server.</p>

$ curl -H "Host: app.local" http://localhost/exists
<h1>Web Application</h1><p>Host: web.local</p>
```

---

## Bonus Challenge 2: Path Type Comparison

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

Test:

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

---

## Bonus Challenge 3: Basic Authentication

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

$ kubectl get secret basic-auth -n ingress-lab -o yaml
# You'll see the auth file encoded in base64
```

Create Ingress with authentication:

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

Test:

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

---

## Bonus Challenge 4: Canary Deployments

Create a v2 deployment:

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
```

Create canary Ingress (10% traffic to v2):

```yaml
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

Test:

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

Test:

```bash
# Normal users get v1
$ curl -H "Host: canary.local" http://localhost
<h1>Web Application</h1>

# Users with header get v2
$ curl -H "Host: canary.local" -H "X-Canary: true" http://localhost
<h1>Web Application V2</h1>
```

---

## Summary

All tasks and bonus challenges completed successfully! You've mastered:

- Installing and configuring NGINX Ingress Controller
- Path-based routing for multiple services
- Host-based routing for multiple domains
- Combined path and host routing
- TLS/HTTPS termination with self-signed certificates
- Custom annotations (rate limiting, CORS, timeouts)
- Default backends for custom 404 pages
- PathType differences (Prefix vs Exact)
- HTTP Basic Authentication
- Canary deployments with traffic splitting

## Key Takeaways

1. **Single Entry Point**: Ingress provides one external endpoint for multiple services
2. **Cost Effective**: One LoadBalancer instead of many
3. **HTTP Features**: Path/host routing, TLS termination, rewrites
4. **Annotations**: Controller-specific features through annotations
5. **Production Ready**: Use with cert-manager for automated TLS certificates
