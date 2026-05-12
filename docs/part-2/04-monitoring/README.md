# Monitoring and Logging

Duration: 45 minutes (20 min theory + 25 min lab)

## Introduction

Observability is critical for production Kubernetes clusters. You need to monitor metrics, collect logs, and trace requests across services.

**Three Pillars of Observability:**

1. **Metrics** - Numeric measurements over time (CPU, memory, request rate)
2. **Logs** - Discrete events (errors, warnings, debug info)
3. **Traces** - Request flow through distributed systems

## Monitoring Stack

**Popular Stack:**

- **Prometheus** - Metrics collection and storage
- **Grafana** - Visualization and dashboards
- **Loki** - Log aggregation
- **Jaeger/Tempo** - Distributed tracing

## Prometheus

Time-series database and monitoring system:

**Architecture:**

```mermaid
flowchart LR
    Targets["Targets<br/>(exporters)"]

    Prometheus["Prometheus<br/>(metrics)"]

    Grafana["Grafana<br/>(dashboards)"]

    Prometheus -->|scrape| Targets
    Grafana -->|query| Prometheus
```

**Key Concepts:**

- **Scraping** - Prometheus pulls metrics from endpoints
- **Targets** - Services exposing /metrics endpoint
- **ServiceMonitor** - CRD defining what to scrape
- **AlertManager** - Handles alert routing

### Installing Prometheus Stack

```bash
# Add Prometheus Community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

This installs:

- Prometheus Operator
- Prometheus server
- Grafana
- AlertManager
- Node Exporter
- Kube State Metrics

## Prometheus Metrics

Four metric types:

1. **Counter** - Only increases (requests_total)
2. **Gauge** - Can increase or decrease (memory_usage)
3. **Histogram** - Samples observations (request_duration)
4. **Summary** - Similar to histogram, calculates quantiles

### Exposing Metrics

Simple Go metrics endpoint:

```go
import "github.com/prometheus/client_golang/prometheus/promhttp"

http.Handle("/metrics", promhttp.Handler())
```

Example metrics output:

```text
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234
http_requests_total{method="POST",status="201"} 567

# HELP memory_usage_bytes Current memory usage
# TYPE memory_usage_bytes gauge
memory_usage_bytes 1048576000
```

## ServiceMonitor

Tells Prometheus what to scrape:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-metrics
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

Corresponding Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
  - name: metrics
    port: 9090
```

## PromQL Basics

Prometheus Query Language:

```promql
# Current value
http_requests_total

# Filter by labels
http_requests_total{method="GET"}

# Rate over time
rate(http_requests_total[5m])

# Aggregation
sum(rate(http_requests_total[5m])) by (method)

# Percentiles
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

## Alerting

### PrometheusRule

Define alert conditions:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
spec:
  groups:
  - name: myapp
    interval: 30s
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m]))
        / sum(rate(http_requests_total[5m]))
        > 0.05
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value | humanizePercentage }}"

    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
```

### AlertManager Configuration

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
type: Opaque
stringData:
  alertmanager.yaml: |
    global:
      slack_api_url: 'https://hooks.slack.com/services/XXX'

    route:
      receiver: 'slack-notifications'
      group_by: ['alertname', 'cluster']
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 12h
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty'

    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: 'YOUR_KEY'
```

## Grafana Dashboards

Access Grafana:

```bash
# Get Grafana password
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode

# Port-forward to Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

# Open http://localhost:3000
# Username: admin
# Password: (from above command)
```

### Creating Dashboard

```json
{
  "dashboard": {
    "title": "My Application",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (method)"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m]))"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

## Loki for Logs

Loki is like Prometheus for logs - it indexes labels, not content.

### Installing Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set grafana.enabled=false
```

### LogQL Queries

```logql
# All logs from namespace
{namespace="production"}

# Filter by pod
{namespace="production", pod="myapp-xxx"}

# Search content
{namespace="production"} |= "error"

# Regex filter
{namespace="production"} |~ "error|exception"

# Rate of logs
rate({namespace="production"}[5m])

# Count by level
sum(count_over_time({namespace="production"} |= "error" [5m])) by (pod)
```

### Promtail Configuration

Promtail ships logs to Loki:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080

    clients:
      - url: http://loki:3100/loki/api/v1/push

    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_container_name]
            target_label: container
```

## Distributed Tracing

Track requests across microservices:

**Jaeger Architecture:**

```text
App → Agent → Collector → Storage → Query UI
```

### Installing Jaeger

```bash
kubectl create namespace observability
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.42.0/jaeger-operator.yaml -n observability

# Create Jaeger instance
kubectl apply -f - <<EOF
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: allInOne
  ingress:
    enabled: false
  allInOne:
    image: jaegertracing/all-in-one:latest
    options:
      log-level: debug
  storage:
    type: memory
    options:
      memory:
        max-traces: 100000
EOF
```

## Kubernetes Metrics

Built-in metrics:

```bash
# Node metrics
kubectl top nodes

# Pod metrics
kubectl top pods

# Get metrics from metrics-server
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
```

## Best Practices

1. **Label consistently** - Use standard labels (app, version, env)
2. **Alert on symptoms** - Not causes (high error rate, not high CPU)
3. **Use recording rules** - Pre-compute expensive queries
4. **Set retention** - Balance storage vs history (15-30 days typical)
5. **Monitor the monitors** - Alert if Prometheus is down
6. **Dashboard organization** - Separate operational vs business metrics
7. **Log levels** - Use appropriate levels (ERROR, WARN, INFO, DEBUG)
8. **Sampling** - Sample traces in high-traffic scenarios
9. **Cardinality** - Avoid high-cardinality labels (user IDs, timestamps)
10. **SLOs/SLIs** - Define service level objectives

## Common Metrics to Monitor

**Infrastructure:**

- Node CPU/Memory usage
- Pod restarts
- Disk usage
- Network I/O

**Application:**

- Request rate, errors, duration (RED method)
- Queue depth
- Database connection pool
- Cache hit rate

**Business:**

- User signups
- Transaction value
- Feature usage

## Next Steps

- Complete hands-on lab in `lab/instructions.md`
- Create custom dashboards in Grafana
- Set up alert rules
- Implement distributed tracing
- Configure log aggregation

## Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)
