# Manifest Organization and Best Practices

**Duration:** 25 minutes

## Learning Objectives

- Organize Kubernetes manifests effectively
- Follow YAML best practices
- Use labels and annotations properly
- Implement naming conventions
- Understand resource management patterns
- Learn configuration strategies

## Manifest File Organization

### Single vs Multiple Files

**Single file with multiple resources:**

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: myapp
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: myapp
```

**Separate files:**

```text
manifests/
├── namespace.yaml
├── deployment.yaml
└── service.yaml
```

**Best practice:** Use separate files for different resource types, but group related resources together.

### Directory Structure Patterns

#### Pattern 1: By Environment

```text
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── dev/
│   ├── kustomization.yaml
│   └── patches/
├── staging/
│   ├── kustomization.yaml
│   └── patches/
└── prod/
    ├── kustomization.yaml
    └── patches/
```

#### Pattern 2: By Component

```text
k8s/
├── frontend/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── backend/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── database/
    ├── statefulset.yaml
    ├── service.yaml
    └── pvc.yaml
```

#### Pattern 3: By Resource Type

```text
k8s/
├── namespaces/
├── deployments/
├── services/
├── configmaps/
├── secrets/
└── ingresses/
```

#### Pattern 4: Hybrid (Recommended)

```text
k8s/
├── base/                      # Common configs
│   ├── namespace.yaml
│   └── common-configmap.yaml
├── apps/                      # Applications
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   └── backend/
│       ├── deployment.yaml
│       └── service.yaml
├── infrastructure/            # Infrastructure components
│   ├── monitoring/
│   └── logging/
└── environments/              # Environment-specific
    ├── dev/
    ├── staging/
    └── production/
```

## Naming Conventions

### Resource Names

```yaml
# Good: descriptive, consistent, lowercase with hyphens
metadata:
  name: web-frontend-deployment
  name: api-backend-service
  name: postgres-primary-statefulset

# Avoid: unclear, inconsistent
metadata:
  name: deployment1
  name: webApp
  name: my_service
```

**Rules:**

- Use lowercase letters, numbers, hyphens
- No underscores or special characters
- Be descriptive but concise
- Include component and purpose
- Maximum 253 characters

### Namespace Naming

```yaml
# Environment-based
metadata:
  namespace: production
  namespace: staging
  namespace: development

# Team-based
metadata:
  namespace: team-platform
  namespace: team-data

# Application-based
metadata:
  namespace: ecommerce-frontend
  namespace: ecommerce-backend
```

## Labels and Annotations

### Labels for Organization

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  labels:
    # Recommended labels
    app.kubernetes.io/name: frontend
    app.kubernetes.io/instance: web-frontend-prod
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/managed-by: helm

    # Custom labels
    team: platform
    environment: production
    cost-center: engineering
```

**Recommended Kubernetes labels:**

- `app.kubernetes.io/name`: Application name
- `app.kubernetes.io/instance`: Unique instance identifier
- `app.kubernetes.io/version`: Application version
- `app.kubernetes.io/component`: Component within the application
- `app.kubernetes.io/part-of`: Higher-level application
- `app.kubernetes.io/managed-by`: Tool managing the resource

### Annotations for Metadata

```yaml
metadata:
  annotations:
    # Documentation
    description: "Frontend web server for ecommerce application"
    documentation: "https://wiki.company.com/ecommerce-frontend"

    # Ownership
    owner: "platform-team@company.com"
    slack-channel: "#team-platform"

    # Build information
    git-commit: "a1b2c3d4"
    build-date: "2024-01-15"
    ci-pipeline: "https://ci.company.com/build/12345"

    # Tool-specific
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

**Use labels for:**

- Selection and filtering
- Resource grouping
- Queries

**Use annotations for:**

- Documentation
- Tool configuration
- Non-identifying metadata

## Resource Management

### Always Specify Resources

```yaml
# Good: Explicit resource requests and limits
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"

# Bad: No resources specified
spec:
  containers:
  - name: app
    image: myapp:1.0
    # No resources - pod can consume all node resources!
```

**Guidelines:**

- Always set `requests` (for scheduling)
- Set `limits` to prevent resource exhaustion
- Use [`m` for millicores](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) (1000m = 1 CPU)
- Use `Mi`/`Gi` for memory (powers of 1024)

### Resource Patterns

```yaml
# Small application
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"

# Medium application
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

# Large application
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

## Configuration Strategies

### 1. ConfigMaps for Configuration

```yaml
# Separate config from code
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.host: "postgres.default.svc.cluster.local"
  database.port: "5432"
  log.level: "info"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        envFrom:
        - configMapRef:
            name: app-config
```

### 2. Secrets for Sensitive Data

```yaml
# Never commit secrets to git!
# Use sealed-secrets, external-secrets, or vault
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:  # Automatically base64 encoded
  username: admin
  password: changeme  # Use secret management tools!
```

### 3. Environment-Specific Values

Use Kustomize overlays or Helm values:

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1  # Will be overridden

# overlays/production/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 5  # Production uses 5 replicas
```

## Health Checks

### Always Define Probes

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080

    # Liveness: Is the app alive?
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3

    # Readiness: Is the app ready for traffic?
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3

    # Startup: For slow-starting apps
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 30  # 30 * 5 = 150 seconds max startup time
```

## Security Best Practices

### Run as Non-Root

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

### Use Specific Image Tags

```yaml
# Good: Specific version
image: nginx:1.25.3-alpine

# Avoid: Latest tag (unpredictable)
image: nginx:latest

# Avoid: No tag (defaults to latest)
image: nginx
```

### Resource Limits

```yaml
# Prevent resource exhaustion
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
    ephemeral-storage: "1Gi"
```

## Deployment Strategies

### Rolling Update (Recommended)

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max 1 extra pod during update
      maxUnavailable: 1  # Max 1 pod down during update
```

### Recreate (Downtime)

```yaml
spec:
  strategy:
    type: Recreate  # All pods terminated before new ones created
```

## Documentation in Manifests

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  labels:
    app: frontend
  annotations:
    # Purpose and description
    description: "Frontend web application serving user traffic"

    # Dependencies
    dependencies: "backend-api, postgres, redis"

    # Runbook
    runbook: "https://wiki.company.com/runbooks/frontend"

    # Oncall
    pagerduty-service-id: "PX7XX"

    # Monitoring
    dashboard: "https://grafana.company.com/dashboard/frontend"
spec:
  # ... rest of configuration
```

## Validation and Testing

### Use `kubectl` Validation

```bash
# Dry-run with server validation
kubectl apply -f manifest.yaml --dry-run=server

# Validate without applying
kubectl apply -f manifest.yaml --validate=true --dry-run=client

# Diff before applying
kubectl diff -f manifest.yaml
```

### Linting Tools

```bash
# kubeval - validate against Kubernetes schemas
kubeval manifest.yaml

# yamllint - YAML syntax checking
yamllint manifest.yaml

# kube-score - best practices scoring
kube-score score manifest.yaml
```

## Common Pitfalls

1. **No resource limits**: Pods can starve other workloads
2. **Missing health checks**: Failed pods stay in rotation
3. **Using latest tags**: Unpredictable versions
4. **No labels**: Can't select or organize resources
5. **Large single files**: Hard to maintain and review
6. **Secrets in git**: Security vulnerability
7. **No replica count**: Single point of failure
8. **Missing namespace**: Everything in default
9. **Inconsistent naming**: Hard to find resources
10. **No documentation**: Future maintainers confused

## Checklist for Production Manifests

- [ ] Resource requests and limits defined
- [ ] Liveness and readiness probes configured
- [ ] Running as non-root user
- [ ] Specific image tags (not latest)
- [ ] Appropriate labels applied
- [ ] Documentation in annotations
- [ ] Multiple replicas for HA
- [ ] Secrets managed externally
- [ ] Namespace specified
- [ ] Resource names follow conventions

## Docker Compose to Kubernetes Comparison

| Compose              | Kubernetes                  | Notes                                  |
| -------------------- | --------------------------- | -------------------------------------- |
| `docker-compose.yml` | Multiple YAML files         | K8s separates by resource type         |
| Service name         | Deployment + Service        | K8s splits compute and networking      |
| `environment:`       | ConfigMap / Secret          | K8s separates config from workloads    |
| `volumes:`           | PVC + Volume mount          | K8s has persistent storage abstraction |
| `ports:`             | Service spec                | K8s Services handle port mapping       |
| `depends_on:`        | Init containers / readiness | K8s has more sophisticated ordering    |
| `restart:`           | restartPolicy               | Different in spec                      |

## Key Takeaways

- Organize manifests logically (by component or environment)
- Use descriptive, consistent naming conventions
- Always specify resource requests and limits
- Implement health checks for all applications
- Use labels for selection, annotations for documentation
- Separate configuration from code using ConfigMaps
- Never commit secrets to version control
- Use specific image tags, not `latest`
- Follow security best practices (non-root, read-only filesystem)
- Document your manifests with annotations

## Check Your Understanding

1. What's the difference between labels and annotations?
2. Why should you always specify resource limits?
3. What's the risk of using `image: nginx:latest`?
4. When should you use ConfigMaps vs Secrets?
5. What are the three types of health probes in Kubernetes?

## Next Steps

Continue to [Final Lab: Complete Application Deployment](../11-final-lab/README.md) to apply everything you've learned.
