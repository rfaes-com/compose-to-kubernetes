# Helm Lab

**Duration:** 25 minutes

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
helm version
# version.BuildInfo{Version:"v3.x.x", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.x.x"}

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm search repo nginx
# NAME                            CHART VERSION   APP VERSION   DESCRIPTION
# bitnami/nginx                   15.x.x          1.25.x        NGINX Open Source is a web server...
# bitnami/nginx-ingress-controller 10.x.x         1.9.x         NGINX Ingress Controller...

# Get chart information
helm show chart bitnami/nginx

# Get default values
helm show values bitnami/nginx > nginx-defaults.yaml
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
helm install my-nginx bitnami/nginx --set replicaCount=2

# Output:
# NAME: my-nginx
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1

kubectl get pods -l app.kubernetes.io/instance=my-nginx
# NAME                        READY   STATUS    RESTARTS   AGE
# my-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
# my-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

kubectl get svc my-nginx

# Test the nginx deployment
kubectl port-forward svc/my-nginx 8080:80
# Visit http://localhost:8080 - should see nginx welcome page
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
helm upgrade my-nginx bitnami/nginx --set replicaCount=2

# Output:
# Release "my-nginx" has been upgraded. Happy Helming!
# NAME: my-nginx
# ...
# STATUS: deployed
# REVISION: 2

# Verify
kubectl get pods -l app.kubernetes.io/name=my-nginx
# Output shows updated pods
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
helm create myapp

# This creates:
# myapp/
#   Chart.yaml
#   values.yaml
#   templates/
#     deployment.yaml
#     service.yaml
#     ingress.yaml
#     ...
```

**Modify `myapp/values.yaml`:**

```yaml
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25.3"

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

**Install the chart:**

```bash
helm install myapp ./myapp

# Output:
# NAME: myapp
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1

kubectl get pods -l app.kubernetes.io/name=myapp
# NAME                     READY   STATUS    RESTARTS   AGE
# myapp-xxxxxxxxxx-xxxxx   1/1     Running   0          20s
# myapp-xxxxxxxxxx-xxxxx   1/1     Running   0          20s
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

**Add environment variable to `myapp/templates/deployment.yaml`:**

```yaml
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        # Add this section:
        env:
        - name: APP_ENV
          value: {{ .Values.environment | default "development" | quote }}
        - name: APP_VERSION
          value: {{ .Chart.Version | quote }}
```

**Add to `myapp/values.yaml`:**

```yaml
environment: production
```

**Test template rendering:**

```bash
# Render templates without installing
helm template myapp ./myapp

# Check specific values
helm template myapp ./myapp --set environment=staging
# Output will show rendered YAML with APP_ENV=staging
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
helm upgrade myapp ./myapp --set replicaCount=3

# Output:
# Release "myapp" has been upgraded. Happy Helming!
# NAME: myapp
# LAST DEPLOYED: ...
# STATUS: deployed
# REVISION: 2

kubectl get pods -l app.kubernetes.io/name=myapp
# Output shows 3 pods

helm history myapp
# REVISION  UPDATED                   STATUS      CHART         APP VERSION  DESCRIPTION
# 1         Mon Jan 1 10:00:00 2024   superseded  myapp-0.1.0   1.16.0       Install complete
# 2         Mon Jan 1 10:05:00 2024   deployed    myapp-0.1.0   1.16.0       Upgrade complete
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
helm rollback myapp 1

# Output:
# Rollback was a success! Happy Helming!

kubectl get pods -l app.kubernetes.io/name=myapp
# Output shows 2 pods again

helm history myapp
# REVISION  UPDATED                   STATUS      CHART         APP VERSION  DESCRIPTION
# 1         Mon Jan 1 10:00:00 2024   superseded  myapp-0.1.0   1.16.0       Install complete
# 2         Mon Jan 1 10:05:00 2024   superseded  myapp-0.1.0   1.16.0       Upgrade complete
# 3         Mon Jan 1 10:10:00 2024   deployed    myapp-0.1.0   1.16.0       Rollback to 1
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

**Create `myapp/templates/configmap.yaml`:**

```yaml
{{- if .Values.config.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.config.data }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
```

**Add to `myapp/values.yaml`:**

```yaml
config:
  enabled: true
  data:
    log_level: "info"
    max_connections: "100"
```

**Mount ConfigMap in `myapp/templates/deployment.yaml`:**

```yaml
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        # ... existing config ...
        {{- if .Values.config.enabled }}
        envFrom:
        - configMapRef:
            name: {{ include "myapp.fullname" . }}-config
        {{- end }}
```

**Test:**

```bash
helm upgrade myapp ./myapp --install

kubectl get configmap | grep myapp

# Disable config
helm upgrade myapp ./myapp --set config.enabled=false

# Verify ConfigMap removed
kubectl get configmap | grep myapp
# (should not appear)
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
helm package myapp/

# Output:
# Successfully packaged chart and saved it to: /path/to/myapp-0.1.0.tgz

ls -lh myapp-0.1.0.tgz

helm uninstall myapp
helm install myapp myapp-0.1.0.tgz

# Output:
# NAME: myapp
# ...
# STATUS: deployed
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

**Create new chart:**

```bash
helm create webapp
cd webapp
```

**Modify `webapp/Chart.yaml` to add dependency:**

```yaml
apiVersion: v2
name: webapp
description: A Helm chart for webapp with Redis
type: application
version: 0.1.0
appVersion: "1.0"

dependencies:
- name: redis
  version: "18.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: redis.enabled
  tags:
    - cache
```

**Modify `webapp/values.yaml`:**

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.25.3"

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      enabled: false
```

**Update and install:**

```bash
helm dependency update webapp/

# Output:
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "bitnami" chart repository
# Saving 1 charts
# Downloading redis from repo https://charts.bitnami.com/bitnami

ls -la webapp/charts/
# redis-18.x.x.tgz

helm install webapp ./webapp

kubectl get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# webapp-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
# webapp-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
# webapp-redis-master-0     1/1     Running   0          30s

# Test disabling Redis
helm upgrade webapp ./webapp --set redis.enabled=false

# Verify only webapp pods remain
kubectl get pods
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

**Create `values-dev.yaml`:**

```yaml
replicaCount: 1
environment: development
resources:
  limits:
    cpu: 50m
    memory: 64Mi
  requests:
    cpu: 25m
    memory: 32Mi
```

**Create `values-prod.yaml`:**

```yaml
replicaCount: 3
environment: production
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

**Install with different values:**

```bash
helm install myapp-dev ./myapp -f myapp/values-dev.yaml
helm install myapp-prod ./myapp -f myapp/values-prod.yaml

kubectl get pods
kubectl describe pod myapp-dev-xxx | grep -A 5 "Limits"
kubectl describe pod myapp-prod-xxx | grep -A 5 "Limits"
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

**Create `myapp/templates/_helpers.tpl`:**

```yaml
{{/*
Create a default fully qualified app name with environment
*/}}
{{- define "myapp.fullname.env" -}}
{{- include "myapp.fullname" . }}-{{ .Values.environment | default "dev" }}
{{- end }}

{{/*
Common labels with custom additions
*/}}
{{- define "myapp.customLabels" -}}
{{ include "myapp.labels" . }}
environment: {{ .Values.environment | default "development" }}
version: {{ .Chart.Version }}
{{- end }}

{{/*
Resource limits and requests template
*/}}
{{- define "myapp.resources" -}}
resources:
  limits:
    cpu: {{ .Values.resources.limits.cpu | default "100m" }}
    memory: {{ .Values.resources.limits.memory | default "128Mi" }}
  requests:
    cpu: {{ .Values.resources.requests.cpu | default "50m" }}
    memory: {{ .Values.resources.requests.memory | default "64Mi" }}
{{- end }}
```

**Use in `myapp/templates/deployment.yaml`:**

```yaml
metadata:
  labels:
    {{- include "myapp.customLabels" . | nindent 4 }}
spec:
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        {{- include "myapp.resources" . | nindent 8 }}
```

</details>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

**Create `myapp/templates/tests/test-connection.yaml`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "myapp.fullname" . }}-test-connection"
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "myapp.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

**Run test:**

```bash
helm install myapp ./myapp
helm test myapp

# Output:
# NAME: myapp
# ...
# TEST SUITE:     myapp-test-connection
# Last Started:   Mon Jan 1 10:00:00 2024
# Last Completed: Mon Jan 1 10:00:05 2024
# Phase:          Succeeded
```

**Hook example (`myapp/templates/job-init.yaml`):**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "myapp.fullname" . }}-init"
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    metadata:
      name: "{{ include "myapp.fullname" . }}-init"
    spec:
      restartPolicy: Never
      containers:
      - name: init
        image: busybox
        command: ['sh', '-c', 'echo "Performing initialization..."; sleep 5; echo "Done!"']
```

```bash
kubectl get pods -w &
helm install myapp ./myapp
# 1. myapp-init job pod runs first (pre-install hook)
# 2. After completion, regular pods start
# 3. Hook pod is deleted (before-hook-creation policy)

helm upgrade myapp ./myapp --set replicaCount=3
```

</details>

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

## Key takeaways

1. **Helm charts** package Kubernetes manifests with templating and versioning into a single deployable unit
2. **Values files** separate configuration from chart structure — customize without modifying templates
3. **`helm upgrade --install`** is idempotent — safe to run repeatedly in CI/CD pipelines
4. **Rollbacks** restore a previous release state in seconds with `helm rollback`
5. **Chart dependencies** allow composing complex applications from reusable sub-charts
6. **Hooks** enable lifecycle actions such as database migrations before upgrades and cleanups after deletion

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../03-gitops/README.md).
