# Lab Solutions: Monitoring and Logging

Complete solutions for the Monitoring and Logging lab exercises.

## Task 1: Install the Monitoring Stack

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

## Task 2: Deploy a Metrics-Enabled Application

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

Test the app and metrics endpoint:

```bash
kubectl port-forward -n observability-lab svc/podinfo 9898:80 9797:9797
```

In another terminal:

```bash
curl http://localhost:9898
curl http://localhost:9797/metrics | head
```

**Expected output:**

```text
# HELP http_requests_total The total number of HTTP requests.
# TYPE http_requests_total counter
```

## Task 3: Configure Prometheus Scraping

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

## Task 4: Query Application Metrics

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

**Expected result:**

```text
Two podinfo targets with value 1
```

```promql
sum(rate(http_requests_total{namespace="observability-lab"}[1m])) by (status)
```

**Expected result:**

```text
Request-rate series grouped by status, including 200 and 404 after the generated traffic.
```

## Task 5: Explore Metrics in Grafana

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

**Expected result:**

```text
Grafana displays podinfo request rate grouped by HTTP status code.
```

## Task 6: Install Loki and Query Logs

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

**Expected result:**

```text
Recent podinfo access logs appear in Grafana Explore.
```

## Task 7: Create and Test an Alert

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

**Expected result:**

```text
PodinfoTargetMissing moves from pending to firing after one minute.
```

Restore the app:

```bash
kubectl scale deployment podinfo -n observability-lab --replicas=2
kubectl wait --for=condition=Available deployment/podinfo \
  -n observability-lab --timeout=90s
```

## Verification

```bash
kubectl get pods -n monitoring
kubectl get pods,svc,servicemonitor,prometheusrule -n observability-lab
kubectl get endpoints podinfo -n observability-lab
```

Expected outcomes:

- Monitoring components are running in the `monitoring` namespace
- Podinfo has two running Pods
- The podinfo Service has endpoints
- Prometheus shows the podinfo target as `UP`
- Grafana can query Prometheus metrics
- Grafana can query Loki logs
- The alert fires when podinfo is scaled to zero

## Cleanup

```bash
kubectl delete namespace observability-lab
helm uninstall loki -n monitoring
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```
