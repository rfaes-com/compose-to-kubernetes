# Helm Lab Solutions

## Task 1: Install Helm

```bash
# Helm should already be installed (check version)
helm version

# Output:
# version.BuildInfo{Version:"v3.x.x", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.x.x"}
```

## Task 2: Add Repository and Search

```bash
# Add Bitnami repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repositories
helm repo update

# Search for nginx
helm search repo nginx

# Output includes:
# NAME                            CHART VERSION   APP VERSION   DESCRIPTION
# bitnami/nginx                   15.x.x          1.25.x        NGINX Open Source is a web server...
# bitnami/nginx-ingress-controller 10.x.x         1.9.x         NGINX Ingress Controller...

# Get chart information
helm show chart bitnami/nginx

# Get default values
helm show values bitnami/nginx > nginx-defaults.yaml
```

## Task 3: Install a Chart

```bash
# Install nginx with custom replica count
helm install my-nginx bitnami/nginx --set replicaCount=2

# Output:
# NAME: my-nginx
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1

# Verify installation
kubectl get pods -l app.kubernetes.io/instance=my-nginx

# Output:
# NAME                        READY   STATUS    RESTARTS   AGE
# my-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
# my-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

# Check service
kubectl get svc my-nginx

# Test the nginx deployment
kubectl port-forward svc/my-nginx 8080:80
# Visit http://localhost:8080 - should see nginx welcome page
```

## Task 4: Create Your Own Chart

```bash
# Create new chart
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

# View the chart structure
tree myapp
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
# Install from local directory
helm install myapp ./myapp

# Output:
# NAME: myapp
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1

# Verify
kubectl get pods -l app.kubernetes.io/name=myapp

# Output:
# NAME                     READY   STATUS    RESTARTS   AGE
# myapp-xxxxxxxxxx-xxxxx   1/1     Running   0          20s
# myapp-xxxxxxxxxx-xxxxx   1/1     Running   0          20s
```

## Task 5: Template Experimentation

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

## Task 6: Upgrade Release

```bash
# Upgrade with different replica count
helm upgrade myapp ./myapp --set replicaCount=3

# Output:
# Release "myapp" has been upgraded. Happy Helming!
# NAME: myapp
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# REVISION: 2

# Verify 3 pods are running
kubectl get pods -l app.kubernetes.io/name=myapp

# Output shows 3 pods

# Check upgrade history
helm history myapp

# Output:
# REVISION  UPDATED                   STATUS      CHART         APP VERSION  DESCRIPTION
# 1         Mon Jan 1 10:00:00 2024   superseded  myapp-0.1.0   1.16.0       Install complete
# 2         Mon Jan 1 10:05:00 2024   deployed    myapp-0.1.0   1.16.0       Upgrade complete
```

## Task 7: Rollback

```bash
# Rollback to previous revision
helm rollback myapp 1

# Output:
# Rollback was a success! Happy Helming!

# Verify back to 2 replicas
kubectl get pods -l app.kubernetes.io/name=myapp

# Output shows 2 pods again

# Check history
helm history myapp

# Output:
# REVISION  UPDATED                   STATUS      CHART         APP VERSION  DESCRIPTION
# 1         Mon Jan 1 10:00:00 2024   superseded  myapp-0.1.0   1.16.0       Install complete
# 2         Mon Jan 1 10:05:00 2024   superseded  myapp-0.1.0   1.16.0       Upgrade complete
# 3         Mon Jan 1 10:10:00 2024   deployed    myapp-0.1.0   1.16.0       Rollback to 1
```

## Task 8: Conditional Resources

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
# Install with config enabled
helm upgrade myapp ./myapp --install

# Verify ConfigMap created
kubectl get configmap | grep myapp

# Disable config
helm upgrade myapp ./myapp --set config.enabled=false

# Verify ConfigMap removed
kubectl get configmap | grep myapp
# (should not appear)
```

## Task 9: Package Chart

```bash
# Package the chart
helm package myapp/

# Output:
# Successfully packaged chart and saved it to: /path/to/myapp-0.1.0.tgz

# Verify package
ls -lh myapp-0.1.0.tgz

# Install from package
helm uninstall myapp
helm install myapp myapp-0.1.0.tgz

# Output:
# NAME: myapp
# ...
# STATUS: deployed
```

## Task 10: Chart with Dependencies

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

**Update dependencies:**

```bash
# Download dependencies
helm dependency update webapp/

# Output:
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "bitnami" chart repository
# Update Complete. ⎈Happy Helming!⎈
# Saving 1 charts
# Downloading redis from repo https://charts.bitnami.com/bitnami
# Deleting outdated charts

# Verify dependencies downloaded
ls -la webapp/charts/

# Output:
# redis-18.x.x.tgz

# Install chart with dependency
helm install webapp ./webapp

# Verify both webapp and redis are running
kubectl get pods

# Output:
# NAME                      READY   STATUS    RESTARTS   AGE
# webapp-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
# webapp-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
# webapp-redis-master-0     1/1     Running   0          30s

# Test disabling Redis
helm upgrade webapp ./webapp --set redis.enabled=false

# Verify only webapp pods remain
kubectl get pods
```

## Bonus Challenge 1: Named Templates

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

## Bonus Challenge 2: Multiple Environments

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
# Development
helm install myapp-dev ./myapp -f myapp/values-dev.yaml

# Production
helm install myapp-prod ./myapp -f myapp/values-prod.yaml

# Verify different configurations
kubectl get pods
kubectl describe pod myapp-dev-xxx | grep -A 5 "Limits"
kubectl describe pod myapp-prod-xxx | grep -A 5 "Limits"
```

## Bonus Challenge 3: Chart Testing

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
# Install chart
helm install myapp ./myapp

# Run test
helm test myapp

# Output:
# NAME: myapp
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# TEST SUITE:     myapp-test-connection
# Last Started:   Mon Jan 1 10:00:00 2024
# Last Completed: Mon Jan 1 10:00:05 2024
# Phase:          Succeeded
```

## Bonus Challenge 4: Hooks

**Create `myapp/templates/job-init.yaml`:**

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

**Install and observe:**

```bash
# Install with watch
kubectl get pods -w &
helm install myapp ./myapp

# You'll see:
# 1. myapp-init job pod runs first (pre-install hook)
# 2. After completion, regular pods start
# 3. Hook pod is deleted (before-hook-creation policy)

# Upgrade and see hook run again
helm upgrade myapp ./myapp --set replicaCount=3
```

## Cleanup

```bash
# Uninstall all releases
helm uninstall my-nginx
helm uninstall myapp
helm uninstall webapp
helm uninstall myapp-dev
helm uninstall myapp-prod

# Remove chart packages
rm -f *.tgz

# Verify cleanup
helm list
kubectl get pods
```

## Key Takeaways

1. **Helm simplifies** Kubernetes deployments with templating
2. **Values files** enable environment-specific configurations
3. **Dependencies** manage complex application stacks
4. **Templating** provides flexibility and reusability
5. **Hooks** enable pre/post installation actions
6. **Testing** validates chart functionality
7. **Version control** enables easy rollbacks
8. **Packaging** makes charts distributable

## Next Steps

- Explore Helm Hub/ArtifactHub for public charts
- Create charts for your applications
- Set up a private chart repository
- Integrate Helm with CI/CD pipelines
- Use Helmfile for managing multiple releases
