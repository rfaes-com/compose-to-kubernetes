# Configuration Management: ConfigMaps & Secrets

**Duration:** 35 minutes (20 min theory + 15 min lab)
**Format:** Presentation + Hands-on Lab

## Learning Objectives

- Externalize configuration using ConfigMaps
- Store sensitive data securely with Secrets
- Consume configuration in Pods
- Map Docker Compose environment variables to Kubernetes

## The Problem: Hardcoded Configuration

Don't do this:

```yaml
spec:
  containers:
    - name: app
      image: myapp:v1
      env:
        - name: DATABASE_HOST
          value: "postgres.example.com"     # Hardcoded!
        - name: API_KEY
          value: "secret123"                 # Exposed in YAML!
```

**Problems:**

- Configuration tied to image
- Secrets visible in version control
- Different values per environment require different images

**Solution:** ConfigMaps and Secrets

## ConfigMaps

**ConfigMap** stores non-sensitive configuration data as key-value pairs.

### Creating ConfigMaps

#### From Literal Values

```bash
kubectl create configmap app-config \
  --from-literal=database.host=postgres \
  --from-literal=database.port=5432 \
  --from-literal=log.level=info
```

#### From File

```bash
# config.properties
database.host=postgres
database.port=5432
log.level=info

kubectl create configmap app-config --from-file=config.properties
```

#### Declarative (YAML)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.host: "postgres"
  database.port: "5432"
  log.level: "info"
  app.properties: |
    feature.flag.enabled=true
    max.connections=100
```

### Using ConfigMaps in Pods

#### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: myapp:v1
      env:
        # Single value
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.host

        # All keys as environment variables
        - envFrom:
            - configMapRef:
                name: app-config
```

#### As Volume Mounts (Files)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: myapp:v1
      volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: app-config
```

Files appear at `/etc/config/`:

- `/etc/config/database.host`
- `/etc/config/database.port`
- `/etc/config/log.level`

## Secrets

**Secret** stores sensitive data (passwords, tokens, keys).

**Note:** Secrets are base64 encoded, NOT encrypted by default. Use encryption at rest for production.

### Creating Secrets

#### From Literal Values

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secretpass123
```

#### From Files

```bash
echo -n 'admin' > username.txt
echo -n 'secretpass123' > password.txt

kubectl create secret generic db-credentials \
  --from-file=username=username.txt \
  --from-file=password=password.txt
```

#### Declarative (YAML)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=              # base64 encoded "admin"
  password: c2VjcmV0cGFzczEyMw==  # base64 encoded "secretpass123"
```

**Encoding/Decoding:**

```bash
# Encode
echo -n 'admin' | base64
# Output: YWRtaW4=

# Decode
echo 'YWRtaW4=' | base64 --decode
# Output: admin
```

**Using stringData (not encoded):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:              # Plain text, Kubernetes encodes it
  username: admin
  password: secretpass123
```

### Using Secrets in Pods

#### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: myapp:v1
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

#### As Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: myapp:v1
      volumeMounts:
        - name: credentials
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: credentials
      secret:
        secretName: db-credentials
```

Files appear at:

- `/etc/secrets/username`
- `/etc/secrets/password`

## ConfigMap vs Secret

| ConfigMap              | Secret                                |
| ---------------------- | ------------------------------------- |
| Non-sensitive data     | Sensitive data                        |
| Plain text in etcd     | Base64 encoded (configure encryption) |
| Config files, settings | Passwords, API keys, certificates     |
| Visible in kubectl get | Hidden by default in kubectl get      |

## Secret Types

```bash
# Generic/Opaque (most common)
kubectl create secret generic my-secret --from-literal=key=value

# Docker registry credentials
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.com \
  --docker-username=user \
  --docker-password=pass

# TLS certificates
kubectl create secret tls tls-secret \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key

# SSH keys
kubectl create secret generic ssh-key \
  --from-file=ssh-privatekey=~/.ssh/id_rsa
```

## Docker Compose Comparison

### Docker Compose

```yaml
version: '3.8'
services:
  app:
    image: myapp:v1
    environment:
      - DATABASE_HOST=postgres
      - DATABASE_PORT=5432
    env_file:
      - .env
    secrets:
      - db_password

secrets:
  db_password:
    file: ./db_password.txt
```

### Kubernetes

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres"
  DATABASE_PORT: "5432"
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  db_password: "secretpass123"
---
# Deployment using both
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:v1
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: db_password
```

## Updating Configuration

### ConfigMaps/Secrets Don't Auto-Reload

When you update a ConfigMap or Secret, **Pods don't automatically restart**.

**Options:**

1. **Restart Pods manually:** `kubectl rollout restart deployment/myapp`
2. **Use volume mounts:** Files update automatically (with delay)
3. **Use tools:** Reloader, Stakater, etc.

```bash
# Update ConfigMap
kubectl edit configmap app-config

# Restart Deployment to pick up changes
kubectl rollout restart deployment/myapp
```

## Immutable ConfigMaps/Secrets

Prevent accidental changes:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
immutable: true
data:
  key: value
```

Once created, cannot be modified. Must delete and recreate.

## Best Practices

1. **Use Secrets for sensitive data** - never ConfigMaps
2. **Don't commit Secrets to Git** - use sealed-secrets or external-secrets
3. **Use volume mounts for large configs** - env vars have size limits
4. **Version your ConfigMaps/Secrets** - app-config-v1, app-config-v2
5. **Limit Secret access with RBAC** - not all Pods should access all Secrets
6. **Enable encryption at rest** for Secrets in production
7. **Use stringData for readability** when creating Secrets manually

## Lab: ConfigMaps and Secrets

**Time:** 15 minutes

Practice creating and using ConfigMaps and Secrets.

See [lab/instructions.md](lab/instructions.md)

## Examples

Check [examples/](examples/) for sample manifests.

## Key Takeaways

- **ConfigMaps** for non-sensitive configuration
- **Secrets** for sensitive data (passwords, keys)
- **Multiple ways to consume:** environment variables or volume mounts
- **Updates don't auto-reload** - restart Pods or use volumes
- **Never commit Secrets to version control**

## Next Section

Now we can configure applications. Let's add persistent storage!

**Next:** [07-storage - Storage in Kubernetes](../07-storage/README.md)

## Check Your Understanding

1. When would you use a ConfigMap vs a Secret?
2. How do you use a ConfigMap as environment variables in a Pod?
3. Are Secrets encrypted by default?
4. What happens when you update a ConfigMap used by running Pods?
5. How can you make a Secret immutable?

<details>
<summary>Click for answers</summary>

1. **ConfigMap: non-sensitive config (settings, feature flags). Secret: sensitive data (passwords, API keys)**
2. **Use `envFrom` with `configMapRef` or `env` with `valueFrom.configMapKeyRef`**
3. **No, only base64 encoded Enable encryption at rest for production**
4. **Running Pods won't pick up changes unless restarted (volumes update with delay)**
5. **Set `immutable: true` in the Secret manifest**

</details>
