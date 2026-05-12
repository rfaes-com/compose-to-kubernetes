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

### Task 4: Generate Load and Watch Scale-Up

Create traffic that pushes CPU above the HPA target.

**Requirements:**

- Run a temporary load generator Pod
- Continuously request the `php-apache` Service
- Watch HPA desired replicas increase
- Confirm the Deployment scales above 1 replica
- Check current CPU usage with `kubectl top pods`

### Task 5: Tune HPA Behavior

Update the HPA to scale up quickly and scale down more slowly.

**Requirements:**

- Keep min replicas at 1 and max replicas at 6
- Keep CPU target at 50%
- Add scale-up behavior with no stabilization window
- Add scale-down behavior with a 120 second stabilization window
- Limit scale-down to 50% per minute
- Verify the behavior appears in the HPA YAML

### Task 6: Stop Load and Observe Scale-Down

Stop the load generator and observe HPA stabilization.

**Requirements:**

- Delete the load generator Pod
- Watch CPU usage drop
- Observe that scale-down is slower than scale-up
- Confirm the Deployment eventually returns toward the minimum replica count

### Task 7: Break and Fix Metrics Intentionally

Create a Deployment that cannot be autoscaled by CPU, then explain why.

**Requirements:**

- Deployment name: `no-requests`
- Image: `nginx:1.25-alpine`
- Replicas: 1
- Do not set CPU requests
- Create an HPA targeting CPU utilization
- Observe the HPA warning or unknown metric state
- Fix the Deployment by adding CPU requests

## Hints

<details>
<summary>Hint for Task 1: Metrics Server on kind</summary>

Install Metrics Server:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Kind clusters commonly need this argument:

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

Wait and verify:

```bash
kubectl rollout status deployment/metrics-server -n kube-system
kubectl top nodes
kubectl top pods --all-namespaces
```

</details>

<details>
<summary>Hint for Task 2: Resource requests for HPA</summary>

HPA CPU utilization needs CPU requests. The container resources should look like:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

The Service selector should match `app=php-apache`.

</details>

<details>
<summary>Hint for Task 3: CPU utilization metric</summary>

An `autoscaling/v2` HPA can use this metric block:

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50
```

Watch the HPA:

```bash
kubectl get hpa php-apache --watch
```

</details>

<details>
<summary>Hint for Task 4: Generating CPU load</summary>

Run a temporary busybox loop:

```bash
kubectl run load-generator --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

Watch scaling:

```bash
kubectl get hpa php-apache --watch
kubectl get deployment php-apache --watch
kubectl top pods -n autoscaling-lab
```

</details>

<details>
<summary>Hint for Task 5: HPA behavior policies</summary>

Scale-down behavior can be slower than scale-up:

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

<details>
<summary>Hint for Task 6: Stopping load</summary>

Delete the load generator:

```bash
kubectl delete pod load-generator --ignore-not-found
```

Then watch:

```bash
kubectl get hpa php-apache --watch
kubectl get deployment php-apache --watch
```

</details>

<details>
<summary>Hint for Task 7: Missing CPU requests</summary>

Create the HPA:

```bash
kubectl autoscale deployment no-requests --cpu-percent=50 --min=1 --max=3
```

Inspect the problem:

```bash
kubectl describe hpa no-requests
```

Patch in a CPU request:

```bash
kubectl set resources deployment no-requests --requests=cpu=100m
```

</details>

## Verification

Check your work:

```bash
kubectl top nodes
kubectl get deployment,hpa,pods -n autoscaling-lab
kubectl describe hpa php-apache -n autoscaling-lab
```

Expected outcomes:

- Metrics Server returns node and pod metrics
- `php-apache` scales up under load
- HPA behavior policies are visible in YAML
- Scale-down waits for stabilization
- The `no-requests` HPA demonstrates why CPU requests are required

## Cleanup

```bash
kubectl delete namespace autoscaling-lab
```

## Check Your Understanding

1. Why does HPA need CPU requests for utilization-based scaling?
2. What is the difference between current replicas and desired replicas?
3. Why might scale-down intentionally be slower than scale-up?
4. What happens if max replicas is too low?
5. When would custom metrics be better than CPU metrics?

## Bonus Challenges

- Add a memory-based HPA metric.
- Change max replicas to 10 and compare scale-up behavior.
- Create a PodDisruptionBudget for `php-apache`.
- Research how Prometheus Adapter exposes custom metrics to HPA.
