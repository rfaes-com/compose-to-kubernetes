# Docker Compose to Kubernetes Mapping

Side-by-side comparison of Docker Compose and Kubernetes concepts.

## Core Concepts

| Docker Compose       | Kubernetes                     | Notes                                                 |
| -------------------- | ------------------------------ | ----------------------------------------------------- |
| `docker-compose.yml` | Multiple YAML manifests        | K8s typically uses multiple files or `---` separators |
| Service              | Deployment + Service           | Deployment manages pods, Service exposes them         |
| Container            | Container (in Pod)             | K8s adds Pod abstraction layer                        |
| Volume               | PersistentVolumeClaim + Volume | K8s separates storage claim from usage                |
| Network              | Service + DNS                  | K8s uses Services and built-in DNS                    |
| Environment          | ConfigMap + Secret             | K8s distinguishes config from secrets                 |

## Service Definition

### Docker Compose

```yaml
services:
  web:
    image: nginx:1.25-alpine
    container_name: my-nginx
    ports:
      - "8080:80"
    environment:
      - ENV=production
      - DEBUG=false
    volumes:
      - ./html:/usr/share/nginx/html
      - web-data:/var/log/nginx
    restart: always
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

### Kubernetes

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
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
        env:
        - name: ENV
          value: production
        - name: DEBUG
          value: "false"
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        - name: web-data
          mountPath: /var/log/nginx
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
      volumes:
      - name: html
        hostPath:
          path: /path/to/html
      - name: web-data
        persistentVolumeClaim:
          claimName: web-data-pvc
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 8080
    targetPort: 80
  type: LoadBalancer
```

## Ports Mapping

| Docker Compose     | Kubernetes                           | Behavior                 |
| ------------------ | ------------------------------------ | ------------------------ |
| `ports: "8080:80"` | Service `port: 8080, targetPort: 80` | External → Internal      |
| `expose: 80`       | Service `port: 80` (ClusterIP)       | Internal only            |
| `N/A`              | Service `type: NodePort`             | Accessible on node IP    |
| Host network       | `hostNetwork: true`                  | Share host network stack |

### Examples

**Docker Compose:**

```yaml
services:
  api:
    image: myapi:latest
    ports:
      - "3000:3000"      # Published port
    expose:
      - "9090"           # Internal only
```

**Kubernetes:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
  - name: http
    port: 3000
    targetPort: 3000
  - name: metrics
    port: 9090
    targetPort: 9090
  type: ClusterIP        # Internal by default
  # Or type: LoadBalancer for external access
```

## Environment Variables

### Docker Compose

```yaml
services:
  app:
    environment:
      - DATABASE_URL=postgres://db:5432/mydb
      - SECRET_KEY=mysecret
    env_file:
      - .env
      - .env.production
```

### Kubernetes

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgres://db:5432/mydb"
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
stringData:
  SECRET_KEY: mysecret
---
# Pod/Deployment
spec:
  containers:
  - name: app
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_URL
    - name: SECRET_KEY
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: SECRET_KEY
    # Or import all keys
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secret
```

## Volumes

### Docker Compose

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - db-data:/var/lib/postgresql/data      # Named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Bind mount
      - type: tmpfs                            # tmpfs
        target: /tmp

volumes:
  db-data:
    driver: local
```

### Kubernetes

```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
# ConfigMap for init script
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-init
data:
  init.sql: |
    CREATE TABLE users (...);
---
# Pod/Deployment
spec:
  containers:
  - name: postgres
    volumeMounts:
    - name: db-data
      mountPath: /var/lib/postgresql/data
    - name: init-script
      mountPath: /docker-entrypoint-initdb.d
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: db-data
    persistentVolumeClaim:
      claimName: db-data
  - name: init-script
    configMap:
      name: db-init
  - name: tmp
    emptyDir: {}
```

## Volume Types Comparison

| Docker Compose | Kubernetes            | Use Case            |
| -------------- | --------------------- | ------------------- |
| Named volume   | PersistentVolumeClaim | Persistent data     |
| Bind mount     | hostPath              | Local development   |
| tmpfs          | emptyDir              | Temporary data      |
| N/A            | emptyDir + Memory     | RAM disk            |
| N/A            | configMap             | Configuration files |
| N/A            | secret                | Sensitive files     |

## Networks

### Docker Compose

```yaml
services:
  frontend:
    networks:
      - frontend-net

  backend:
    networks:
      - frontend-net
      - backend-net

  database:
    networks:
      - backend-net

networks:
  frontend-net:
  backend-net:
    internal: true
```

### Kubernetes

```yaml
# In Kubernetes, pods in the same namespace can communicate by default
# Use NetworkPolicies for restrictions

# Allow frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
---
# Isolate database (only from backend)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
```

## Service Discovery

### Docker Compose

```yaml
services:
  frontend:
    image: frontend:latest
    environment:
      - API_URL=http://backend:3000  # Use service name

  backend:
    image: backend:latest
```

### Kubernetes

```yaml
# Pods use Service DNS names
# Format: <service-name>.<namespace>.svc.cluster.local

spec:
  containers:
  - name: frontend
    env:
    - name: API_URL
      # Short form (same namespace)
      value: "http://backend:3000"
      # Or full FQDN
      # value: "http://backend.default.svc.cluster.local:3000"
```

## Health Checks

### Docker Compose

```yaml
services:
  api:
    image: api:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Kubernetes

```yaml
spec:
  containers:
  - name: api
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 40
      periodSeconds: 30
      timeoutSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /health
        port: 3000
      failureThreshold: 30
      periodSeconds: 10
```

## Restart Policies

| Docker Compose            | Kubernetes                 | Behavior                 |
| ------------------------- | -------------------------- | ------------------------ |
| `restart: "no"`           | `restartPolicy: Never`     | Never restart            |
| `restart: always`         | `restartPolicy: Always`    | Always restart           |
| `restart: on-failure`     | `restartPolicy: OnFailure` | Restart on non-zero exit |
| `restart: unless-stopped` | N/A                        | Not applicable to K8s    |

## Resource Limits

### Docker Compose

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

### Kubernetes

```yaml
spec:
  containers:
  - name: app
    resources:
      limits:
        cpu: 500m        # 0.5 CPU
        memory: 512Mi
      requests:
        cpu: 250m        # 0.25 CPU
        memory: 256Mi
```

## Dependencies

### Docker Compose

```yaml
services:
  web:
    depends_on:
      db:
        condition: service_healthy

  db:
    healthcheck:
      test: ["CMD", "pg_isready"]
```

### Kubernetes

```yaml
# Use Init Containers to wait for dependencies
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command:
    - 'sh'
    - '-c'
    - |
      until nc -z db 5432; do
        echo "Waiting for database..."
        sleep 2
      done
  containers:
  - name: web
    image: web:latest
```

## Scaling

| Docker Compose                    | Kubernetes                                  | Command      |
| --------------------------------- | ------------------------------------------- | ------------ |
| `docker-compose up --scale web=3` | `kubectl scale deployment web --replicas=3` | Imperative   |
| `deploy.replicas: 3`              | `spec.replicas: 3`                          | Declarative  |
| N/A                               | HorizontalPodAutoscaler                     | Auto-scaling |

### Auto-scaling in Kubernetes

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

## Update Strategy

### Docker Compose

```yaml
services:
  app:
    deploy:
      update_config:
        parallelism: 2
        delay: 10s
        order: start-first
```

### Kubernetes

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 1 extra pod during update
      maxUnavailable: 1     # 1 pod can be down
  minReadySeconds: 10       # Wait before considering ready
```

## Labels and Annotations

### Docker Compose

```yaml
services:
  app:
    labels:
      com.example.description: "Web application"
      com.example.team: "platform"
```

### Kubernetes

```yaml
metadata:
  labels:
    app: web
    version: v1
    tier: frontend
  annotations:
    description: "Web application"
    team: "platform"
```

## Complete Example Comparison

### Docker Compose (3-Tier App)

```yaml
version: '3.8'

services:
  frontend:
    image: nginx:1.25-alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - backend
    networks:
      - frontend-net

  backend:
    image: node:20-alpine
    environment:
      - DATABASE_URL=postgres://postgres:secret@database:5432/mydb
      - NODE_ENV=production
    depends_on:
      database:
        condition: service_healthy
    networks:
      - frontend-net
      - backend-net
    deploy:
      replicas: 2

  database:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=mydb
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 10s
    networks:
      - backend-net

volumes:
  db-data:

networks:
  frontend-net:
  backend-net:
```

### Kubernetes (3-Tier App)

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
# database-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: myapp
stringData:
  password: secret
---
# database-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
  namespace: myapp
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
# database-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: myapp
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
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: POSTGRES_DB
          value: mydb
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command: ["pg_isready"]
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["pg_isready"]
          periodSeconds: 5
      volumes:
      - name: db-data
        persistentVolumeClaim:
          claimName: db-data
---
# database-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: myapp
spec:
  selector:
    app: database
  ports:
  - port: 5432
  clusterIP: None  # Headless service
---
# backend-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: myapp
data:
  NODE_ENV: production
  DATABASE_HOST: database
  DATABASE_PORT: "5432"
  DATABASE_NAME: mydb
---
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myapp
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
      initContainers:
      - name: wait-for-db
        image: busybox:1.36
        command:
        - sh
        - -c
        - until nc -z database 5432; do sleep 2; done
      containers:
      - name: node
        image: node:20-alpine
        envFrom:
        - configMapRef:
            name: backend-config
        env:
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        ports:
        - containerPort: 3000
---
# backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: myapp
spec:
  selector:
    app: backend
  ports:
  - port: 3000
---
# frontend-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-html
  namespace: myapp
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body><h1>Welcome</h1></body>
    </html>
---
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: myapp
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
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: frontend-html
---
# frontend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: myapp
spec:
  selector:
    app: frontend
  ports:
  - port: 8080
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

## Key Differences Summary

1. **File Structure**: Compose uses one file; K8s uses multiple manifests
2. **Service Discovery**: Compose uses service names; K8s uses Service DNS
3. **Networking**: Compose uses custom networks; K8s uses Services + NetworkPolicies
4. **Storage**: Compose has simple volumes; K8s has PV/PVC abstraction
5. **Configuration**: Compose uses env vars; K8s separates ConfigMaps/Secrets
6. **Scaling**: Compose has basic scaling; K8s has advanced auto-scaling
7. **Health Checks**: Similar concept, different configuration
8. **Updates**: Compose has basic options; K8s has sophisticated strategies
9. **Dependencies**: Compose has `depends_on`; K8s uses init containers
10. **State Management**: Compose restarts containers; K8s uses StatefulSets

## Migration Checklist

- [ ] Convert service definitions to Deployments
- [ ] Create Services for network exposure
- [ ] Move environment variables to ConfigMaps
- [ ] Move secrets to Secret resources
- [ ] Convert volumes to PVCs
- [ ] Add health checks (liveness/readiness probes)
- [ ] Configure resource limits
- [ ] Set up service discovery (Service DNS)
- [ ] Implement network policies if needed
- [ ] Add labels for organization
- [ ] Configure update strategies
- [ ] Test in development cluster
- [ ] Document any compose-specific features that don't translate
