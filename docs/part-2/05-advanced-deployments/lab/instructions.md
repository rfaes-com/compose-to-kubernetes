# Lab: Advanced Deployment Strategies

**Duration:** 20 minutes

## Objectives

- Configure safer rolling updates
- Deploy blue and green application versions
- Switch production traffic with a Service selector
- Run a canary deployment by replica weighting
- Promote or roll back a canary release

## Prerequisites

- Kind cluster running
- kubectl configured and working
- Basic understanding of Deployments and Services

## Tasks

### Task 1: Create the Lab Namespace

Create a namespace for all deployment strategy resources.

**Requirements:**

- Namespace name: `deploy-strategy-lab`
- Set your current context namespace to `deploy-strategy-lab`
- Verify the namespace exists

### Task 2: Configure an Advanced Rolling Update

Deploy a baseline application with rolling update controls and readiness checks.

**Requirements:**

- Deployment name: `rollout-demo`
- Image: `nginx:1.25-alpine`
- Replicas: 4
- Label: `app=rollout-demo`
- Rolling update strategy
- `maxUnavailable: 0`
- `maxSurge: 1`
- `minReadySeconds: 5`
- Readiness probe on `/`
- Service name: `rollout-demo`
- Service port: 80

### Task 3: Watch a Rolling Update

Update the image and observe how Kubernetes replaces Pods.

**Requirements:**

- Change the Deployment image to `nginx:1.26-alpine`
- Watch the rollout status
- Confirm the new image is running
- Inspect rollout history
- Roll back to the previous revision

### Task 4: Deploy Blue and Green Versions

Deploy two complete versions of the same app side by side.

**Requirements:**

- Blue Deployment name: `web-blue`
- Blue labels: `app=web`, `version=blue`
- Blue page should contain `BLUE version`
- Green Deployment name: `web-green`
- Green labels: `app=web`, `version=green`
- Green page should contain `GREEN version`
- One replica for each Deployment
- Service name: `web`
- Service selector initially points to `app=web,version=blue`

### Task 5: Switch and Roll Back Blue/Green Traffic

Use the Service selector to switch production traffic.

**Requirements:**

- Port-forward the `web` Service
- Confirm traffic initially reaches blue
- Patch the Service selector to `version=green`
- Confirm traffic reaches green
- Patch the Service selector back to `version=blue`
- Confirm rollback is instant

### Task 6: Run a Canary Deployment

Deploy stable and canary versions behind one Service.

**Requirements:**

- Stable Deployment name: `api-stable`
- Stable labels: `app=api`, `track=stable`
- Stable replicas: 4
- Stable page should contain `stable`
- Canary Deployment name: `api-canary`
- Canary labels: `app=api`, `track=canary`
- Canary replicas: 1
- Canary page should contain `canary`
- Service name: `api`
- Service selector should match `app=api` only

### Task 7: Promote or Roll Back the Canary

Observe traffic split and then choose a final action.

**Requirements:**

- Send at least 20 requests to the `api` Service
- Count how many responses go to stable vs canary
- Promote by scaling stable to 0 and canary to 5
- Roll back by scaling canary to 0 and stable to 5
- Leave the app in either promoted or rolled-back state

## Hints

<details>
<summary>Hint for Task 1: Creating and selecting the namespace</summary>

Create the namespace:

```bash
kubectl create namespace deploy-strategy-lab
```

Set it as the default namespace for your current context:

```bash
kubectl config set-context --current --namespace=deploy-strategy-lab
```

</details>

<details>
<summary>Hint for Task 2: Rolling update strategy fields</summary>

The strategy belongs under `spec.strategy`:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
minReadySeconds: 5
```

A simple readiness probe for nginx can use:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 2
  periodSeconds: 3
```

</details>

<details>
<summary>Hint for Task 3: Watching rollout progress</summary>

Update the image:

```bash
kubectl set image deployment/rollout-demo nginx=nginx:1.26-alpine
```

Watch rollout status and history:

```bash
kubectl rollout status deployment/rollout-demo
kubectl rollout history deployment/rollout-demo
kubectl describe deployment rollout-demo
```

Roll back:

```bash
kubectl rollout undo deployment/rollout-demo
```

</details>

<details>
<summary>Hint for Task 4: Creating color-coded pages</summary>

Use nginx with a shell command that writes `index.html` before starting nginx:

```yaml
command: ["/bin/sh", "-c"]
args:
- |
  echo 'BLUE version' > /usr/share/nginx/html/index.html
  nginx -g 'daemon off;'
```

Create the Service selector with both labels:

```yaml
selector:
  app: web
  version: blue
```

</details>

<details>
<summary>Hint for Task 5: Switching a Service selector</summary>

Port-forward the Service:

```bash
kubectl port-forward svc/web 8080:80
```

Patch the selector to green:

```bash
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"green"}}}'
```

Patch it back to blue:

```bash
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"blue"}}}'
```

</details>

<details>
<summary>Hint for Task 6: Canary Service selector</summary>

The canary Service should select only the shared app label:

```yaml
selector:
  app: api
```

Kubernetes load balances across all matching Pods. With 4 stable Pods and 1 canary Pod, the canary receives roughly 20% of requests.

</details>

<details>
<summary>Hint for Task 7: Counting stable and canary responses</summary>

Port-forward the canary Service:

```bash
kubectl port-forward svc/api 8081:80
```

In another terminal, send requests:

```bash
for i in $(seq 1 20); do curl -s http://localhost:8081; done | sort | uniq -c
```

Promote canary:

```bash
kubectl scale deployment api-stable --replicas=0
kubectl scale deployment api-canary --replicas=5
```

Roll back canary:

```bash
kubectl scale deployment api-canary --replicas=0
kubectl scale deployment api-stable --replicas=5
```

</details>

## Verification

Check your work:

```bash
kubectl get deployments,services
kubectl rollout history deployment/rollout-demo
kubectl get endpoints web api
```

Expected outcomes:

- `rollout-demo` rolls forward and back without dropping all replicas
- `web` Service can switch between blue and green
- `api` Service sends traffic to both stable and canary while both have replicas
- Canary can be promoted or rolled back with scaling changes

## Cleanup

```bash
kubectl delete namespace deploy-strategy-lab
kubectl config set-context --current --namespace=default
```

## Check Your Understanding

1. Why does `maxUnavailable: 0` reduce downtime risk?
2. Why is blue/green rollback faster than a Deployment rollback?
3. How does the canary traffic percentage relate to replica counts?
4. What are the limits of replica-weighted canary deployments?
5. When would you choose a progressive delivery controller such as Flagger?
