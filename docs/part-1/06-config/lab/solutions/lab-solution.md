# Lab Solutions: ConfigMaps and Secrets

Complete solutions for the ConfigMaps and Secrets lab exercises.

## Task 1: Create a ConfigMap from Literals

```bash
# Create ConfigMap with multiple key-value pairs
kubectl create configmap app-settings \
  --from-literal=app_name=MyApp \
  --from-literal=app_env=development \
  --from-literal=log_level=debug \
  --from-literal=max_retries=3
```

**Expected output:**

```text
configmap/app-settings created
```

```bash
# Verify ConfigMap
kubectl get configmap app-settings
```

**Expected output:**

```text
NAME           DATA   AGE
app-settings   4      5s
```

```bash
# View ConfigMap details
kubectl describe configmap app-settings
```

**Expected output:**

```text
Name:         app-settings
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
app_env:
----
development
app_name:
----
MyApp
log_level:
----
debug
max_retries:
----
3
```

```bash
# View as YAML
kubectl get configmap app-settings -o yaml
```

**Expected output:**

```yaml
apiVersion: v1
data:
  app_env: development
  app_name: MyApp
  log_level: debug
  max_retries: "3"
kind: ConfigMap
metadata:
  name: app-settings
  namespace: default
```

## Task 2: Create a ConfigMap from a File

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

# Verify file created
cat app.conf
```

**Expected output:**

```text
[database]
host=postgres
port=5432
name=mydb

[cache]
host=redis
port=6379
```

```bash
# Create ConfigMap from file
kubectl create configmap app-config-file --from-file=app.conf
```

**Expected output:**

```text
configmap/app-config-file created
```

```bash
# View ConfigMap
kubectl get configmap app-config-file -o yaml
```

**Expected output:**

```yaml
apiVersion: v1
data:
  app.conf: |
    [database]
    host=postgres
    port=5432
    name=mydb

    [cache]
    host=redis
    port=6379
kind: ConfigMap
metadata:
  name: app-config-file
  namespace: default
```

## Task 3: Use ConfigMap as Environment Variables

```bash
# Create Pod with ConfigMap as environment variables
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
        env | grep -E 'app_|log_|max_' | sort
        echo "Sleeping..."
        sleep 3600
    envFrom:
    - configMapRef:
        name: app-settings
  restartPolicy: Never
EOF
```

**Expected output:**

```text
pod/env-test created
```

```bash
# Wait for pod to start
kubectl wait --for=condition=Ready pod/env-test --timeout=30s

# View pod logs
kubectl logs env-test
```

**Expected output:**

```text
Environment variables from ConfigMap:
app_env=development
app_name=MyApp
log_level=debug
max_retries=3
Sleeping...
```

```bash
# Verify environment variables inside pod
kubectl exec env-test -- env | grep -E 'app_|log_|max_' | sort
```

**Expected output:**

```text
app_env=development
app_name=MyApp
log_level=debug
max_retries=3
```

## Task 4: Mount ConfigMap as Volume

```bash
# Create Deployment with ConfigMap mounted as volume
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
```

**Expected output:**

```text
deployment.apps/config-volume-test created
```

```bash
# Wait for deployment
kubectl wait --for=condition=Available deployment/config-volume-test --timeout=60s

# Verify pods are running
kubectl get pods -l app=config-test
```

**Expected output:**

```text
NAME                                  READY   STATUS    RESTARTS   AGE
config-volume-test-5c8b9d7f8d-abcde   1/1     Running   0          10s
config-volume-test-5c8b9d7f8d-fghij   1/1     Running   0          10s
```

```bash
# Check if ConfigMap is mounted
kubectl exec deployment/config-volume-test -- ls -la /etc/config
```

**Expected output:**

```text
total 4
drwxrwxrwx    3 root     root           80 Jan  1 00:00 .
drwxr-xr-x    1 root     root         4096 Jan  1 00:00 ..
drwxr-xr-x    2 root     root           60 Jan  1 00:00 ..data
lrwxrwxrwx    1 root     root           15 Jan  1 00:00 app.conf -> ..data/app.conf
```

```bash
# View the mounted configuration file
kubectl exec deployment/config-volume-test -- cat /etc/config/app.conf
```

**Expected output:**

```text
[database]
host=postgres
port=5432
name=mydb

[cache]
host=redis
port=6379
```

## Task 5: Create and Use Secrets

```bash
# Create Secret
kubectl create secret generic db-secret \
  --from-literal=username=dbadmin \
  --from-literal=password=secretpass123
```

**Expected output:**

```text
secret/db-secret created
```

```bash
# View Secret (notice data is hidden)
kubectl get secret db-secret
```

**Expected output:**

```text
NAME        TYPE     DATA   AGE
db-secret   Opaque   2      5s
```

```bash
# Describe Secret (keys shown, values hidden)
kubectl describe secret db-secret
```

**Expected output:**

```text
Name:         db-secret
Namespace:    default
Type:         Opaque

Data
====
password:  13 bytes
username:  7 bytes
```

```bash
# View Secret in YAML (base64 encoded)
kubectl get secret db-secret -o yaml
```

**Expected output:**

```yaml
apiVersion: v1
data:
  password: c2VjcmV0cGFzczEyMw==
  username: ZGJhZG1pbg==
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque
```

```bash
# Decode secret values
echo "Username:"
kubectl get secret db-secret -o jsonpath='{.data.username}' | base64 -d
echo ""
echo "Password:"
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
echo ""
```

**Expected output:**

```text
Username:
dbadmin
Password:
secretpass123
```

```bash
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
        echo "Sleeping..."
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
```

**Expected output:**

```text
pod/secret-test created
```

```bash
# Wait for pod and check logs
kubectl wait --for=condition=Ready pod/secret-test --timeout=30s
kubectl logs secret-test
```

**Expected output:**

```text
Database credentials:
Username: dbadmin
Password: secretpass123
Sleeping...
```

```bash
# Verify environment variables
kubectl exec secret-test -- sh -c 'echo "User: $DB_USER, Pass: $DB_PASS"'
```

**Expected output:**

```text
User: dbadmin, Pass: secretpass123
```

## Task 6: Update ConfigMap and Observe Changes

```bash
# Check current log_level
kubectl get configmap app-settings -o jsonpath='{.data.log_level}'
echo ""
```

**Expected output:**

```text
debug
```

```bash
# Update ConfigMap
kubectl patch configmap app-settings --type merge -p '{"data":{"log_level":"info"}}'
```

**Expected output:**

```text
configmap/app-settings patched
```

```bash
# Verify update
kubectl get configmap app-settings -o jsonpath='{.data.log_level}'
echo ""
```

**Expected output:**

```text
info
```

```bash
# Check environment variable in existing pod (will NOT change)
kubectl exec env-test -- env | grep log_level
```

**Expected output:**

```text
log_level=debug
```

The environment variable still shows "debug" because environment variables are set at pod creation time and don't update automatically.

```bash
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
        echo "Environment variables from ConfigMap:"
        env | grep -E 'app_|log_|max_' | sort
        sleep 3600
    envFrom:
    - configMapRef:
        name: app-settings
  restartPolicy: Never
EOF

# Wait and check new value
kubectl wait --for=condition=Ready pod/env-test --timeout=30s
kubectl exec env-test -- env | grep log_level
```

**Expected output:**

```text
log_level=info
```

Now it shows the updated value!

```bash
# For volume mounts, changes propagate automatically (may take up to 60 seconds)
# Update the app-config-file ConfigMap
kubectl patch configmap app-config-file --type merge -p '{"data":{"app.conf":"[database]\nhost=postgres-updated\nport=5432\n"}}'

# Wait a moment for kubelet to sync
sleep 10

# Check updated file in pod
kubectl exec deployment/config-volume-test -- cat /etc/config/app.conf
```

**Expected output:**

```text
[database]
host=postgres-updated
port=5432
```

The volume mount was updated automatically!

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

# Verify cleanup
kubectl get configmaps,secrets,pods,deployments
```

**Expected output:**

```text
NAME                  TYPE                                  DATA   AGE
secret/default-token  kubernetes.io/service-account-token   3      30m
```

## Bonus Challenges

### Challenge 1: Immutable ConfigMap

Create an immutable ConfigMap that cannot be updated.

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

# Try to update it
kubectl patch configmap immutable-config --type merge -p '{"data":{"setting":"newvalue"}}'
```

**Expected output:**

```text
Error from server (Forbidden): configmaps "immutable-config" is forbidden:
field immutable is immutable
```

Benefits of immutable ConfigMaps:

- Protection against accidental updates
- Better performance (kubelet doesn't need to watch for changes)
- Must delete and recreate to change values

```bash
# Cleanup
kubectl delete configmap immutable-config
```

### Challenge 2: Mount Secret as Files

```bash
# Create secret with multiple keys
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
        echo "Reading secrets from files:"
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

# Check output
kubectl logs secret-files
```

**Expected output:**

```text
Reading secrets from files:
API Key: sk-123456
API Secret: secret-789
```

```bash
# Cleanup
kubectl delete pod secret-files
kubectl delete secret app-secrets
```

### Challenge 3: Selective ConfigMap Mount

Mount only specific keys from a ConfigMap.

```bash
# Create ConfigMap with multiple files
kubectl create configmap multi-file-config \
  --from-literal=file1.txt="Content of file 1" \
  --from-literal=file2.txt="Content of file 2" \
  --from-literal=file3.txt="Content of file 3"

# Mount only file1.txt and file2.txt
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: selective-mount
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c", "ls -la /config && cat /config/* && sleep 3600"]
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    configMap:
      name: multi-file-config
      items:
      - key: file1.txt
        path: file1.txt
      - key: file2.txt
        path: file2.txt
      # file3.txt is NOT mounted
  restartPolicy: Never
EOF

# Check which files are present
kubectl logs selective-mount
```

**Expected output:**

```text
total 12
drwxrwxrwx    3 root     root           100 Jan  1 00:00 .
drwxr-xr-x    1 root     root          4096 Jan  1 00:00 ..
drwxr-xr-x    2 root     root            80 Jan  1 00:00 ..data
lrwxrwxrwx    1 root     root            16 Jan  1 00:00 file1.txt -> ..data/file1.txt
lrwxrwxrwx    1 root     root            16 Jan  1 00:00 file2.txt -> ..data/file2.txt
Content of file 1
Content of file 2
```

Notice file3.txt is not present.

```bash
# Cleanup
kubectl delete pod selective-mount
kubectl delete configmap multi-file-config
```

## Key Takeaways

1. **ConfigMaps** store non-sensitive configuration data as key-value pairs or files
2. **Secrets** store sensitive data with base64 encoding (not encryption by default)
3. **Environment variables** from ConfigMaps/Secrets don't update automatically - require pod restart
4. **Volume mounts** from ConfigMaps/Secrets update automatically (with kubelet sync delay)
5. Use `stringData` in Secrets for plain text (auto-encoded to base64)
6. Immutable ConfigMaps/Secrets prevent accidental updates and improve performance
7. Secrets should be combined with RBAC and encryption at rest for production security

## Common Issues and Solutions

**Issue:** Secret values showing as base64 in pod

- **Cause:** Kubernetes automatically decodes secrets when mounted or used as env vars
- **Solution:** No action needed - this is expected behavior

**Issue:** ConfigMap changes not reflected in pod

- **Cause:** Environment variables don't update automatically
- **Solution:** Restart pod or use volume mounts for auto-updates

**Issue:** Permission denied when accessing mounted secret

- **Cause:** Secret volume mount permissions
- **Solution:** Secret volumes are mounted read-only by default (0644)

**Issue:** ConfigMap too large

- **Cause:** ConfigMaps have a 1MB size limit
- **Solution:** Use PersistentVolume for large files or break into multiple ConfigMaps
