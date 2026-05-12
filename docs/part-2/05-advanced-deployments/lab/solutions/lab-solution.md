# Lab Solutions: Advanced Deployment Strategies

Complete solutions for the Advanced Deployment Strategies lab exercises.

## Task 1: Create the Lab Namespace

```bash
kubectl create namespace deploy-strategy-lab
kubectl config set-context --current --namespace=deploy-strategy-lab
kubectl get namespace deploy-strategy-lab
```

## Task 2: Configure an Advanced Rolling Update

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

## Task 3: Watch a Rolling Update

```bash
kubectl set image deployment/rollout-demo nginx=nginx:1.26-alpine
kubectl rollout status deployment/rollout-demo
kubectl get pods -l app=rollout-demo
kubectl describe deployment rollout-demo | grep Image
kubectl rollout history deployment/rollout-demo
kubectl rollout undo deployment/rollout-demo
kubectl rollout status deployment/rollout-demo
```

**Expected result:**

```text
The Deployment keeps available replicas during the update and returns to nginx:1.25-alpine after rollback.
```

## Task 4: Deploy Blue and Green Versions

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

## Task 5: Switch and Roll Back Blue/Green Traffic

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

## Task 6: Run a Canary Deployment

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

## Task 7: Promote or Roll Back the Canary

```bash
kubectl port-forward svc/api 8081:80
```

In another terminal:

```bash
for i in $(seq 1 20); do curl -s http://localhost:8081; done | sort | uniq -c
```

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

## Verification

```bash
kubectl get deployments,services
kubectl rollout history deployment/rollout-demo
kubectl get endpoints web api
```

## Cleanup

```bash
kubectl delete namespace deploy-strategy-lab
kubectl config set-context --current --namespace=default
```
