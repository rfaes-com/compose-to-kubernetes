# YAML Basics

A quick reference for YAML syntax used throughout Kubernetes manifests.

## What is YAML?

**YAML** (YAML Ain't Markup Language) is a human-readable data serialization format commonly used for configuration files. Kubernetes uses YAML to define resources.

## Basic Syntax

### Key-Value Pairs

```yaml
name: nginx
version: 1.21
```

### Nested Objects (Indentation Matters!)

```yaml
metadata:
  name: my-app
  namespace: default
  labels:
    app: web
    tier: frontend
```

**Important:** YAML uses **spaces** for indentation, NOT tabs. Standard is 2 spaces.

### Lists (Arrays)

```yaml
# Method 1: Dash notation
ports:
  - 80
  - 443
  - 8080

# Method 2: Inline (JSON-style)
ports: [80, 443, 8080]
```

### Lists of Objects

```yaml
containers:
  - name: nginx
    image: nginx:latest
    ports:
      - containerPort: 80
  - name: sidecar
    image: busybox:latest
    command: ["sleep", "3600"]
```

## Common Patterns in Kubernetes

### Complete Pod Example

```yaml
apiVersion: v1                    # API version
kind: Pod                         # Resource type
metadata:                         # Metadata section
  name: my-pod
  labels:
    app: web
spec:                            # Specification section
  containers:
    - name: nginx
      image: nginx:1.21
      ports:
        - containerPort: 80
```

### Multi-Document YAML

Use `---` to separate multiple resources in one file:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  key: value
---
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: web
  ports:
    - port: 80
```

## Data Types

### Strings

```yaml
# Unquoted (most common)
name: my-app

# Quoted (use when string contains special characters)
name: "my-app: production"
name: 'special:characters'

# Multi-line strings
description: |
  This is a multi-line string.
  Each line is preserved.
  Including newlines.

# Folded multi-line (newlines become spaces)
description: >
  This is also multi-line
  but will be folded into
  a single line with spaces.
```

### Numbers

```yaml
age: 30                 # Integer
price: 29.99           # Float
port: 8080             # Integer
memory: "512Mi"        # String (Kubernetes resource format)
```

### Booleans

```yaml
# These are all valid boolean true:
enabled: true
enabled: True
enabled: TRUE
enabled: yes

# These are all valid boolean false:
enabled: false
enabled: False
enabled: FALSE
enabled: no
```

## 🚦 Special Characters

### Comments

```yaml
# This is a comment
name: my-app  # Inline comment
# Comments are ignored by parsers
```

### Null Values

```yaml
description: null
# or simply omit the value
description:
```

### Anchors and Aliases (Reusing Values)

```yaml
# Define an anchor with &
defaults: &default-labels
  env: production
  team: platform

# Reference it with *
service1:
  labels:
    <<: *default-labels  # Merge
    app: web

service2:
  labels:
    <<: *default-labels
    app: api

# Results in:
# service1.labels = {env: production, team: platform, app: web}
# service2.labels = {env: production, team: platform, app: api}
```

## Common Mistakes

### 1. Tabs vs Spaces

```yaml
# WRONG - uses tabs (invisible here)
metadata:
  name: my-app

# CORRECT - uses spaces
metadata:
  name: my-app
```

### 2. Incorrect Indentation

```yaml
# WRONG - inconsistent indentation
metadata:
  name: my-app
   labels:
    app: web

# CORRECT - consistent 2-space indentation
metadata:
  name: my-app
  labels:
    app: web
```

### 3. List Item Indentation

```yaml
# WRONG - list items not aligned
containers:
- name: nginx
  image: nginx
- name: redis
  image: redis

# CORRECT - list items aligned
containers:
  - name: nginx
    image: nginx
  - name: redis
    image: redis
```

### 4. Mixing Formats

```yaml
# WRONG - inconsistent list format
containers:
  - name: nginx
  - image: nginx
    name: redis  # Properties should be grouped

# CORRECT
containers:
  - name: nginx
    image: nginx
  - name: redis
    image: redis
```

### 5. Quotes Around Numbers

```yaml
# Be careful with numbers in strings
port: "8080"    # This is a STRING
port: 8080      # This is a NUMBER

# In Kubernetes, resource values are strings:
memory: "512Mi"      # CORRECT
cpu: "0.5"           # CORRECT
replicas: 3          # CORRECT (integer)
```

## 🔍 Validation Tools

### kubectl (Kubernetes)

```bash
# Validate syntax (doesn't create resources)
kubectl apply -f myfile.yaml --dry-run=client

# Validate against the cluster
kubectl apply -f myfile.yaml --dry-run=server
```

### Online Validators

- [YAML Lint](http://www.yamllint.com/)
- [YAML Validator](https://codebeautify.org/yaml-validator)

### Editor Support

Most modern editors support YAML:

- **VS Code**: YAML extension by Red Hat
- **vim**: Built-in syntax highlighting
- **IntelliJ**: Built-in support

## Kubernetes-Specific Conventions

### Required Fields (Most Resources)

```yaml
apiVersion: v1        # Required
kind: Pod             # Required
metadata:             # Required
  name: my-pod        # Required
spec:                 # Required (for most resources)
  # ... resource-specific fields
```

### Label Naming Conventions

```yaml
metadata:
  labels:
    # Recommended format: domain/name
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-production
    app.kubernetes.io/version: "1.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: myproject

    # Simple format (also valid)
    app: myapp
    tier: frontend
    environment: production
```

### Resource Naming Rules

- Lowercase alphanumeric characters, `-`, or `.`
- Must start and end with alphanumeric character
- Max 253 characters (for most resources)

```yaml
# VALID names
my-app
my.app
my-app-123
web-frontend

# INVALID names
My-App           # uppercase not allowed
my_app           # underscores not allowed
-myapp           # can't start with dash
myapp-           # can't end with dash
```

## Practice Exercise

Try to spot the errors in this YAML:

```yaml
apiVersion: v1
kind: pod
metadata:
  Name: my-pod
  labels:
  app: web
spec:
  containers:
  - name: nginx
  image: nginx
    ports:
    - containerport: 80
```

<details>
<summary>Click to see the corrections</summary>

```yaml
apiVersion: v1
kind: Pod                    # Should be capitalized
metadata:
  name: my-pod              # Should be lowercase 'name'
  labels:
    app: web                # Needs proper indentation
spec:
  containers:
    - name: nginx           # Proper list indentation
      image: nginx          # Same level as 'name'
      ports:
        - containerPort: 80  # Proper camelCase
```

Errors:

1. `kind: pod` → should be `Pod` (capitalized)
2. `Name:` → should be `name:` (lowercase)
3. `app: web` → incorrect indentation under labels
4. `containers` list items → inconsistent indentation
5. `containerport` → should be `containerPort` (camelCase)

</details>

## Additional Resources

- [Official YAML Specification](https://yaml.org/spec/)
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)
- [kubectl Cheatsheet](../kubectl-cheatsheet.md)

**Pro Tip:** Use `kubectl explain` to see the structure of any Kubernetes resource:

```bash
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
```
