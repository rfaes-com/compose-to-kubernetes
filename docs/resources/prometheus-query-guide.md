# Prometheus Query Guide

Quick reference for PromQL (Prometheus Query Language) queries useful for monitoring Kubernetes workloads.

## PromQL Basics

### Data Types

| Type           | Description                                   | Example                   |
| -------------- | --------------------------------------------- | ------------------------- |
| Instant vector | Set of time series with one sample per series | `http_requests_total`     |
| Range vector   | Set of time series with a range of samples    | `http_requests_total[5m]` |
| Scalar         | A single numeric floating-point value         | `1.5`                     |
| String         | A simple string value (rarely used)           | `"hello"`                 |

### Selectors

```promql
# Match exact label value
http_requests_total{job="api"}

# Negative match
http_requests_total{job!="api"}

# Regex match
http_requests_total{job=~"api.*"}

# Negative regex match
http_requests_total{job!~"api.*"}

# Multiple label matchers
http_requests_total{job="api", status="200"}
```

### Time Range

```promql
# Last 5 minutes of samples
http_requests_total[5m]

# Offset (5 minutes ago)
http_requests_total offset 5m

# Range with offset
http_requests_total[5m] offset 1h
```

## Kubernetes Pod Queries

### CPU Usage

```promql
# CPU usage per pod (cores)
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)

# CPU usage as percentage of requested
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)
  /
sum(kube_pod_container_resource_requests{resource="cpu", container!=""}) by (pod)

# Top 10 pods by CPU usage
topk(10, sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace))
```

### Memory Usage

```promql
# Memory working set per pod (bytes)
sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace)

# Memory usage in MiB
sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace) / 1024 / 1024

# Memory usage as percentage of requested
sum(container_memory_working_set_bytes{container!=""}) by (pod)
  /
sum(kube_pod_container_resource_requests{resource="memory", container!=""}) by (pod)

# Top 10 pods by memory usage
topk(10, sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace))
```

### Pod Status

```promql
# Number of running pods per namespace
count(kube_pod_status_phase{phase="Running"}) by (namespace)

# Pods not in Running or Succeeded state
count(kube_pod_status_phase{phase!~"Running|Succeeded"}) by (pod, namespace, phase)

# Pod restarts in the last hour
increase(kube_pod_container_status_restarts_total[1h]) > 0

# OOMKilled pods
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}
```

## Kubernetes Deployment Queries

```promql
# Desired vs available replicas
kube_deployment_spec_replicas
kube_deployment_status_replicas_available

# Deployments with unavailable replicas
kube_deployment_status_replicas_unavailable > 0

# Deployment rollout progress
kube_deployment_status_replicas_updated / kube_deployment_spec_replicas

# Replica mismatch (desired != available)
kube_deployment_spec_replicas != kube_deployment_status_replicas_available
```

## Kubernetes Node Queries

### Node CPU

```promql
# Node CPU utilization (%)
100 - (avg by (node) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# CPU usage per node
sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (node)
```

### Node Memory

```promql
# Node memory utilization (%)
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Available memory per node
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024    # GiB

# Memory pressure
node_memory_MemAvailable_bytes < 0.1 * node_memory_MemTotal_bytes
```

### Node Disk

```promql
# Disk usage per node filesystem (%)
100 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes * 100)

# Disk I/O
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])
```

### Node Network

```promql
# Network receive rate per node
rate(node_network_receive_bytes_total{device!~"lo|veth.*"}[5m])

# Network transmit rate per node
rate(node_network_transmit_bytes_total{device!~"lo|veth.*"}[5m])
```

## HTTP / Application Metrics

### Request Rate

```promql
# Total request rate (per second)
sum(rate(http_requests_total[5m]))

# Request rate by status code
sum(rate(http_requests_total[5m])) by (status_code)

# Error rate (5xx)
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# Error rate by service
sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
  /
sum(rate(http_requests_total[5m])) by (service)
```

### Latency

```promql
# 50th percentile (median) latency
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# 95th percentile latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# 99th percentile latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Average latency
sum(rate(http_request_duration_seconds_sum[5m]))
  /
sum(rate(http_request_duration_seconds_count[5m]))

# Latency by service
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)
```

### Apdex Score

```promql
# Apdex with 300ms satisfying threshold, 1.2s tolerating threshold
(
  sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
  +
  sum(rate(http_request_duration_seconds_bucket{le="1.2"}[5m]))
) / 2
  /
sum(rate(http_request_duration_seconds_count[5m]))
```

## SLI / SLO Queries

```promql
# Availability SLI (success rate over 30 days)
sum_over_time(up[30d]) / count_over_time(up[30d])

# Error budget remaining
1 - (
  sum(rate(http_requests_total{status_code=~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
) / (1 - 0.999)     # 99.9% SLO

# Burn rate (how fast we're consuming error budget)
sum(rate(http_requests_total{status_code=~"5.."}[1h]))
  /
sum(rate(http_requests_total[1h]))
  /
(1 - 0.999)
```

## Kubernetes Resource Quotas

```promql
# CPU quota usage per namespace
sum(kube_pod_container_resource_requests{resource="cpu"}) by (namespace)
  /
sum(kube_resourcequota{resource="requests.cpu", type="hard"}) by (namespace)

# Memory quota usage per namespace
sum(kube_pod_container_resource_requests{resource="memory"}) by (namespace)
  /
sum(kube_resourcequota{resource="requests.memory", type="hard"}) by (namespace)
```

## Ingress Metrics (NGINX)

```promql
# Request rate through Ingress
sum(rate(nginx_ingress_controller_requests[5m])) by (ingress, namespace)

# Error rate through Ingress
sum(rate(nginx_ingress_controller_requests{status=~"[45].."}[5m])) by (ingress)
  /
sum(rate(nginx_ingress_controller_requests[5m])) by (ingress)

# Ingress latency (p99)
histogram_quantile(0.99,
  sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) by (le, ingress)
)

# Active connections
avg(nginx_ingress_controller_nginx_process_connections{state="active"})
```

## Alerting Rules Examples

```yaml
groups:
  - name: kubernetes
    rules:
      # Pod crash looping
      - alert: PodCrashLooping
        expr: |
          increase(kube_pod_container_status_restarts_total[1h]) > 5
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
          description: "Container {{ $labels.container }} has restarted {{ $value }} times in the last hour"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          (container_memory_working_set_bytes{container!=""}
            / kube_pod_container_resource_limits{resource="memory", container!=""}) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage in {{ $labels.namespace }}/{{ $labels.pod }}"

      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
            /
          sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.service }}: {{ $value | humanizePercentage }}"

      # Node not ready
      - alert: NodeNotReady
        expr: kube_node_status_condition{condition="Ready", status="true"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.node }} is not ready"
```

## Useful Functions

| Function               | Description                              | Example                                             |
| ---------------------- | ---------------------------------------- | --------------------------------------------------- |
| `rate()`               | Per-second rate of increase over a range | `rate(requests_total[5m])`                          |
| `irate()`              | Instantaneous rate (last two samples)    | `irate(requests_total[5m])`                         |
| `increase()`           | Total increase over a range              | `increase(requests_total[1h])`                      |
| `sum()`                | Sum across dimensions                    | `sum(requests_total) by (job)`                      |
| `avg()`                | Average across dimensions                | `avg(cpu_usage) by (node)`                          |
| `max()`                | Maximum across dimensions                | `max(memory_usage) by (pod)`                        |
| `min()`                | Minimum across dimensions                | `min(memory_usage) by (pod)`                        |
| `count()`              | Count of elements                        | `count(up == 1)`                                    |
| `topk()`               | Top k elements                           | `topk(5, requests_total)`                           |
| `bottomk()`            | Bottom k elements                        | `bottomk(5, requests_total)`                        |
| `histogram_quantile()` | Quantile from histogram                  | `histogram_quantile(0.95, ...)`                     |
| `absent()`             | Returns 1 if no data                     | `absent(up{job="api"})`                             |
| `predict_linear()`     | Linear prediction                        | `predict_linear(disk_free[1h], 4*3600)`             |
| `label_replace()`      | Modify label values                      | `label_replace(metric, "dst", "$1", "src", "(.*)")` |
