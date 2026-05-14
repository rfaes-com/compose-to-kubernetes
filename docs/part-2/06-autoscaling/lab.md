# Lab: Autoscaling

**Duration:** 25 minutes

## Objectives

- Install and verify Metrics Server
- Deploy an application with resource requests
- Configure a HorizontalPodAutoscaler
- Generate CPU load and observe scale-up
- Tune HPA scale-down behavior
- Understand why resource requests matter

## Prerequisites

- Kind cluster running
- kubectl configured and working
- Basic understanding of Deployments and resource requests

## Tasks

### Task 1: Install Metrics Server

Install Metrics Server and make it work in kind.

**Requirements:**

- Apply the upstream Metrics Server manifest
- Patch the Metrics Server Deployment for kind TLS behavior
- Verify `kubectl top nodes` works
- Verify `kubectl top pods --all-namespaces` works

<details class="hint" markdown="1">
<summary>Hint</summary>

Apply the manifest and patch the args:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

Wait for it to be ready:

```bash
kubectl rollout status deployment/metrics-server -n kube-system
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

kubectl rollout status deployment/metrics-server -n kube-system
kubectl top nodes
kubectl top pods --all-namespaces
```

**Expected result:** `kubectl top nodes` shows node CPU and memory usage. `kubectl top pods` shows Pod metrics across all namespaces.

</details>

### Task 2: Deploy a CPU-Bound Application

Deploy an application that can consume CPU when requested.

**Requirements:**

- Namespace: `autoscaling-lab`
- Deployment name: `php-apache`
- Image: `registry.k8s.io/hpa-example`
- Label: `app=php-apache`
- Replicas: 1
- Container port: 80
- CPU request: `200m`
- CPU limit: `500m`
- Memory request: `128Mi`
- Memory limit: `256Mi`
- Service name: `php-apache`
- Service port: 80

<details class="hint" markdown="1">
<summary>Hint</summary>

Create the namespace first:

```bash
kubectl create namespace autoscaling-lab
kubectl config set-context --current --namespace=autoscaling-lab
```

The image `registry.k8s.io/hpa-example` runs a simple PHP app that computes square roots in a loop when requested. The important thing is to specify CPU requests so that the HPA can calculate utilization.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl create namespace autoscaling-lab
kubectl config set-context --current --namespace=autoscaling-lab
```

Create `php-apache-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  labels:
    app: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

Create `php-apache-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    app: php-apache
  ports:
  - port: 80
    targetPort: 80
```

Apply and verify:

```bash
kubectl apply -f php-apache-deployment.yaml
kubectl apply -f php-apache-service.yaml
kubectl wait --for=condition=Available deployment/php-apache --timeout=90s
kubectl get pods,svc
```

</details>

### Task 3: Create a CPU-Based HPA

Create a HorizontalPodAutoscaler for the application.

**Requirements:**

- HPA name: `php-apache`
- Target Deployment: `php-apache`
- Minimum replicas: 1
- Maximum replicas: 6
- Target CPU utilization: 50%
- Use `autoscaling/v2`
- Verify the HPA can read CPU metrics

<details class="hint" markdown="1">
<summary>Hint</summary>

The `autoscaling/v2` spec uses a `metrics` array:

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50
```

After applying, run:

```bash
kubectl get hpa php-apache
kubectl describe hpa php-apache
```

The HPA status shows current CPU utilization and desired replicas.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `php-apache-hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply and verify:

```bash
kubectl apply -f php-apache-hpa.yaml
kubectl get hpa php-apache
kubectl describe hpa php-apache
```

**Expected output:**

```text
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   1%/50%    1         6         1          30s
```

The current CPU usage should be very low at this point (no traffic yet).

</details>

### Task 4: Generate Load and Watch Scale-Up

Create traffic that pushes CPU above the HPA target.

**Requirements:**

- Run a temporary load generator Pod
- Continuously request the `php-apache` Service
- Watch HPA desired replicas increase
- Confirm the Deployment scales above 1 replica
- Check current CPU usage with `kubectl top pods`

<details class="hint" markdown="1">
<summary>Hint</summary>

Run a busy loop in a Pod that sends HTTP requests:

```bash
kubectl run load-generator --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

Watch the HPA in a separate terminal:

```bash
kubectl get hpa php-apache --watch
```

Also check deployment replicas and Pod CPU:

```bash
kubectl get deployment php-apache
kubectl top pods
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl run load-generator --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

kubectl get hpa php-apache --watch
kubectl get deployment php-apache
kubectl top pods
```

**Expected result:** The HPA target rises above 50%, desired replicas increase from 1 toward 6, and `kubectl top pods` shows higher CPU usage across scaled Pods.

</details>

### Task 5: Tune HPA Behavior

Update the HPA to scale up quickly and scale down more slowly.

**Requirements:**

- Keep min replicas at 1 and max replicas at 6
- Keep CPU target at 50%
- Add scale-up behavior with no stabilization window
- Add scale-down behavior with a 120 second stabilization window
- Limit scale-down to 50% per minute
- Verify the behavior appears in the HPA YAML

<details class="hint" markdown="1">
<summary>Hint</summary>

Add a `behavior` block to the HPA spec:

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 120
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
      periodSeconds: 30
    selectPolicy: Max
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

Update `php-apache-hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 120
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
        periodSeconds: 30
      selectPolicy: Max
```

Apply and inspect:

```bash
kubectl apply -f php-apache-hpa.yaml
kubectl get hpa php-apache -o yaml
```

</details>

### Task 6: Stop Load and Observe Scale-Down

Stop the load generator and observe HPA stabilization.

**Requirements:**

- Delete the load generator Pod
- Watch CPU usage drop
- Observe that scale-down is slower than scale-up
- Confirm the Deployment eventually returns toward the minimum replica count

<details class="hint" markdown="1">
<summary>Hint</summary>

Delete the load generator:

```bash
kubectl delete pod load-generator --ignore-not-found
```

Then watch the HPA and Deployment:

```bash
kubectl get hpa php-apache --watch
kubectl get deployment php-apache --watch
```

Because of the 120-second stabilization window, the Deployment will not scale down immediately.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl delete pod load-generator --ignore-not-found
kubectl get hpa php-apache --watch
kubectl get deployment php-apache --watch
```

**Expected result:** CPU usage drops quickly, but replicas reduce more slowly because of the 120-second scale-down stabilization window. The Deployment will eventually return toward 1 replica.

</details>

## Verification

Check your work:

```bash
kubectl get pods -n autoscaling-lab
kubectl get hpa php-apache -n autoscaling-lab
kubectl describe hpa php-apache -n autoscaling-lab
kubectl top pods -n autoscaling-lab
```

Expected outcomes:

- Metrics Server is running and `kubectl top` works
- HPA reports current CPU utilization
- Deployment scaled up under load
- Scale-down is slower than scale-up due to the stabilization window

## Cleanup

```bash
kubectl delete namespace autoscaling-lab
kubectl config set-context --current --namespace=default
```

## Key takeaways

1. **Metrics Server** is required for HPA — without it, `kubectl top` and CPU-based autoscaling both fail
2. **Resource requests must be set** on Pods for HPA to calculate CPU utilization percentage
3. **HPA scales up quickly** but uses a stabilization window to prevent flapping on scale-down
4. **`scaleDown` behavior policies** give fine-grained control over how fast and how many Pods are removed at once
5. **Custom and external metrics** allow HPA to scale on application-level signals such as queue depth or request latency

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../07-security/README.md).
