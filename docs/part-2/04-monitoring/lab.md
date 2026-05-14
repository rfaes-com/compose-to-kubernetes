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

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.service.type=ClusterIP \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.ruleSelectorNilUsesHelmValues=false
```

Wait for the main components:

```bash
kubectl wait --for=condition=Available deployment/monitoring-grafana \
  -n monitoring --timeout=180s

kubectl wait --for=condition=Ready pod \
  -l app.kubernetes.io/name=prometheus \
  -n monitoring --timeout=180s

kubectl get pods -n monitoring
```

**Expected output:**

```text
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          2m
monitoring-grafana-xxxxxxxxxx-xxxxx                      3/3     Running   0          2m
monitoring-kube-prometheus-operator-xxxxxxxxxx-xxxxx     1/1     Running   0          2m
monitoring-kube-state-metrics-xxxxxxxxxx-xxxxx           1/1     Running   0          2m
monitoring-prometheus-node-exporter-xxxxx                1/1     Running   0          2m
prometheus-monitoring-kube-prometheus-prometheus-0        2/2     Running   0          2m
```

</details>

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
- CPU request: `100m`, Memory request: `64Mi`
- CPU limit: `250m`, Memory limit: `128Mi`
- Service name: `podinfo`
- Service port `http`: 80 → target port `http`
- Service port `metrics`: 9797 → target port `http`

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl create namespace observability-lab
```

Create `podinfo-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: observability-lab
  labels:
    app: podinfo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfo
        image: ghcr.io/stefanprodan/podinfo:6.5.3
        ports:
        - name: http
          containerPort: 9898
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 250m
            memory: 128Mi
```

Create `podinfo-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  namespace: observability-lab
  labels:
    app: podinfo
spec:
  selector:
    app: podinfo
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: metrics
    port: 9797
    targetPort: http
```

Apply and verify:

```bash
kubectl apply -f podinfo-deployment.yaml
kubectl apply -f podinfo-service.yaml

kubectl wait --for=condition=Available deployment/podinfo \
  -n observability-lab --timeout=90s

kubectl get pods,svc -n observability-lab
```

**Expected output:**

```text
NAME                           READY   STATUS    RESTARTS   AGE
pod/podinfo-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
pod/podinfo-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)            AGE
service/podinfo   ClusterIP   10.96.xxx.xxx   <none>        80/TCP,9797/TCP    30s
```

Test metrics endpoint:

```bash
kubectl port-forward -n observability-lab svc/podinfo 9898:80 9797:9797
curl http://localhost:9797/metrics | head
# HELP http_requests_total The total number of HTTP requests.
# TYPE http_requests_total counter
```

</details>

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

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `podinfo-servicemonitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: podinfo
  namespace: observability-lab
  labels:
    app: podinfo
spec:
  selector:
    matchLabels:
      app: podinfo
  namespaceSelector:
    matchNames:
    - observability-lab
  endpoints:
  - port: metrics
    path: /metrics
    interval: 15s
```

Apply and inspect:

```bash
kubectl apply -f podinfo-servicemonitor.yaml
kubectl get servicemonitor -n observability-lab
kubectl describe servicemonitor podinfo -n observability-lab
```

Port-forward Prometheus:

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Open <http://localhost:9090/targets>.

**Expected result:**

```text
observability-lab/podinfo/0 (2/2 up)
```

</details>

### Task 4: Query Application Metrics

Use Prometheus to inspect the podinfo target and request metrics.

**Requirements:**

- Confirm Prometheus can scrape podinfo
- Generate successful and failed HTTP requests
- Query request rate grouped by status code
- Identify which status code appears after requesting a missing path

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

Generate traffic:

```bash
kubectl port-forward -n observability-lab svc/podinfo 9898:80
```

In another terminal:

```bash
for i in $(seq 1 30); do curl -s http://localhost:9898 > /dev/null; done
for i in $(seq 1 5); do curl -s http://localhost:9898/not-found > /dev/null; done
```

Run these PromQL queries in Prometheus:

```promql
up{namespace="observability-lab"}
```

**Expected result:** Two podinfo targets with value 1

```promql
sum(rate(http_requests_total{namespace="observability-lab"}[1m])) by (status)
```

**Expected result:** Request-rate series grouped by status, including 200 and 404 after the generated traffic.

</details>

### Task 5: Explore Metrics in Grafana

Open Grafana and inspect both built-in Kubernetes dashboards and your application metrics.

**Requirements:**

- Retrieve the Grafana admin password from the `monitoring-grafana` Secret
- Port-forward Grafana to local port 3000
- Open a Kubernetes namespace dashboard for `observability-lab`
- Use Explore to run a PromQL query for podinfo request rate

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
echo ""

kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Open <http://localhost:3000>.

- Username: `admin`
- Password: decoded from the `monitoring-grafana` Secret

Open **Dashboards** and choose **Kubernetes / Compute Resources / Namespace (Pods)**. Select `observability-lab`.

In **Explore**, select the Prometheus data source and run:

```promql
sum(rate(http_requests_total{namespace="observability-lab"}[1m])) by (status)
```

**Expected result:** Grafana displays podinfo request rate grouped by HTTP status code.

</details>

### Task 6: Install Loki and Query Logs

Install Loki with Promtail and query podinfo logs from Grafana.

**Requirements:**

- Add the Grafana Helm repository
- Install `grafana/loki-stack` into the `monitoring` namespace
- Enable Promtail
- Do not install a second Grafana instance
- Add Loki as a Grafana data source
- Query logs for the `observability-lab` namespace

<details class="hint" markdown="1">
<summary>Hint</summary>

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

Try these LogQL queries:

```logql
{namespace="observability-lab"}
```

```logql
{namespace="observability-lab", app="podinfo"}
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set grafana.enabled=false
```

Verify Loki and Promtail:

```bash
kubectl wait --for=condition=Ready pod \
  -l app=loki \
  -n monitoring --timeout=120s

kubectl get pods -n monitoring | grep promtail
```

Add a Grafana data source:

```text
Type: Loki
URL:  http://loki:3100
```

Generate logs:

```bash
kubectl port-forward -n observability-lab svc/podinfo 9898:80
```

In another terminal:

```bash
for i in $(seq 1 10); do curl -s http://localhost:9898/version; done
for i in $(seq 1 3); do curl -s http://localhost:9898/headers; done
```

Run these LogQL queries in Grafana Explore:

```logql
{namespace="observability-lab"}
```

```logql
{namespace="observability-lab", app="podinfo"}
```

**Expected result:** Recent podinfo access logs appear in Grafana Explore.

</details>

### Task 7: Create and Test an Alert

Create a PrometheusRule that fires when Prometheus cannot find a podinfo target.

**Requirements:**

- PrometheusRule name: `podinfo-alerts`
- Namespace: `observability-lab`
- Alert name: `PodinfoTargetMissing`
- Expression should detect absence of the podinfo `up` metric
- Alert should become pending/firing after podinfo is scaled to zero
- Restore podinfo to 2 replicas after the test

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `podinfo-alert.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: podinfo-alerts
  namespace: observability-lab
  labels:
    app: podinfo
spec:
  groups:
  - name: podinfo.rules
    rules:
    - alert: PodinfoTargetMissing
      expr: absent(up{namespace="observability-lab", service="podinfo"}) == 1
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Podinfo target is missing"
        description: "Prometheus cannot find a podinfo metrics target."
```

Apply it:

```bash
kubectl apply -f podinfo-alert.yaml
kubectl get prometheusrule -n observability-lab
```

Open <http://localhost:9090/rules> and verify that `PodinfoTargetMissing` is loaded.

Scale the app down:

```bash
kubectl scale deployment podinfo -n observability-lab --replicas=0
```

Open <http://localhost:9090/alerts>.

**Expected result:** `PodinfoTargetMissing` moves from pending to firing after one minute.

Restore the app:

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

Expected outcomes:

- `observability-lab/podinfo` target is `UP`
- `up{namespace="observability-lab"}` returns podinfo samples
- `http_requests_total{namespace="observability-lab"}` appears after traffic is generated
- `PodinfoTargetMissing` appears on the Prometheus alerts page
- Grafana can query Prometheus metrics and Loki logs

## Cleanup

```bash
kubectl delete namespace observability-lab
helm uninstall loki -n monitoring
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

## Bonus Challenges

- Create a Grafana dashboard panel for request rate by HTTP status.
- Add an alert that fires when podinfo has fewer than two available replicas.
- Use LogQL to count podinfo log lines over a five-minute window.
- Compare `kubectl logs` output with the same logs in Loki.

## Key takeaways

1. **Prometheus** scrapes metrics via a pull model; **ServiceMonitor** resources define which Services to scrape and how
2. **PromQL** enables powerful metric queries — range vectors and aggregation functions unlock deep analysis
3. **Grafana** unifies Prometheus metrics and Loki logs in a single observability dashboard
4. **Loki** provides log aggregation without indexing full log content, keeping storage costs low
5. **`absent()` in alert rules** catches missing metrics — useful when a target disappearing is itself the problem
6. **Label selectors** connect ServiceMonitors to Services; a mismatch silently produces no data
7. **Service `targetPort`** must match the container port where metrics are actually exposed — a misconfigured port means Prometheus scrapes the wrong endpoint and returns no data

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../05-advanced-deployments/README.md).
