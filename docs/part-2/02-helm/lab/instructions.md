# Helm Lab

Duration: 25 minutes

## Objectives

- Create a Helm chart from scratch
- Use templating and values
- Install and upgrade releases
- Work with chart repositories
- Package and distribute charts

## Prerequisites

- kind cluster running
- Helm 3 installed
- kubectl configured

## Lab Tasks

### Task 1: Verify Helm Installation

```bash
# Check Helm version
helm version

# Add Bitnami repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repository index
helm repo update

# Search for charts
helm search repo nginx
```

**Expected Output:**
Helm version 3.x and list of nginx charts from Bitnami.

---

### Task 2: Install a Chart from Repository

Install NGINX from the Bitnami repository:

```bash
# Create namespace
kubectl create namespace helm-demo

# Install NGINX chart
helm install my-nginx bitnami/nginx \
  --namespace helm-demo \
  --set service.type=NodePort

# List releases
helm list -n helm-demo

# Check deployed resources
kubectl get all -n helm-demo

# Get release status
helm status my-nginx -n helm-demo
```

**Expected Output:**
NGINX deployment, service, and pods running in helm-demo namespace.

---

### Task 3: Customize with Values

Create a custom values file to override defaults:

Create `custom-values.yaml`:

```yaml
replicaCount: 3

service:
  type: NodePort
  nodePorts:
    http: 30080

resources:
  limits:
    cpu: 300m
    memory: 384Mi
  requests:
    cpu: 150m
    memory: 192Mi
```

Upgrade the release with custom values:

```bash
# Upgrade with custom values
helm upgrade my-nginx bitnami/nginx \
  --namespace helm-demo \
  -f custom-values.yaml

# Verify changes
kubectl get pods -n helm-demo
kubectl get svc -n helm-demo

# Check revision history
helm history my-nginx -n helm-demo
```

Test the service:

```bash
curl http://localhost:30080
```

---

### Task 4: Create Your Own Chart

Create a custom Helm chart for a simple application:

```bash
# Create chart scaffold
helm create myapp

# Explore the structure
tree myapp

# Chart structure:
# myapp/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# └── templates/
#     ├── deployment.yaml
#     ├── service.yaml
#     ├── ingress.yaml
#     └── ...
```

Customize the chart:

Edit `myapp/values.yaml`:

```yaml
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25-alpine"

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

---

### Task 5: Template and Install Your Chart

Preview the rendered templates:

```bash
# Render templates (dry run)
helm template myapp ./myapp

# Validate the chart
helm lint ./myapp

# Install with dry-run to see what would be created
helm install test-release ./myapp --dry-run --debug
```

Install the chart:

```bash
# Install the chart
helm install myapp-release ./myapp --namespace helm-demo

# Verify installation
helm list -n helm-demo
kubectl get all -n helm-demo -l app.kubernetes.io/instance=myapp-release

# Get the rendered manifest
helm get manifest myapp-release -n helm-demo
```

---

### Task 6: Override Values

Install another release with different values:

```bash
# Install with overrides
helm install myapp-prod ./myapp \
  --namespace helm-demo \
  --set replicaCount=5 \
  --set image.tag=1.26-alpine \
  --set service.port=9090

# List both releases
helm list -n helm-demo

# Compare pods
kubectl get pods -n helm-demo -l app.kubernetes.io/name=myapp
```

---

### Task 7: Upgrade and Rollback

Upgrade the release:

```bash
# Upgrade with new values
helm upgrade myapp-release ./myapp \
  --namespace helm-demo \
  --set replicaCount=4

# View history
helm history myapp-release -n helm-demo

# Check the change
kubectl get deployment -n helm-demo
```

Rollback if needed:

```bash
# Rollback to previous version
helm rollback myapp-release -n helm-demo

# Verify rollback
helm history myapp-release -n helm-demo
kubectl get deployment -n helm-demo
```

---

### Task 8: Add Conditional Resources

Modify your chart to conditionally create an Ingress.

Edit `myapp/templates/ingress.yaml` (if not exists, create it):

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  ingressClassName: nginx
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ include "myapp.fullname" . }}
            port:
              number: {{ .Values.service.port }}
{{- end }}
```

Update `myapp/values.yaml`:

```yaml
ingress:
  enabled: false
  host: myapp.local
```

Test with Ingress enabled:

```bash
# Upgrade with Ingress enabled
helm upgrade myapp-release ./myapp \
  --namespace helm-demo \
  --set ingress.enabled=true \
  --set ingress.host=myapp.local

# Verify Ingress created
kubectl get ingress -n helm-demo
```

---

### Task 9: Package and Share

Package your chart for distribution:

```bash
# Package the chart
helm package ./myapp

# This creates: myapp-0.1.0.tgz

# Install from package
helm install myapp-packaged ./myapp-0.1.0.tgz --namespace helm-demo

# List all releases
helm list -n helm-demo
```

---

### Task 10: Work with Chart Dependencies

Add a dependency to your chart.

Edit `myapp/Chart.yaml`:

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"

dependencies:
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

Update `myapp/values.yaml`:

```yaml
# ... existing values ...

redis:
  enabled: true
  master:
    persistence:
      enabled: false
```

Download dependencies and install:

```bash
# Update dependencies
helm dependency update ./myapp

# This downloads charts to myapp/charts/

# List dependencies
helm dependency list ./myapp

# Install with dependency
helm install myapp-with-redis ./myapp \
  --namespace helm-demo \
  --set redis.enabled=true

# Verify Redis is deployed
kubectl get pods -n helm-demo | grep redis
```

---

## Verification

Check all your releases:

```bash
# List all releases in namespace
helm list -n helm-demo

# Get details of each release
helm status my-nginx -n helm-demo
helm status myapp-release -n helm-demo

# View all resources
kubectl get all -n helm-demo
```

## Cleanup

```bash
# Uninstall all releases
helm uninstall my-nginx -n helm-demo
helm uninstall myapp-release -n helm-demo
helm uninstall myapp-prod -n helm-demo
helm uninstall myapp-packaged -n helm-demo
helm uninstall myapp-with-redis -n helm-demo

# Delete namespace
kubectl delete namespace helm-demo

# Remove repository (optional)
helm repo remove bitnami
```

## Bonus Challenges

### Challenge 1: Multi-Environment Values

Create separate values files for dev, staging, and prod environments.

Create `values-dev.yaml`:

```yaml
replicaCount: 1
resources:
  limits:
    cpu: 100m
    memory: 128Mi
```

Create `values-prod.yaml`:

```yaml
replicaCount: 5
resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

Install for different environments:

```bash
helm install myapp-dev ./myapp -f values-dev.yaml
helm install myapp-prod ./myapp -f values-prod.yaml
```

### Challenge 2: Add Health Checks

Modify `myapp/templates/deployment.yaml` to include liveness and readiness probes using values.

Add to `values.yaml`:

```yaml
probes:
  liveness:
    httpGet:
      path: /
      port: http
    initialDelaySeconds: 30
    periodSeconds: 10
  readiness:
    httpGet:
      path: /
      port: http
    initialDelaySeconds: 5
    periodSeconds: 5
```

Update deployment template to use these values.

### Challenge 3: Create a Helper Function

Add a helper function to `myapp/templates/_helpers.tpl` that generates resource names with environment prefix.

```yaml
{{/*
Generate name with environment
*/}}
{{- define "myapp.envName" -}}
{{- if .Values.environment }}
{{- printf "%s-%s" .Values.environment (include "myapp.fullname" .) }}
{{- else }}
{{- include "myapp.fullname" . }}
{{- end }}
{{- end }}
```

Use it in templates:

```yaml
metadata:
  name: {{ include "myapp.envName" . }}
```

### Challenge 4: Chart Hooks

Add a pre-install hook that runs a job before the application deploys.

Create `myapp/templates/pre-install-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-preinstall
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: pre-install
        image: busybox
        command: ['sh', '-c', 'echo "Running pre-install checks..."; sleep 5; echo "Done!"']
      restartPolicy: Never
```

## Troubleshooting Tips

**Chart validation fails:**

- Run `helm lint ./myapp` to see specific errors
- Check YAML syntax and indentation
- Verify template syntax with `helm template`

**Release fails to install:**

- Check logs: `kubectl logs <pod-name>`
- Describe resources: `kubectl describe pod <pod-name>`
- Review release: `helm status <release-name>`

**Values not applied:**

- Verify precedence: defaults < values file < --set
- Check spelling of value keys
- Use `helm get values <release-name>` to see applied values

**Template errors:**

- Use `--dry-run --debug` to see rendered templates
- Check required values with `{{ required "message" .Values.key }}`

## Summary

You've learned to:

- Install charts from repositories
- Create custom Helm charts from scratch
- Use templating and values for customization
- Install, upgrade, and rollback releases
- Work with multiple releases
- Add conditional resources
- Package and distribute charts
- Manage chart dependencies
- Use Helm hooks for lifecycle management

Helm provides powerful package management for Kubernetes, making it easier to deploy, version, and manage complex applications.
