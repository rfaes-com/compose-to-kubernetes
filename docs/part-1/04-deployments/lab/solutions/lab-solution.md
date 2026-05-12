# Lab Solution: Working with Deployments

## Task 1: Create a Deployment

```bash
# 1. Create Deployment
kubectl create deployment web-app --image=nginx:1.21 --replicas=3

# Output:
# deployment.apps/web-app created

# 2. Verify Deployment
kubectl get deployments

# Output:
# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# web-app   3/3     3            3           10s

# 3. List ReplicaSets
kubectl get replicasets

# Output:
# NAME                 DESIRED   CURRENT   READY   AGE
# web-app-7d8f5c5b6d   3         3         3       15s

# 4. List Pods
kubectl get pods -l app=web-app

# Output:
# NAME                       READY   STATUS    RESTARTS   AGE
# web-app-7d8f5c5b6d-a1b2c   1/1     Running   0          20s
# web-app-7d8f5c5b6d-d3e4f   1/1     Running   0          20s
# web-app-7d8f5c5b6d-g5h6i   1/1     Running   0          20s

# 5. Describe Deployment
kubectl describe deployment web-app
```

## Task 2: Test Self-Healing

```bash
# 1. Watch Pods (terminal 1)
kubectl get pods -l app=web-app -w

# 2. Delete a Pod (terminal 2)
kubectl delete pod web-app-7d8f5c5b6d-a1b2c

# 3. Observe in terminal 1:
# web-app-7d8f5c5b6d-a1b2c   1/1     Terminating   0          2m
# web-app-7d8f5c5b6d-j7k8l   0/1     Pending       0          0s
# web-app-7d8f5c5b6d-j7k8l 0/1     ContainerCreating   0          0s
# web-app-7d8f5c5b6d-j7k8l   1/1     Running             0          2s

# 4. Stop watching (Ctrl+C)
```

## Task 3: Scale the Deployment

```bash
# 1. Scale to 5 replicas
kubectl scale deployment web-app --replicas=5

# Output:
# deployment.apps/web-app scaled

# 2. Watch Pods being created
kubectl get pods -l app=web-app -w

# 3. Scale down to 2
kubectl scale deployment web-app --replicas=2

# 4. Observe termination
# (Pods will show Terminating status)

# 5. Verify final state
kubectl get deployment web-app

# Output:
# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# web-app   2/2     2            2           5m
```

## Task 4: Rolling Update

```bash
# 1. Update image
kubectl set image deployment/web-app nginx=nginx:1.22

# Output:
# deployment.apps/web-app image updated

# 2. Watch rollout
kubectl rollout status deployment/web-app

# Output:
# Waiting for deployment "web-app" rollout to finish: 1 out of 2 new replicas have been updated...
# Waiting for deployment "web-app" rollout to finish: 1 old replicas are pending termination...
# deployment "web-app" successfully rolled out

# 3. Check history
kubectl rollout history deployment/web-app

# Output:
# deployment.apps/web-app
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# 4. List ReplicaSets
kubectl get replicasets

# Output:
# NAME                 DESIRED   CURRENT   READY   AGE
# web-app-7d8f5c5b6d   0         0         0       10m    (old)
# web-app-5b9c8d7e6f   2         2         2       1m     (new)

# 5. Verify image
kubectl describe pods -l app=web-app | grep Image:

# Output:
# Image:          nginx:1.22
# Image:          nginx:1.22
```

## Task 5: Rollback

```bash
# 1. Break deployment with bad image
kubectl set image deployment/web-app nginx=nginx:broken

# Output:
# deployment.apps/web-app image updated

# 2. Watch rollout (it will hang)
kubectl rollout status deployment/web-app

# Output:
# Waiting for deployment "web-app" rollout to finish: 1 out of 2 new replicas have been updated...
# (hangs here)

# Cancel with Ctrl+C

# 3. Check Pod status
kubectl get pods -l app=web-app

# Output:
# NAME                       READY   STATUS             RESTARTS   AGE
# web-app-5b9c8d7e6f-a1b2c   1/1     Running            0          5m
# web-app-5b9c8d7e6f-d3e4f   1/1     Running            0          5m
# web-app-9f8e7d6c5b-g5h6i   0/1     ImagePullBackOff   0          30s

# Describe the failing Pod
kubectl describe pod web-app-9f8e7d6c5b-g5h6i

# Shows: Failed to pull image "nginx:broken": rpc error: manifest unknown

# 4. Rollback
kubectl rollout undo deployment/web-app

# Output:
# deployment.apps/web-app rolled back

# 5. Verify recovery
kubectl get pods -l app=web-app

# Output:
# NAME                       READY   STATUS    RESTARTS   AGE
# web-app-5b9c8d7e6f-a1b2c   1/1     Running   0          7m
# web-app-5b9c8d7e6f-d3e4f   1/1     Running   0          7m

kubectl describe pods -l app=web-app | grep Image:

# Output:
# Image:          nginx:1.22
# Image:          nginx:1.22
```

## Task 6: Cleanup

```bash
# 1. Delete Deployment
kubectl delete deployment web-app

# Output:
# deployment.apps "web-app" deleted

# 2. Verify all resources deleted
kubectl get deployments
kubectl get replicasets
kubectl get pods -l app=web-app

# All should return "No resources found"
```

## Bonus Challenge Solutions

### Bonus 1: Declarative Updates

```bash
# Create YAML
kubectl get deployment web-app -o yaml > web-app-deployment.yaml

# Edit the file (change replicas and image)
# Then apply
kubectl apply -f web-app-deployment.yaml
```

### Bonus 2: Custom Strategy

```yaml
# deployment-zero-downtime.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero-downtime-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # Can have up to 5 Pods during update
      maxUnavailable: 0    # Always keep 3 running (zero downtime)
  selector:
    matchLabels:
      app: zd-app
  template:
    metadata:
      labels:
        app: zd-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment-zero-downtime.yaml
kubectl set image deployment/zero-downtime-app nginx=nginx:1.22
kubectl get pods -w  # Watch the update - always 3+ Pods running
```

### Bonus 3: Revision History

```bash
# View all revisions with details
kubectl rollout history deployment/web-app

# View specific revision
kubectl rollout history deployment/web-app --revision=2

# Rollback to specific revision
kubectl rollout undo deployment/web-app --to-revision=1
```

### Bonus 4: Pause and Resume

```bash
# Create deployment
kubectl create deployment pause-test --image=nginx:1.21 --replicas=3

# Pause rollout
kubectl rollout pause deployment/pause-test

# Make multiple changes
kubectl set image deployment/pause-test nginx=nginx:1.22
kubectl set resources deployment/pause-test -c=nginx --limits=cpu=200m,memory=512Mi

# Changes are recorded but not applied

# Resume to apply all changes at once
kubectl rollout resume deployment/pause-test

# Watch the single rollout with all changes
kubectl rollout status deployment/pause-test
```

## Key Learnings

1. **Deployments provide self-healing:** Pods are automatically replaced
2. **Scaling is dynamic:** Easy to scale up or down
3. **Rolling updates minimize downtime:** Gradual replacement of Pods
4. **Rollback is instant:** Revert to previous working state
5. **ReplicaSets are managed automatically:** One for each revision
6. **Update strategy controls rollout behavior:** maxSurge and maxUnavailable

Excellent work! You now understand Deployments and can manage production-grade applications.

**Next:** [05-services - Services & Networking](../../../05-services/README.md)
