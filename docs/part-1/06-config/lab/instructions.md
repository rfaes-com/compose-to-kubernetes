# Lab: ConfigMaps and Secrets

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

**Hints:**

```bash
kubectl create configmap app-settings \
  --from-literal=app_name=MyApp \
  --from-literal=app_env=development \
  --from-literal=log_level=debug \
  --from-literal=max_retries=3
```

### Task 2: Create a ConfigMap from a File

Create a configuration file and use it to create a ConfigMap.

**Requirements:**

- Create a file named `app.conf` with sample configuration
- ConfigMap name: `app-config-file`
- Load the file content into the ConfigMap

**Hints:**

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

### Task 3: Use ConfigMap as Environment Variables

Create a Pod that uses the ConfigMap as environment variables.

**Requirements:**

- Pod name: `env-test`
- Image: `busybox:1.36`
- Load all keys from `app-settings` ConfigMap as environment variables
- Command that prints all environment variables and sleeps

**Hints:**

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
```

### Task 4: Mount ConfigMap as Volume

Create a Deployment that mounts the configuration file as a volume.

**Requirements:**

- Deployment name: `config-volume-test`
- Image: `nginx:1.25-alpine`
- Mount `app-config-file` ConfigMap to `/etc/config`
- 2 replicas

**Hints:**

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
```

### Task 5: Create and Use Secrets

Create a Secret for database credentials and use it in a Pod.

**Requirements:**

- Secret name: `db-secret`
- Keys: `username` = "dbadmin", `password` = "secretpass123"
- Create a Pod that uses these secrets as environment variables
- Pod name: `secret-test`

**Hints:**

```bash
# Create secret
kubectl create secret generic db-secret \
  --from-literal=username=dbadmin \
  --from-literal=password=secretpass123

# Create Pod using secret
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
```

### Task 6: Update ConfigMap and Observe Changes

Update the ConfigMap and see how it affects mounted volumes vs environment variables.

**Requirements:**

- Update `app-settings` ConfigMap
- Check if environment variables in `env-test` pod changed
- Check if volume mounts in `config-volume-test` pods updated

**Hints:**

```bash
# Update ConfigMap
kubectl patch configmap app-settings --type merge -p '{"data":{"log_level":"info"}}'

# Check environment variables (will NOT change without restart)
kubectl exec env-test -- env | grep log_level

# Delete and recreate pod to see new value
kubectl delete pod env-test
# (Recreate the pod from Task 3)

# For volume mounts, check if file updated (may take up to 60 seconds)
kubectl exec deployment/config-volume-test -- cat /etc/config/app.conf
```

## Verification

Check your work:

```bash
# List ConfigMaps
kubectl get configmaps

# List Secrets (values are hidden)
kubectl get secrets

# Describe to see keys (but not values)
kubectl describe configmap app-settings
kubectl describe secret db-secret

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

## Check Your Understanding

1. What is the difference between ConfigMap and Secret?
2. How do environment variables from ConfigMaps differ from volume mounts in terms of updates?
3. Why are Secret values base64 encoded? Does this provide security?
4. When should you use `stringData` vs `data` in a Secret manifest?
5. How can you make a ConfigMap or Secret immutable? Why would you want to?

## Next Steps

Proceed to [Persistent Storage](../../07-storage/README.md) to learn about volumes and persistent storage.
