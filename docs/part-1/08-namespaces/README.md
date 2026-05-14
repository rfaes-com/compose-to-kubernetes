# Namespaces and Resource Quotas

**Duration:** 55 minutes (30 minutes theory + 25 minutes lab)

## Learning Objectives

- Understand Kubernetes Namespaces for resource isolation
- Create and manage multiple namespaces
- Use ResourceQuotas to limit resource consumption
- Apply LimitRanges for default resource limits
- Implement NetworkPolicies for namespace isolation
- Learn namespace best practices

## Namespaces Concept

Namespaces provide a way to divide cluster resources between multiple users, teams, or applications.

### Default Namespaces

```bash
kubectl get namespaces
```

**Default namespaces:**

- **default**: Resources with no namespace specified
- **kube-system**: Kubernetes system components
- **kube-public**: Publicly readable resources
- **kube-node-lease**: Node heartbeat information

### When to Use Namespaces

**Use namespaces for:**

- Multi-tenant environments (dev, staging, production)
- Team isolation (team-a, team-b)
- Application separation (frontend, backend, database)
- Resource quota enforcement
- Access control via RBAC

**Don't need namespaces for:**

- Small clusters with few resources
- Single application/team
- When simple labels are sufficient

## Creating and Using Namespaces

### Create Namespace

```bash
# Using kubectl
kubectl create namespace development

# Using YAML
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
EOF
```

### Working with Namespaces

```bash
# List resources in specific namespace
kubectl get pods -n development

# Create resource in namespace
kubectl run nginx --image=nginx:1.25-alpine -n development

# Delete namespace (deletes all resources in it!)
kubectl delete namespace development
```

### Set Default Namespace

```bash
# Set default namespace for current context
kubectl config set-context --current --namespace=development

# Verify
kubectl config view --minify | grep namespace:
```

## ResourceQuotas

Limit total resource consumption per namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    # Compute resources
    requests.cpu: "10"           # Total CPU requests
    requests.memory: 20Gi        # Total memory requests
    limits.cpu: "20"             # Total CPU limits
    limits.memory: 40Gi          # Total memory limits

    # Object counts
    pods: "20"                   # Max number of Pods
    services: "10"               # Max number of Services
    persistentvolumeclaims: "5"  # Max number of PVCs

    # Storage
    requests.storage: 100Gi      # Total storage requests
```

**Important:** When ResourceQuota is active for CPU/memory, all Pods must specify requests and limits.

### Check Quota Usage

```bash
kubectl get resourcequota -n development
kubectl describe resourcequota compute-quota -n development
```

## LimitRange

Define default and min/max resource constraints for pods and containers.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: development
spec:
  limits:
  # Container limits
  - type: Container
    default:  # Default limits (if not specified)
      cpu: 500m
      memory: 512Mi
    defaultRequest:  # Default requests (if not specified)
      cpu: 200m
      memory: 256Mi
    max:  # Maximum allowed
      cpu: "2"
      memory: 2Gi
    min:  # Minimum required
      cpu: 100m
      memory: 128Mi

  # Pod limits (sum of all containers)
  - type: Pod
    max:
      cpu: "4"
      memory: 4Gi

  # PVC limits
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 10Gi
```

**How it works:**

1. When Pod is created, LimitRange is consulted
2. If resources not specified, defaults are applied
3. If resources exceed max or below min, Pod is rejected

## Example: Multi-Environment Setup

### Development Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    env: development
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "1"
      memory: 1Gi
```

### Production Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    max:
      cpu: "4"
      memory: 8Gi
    min:
      cpu: 100m
      memory: 64Mi
```

## Cross-Namespace Communication

Pods can communicate across namespaces using fully qualified DNS names:

```text
<service-name>.<namespace>.svc.cluster.local
```

Example:

```bash
# Service in 'backend' namespace
kubectl create service clusterip api --tcp=8080:8080 -n backend

# Access from 'frontend' namespace
curl http://api.backend.svc.cluster.local:8080
```

## NetworkPolicies for Namespace Isolation

Restrict traffic between namespaces (requires CNI plugin support like Calico or Cilium).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: production
spec:
  podSelector: {}  # Apply to all pods in namespace
  policyTypes:
  - Ingress
  ingress:
  # Allow from same namespace
  - from:
    - podSelector: {}
  # Allow from specific namespace
  - from:
    - namespaceSelector:
        matchLabels:
          env: production
```

**Note:** kind doesn't include NetworkPolicy support by default. You'd need to install a CNI plugin.

## Namespace Management Best Practices

1. **Use namespaces for environments**: dev, staging, production
2. **Apply ResourceQuotas**: Prevent resource exhaustion
3. **Set LimitRanges**: Provide defaults, prevent extreme requests
4. **Label namespaces**: For organization and selection
5. **Use RBAC**: Control who can access each namespace
6. **Document naming conventions**: team-name-env, app-name-env
7. **Monitor quota usage**: Alert before hitting limits
8. **Clean up unused namespaces**: Delete to free resources
9. **Avoid excessive namespaces**: Too many can be hard to manage
10. **Use NetworkPolicies**: Isolate sensitive workloads

## Common Patterns

### Pattern 1: Environment Isolation

```text
development/
  - all features in development
staging/
  - release candidates
production/
  - live applications
```

### Pattern 2: Team Isolation

```text
team-alpha/
  - Team Alpha's applications
team-beta/
  - Team Beta's applications
shared-services/
  - Common services (monitoring, logging)
```

### Pattern 3: Application Isolation

```text
frontend/
backend/
database/
cache/
```

## Troubleshooting

```bash
# Check quota status
kubectl describe quota -n <namespace>

# Check why pod was rejected
kubectl describe pod <pod> -n <namespace>
# Look for: "exceeded quota" or "violates LimitRange"

# View current resource usage
kubectl top pods -n <namespace>
kubectl top nodes

# List all resources in namespace
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -n <namespace>
```

## Docker Compose to Kubernetes: Namespaces

Docker Compose doesn't have a direct equivalent to namespaces, but conceptually:

**Docker Compose:**

- Projects (docker-compose.yml in different directories)
- Each project has its own network and volumes
- Isolation through project names

**Kubernetes:**

- Namespaces provide resource isolation
- Resources can't have same name in namespace
- More sophisticated with quotas and RBAC

## Key Differences from Docker Compose

| Feature         | Docker Compose        | Kubernetes                          |
| --------------- | --------------------- | ----------------------------------- |
| Isolation       | Project names         | Namespaces                          |
| Resource Limits | Per container         | Per namespace (quotas)              |
| DNS             | service-name          | service.namespace.svc.cluster.local |
| Default         | "default" project     | "default" namespace                 |
| Deletion        | `docker-compose down` | `kubectl delete namespace <name>`   |

## Examples

Check [examples/](examples/index.md) for sample manifests.

## Key takeaways

- Namespaces provide logical isolation within a cluster
- ResourceQuotas limit total resource consumption per namespace
- LimitRanges provide defaults and boundaries for individual resources
- Pods can communicate across namespaces using FQDN
- Deleting a namespace deletes all resources within it
- Use namespaces for environments, teams, or applications

## Check your understanding

1. What is the purpose of Kubernetes namespaces?
2. What happens when you delete a namespace?
3. How do ResourceQuotas differ from LimitRanges?
4. How can a Pod in namespace 'frontend' access a Service in namespace 'backend'?
5. Why must Pods specify resource requests/limits when ResourceQuota is active?

<details class="solution" markdown="1">
<summary>Solution</summary>

1. **Namespaces provide logical isolation — grouping resources, applying quotas, and controlling access by team/environment**
2. **All resources in the namespace are deleted along with it — including Pods, Services, PVCs, etc.**
3. **ResourceQuota limits the total consumption for a namespace; LimitRange sets defaults and min/max for individual containers**
4. **Use the FQDN: `http://service-name.backend.svc.cluster.local`**
5. **If ResourceQuota sets CPU/memory limits for the namespace, every Pod must specify requests/limits so Kubernetes can track usage correctly**

</details>

## Hands-on

Apply the concepts from this section in the [lab](lab.md) exercises.

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../09-tools/README.md)
