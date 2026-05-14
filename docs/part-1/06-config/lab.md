# Lab

**Duration:** 25 minutes

## Objectives

- Create and manage ConfigMaps from literals and files
- Create and manage Secrets
- Use ConfigMaps as environment variables
- Mount ConfigMaps and Secrets as volumes
- Update configuration and observe changes

## Prerequisites

- Kind cluster running
- kubectl configured and working

## Tasks

### Task 1: Create a ConfigMap from Literals

Create a ConfigMap with application settings using kubectl.

**Requirements:**

- ConfigMap name: `app-settings`
- Keys and values:
  - `app_name`: "MyApp"
  - `app_env`: "development"
  - `log_level`: "debug"
  - `max_retries`: "3"

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
kubectl create configmap app-settings \
  --from-literal=app_name=MyApp \
  --from-literal=app_env=development \
  --from-literal=log_level=debug \
  --from-literal=max_retries=3
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create ConfigMap with multiple key-value pairs
kubectl create configmap app-settings \
  --from-literal=app_name=MyApp \
  --from-literal=app_env=development \
  --from-literal=log_level=debug \
  --from-literal=max_retries=3

# Output:
# configmap/app-settings created

# Verify ConfigMap
kubectl get configmap app-settings

# Output:
# NAME           DATA   AGE
# app-settings   4      5s

# View ConfigMap details
kubectl describe configmap app-settings

# View as YAML
kubectl get configmap app-settings -o yaml
```

</details>

### Task 2: Create a ConfigMap from a File

Create a configuration file and use it to create a ConfigMap.

**Requirements:**

- Create a file named `app.conf` with sample configuration
- ConfigMap name: `app-config-file`
- Load the file content into the ConfigMap

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Create configuration file
cat > app.conf <<EOF
[database]
host=postgres
port=5432
name=mydb

[cache]
host=redis
port=6379
EOF

# Create ConfigMap from file
kubectl create configmap app-config-file --from-file=app.conf
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create configuration file
cat > app.conf <<EOF
[database]
host=postgres
port=5432
name=mydb

[cache]
host=redis
port=6379
EOF

# Create ConfigMap from file
kubectl create configmap app-config-file --from-file=app.conf

# Output:
# configmap/app-config-file created

# View ConfigMap
kubectl get configmap app-config-file -o yaml
```

</details>

### Task 3: Use ConfigMap as Environment Variables

Create a Pod that uses the ConfigMap as environment variables.

**Requirements:**

- Pod name: `env-test`
- Image: `busybox:1.36`
- Load all keys from `app-settings` ConfigMap as environment variables
- Command that prints all environment variables and sleeps

<details class="hint" markdown="1">
<summary>Hint</summary>

Use `envFrom` with `configMapRef` in the Pod spec:

```yaml
envFrom:
- configMapRef:
    name: app-settings
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: env-test
spec:
  containers:
  - name: test
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
      - |
        echo "Environment variables from ConfigMap:"
        env | grep -E 'app_|log_|max_'
        sleep 3600
    envFrom:
    - configMapRef:
        name: app-settings
  restartPolicy: Never
EOF

# Wait for pod to start
kubectl wait --for=condition=Ready pod/env-test --timeout=30s

# View pod logs
kubectl logs env-test

# Output:
# Environment variables from ConfigMap:
# app_env=development
# app_name=MyApp
# log_level=debug
# max_retries=3
```

</details>

### Task 4: Mount ConfigMap as Volume

Create a Deployment that mounts the configuration file as a volume.

**Requirements:**

- Deployment name: `config-volume-test`
- Image: `nginx:1.25-alpine`
- Mount `app-config-file` ConfigMap to `/etc/config`
- 2 replicas

<details class="hint" markdown="1">
<summary>Hint</summary>

In the Pod spec, define a volume from a ConfigMap and mount it:

```yaml
volumes:
- name: config
  configMap:
    name: app-config-file
containers:
- name: nginx
  volumeMounts:
  - name: config
    mountPath: /etc/config
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-volume-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: config-test
  template:
    metadata:
      labels:
        app: config-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        volumeMounts:
        - name: config
          mountPath: /etc/config
      volumes:
      - name: config
        configMap:
          name: app-config-file
EOF

# Wait for deployment
kubectl wait --for=condition=Available deployment/config-volume-test --timeout=60s

# Check if ConfigMap is mounted
kubectl exec deployment/config-volume-test -- cat /etc/config/app.conf

# Output:
# [database]
# host=postgres
# port=5432
# name=mydb
#
# [cache]
# host=redis
# port=6379
```

</details>

### Task 5: Create and Use Secrets

Create a Secret for database credentials and use it in a Pod.

**Requirements:**

- Secret name: `db-secret`
- Keys: `username` = "dbadmin", `password` = "secretpass123"
- Create a Pod that uses these secrets as environment variables
- Pod name: `secret-test`

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Create secret
kubectl create secret generic db-secret \
  --from-literal=username=dbadmin \
  --from-literal=password=secretpass123
```

Use `valueFrom.secretKeyRef` to reference individual keys as env vars.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create secret
kubectl create secret generic db-secret \
  --from-literal=username=dbadmin \
  --from-literal=password=secretpass123

# Output:
# secret/db-secret created

# View Secret (notice data is hidden)
kubectl get secret db-secret
kubectl describe secret db-secret

# Decode secret values
echo "Username:"
kubectl get secret db-secret -o jsonpath='{.data.username}' | base64 -d
echo ""
echo "Password:"
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
echo ""

# Create Pod using Secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-test
spec:
  containers:
  - name: test
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
      - |
        echo "Database credentials:"
        echo "Username: \$DB_USER"
        echo "Password: \$DB_PASS"
        sleep 3600
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
  restartPolicy: Never
EOF

# Wait and check logs
kubectl wait --for=condition=Ready pod/secret-test --timeout=30s
kubectl logs secret-test

# Output:
# Database credentials:
# Username: dbadmin
# Password: secretpass123
```

</details>

### Task 6: Update ConfigMap and Observe Changes

Update the ConfigMap and see how it affects mounted volumes vs environment variables.

**Requirements:**

- Update `app-settings` ConfigMap (change `log_level` to "info")
- Check if environment variables in `env-test` pod changed
- Check if volume mounts in `config-volume-test` pods updated

<details class="hint" markdown="1">
<summary>Hint</summary>

```bash
# Update ConfigMap
kubectl patch configmap app-settings --type merge -p '{"data":{"log_level":"info"}}'

# Environment variables require a pod restart to pick up changes
# Volume mounts update automatically (within ~60 seconds)
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Update ConfigMap
kubectl patch configmap app-settings --type merge -p '{"data":{"log_level":"info"}}'

# Output:
# configmap/app-settings patched

# Check environment variable in existing pod (will NOT change)
kubectl exec env-test -- env | grep log_level

# Output:
# log_level=debug
# (still shows old value - env vars are set at pod creation time)

# Delete and recreate pod to get new value
kubectl delete pod env-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: env-test
spec:
  containers:
  - name: test
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
      - |
        env | grep -E 'app_|log_|max_'
        sleep 3600
    envFrom:
    - configMapRef:
        name: app-settings
  restartPolicy: Never
EOF

kubectl wait --for=condition=Ready pod/env-test --timeout=30s
kubectl exec env-test -- env | grep log_level

# Output:
# log_level=info
# (now shows updated value)

# For volume mounts, changes propagate automatically (may take up to 60 seconds)
# Check updated file in pod
sleep 15
kubectl exec deployment/config-volume-test -- cat /etc/config/app.conf
```

</details>

## Verification

```bash
# List ConfigMaps
kubectl get configmaps

# List Secrets (values are hidden)
kubectl get secrets

# View ConfigMap data
kubectl get configmap app-settings -o yaml

# View Secret data (base64 encoded)
kubectl get secret db-secret -o yaml

# Decode secret manually
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
echo ""
```

## Cleanup

```bash
# Delete pods
kubectl delete pod env-test secret-test

# Delete deployment
kubectl delete deployment config-volume-test

# Delete ConfigMaps
kubectl delete configmap app-settings app-config-file

# Delete Secret
kubectl delete secret db-secret

# Delete local file
rm -f app.conf
```

## Bonus Challenges

**1. Immutable ConfigMap:** Create an immutable ConfigMap and verify it cannot be updated

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
data:
  setting: "value"
immutable: true
EOF

# Try to update it (will fail)
kubectl patch configmap immutable-config --type merge -p '{"data":{"setting":"newvalue"}}'

# Output:
# Error from server (Forbidden): configmaps "immutable-config" is forbidden:
# field immutable is immutable

# Cleanup
kubectl delete configmap immutable-config
```

</details>

**2. Mount Secret as Files:** Mount Secret keys as individual files in a volume

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create secret
kubectl create secret generic app-secrets \
  --from-literal=api-key=sk-123456 \
  --from-literal=api-secret=secret-789

# Create pod that mounts secret as files
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-files
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
      - |
        echo "API Key: \$(cat /etc/secrets/api-key)"
        echo "API Secret: \$(cat /etc/secrets/api-secret)"
        sleep 3600
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: app-secrets
  restartPolicy: Never
EOF

kubectl wait --for=condition=Ready pod/secret-files --timeout=30s
kubectl logs secret-files

# Output:
# API Key: sk-123456
# API Secret: secret-789

# Cleanup
kubectl delete pod secret-files
kubectl delete secret app-secrets
```

</details>

## Common issues

| Issue                              | Solution                                                                            |
| ---------------------------------- | ----------------------------------------------------------------------------------- |
| ConfigMap changes not in pod       | Env vars require pod restart; volume mounts update automatically within ~60 seconds |
| Secret value shows base64 in shell | Pipe through `base64 -d` to decode, or use env var injection                        |
| ConfigMap too large                | ConfigMaps have a 1MB limit; use PersistentVolume for large files                   |
| Permission denied on secret mount  | Secret volumes are mounted read-only by default (0644)                              |

## Key takeaways

1. **ConfigMaps** store non-sensitive configuration as key-value pairs or file content
2. **Secrets** store sensitive data; values are base64-encoded but not encrypted by default
3. **Environment variables** from ConfigMaps/Secrets are fixed at pod creation — restart required for updates
4. **Volume mounts** from ConfigMaps/Secrets update automatically with a short delay
5. **Never commit Secrets to version control** — use external secret management in production

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../07-storage/README.md)
