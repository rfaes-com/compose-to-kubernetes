# Final Lab

**Duration:** 45-60 minutes

## Overview

This comprehensive lab ties together everything from Part 1. You'll deploy a complete 3-tier application (frontend, backend, database) to Kubernetes, converting it from a Docker Compose configuration.

## Learning Objectives

- Convert a complete Docker Compose application to Kubernetes
- Apply all concepts learned: Pods, Deployments, Services, ConfigMaps, Secrets, Storage
- Implement health checks and resource limits
- Set up networking between components
- Verify end-to-end functionality
- Troubleshoot issues

## Application Architecture

```mermaid
flowchart TB
    Frontend["Frontend<br/>(nginx)<br/>nginx serving static files<br/>Port: 80"]

    Backend["Backend<br/>(nginx)<br/>API server<br/>Port: 8080"]

    Database["Database<br/>(postgres)<br/>PostgreSQL<br/>Port: 5432"]

    Frontend --> Backend
    Backend --> Database
```

## Starting Point: Docker Compose

Here's the application in Docker Compose:

```yaml
services:
  frontend:
    image: nginx:1.25-alpine
    ports:
      - "8080:80"
    volumes:
      - ./frontend/html:/usr/share/nginx/html:ro
    environment:
      - BACKEND_URL=http://backend:8080
    depends_on:
      - backend
    networks:
      - app-network

  backend:
    image: nginx:1.25-alpine
    ports:
      - "8081:80"
    volumes:
      - ./backend/config:/etc/nginx/conf.d:ro
    environment:
      - DB_HOST=database
      - DB_PORT=5432
      - DB_NAME=appdb
      - DB_USER=appuser
    depends_on:
      - database
    networks:
      - app-network

  database:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=secret123
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-network

networks:
  app-network:

volumes:
  db-data:
```

## Your Mission

Convert this Docker Compose application to Kubernetes manifests and deploy it.

## Requirements

### 1. Namespace

- Create namespace: `myapp`
- All resources should be in this namespace

### 2. Database (PostgreSQL)

- Use a StatefulSet (persistent identity)
- 1 replica
- Persistent storage: 1Gi PVC
- Create Secret for database credentials
- Expose via ClusterIP Service on port 5432
- Resource limits: 256Mi memory, 250m CPU
- Health checks: PostgreSQL readiness

### 3. Backend API

- Use a Deployment
- 2 replicas
- ConfigMap for non-sensitive configuration
- Secret for database password
- Environment variables for database connection
- Expose via ClusterIP Service on port 8080
- Resource limits: 128Mi memory, 100m CPU
- Liveness and readiness probes

### 4. Frontend

- Use a Deployment
- 3 replicas
- ConfigMap for nginx configuration
- Expose via NodePort Service on port 30080
- Resource limits: 64Mi memory, 50m CPU
- Health checks on port 80

### 5. Best Practices

- Use appropriate labels
- Include annotations for documentation
- Use specific image tags
- Run as non-root where possible
- Implement proper health checks
- Set resource requests and limits

## Lab Tasks

### Task 1: Create Namespace

Create a namespace for the application with appropriate labels.

### Task 2: Create Database Secret

Create a Secret containing database credentials.

### Task 3: Create Database PVC

Create a PersistentVolumeClaim for database storage.

### Task 4: Deploy PostgreSQL

Create a StatefulSet for PostgreSQL with:

- PVC mounted at `/var/lib/postgresql/data`
- Environment variables from Secret
- Resource limits
- Readiness probe

### Task 5: Create Database Service

Expose PostgreSQL with a headless Service.

### Task 6: Create Backend ConfigMap

Create a ConfigMap with backend configuration.

### Task 7: Deploy Backend

Create a Deployment for the backend with:

- Environment variables from ConfigMap and Secret
- Resource limits
- Health probes

### Task 8: Create Backend Service

Expose the backend via ClusterIP Service.

### Task 9: Create Frontend ConfigMap

Create a ConfigMap for frontend configuration (nginx config).

### Task 10: Deploy Frontend

Create a Deployment for the frontend with:

- ConfigMap volume mount
- Resource limits
- Health checks

### Task 11: Create Frontend Service

Expose the frontend via NodePort Service on port 30080.

### Task 12: Verify End-to-End

Test the complete application:

- Access frontend from browser
- Verify frontend can reach backend
- Verify backend can reach database
- Check logs from all components

### Task 13: Scale and Update

- Scale backend to 4 replicas
- Update frontend image version
- Verify zero-downtime rolling update

## Hints

<details markdown="1">
<summary>Hint for Task 1: Create Namespace</summary>

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    name: myapp
    environment: learning
EOF
```

</details>

<details markdown="1">
<summary>Hint for Task 2: Create Database Secret</summary>

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=appuser \
  --from-literal=password=secret123 \
  --from-literal=database=appdb \
  -n myapp
```

</details>

<details markdown="1">
<summary>Hint for Task 3: Create Database PVC</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: myapp
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path  # or "standard" depending on your cluster
```

</details>

<details markdown="1">
<summary>Hint for Task 4: Deploy PostgreSQL StatefulSet</summary>

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: myapp
  labels:
    app: database
    tier: data
spec:
  serviceName: database
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
        tier: data
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: database
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

</details>

<details markdown="1">
<summary>Hint for Task 5: Create Database Service</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: myapp
  labels:
    app: database
spec:
  clusterIP: None  # Headless service
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
    name: postgres
```

</details>

<details markdown="1">
<summary>Hint for Task 6: Create Backend ConfigMap</summary>

```bash
kubectl create configmap backend-config \
  --from-literal=db.host=database.myapp.svc.cluster.local \
  --from-literal=db.port=5432 \
  --from-literal=log.level=info \
  -n myapp
```

</details>

<details markdown="1">
<summary>Hint for Task 7: Deploy Backend</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myapp
  labels:
    app: backend
    tier: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      containers:
      - name: backend
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
          name: http
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: db.host
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: db.port
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: database
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

</details>

<details markdown="1">
<summary>Hint for Task 8: Create Backend Service</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: myapp
  labels:
    app: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 80
    name: http
```

</details>

<details markdown="1">
<summary>Hint for Task 9: Create Frontend ConfigMap</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: myapp
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>My Application</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 50px; }
            .container { max-width: 800px; margin: 0 auto; }
            h1 { color: #333; }
            .status { padding: 20px; background: #f0f0f0; border-radius: 5px; }
            button { padding: 10px 20px; font-size: 16px; cursor: pointer; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Welcome to My Kubernetes Application!</h1>
            <div class="status">
                <h2>Application Status</h2>
                <p>Frontend: Running on Kubernetes</p>
                <p>Backend API: <span id="backend-status">Checking...</span></p>
                <button onclick="checkBackend()">Test Backend Connection</button>
            </div>
            <h2>Architecture</h2>
            <ul>
                <li>Frontend: nginx (3 replicas)</li>
                <li>Backend: nginx (2 replicas)</li>
                <li>Database: PostgreSQL (StatefulSet)</li>
            </ul>
        </div>
        <script>
            function checkBackend() {
                document.getElementById('backend-status').textContent = 'Connected!';
            }
            checkBackend();
        </script>
    </body>
    </html>
```

</details>

<details markdown="1">
<summary>Hint for Task 10: Deploy Frontend</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: myapp
  labels:
    app: frontend
    tier: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: frontend
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
          name: http
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
        resources:
          requests:
            memory: "32Mi"
            cpu: "25m"
          limits:
            memory: "64Mi"
            cpu: "50m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: html
        configMap:
          name: frontend-config
          items:
          - key: index.html
            path: index.html
```

</details>

<details markdown="1">
<summary>Hint for Task 11: Create Frontend Service</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: myapp
  labels:
    app: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    name: http
```

</details>

## Verification Steps

```bash
# 1. Check all resources are created
kubectl get all -n myapp

# 2. Check PVC is bound
kubectl get pvc -n myapp

# 3. Check pods are running
kubectl get pods -n myapp

# 4. Check services
kubectl get svc -n myapp

# 5. Check database is ready
kubectl exec -n myapp statefulset/database -- psql -U appuser -d appdb -c '\l'

# 6. Check backend can see database
kubectl exec -n myapp deployment/backend -- printenv | grep DB_

# 7. Access frontend
curl http://localhost:30080

# or in browser: http://localhost:30080

# 8. Check logs
kubectl logs -n myapp -l app=frontend --tail=20
kubectl logs -n myapp -l app=backend --tail=20
kubectl logs -n myapp -l app=database --tail=20
```

## Testing and Validation

### 1. Scale Backend

```bash
kubectl scale deployment backend -n myapp --replicas=4
kubectl get pods -n myapp -l app=backend
```

### 2. Rolling Update

```bash
# Update frontend image
kubectl set image deployment/frontend frontend=nginx:1.25.3-alpine -n myapp

# Watch the rollout
kubectl rollout status deployment/frontend -n myapp

# Check rollout history
kubectl rollout history deployment/frontend -n myapp
```

### 3. Test Database Persistence

```bash
# Write data to database
kubectl exec -n myapp statefulset/database -- psql -U appuser -d appdb -c "CREATE TABLE test (id serial PRIMARY KEY, name VARCHAR(50));"
kubectl exec -n myapp statefulset/database -- psql -U appuser -d appdb -c "INSERT INTO test (name) VALUES ('Kubernetes');"

# Delete and recreate StatefulSet
kubectl delete pod database-0 -n myapp

# Wait for pod to restart
kubectl wait --for=condition=Ready pod/database-0 -n myapp --timeout=60s

# Verify data persisted
kubectl exec -n myapp statefulset/database -- psql -U appuser -d appdb -c "SELECT * FROM test;"
```

## Troubleshooting

If something doesn't work:

```bash
# Check pod status
kubectl get pods -n myapp

# Describe pod for events
kubectl describe pod <pod-name> -n myapp

# Check logs
kubectl logs <pod-name> -n myapp

# Check service endpoints
kubectl get endpoints -n myapp

# Test DNS resolution
kubectl run test --image=busybox -n myapp --rm -it -- nslookup database.myapp.svc.cluster.local

# Test connectivity
kubectl run test --image=curlimages/curl -n myapp --rm -it -- curl http://backend:8080
```

## Cleanup

```bash
# Delete everything
kubectl delete namespace myapp

# Verify deletion
kubectl get all -n myapp
```

## Success Criteria

You've successfully completed this lab when:

- [ ] All pods are running (frontend: 3, backend: 2, database: 1)
- [ ] Frontend is accessible at `http://localhost:30080`
- [ ] Backend can connect to database
- [ ] Data persists in PostgreSQL after pod restart
- [ ] You can scale deployments up and down
- [ ] Rolling updates work without downtime
- [ ] All health checks are passing

## Bonus Challenges

1. **Add resource quotas** to the namespace
2. **Implement HorizontalPodAutoscaler** for frontend and backend
3. **Add Ingress** instead of NodePort
4. **Create a Redis cache** layer between backend and database
5. **Add monitoring** with Prometheus annotations
6. **Implement NetworkPolicies** to restrict traffic

## Key Takeaways

- Docker Compose services map to Kubernetes Deployments + Services
- Volumes in Compose become PVCs in Kubernetes
- Environment variables use ConfigMaps and Secrets
- Services provide DNS-based discovery
- StatefulSets provide persistent identity for databases
- Health checks ensure traffic only goes to ready pods
- Resource limits prevent noisy neighbor problems
- Labels enable powerful selection and organization

## Congratulations

You've completed Part 1 of the workshop! You've learned:

- Kubernetes architecture and concepts
- Pods, Deployments, Services, and StatefulSets
- ConfigMaps, Secrets, and persistent storage
- Namespaces and resource management
- kubectl and k9s for cluster interaction
- Best practices for manifest organization

Ready for Part 2? Continue to advanced topics including:

- Ingress and load balancing
- Helm package management
- GitOps with Flux
- Monitoring and logging
- Advanced deployment strategies

## Next Steps

1. Review any concepts that were challenging
2. Experiment with the deployed application
3. Try the bonus challenges
4. Take a break before Part 2!
