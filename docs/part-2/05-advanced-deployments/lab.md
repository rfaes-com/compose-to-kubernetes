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

<details class="hint" markdown="1">
<summary>Hint</summary>

Create the namespace:

```bash
kubectl create namespace deploy-strategy-lab
```

Set it as the default namespace for your current context:

```bash
kubectl config set-context --current --namespace=deploy-strategy-lab
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl create namespace deploy-strategy-lab
kubectl config set-context --current --namespace=deploy-strategy-lab
kubectl get namespace deploy-strategy-lab
```

</details>

### Task 2: Configure an Advanced Rolling Update

Deploy a baseline application with rolling update controls and readiness checks.

**Requirements:**

- Deployment name: `rollout-demo`
- Image: `nginx:1.25-alpine`
- Replicas: 4
- Label: `app=rollout-demo`
- Rolling update strategy with `maxUnavailable: 0` and `maxSurge: 1`
- `minReadySeconds: 5`
- Readiness probe on `/`
- Service name: `rollout-demo`, port: 80

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `rollout-demo-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rollout-demo
spec:
  replicas: 4
  minReadySeconds: 5
  selector:
    matchLabels:
      app: rollout-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: rollout-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 3
```

Create `rollout-demo-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rollout-demo
spec:
  selector:
    app: rollout-demo
  ports:
  - port: 80
    targetPort: 80
```

Apply and verify:

```bash
kubectl apply -f rollout-demo-deployment.yaml
kubectl apply -f rollout-demo-service.yaml
kubectl rollout status deployment/rollout-demo
kubectl get pods -l app=rollout-demo
```

</details>

### Task 3: Watch a Rolling Update

Update the image and observe how Kubernetes replaces Pods.

**Requirements:**

- Change the Deployment image to `nginx:1.26-alpine`
- Watch the rollout status
- Confirm the new image is running
- Inspect rollout history
- Roll back to the previous revision

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl set image deployment/rollout-demo nginx=nginx:1.26-alpine
kubectl rollout status deployment/rollout-demo
kubectl get pods -l app=rollout-demo
kubectl describe deployment rollout-demo | grep Image
kubectl rollout history deployment/rollout-demo
kubectl rollout undo deployment/rollout-demo
kubectl rollout status deployment/rollout-demo
```

**Expected result:** The Deployment keeps available replicas during the update and returns to `nginx:1.25-alpine` after rollback.

</details>

### Task 4: Deploy Blue and Green Versions

Deploy two complete versions of the same app side by side.

**Requirements:**

- Blue Deployment name: `web-blue`, labels: `app=web, version=blue`, page contains `BLUE version`
- Green Deployment name: `web-green`, labels: `app=web, version=green`, page contains `GREEN version`
- One replica for each Deployment
- Service name: `web`, selector initially points to `app=web, version=blue`

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `web-blue.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-blue
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      version: blue
  template:
    metadata:
      labels:
        app: web
        version: blue
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo 'BLUE version' > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
```

Create `web-green.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-green
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      version: green
  template:
    metadata:
      labels:
        app: web
        version: green
    spec:
      containers:
      - name: nginx
        image: nginx:1.26-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo 'GREEN version' > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
```

Create `web-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
    version: blue
  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f web-blue.yaml
kubectl apply -f web-green.yaml
kubectl apply -f web-service.yaml
kubectl wait --for=condition=Available deployment/web-blue --timeout=60s
kubectl wait --for=condition=Available deployment/web-green --timeout=60s
```

</details>

### Task 5: Switch and Roll Back Blue/Green Traffic

Use the Service selector to switch production traffic.

**Requirements:**

- Port-forward the `web` Service
- Confirm traffic initially reaches blue
- Patch the Service selector to `version=green`
- Confirm traffic reaches green
- Patch the Service selector back to `version=blue`
- Confirm rollback is instant

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl port-forward svc/web 8080:80
```

In another terminal:

```bash
curl http://localhost:8080
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"green"}}}'
curl http://localhost:8080
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"blue"}}}'
curl http://localhost:8080
```

**Expected result:**

```text
BLUE version
GREEN version
BLUE version
```

</details>

### Task 6: Run a Canary Deployment

Deploy stable and canary versions behind one Service.

**Requirements:**

- Stable Deployment name: `api-stable`, labels: `app=api, track=stable`, 4 replicas, page contains `stable`
- Canary Deployment name: `api-canary`, labels: `app=api, track=canary`, 1 replica, page contains `canary`
- Service name: `api`, selector matches `app=api` only

<details class="hint" markdown="1">
<summary>Hint</summary>

The canary Service should select only the shared app label:

```yaml
selector:
  app: api
```

Kubernetes load balances across all matching Pods. With 4 stable Pods and 1 canary Pod, the canary receives roughly 20% of requests.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

Create `api-stable.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-stable
spec:
  replicas: 4
  selector:
    matchLabels:
      app: api
      track: stable
  template:
    metadata:
      labels:
        app: api
        track: stable
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo 'stable' > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
```

Create `api-canary.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
      track: canary
  template:
    metadata:
      labels:
        app: api
        track: canary
    spec:
      containers:
      - name: nginx
        image: nginx:1.26-alpine
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo 'canary' > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
```

Create `api-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f api-stable.yaml
kubectl apply -f api-canary.yaml
kubectl apply -f api-service.yaml
kubectl wait --for=condition=Available deployment/api-stable --timeout=60s
kubectl wait --for=condition=Available deployment/api-canary --timeout=60s
```

</details>

### Task 7: Promote or Roll Back the Canary

Observe traffic split and then choose a final action.

**Requirements:**

- Send at least 20 requests to the `api` Service
- Count how many responses go to stable vs canary
- Promote by scaling stable to 0 and canary to 5
- Roll back by scaling canary to 0 and stable to 5
- Leave the app in either promoted or rolled-back state

<details class="hint" markdown="1">
<summary>Hint</summary>

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

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl port-forward svc/api 8081:80
```

In another terminal:

```bash
for i in $(seq 1 20); do curl -s http://localhost:8081; done | sort | uniq -c
```

**Expected result:** ~16 `stable` and ~4 `canary` responses (approximately 80/20 split with 4+1 replicas).

Promote:

```bash
kubectl scale deployment api-stable --replicas=0
kubectl scale deployment api-canary --replicas=5
```

Or roll back:

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

## Key takeaways

1. **`maxUnavailable: 0` with `maxSurge`** ensures zero-downtime rolling updates by always keeping old Pods running during the rollout
2. **Blue/green deployments** enable instant traffic switching and fast rollback simply by updating a Service selector
3. **Canary releases** expose a controlled percentage of traffic to new code by managing replica ratios between stable and canary Deployments
4. **Service selectors** are the traffic control plane — changing a label match redirects all traffic without touching Pods
5. **Progressive delivery controllers** like Flagger automate canary promotion based on real metrics, removing the need for manual replica management
6. **Replica-weighted canary has limits** — traffic split is coarse-grained and tied to replica count, making precise percentages impossible at low replica numbers

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../06-autoscaling/README.md).
