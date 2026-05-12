# Advanced Deployment Strategies

Duration: 40 minutes (20 min theory + 20 min lab)

## Introduction

Basic deployments with rolling updates work for many scenarios, but production systems often require more sophisticated deployment strategies to minimize risk and downtime.

## Deployment Strategies Overview

| Strategy           | Description               | Downtime | Risk     | Rollback Speed |
| ------------------ | ------------------------- | -------- | -------- | -------------- |
| **Recreate**       | Stop all, deploy new      | Yes      | High     | Slow           |
| **Rolling Update** | Gradual replacement       | No       | Medium   | Medium         |
| **Blue/Green**     | Two complete environments | No       | Low      | Instant        |
| **Canary**         | Gradual traffic shift     | No       | Very Low | Instant        |
| **A/B Testing**    | Feature-based routing     | No       | Low      | Instant        |

## Rolling Updates (Advanced)

### Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Max pods above desired count
      maxUnavailable: 1  # Max pods unavailable during update
  minReadySeconds: 30    # Wait before considering pod ready
  progressDeadlineSeconds: 600  # Timeout for deployment progress
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

**Best Practices:**

- Set `maxUnavailable` to 0 for critical services
- Use `maxSurge` to speed up deployments
- Always configure readiness probes
- Set appropriate `minReadySeconds`

## Blue/Green Deployments

Deploy new version alongside old, then switch traffic instantly.

### Implementation with Services

```yaml
# Blue deployment (v1 - current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:v1
---
# Green deployment (v2 - new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:v2
---
# Service points to blue initially
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # Switch to 'green' when ready
  ports:
  - port: 80
    targetPort: 8080
```

**Switching Process:**

1. Deploy green environment
2. Test green environment internally
3. Update Service selector to point to green
4. Monitor for issues
5. Keep blue running for quick rollback
6. Delete blue after success confirmation

### Advantages:

- Instant switch and rollback
- Full testing before switch
- Zero downtime

### Disadvantages:

- Double resources needed
- Database migrations can be tricky
- Session handling between versions

## Canary Deployments

Gradually shift traffic to new version while monitoring.

### Using Labels and Multiple Services

```yaml
# Stable deployment (v1 - 90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: app
        image: myapp:v1
---
# Canary deployment (v2 - 10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: app
        image: myapp:v2
---
# Service routes to both (weighted by replicas)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # Matches both stable and canary
  ports:
  - port: 80
    targetPort: 8080
```

**Traffic Distribution:**

- 9 stable pods + 1 canary pod = ~10% to canary
- Gradually increase canary replicas
- Monitor metrics and errors
- Rollback by deleting canary deployment
- Promote by switching stable to v2

### Progressive Canary

```text
1. Deploy canary (1 replica) - 10% traffic
2. Monitor metrics for 10 minutes
3. If healthy, scale to 3 replicas - 30% traffic
4. Monitor for 20 minutes
5. If healthy, scale to 5 replicas - 50% traffic
6. Monitor for 30 minutes
7. Promote canary to stable
8. Scale down old stable
```

## Flagger for Progressive Delivery

Flagger automates canary deployments:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
    webhooks:
    - name: load-test
      url: http://flagger-loadtester/
```

**Flagger Process:**

1. Detects new image in deployment
2. Creates canary deployment
3. Routes initial traffic (10%)
4. Monitors metrics every minute
5. Increases traffic by 10% if healthy
6. Promotes to stable if all checks pass
7. Rolls back automatically if metrics fail

## A/B Testing

Route traffic based on user attributes, not just percentage.

### Using Ingress Annotations

```yaml
# Baseline deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1
---
# New feature deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
---
# Main Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-main
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: myapp-v1
            port:
              number: 80
---
# A/B Test Ingress (canary based on cookie)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ab
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-cookie: "beta_user"
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: myapp-v2
            port:
              number: 80
```

**Routing Rules:**

- Users with `beta_user=always` cookie → v2
- All other users → v1
- Can also route by header, query param, etc.

## Feature Flags

Control features without deployment:

```go
// In application code
if featureFlags.IsEnabled("new-ui", user) {
    return renderNewUI()
} else {
    return renderOldUI()
}
```

**Tools:**

- LaunchDarkly
- Split.io
- Unleash (open-source)
- ConfigCat

**Kubernetes ConfigMap Integration:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  flags.json: |
    {
      "new-ui": {
        "enabled": true,
        "rollout": 25,
        "users": ["beta@example.com"]
      }
    }
```

## Shadow Deployments

Run new version in parallel, compare responses (don't return to users).

```yaml
# Traffic Mirroring with Istio
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp-v1
      weight: 100
    mirror:
      host: myapp-v2
    mirrorPercentage:
      value: 100.0
```

**Use Cases:**

- Performance testing with production load
- Comparing response differences
- Testing new algorithms

## Deployment Health Checks

### Readiness vs Liveness vs Startup

```yaml
spec:
  containers:
  - name: app
    # Startup probe (for slow-starting apps)
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      failureThreshold: 30
      periodSeconds: 10

    # Liveness probe (restart if failing)
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3

    # Readiness probe (remove from service if failing)
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
```

## Deployment Best Practices

1. **Always use readiness probes** - Prevent traffic to unhealthy pods
2. **Set appropriate timeouts** - Don't rush deployments
3. **Monitor during rollout** - Watch error rates, latency
4. **Automate rollback** - Use metrics-based triggers
5. **Test thoroughly** - Use staging environment
6. **Database migrations** - Handle schema changes carefully
7. **Session management** - Use external session store
8. **Gradual rollout** - Start small, increase slowly
9. **Document process** - Clear runbooks for rollback
10. **Practice rollbacks** - Test rollback procedures regularly

## Deployment Metrics to Monitor

- **Error rate** - 5xx responses
- **Latency** - p50, p95, p99 response times
- **Throughput** - Requests per second
- **Saturation** - CPU, memory usage
- **Pod restarts** - CrashLoopBackOff events

## Rollback Strategies

### kubectl rollout

```bash
# Check rollout status
kubectl rollout status deployment/myapp

# View history
kubectl rollout history deployment/myapp

# Rollback to previous
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=2

# Pause rollout
kubectl rollout pause deployment/myapp

# Resume rollout
kubectl rollout resume deployment/myapp
```

## Next Steps

- Complete hands-on lab in `lab/instructions.md`
- Implement blue/green deployment
- Try canary with traffic splitting
- Install Flagger for automated progressive delivery
- Set up deployment monitoring

## Additional Resources

- [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Flagger Documentation](https://docs.flagger.app/)
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)
