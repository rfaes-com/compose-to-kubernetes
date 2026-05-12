# Lab Solutions: Autoscaling

Complete solutions for the Autoscaling lab exercises.

## Task 1: Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

kubectl rollout status deployment/metrics-server -n kube-system
kubectl top nodes
kubectl top pods --all-namespaces
```

## Task 2: Deploy a CPU-Bound Application

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

Apply:

```bash
kubectl apply -f php-apache-deployment.yaml
kubectl apply -f php-apache-service.yaml
kubectl wait --for=condition=Available deployment/php-apache --timeout=90s
```

## Task 3: Create a CPU-Based HPA

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

## Task 4: Generate Load and Watch Scale-Up

```bash
kubectl run load-generator --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

kubectl get hpa php-apache --watch
kubectl get deployment php-apache
kubectl top pods
```

**Expected result:**

```text
The HPA target rises above 50%, desired replicas increase, and the Deployment scales above 1 replica.
```

## Task 5: Tune HPA Behavior

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

## Task 6: Stop Load and Observe Scale-Down

```bash
kubectl delete pod load-generator --ignore-not-found
kubectl get hpa php-apache --watch
kubectl get deployment php-apache --watch
```

**Expected result:**

```text
CPU usage drops quickly, but replicas reduce more slowly because of the scale-down stabilization window.
```

## Task 7: Break and Fix Metrics Intentionally

```bash
kubectl create deployment no-requests --image=nginx:1.25-alpine --replicas=1
kubectl autoscale deployment no-requests --cpu-percent=50 --min=1 --max=3
kubectl describe hpa no-requests
```

**Expected issue:**

```text
The HPA cannot calculate CPU utilization because the target container has no CPU request.
```

Fix it:

```bash
kubectl set resources deployment no-requests --requests=cpu=100m
kubectl rollout status deployment/no-requests
kubectl describe hpa no-requests
```

## Verification

```bash
kubectl top nodes
kubectl get deployment,hpa,pods -n autoscaling-lab
kubectl describe hpa php-apache -n autoscaling-lab
```

## Cleanup

```bash
kubectl delete namespace autoscaling-lab
kubectl config set-context --current --namespace=default
```
