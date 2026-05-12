# Ingress Controllers

Duration: 45 minutes (20 min theory + 25 min lab)

## Introduction

In Part 1, we learned about Services (ClusterIP, NodePort, LoadBalancer). While these work well for basic scenarios, they have limitations:

- **NodePort**: Requires non-standard ports (30000-32767), one Service per port
- **LoadBalancer**: Creates one cloud load balancer per Service (expensive)
- Neither provides HTTP-level features (path routing, SSL termination, virtual hosts)

**Ingress** solves these problems by providing HTTP/HTTPS routing to multiple services through a single entry point.

## What is Ingress?

Ingress is a Kubernetes API object that manages external HTTP(S) access to services in a cluster. It provides:

- Path-based routing (`/api` → api-service, `/web` → web-service)
- Host-based routing (`api.example.com` → api-service, `web.example.com` → web-service)
- SSL/TLS termination
- Name-based virtual hosting
- Load balancing

**Important:** Ingress is just a configuration object. You need an **Ingress Controller** to actually implement the routing.

## Ingress vs Services

```text
Docker Compose (port mapping):
ports:
  - "8080:80"
  - "8081:3000"

Kubernetes Services (NodePort):
Service 1: NodePort 30080 → web:80
Service 2: NodePort 30081 → api:3000

Kubernetes Ingress (single entry point):
http://example.com/     → web:80
http://example.com/api  → api:3000
```

## Ingress Controllers

Popular Ingress Controllers:

- **NGINX Ingress Controller** - Most common, feature-rich
- **Traefik** - Modern, automatic HTTPS
- **HAProxy** - High performance
- **Istio Gateway** - Service mesh integration
- **Contour** - Envoy-based

We'll use **NGINX Ingress Controller** in this workshop.

## Ingress Resource Structure

Basic Ingress manifest:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
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

## Path Types

Kubernetes supports three `pathType` values:

1. **Prefix**: Matches URL path prefix
   - `/foo` matches `/foo`, `/foo/`, `/foo/bar`

2. **Exact**: Exact match only
   - `/foo` matches only `/foo`

3. **ImplementationSpecific**: Depends on Ingress Controller
   - NGINX uses regex matching

## Installing NGINX Ingress Controller

### For kind clusters:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### For cloud providers:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### Verify installation:

```bash
kubectl get pods -n ingress-nginx
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## Path-Based Routing

Route different URL paths to different services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000
```

**Request flow:**

- `http://cluster-ip/web` → web-service:80
- `http://cluster-ip/api` → api-service:3000

## Host-Based Routing

Route different hostnames to different services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000
```

**Request flow:**

- `http://web.example.com/` → web-service:80
- `http://api.example.com/` → api-service:3000

For local testing with kind, add to `/etc/hosts`:

```text
127.0.0.1 web.example.com api.example.com
```

## Default Backend

Specify a fallback service for unmatched requests:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

## TLS/HTTPS Configuration

Secure your Ingress with TLS certificates:

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: example-tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
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

### Generate self-signed certificate for testing:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=example.com"

kubectl create secret tls example-tls \
  --cert=tls.crt --key=tls.key
```

## Common Annotations

NGINX Ingress Controller supports many annotations:

### Rewrite Target

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
```

Rewrites `/api/users` → `/users` at backend

### Custom Timeouts

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
```

### Rate Limiting

```yaml
annotations:
  nginx.ingress.kubernetes.io/limit-rps: "10"
  nginx.ingress.kubernetes.io/limit-connections: "5"
```

### CORS Configuration

```yaml
annotations:
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "*"
  nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE"
```

### Auth Basic

```yaml
annotations:
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth
  nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
```

### Custom Headers

```yaml
annotations:
  nginx.ingress.kubernetes.io/configuration-snippet: |
    more_set_headers "X-Custom-Header: custom-value";
```

## Ingress Class

Multiple Ingress Controllers can coexist in the same cluster:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx  # Use this controller
  rules:
  - host: example.com
    # ...
```

## Debugging Ingress

### Check Ingress status:

```bash
kubectl get ingress
kubectl describe ingress <ingress-name>
```

### Check Ingress Controller logs:

```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

### Check backend service and endpoints:

```bash
kubectl get svc
kubectl get endpoints
```

### Test from inside cluster:

```bash
kubectl run test --image=curlimages/curl:8.5.0 -it --rm -- sh
curl http://web-service
curl -H "Host: example.com" http://ingress-nginx-controller.ingress-nginx
```

## Ingress vs LoadBalancer Comparison

### Using LoadBalancer Services:

```yaml
# 3 services = 3 load balancers (expensive!)
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer
  ports:
  - port: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
spec:
  type: LoadBalancer
  ports:
  - port: 8080
```

**Cost:** 3 load balancers
**Management:** 3 separate IP addresses
**Routing:** None (only L4 routing)

### Using Ingress:

```yaml
# 3 ClusterIP services + 1 Ingress
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  ports:
  - port: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
spec:
  type: ClusterIP
  ports:
  - port: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

**Cost:** 1 load balancer (Ingress Controller)
**Management:** 1 IP address, DNS-based routing
**Routing:** HTTP-level (L7) with path/host routing

## Best Practices

1. **Use IngressClass**: Always specify `ingressClassName` for clarity
2. **Configure TLS**: Use HTTPS in production (cert-manager can automate this)
3. **Set resource limits**: On Ingress Controller pods
4. **Health checks**: Configure proper liveness/readiness probes
5. **Rate limiting**: Protect backend services from overload
6. **Monitoring**: Monitor Ingress Controller metrics
7. **Default backend**: Provide friendly 404 page
8. **Annotations**: Use controller-specific features when needed
9. **Path strategy**: Choose between path-based and host-based routing
10. **Testing**: Test in dev with `/etc/hosts` before DNS changes

## Common Patterns

### Microservices API Gateway

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /orders(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
      - path: /products(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 8080
```

### Multi-Tenant Application

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-tenant
spec:
  ingressClassName: nginx
  rules:
  - host: tenant1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-tenant1
            port:
              number: 80
  - host: tenant2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-tenant2
            port:
              number: 80
```

## Next Steps

- Complete the hands-on lab in `lab/instructions.md`
- Explore other Ingress Controllers (Traefik, Contour)
- Learn about cert-manager for automated TLS certificates
- Study service mesh alternatives (Istio, Linkerd)

## Additional Resources

- [NGINX Ingress Controller Documentation](https://kubernetes.github.io/ingress-nginx/)
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress Annotations Reference](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)
