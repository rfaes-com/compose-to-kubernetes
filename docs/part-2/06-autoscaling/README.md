# Autoscaling

Duration: 45 minutes (20 minutes theory + 25 minutes lab)

## Introduction

Kubernetes can automatically scale applications based on demand, optimizing resource usage and ensuring performance.

**Three Types of Autoscaling:**

1. **Horizontal Pod Autoscaler (HPA)** - Scales number of pods
2. **Vertical Pod Autoscaler (VPA)** - Adjusts CPU/memory requests
3. **Cluster Autoscaler** - Adds/removes nodes

## Horizontal Pod Autoscaler (HPA)

Automatically scales pods based on metrics.

### Metrics Server

Required for HPA to work:

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top nodes
kubectl top pods
```

### Basic HPA Example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Max
```

**How it works:**

1. HPA checks metric every 15 seconds (default)
2. Calculates desired replicas: `ceil(current * (current_metric / target_metric))`
3. Scales pods if needed
4. Waits for stabilization before next scale event

### CPU-Based Autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Memory-Based Autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: memory-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Multiple Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

**Note:** HPA uses the metric that requires the most replicas.

## Custom Metrics

Scale based on application-specific metrics.

### Prometheus Adapter

Expose Prometheus metrics to HPA:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --set prometheus.url=http://prometheus-server.monitoring.svc \
  --namespace monitoring
```

Configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_total$"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

HPA using custom metric:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

## KEDA - Event-Driven Autoscaling

Scale based on external metrics (queues, databases, etc.).

### Installing KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```

### Scaling Based on Queue Length

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaler
spec:
  scaleTargetRef:
    name: worker-deployment
  minReplicaCount: 1
  maxReplicaCount: 30
  triggers:
  - type: rabbitmq
    metadata:
      queueName: tasks
      queueLength: "5"
      host: amqp://rabbitmq.default.svc.cluster.local:5672
```

### Scaling Based on Kafka Lag

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-scaler
spec:
  scaleTargetRef:
    name: consumer-deployment
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.kafka:9092
      consumerGroup: my-consumer-group
      topic: events
      lagThreshold: "100"
```

### Scaling Based on Cron

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cron-scaler
spec:
  scaleTargetRef:
    name: batch-job
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
  - type: cron
    metadata:
      timezone: America/New_York
      start: 0 8 * * *
      end: 0 18 * * *
      desiredReplicas: "10"
```

KEDA Supports 50+ scalers:

- AWS SQS, CloudWatch
- Azure Queue, Service Bus
- GCP Pub/Sub
- Prometheus
- PostgreSQL
- MongoDB
- And many more

## Vertical Pod Autoscaler (VPA)

Automatically adjusts CPU/memory requests.

### Installing VPA

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

### VPA Example

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"  # Or "Off", "Initial", "Recreate"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 1
        memory: 1Gi
      controlledResources: ["cpu", "memory"]
```

**Update Modes:**

- **Off** - Only recommendations, no changes
- **Initial** - Set on pod creation only
- **Recreate** - Delete and recreate pods with new requests
- **Auto** - Update in-place (experimental)

**Note:** VPA and HPA on the same metric can conflict. Use VPA for CPU/memory requests, HPA for replica count.

## Cluster Autoscaler

Automatically adds or removes nodes based on pod resource requests.

### How It Works

1. Pods pending due to insufficient resources → Add nodes
2. Node underutilized for 10+ minutes → Remove node
3. Respects PodDisruptionBudgets during scale-down

### Cloud Provider Integration

```yaml
# AWS
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-config
data:
  config.yaml: |
    autoDiscovery:
      clusterName: my-cluster
      tags:
      - k8s.io/cluster-autoscaler/enabled
      - k8s.io/cluster-autoscaler/my-cluster
```

### Node Autoscaler Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.28.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
```

**Note:** kind clusters don't support Cluster Autoscaler since nodes are containers.

## Autoscaling Best Practices

1. **Set resource requests** - HPA and CA use requests for decisions
2. **Readiness probes** - Required for HPA to know when pods are ready
3. **PodDisruptionBudgets** - Prevent too many pods down during scale-down
4. **Gradual scaling** - Use behavior policies to avoid thrashing
5. **Monitor carefully** - Watch for scaling loops
6. **Cost awareness** - Set max replicas/nodes to control costs
7. **Test scaling** - Load test to verify behavior
8. **Stabilization windows** - Allow time between scale events
9. **Right-size requests** - VPA can help find optimal resource requests
10. **Multiple metrics** - Use custom metrics for accuracy

## PodDisruptionBudget

Ensure availability during voluntary disruptions:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  # Or use maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

Prevents:

- Cluster Autoscaler from draining nodes with critical pods
- kubectl drain from removing too many pods
- Disruptions during HPA scale-down

## Scaling Calculations

### HPA Formula

```text
desiredReplicas = ceil(currentReplicas * (currentMetric / targetMetric))
```

Example:

- Current: 4 replicas
- Current CPU: 90%
- Target CPU: 70%
- Desired = ceil(4 * (90 / 70)) = ceil(5.14) = 6 replicas

### Multiple Metrics

HPA selects the highest replica count from all metrics:

- CPU suggests 6 replicas
- Memory suggests 4 replicas
- Custom metric suggests 8 replicas
- **Result: 8 replicas** (maximum)

## Monitoring Autoscaling

```bash
# HPA status
kubectl get hpa myapp-hpa

# HPA details
kubectl describe hpa myapp-hpa

# HPA events
kubectl get events --field-selector involvedObject.name=myapp-hpa

# VPA recommendations
kubectl describe vpa myapp-vpa

# Cluster Autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler
```

## Advanced HPA Behavior

### Scale-Down Stabilization

Prevent flapping when metrics oscillate:

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # Wait 5 min before scale down
    policies:
    - type: Percent
      value: 50
      periodSeconds: 60  # Max 50% pods removed per minute
```

### Scale-Up Policies

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0  # Scale up immediately
    policies:
    - type: Percent
      value: 100
      periodSeconds: 30  # Double pods every 30 sec
    - type: Pods
      value: 4
      periodSeconds: 60  # Add max 4 pods per minute
    selectPolicy: Max  # Use policy that scales fastest
```

## Key takeaways

- **HPA scales the number of Pod replicas** based on CPU, memory, or custom metrics automatically
- **VPA adjusts resource requests and limits** for individual Pods, optimising resource usage without changing replica count
- **Cluster Autoscaler adds or removes nodes** when Pods cannot be scheduled or nodes are underutilised
- **KEDA enables event-driven autoscaling** from external sources like queues, databases, and HTTP traffic
- **PodDisruptionBudgets protect availability** during scaling events by limiting the number of Pods that can be removed simultaneously

## Check your understanding

1. What is the difference between HPA and VPA?
2. What metrics does HPA use by default, and how can you extend it?
3. At what point does the Cluster Autoscaler add a new node?
4. What Kubernetes resource prevents too many Pods of a deployment from being evicted at once?
5. Why might you use KEDA instead of the standard HPA?

<details class="solution" markdown="1">
<summary>Solution</summary>

1. **HPA scales the number of replicas horizontally; VPA adjusts the CPU/memory resource requests of existing Pods vertically**
2. **CPU utilisation and memory utilisation by default; you can extend it with custom metrics via the Custom Metrics API or with KEDA**
3. **When a Pod cannot be scheduled because no existing node has sufficient available resources**
4. **PodDisruptionBudget (PDB)**
5. **KEDA supports scaling to zero and reacting to external event sources (e.g. queue depth, database rows) that the standard HPA cannot natively handle**

</details>

## Hands-on

Apply the concepts from this section in the [lab](lab.md) exercises.

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../07-security/README.md).

## Further reading

- [HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [KEDA Documentation](https://keda.sh/docs/)
- [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
