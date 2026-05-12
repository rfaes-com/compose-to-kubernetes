# Lab: Monitoring and Logging

**Duration:** 25 minutes

## Objectives

- Install a Prometheus and Grafana monitoring stack
- Deploy an application that exposes Prometheus metrics
- Configure Prometheus scraping with a ServiceMonitor
- Query application metrics with PromQL
- Explore Kubernetes metrics in Grafana
- Install Loki and query application logs
- Create and test a Prometheus alert

## Prerequisites

- Kind cluster running
- kubectl configured and working
- Helm 3 installed
- At least 8GB RAM available for the cluster

## Tasks

### Task 1: Install the Monitoring Stack

Install `kube-prometheus-stack` into a `monitoring` namespace.

**Requirements:**

- Add the Prometheus Community Helm repository
- Create the `monitoring` namespace
- Install the chart with release name `monitoring`
- Configure Prometheus to discover ServiceMonitors and PrometheusRules outside the Helm release labels
- Verify that Grafana, Prometheus, AlertManager, node-exporter, and kube-state-metrics are running

### Task 2: Deploy a Metrics-Enabled Application

Deploy `podinfo` into a dedicated namespace and expose both its HTTP traffic and metrics endpoint.

**Requirements:**

- Namespace: `observability-lab`
- Deployment name: `podinfo`
- Image: `ghcr.io/stefanprodan/podinfo:6.5.3`
- Replicas: 2
- Label: `app=podinfo`
- Container port name: `http`
- Container port: 9898
- CPU request: `100m`
- Memory request: `64Mi`
- CPU limit: `250m`
- Memory limit: `128Mi`
- Service name: `podinfo`
- Service port `http`: 80 to target port `http`
- Service port `metrics`: 9797 to target port `http`

### Task 3: Configure Prometheus Scraping

Create a ServiceMonitor so Prometheus discovers the podinfo metrics endpoint.

**Requirements:**

- ServiceMonitor name: `podinfo`
- Namespace: `observability-lab`
- Select the Service with label `app=podinfo`
- Scrape the `metrics` service port
- Scrape path: `/metrics`
- Scrape interval: `15s`
- Confirm the target appears as `UP` in Prometheus

### Task 4: Query Application Metrics

Use Prometheus to inspect the podinfo target and request metrics.

**Requirements:**

- Confirm Prometheus can scrape podinfo
- Generate successful and failed HTTP requests
- Query request rate grouped by status code
- Identify which status code appears after requesting a missing path

### Task 5: Explore Metrics in Grafana

Open Grafana and inspect both built-in Kubernetes dashboards and your application metrics.

**Requirements:**

- Retrieve the Grafana admin password from the `monitoring-grafana` Secret
- Port-forward Grafana to local port 3000
- Open a Kubernetes namespace dashboard for `observability-lab`
- Use Explore to run a PromQL query for podinfo request rate

### Task 6: Install Loki and Query Logs

Install Loki with Promtail and query podinfo logs from Grafana.

**Requirements:**

- Add the Grafana Helm repository
- Install `grafana/loki-stack` into the `monitoring` namespace
- Enable Promtail
- Do not install a second Grafana instance
- Add Loki as a Grafana data source
- Query logs for the `observability-lab` namespace

### Task 7: Create and Test an Alert

Create a PrometheusRule that fires when Prometheus cannot find a podinfo target.

**Requirements:**

- PrometheusRule name: `podinfo-alerts`
- Namespace: `observability-lab`
- Alert name: `PodinfoTargetMissing`
- Expression should detect absence of the podinfo `up` metric
- Alert should become pending/firing after podinfo is scaled to zero
- Restore podinfo to 2 replicas after the test

## Hints

<details>
<summary>Hint for Task 1: Installing kube-prometheus-stack</summary>

Add the chart repository and create the namespace:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring
```

Install the stack with ServiceMonitor and PrometheusRule selector overrides:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.service.type=ClusterIP \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.ruleSelectorNilUsesHelmValues=false
```

Check the Pods:

```bash
kubectl get pods -n monitoring
```

</details>

<details>
<summary>Hint for Task 2: Exposing podinfo and metrics</summary>

Create the namespace first:

```bash
kubectl create namespace observability-lab
```

Your Service should expose two named ports:

```yaml
ports:
- name: http
  port: 80
  targetPort: http
- name: metrics
  port: 9797
  targetPort: http
```

Apply your manifest and verify the Deployment:

```bash
kubectl apply -f podinfo.yaml
kubectl wait --for=condition=Available deployment/podinfo \
  -n observability-lab --timeout=90s
```

Port-forward both service ports:

```bash
kubectl port-forward -n observability-lab svc/podinfo 9898:80 9797:9797
```

In another terminal:

```bash
curl http://localhost:9898
curl http://localhost:9797/metrics | head
```

</details>

<details>
<summary>Hint for Task 3: Creating the ServiceMonitor</summary>

A ServiceMonitor selects a Service by labels and scrapes a named Service port:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: podinfo
  namespace: observability-lab
spec:
  selector:
    matchLabels:
      app: podinfo
  endpoints:
  - port: metrics
```

Port-forward Prometheus:

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Open <http://localhost:9090/targets> and find the `observability-lab/podinfo` target.

</details>

<details>
<summary>Hint for Task 4: Querying podinfo metrics</summary>

Start with the scrape health:

```promql
up{namespace="observability-lab"}
```

Then query request rate by HTTP status:

```promql
sum(rate(http_requests_total{namespace="observability-lab"}[1m])) by (status)
```

Generate traffic through a port-forward:

```bash
kubectl port-forward -n observability-lab svc/podinfo 9898:80
```

In another terminal:

```bash
for i in $(seq 1 30); do curl -s http://localhost:9898 > /dev/null; done
for i in $(seq 1 5); do curl -s http://localhost:9898/not-found > /dev/null; done
```

</details>

<details>
<summary>Hint for Task 5: Opening Grafana</summary>

Decode the Grafana password:

```bash
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
echo ""
```

Port-forward Grafana:

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Open <http://localhost:3000>.

- Username: `admin`
- Password: the decoded secret value

Try this query in Grafana Explore:

```promql
sum(rate(http_requests_total{namespace="observability-lab"}[1m])) by (status)
```

</details>

<details>
<summary>Hint for Task 6: Installing Loki and querying logs</summary>

Install Loki and Promtail without installing a second Grafana:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set grafana.enabled=false
```

Use this data source URL in Grafana:

```text
http://loki:3100
```

Generate logs through the podinfo port-forward:

```bash
kubectl port-forward -n observability-lab svc/podinfo 9898:80
```

In another terminal:

```bash
for i in $(seq 1 10); do curl -s http://localhost:9898/version; done
for i in $(seq 1 3); do curl -s http://localhost:9898/headers; done
```

Try these LogQL queries:

```logql
{namespace="observability-lab"}
```

```logql
{namespace="observability-lab", app="podinfo"}
```

</details>

<details>
<summary>Hint for Task 7: Alerting when the target disappears</summary>

The expression can use `absent()`:

```promql
absent(up{namespace="observability-lab", service="podinfo"}) == 1
```

Test the alert:

```bash
kubectl apply -f podinfo-alert.yaml
kubectl scale deployment podinfo -n observability-lab --replicas=0
```

Open <http://localhost:9090/alerts> and wait for `PodinfoTargetMissing`.

Restore the application:

```bash
kubectl scale deployment podinfo -n observability-lab --replicas=2
kubectl wait --for=condition=Available deployment/podinfo \
  -n observability-lab --timeout=90s
```

</details>

## Verification

Check your work:

```bash
kubectl get pods -n monitoring
kubectl get pods,svc,servicemonitor,prometheusrule -n observability-lab
kubectl get endpoints podinfo -n observability-lab
```

Prometheus checks:

- `observability-lab/podinfo` target is `UP`
- `up{namespace="observability-lab"}` returns podinfo samples
- `http_requests_total{namespace="observability-lab"}` appears after traffic is generated
- `PodinfoTargetMissing` appears on the Prometheus alerts page

Grafana checks:

- Kubernetes namespace dashboard shows `observability-lab`
- Prometheus Explore can graph podinfo request rate
- Loki Explore shows podinfo logs

## Cleanup

```bash
kubectl delete namespace observability-lab
helm uninstall loki -n monitoring
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

## Check Your Understanding

1. Why does Prometheus need a ServiceMonitor instead of scraping every Service automatically?
2. What labels are used to connect the ServiceMonitor to the Service?
3. Why does the `metrics` Service port target the same container port as `http` for podinfo?
4. What is the difference between PromQL and LogQL?
5. Why does the alert use `absent()` instead of checking for `up == 0`?

## Bonus Challenges

- Create a Grafana dashboard panel for request rate by HTTP status.
- Add an alert that fires when podinfo has fewer than two available replicas.
- Use LogQL to count podinfo log lines over a five-minute window.
- Compare `kubectl logs` output with the same logs in Loki.
