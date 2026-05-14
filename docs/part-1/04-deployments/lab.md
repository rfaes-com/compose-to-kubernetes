# Lab

**Duration:** 20 minutes

## Objectives

- Create and manage Deployments
- Practice scaling applications
- Perform rolling updates
- Rollback a failed update
- Understand self-healing behavior

## Prerequisites

- Cluster running
- Completed previous sections

## Tasks

### Task 1: Create a Deployment

1. Create a Deployment named `web-app` using `nginx:1.21` with 3 replicas

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    Imperative method:

    ```bash
    kubectl create deployment <name> --image=<image> --replicas=<count>
    ```

    </details>

2. Verify the Deployment is created
3. List the ReplicaSet(s) created by the Deployment
4. List the Pods created and note their names
5. Describe the Deployment and examine the events

<details class="solution" markdown="1">
<summary>Solution</summary>

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

</details>

### Task 2: Test Self-Healing

1. Watch the Pods: `kubectl get pods -l app=web-app -w`
2. In another terminal, delete one of the Pods
3. Observe in the watch window:
   - Pod terminates
   - New Pod is immediately created
   - Replica count is maintained
4. Stop watching (Ctrl+C)

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. Watch Pods (terminal 1)
kubectl get pods -l app=web-app -w

# 2. Delete a Pod (terminal 2)
kubectl delete pod web-app-7d8f5c5b6d-a1b2c

# 3. Observe in terminal 1:
# web-app-7d8f5c5b6d-a1b2c   1/1     Terminating   0          2m
# web-app-7d8f5c5b6d-j7k8l   0/1     Pending       0          0s
# web-app-7d8f5c5b6d-j7k8l   0/1     ContainerCreating   0   0s
# web-app-7d8f5c5b6d-j7k8l   1/1     Running             0   2s

# 4. Stop watching (Ctrl+C)
```

</details>

### Task 3: Scale the Deployment

1. Scale the Deployment to 5 replicas

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    ```bash
    kubectl scale deployment <name> --replicas=<count>
    ```

    </details>

2. Watch the new Pods being created
3. Scale back down to 2 replicas
4. Observe Pods being terminated
5. Verify final state has 2 Pods running

<details class="solution" markdown="1">
<summary>Solution</summary>

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

</details>

### Task 4: Rolling Update

1. Update the image to `nginx:1.22`

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    ```bash
    kubectl set image deployment/<name> <container-name>=<new-image>
    ```

    For nginx container named `nginx`:

    ```bash
    kubectl set image deployment/web-app nginx=nginx:1.22
    ```

    </details>

2. Watch the rollout status
3. Check the rollout history
4. List ReplicaSets - you should see two:
   - Old ReplicaSet with 0 Pods
   - New ReplicaSet with 2 Pods
5. Verify all Pods are running the new image: `kubectl describe pods -l app=web-app | grep Image:`

<details class="solution" markdown="1">
<summary>Solution</summary>

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

</details>

### Task 5: Rollback

1. Intentionally break the deployment by updating to a non-existent image: `nginx:broken`
2. Watch the rollout - it will hang because the image doesn't exist
3. Check Pod status - new Pods will be in `ImagePullBackOff`
4. Rollback to the previous version

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    ```bash
    kubectl rollout undo deployment/<name>
    ```

    </details>

5. Verify the Pods are running correctly again with `nginx:1.22`

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. Break deployment with bad image
kubectl set image deployment/web-app nginx=nginx:broken

# Output:
# deployment.apps/web-app image updated

# 2. Watch rollout (it will hang)
kubectl rollout status deployment/web-app

# Output:
# Waiting for deployment "web-app" rollout to finish: 1 out of 2 new replicas have been updated...
# (hangs here - cancel with Ctrl+C)

# 3. Check Pod status
kubectl get pods -l app=web-app

# Output:
# NAME                       READY   STATUS             RESTARTS   AGE
# web-app-5b9c8d7e6f-a1b2c   1/1     Running            0          5m
# web-app-5b9c8d7e6f-d3e4f   1/1     Running            0          5m
# web-app-9f8e7d6c5b-g5h6i   0/1     ImagePullBackOff   0          30s

# Describe the failing Pod to see the error
kubectl describe pod web-app-9f8e7d6c5b-g5h6i

# 4. Rollback
kubectl rollout undo deployment/web-app

# Output:
# deployment.apps/web-app rolled back

# 5. Verify recovery
kubectl get pods -l app=web-app
kubectl describe pods -l app=web-app | grep Image:

# Output:
# Image:          nginx:1.22
# Image:          nginx:1.22
```

</details>

### Task 6: Cleanup

1. Delete the Deployment
2. Verify that the ReplicaSet and all Pods are also deleted

<details class="solution" markdown="1">
<summary>Solution</summary>

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

</details>

## Validation

```bash
# After Task 5, before cleanup
kubectl get deployment web-app
# Should show 2/2 ready

kubectl rollout history deployment/web-app
# Should show multiple revisions

# After cleanup
kubectl get deployments
# Should show no resources
```

## Bonus Challenges

**1. Declarative Updates:** Create a YAML file for the Deployment, modify the replicas and image, then apply

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Export current deployment to YAML
kubectl create deployment web-app --image=nginx:1.21 --replicas=3 --dry-run=client -o yaml > web-app-deployment.yaml

# Edit the file (change replicas to 2 and image to nginx:1.22)
# Then apply
kubectl apply -f web-app-deployment.yaml

# Cleanup
kubectl delete deployment web-app
rm web-app-deployment.yaml
```

</details>

**2. Custom Strategy:** Create a Deployment with `maxSurge: 2` and `maxUnavailable: 0` (zero downtime guarantee)

<details class="solution" markdown="1">
<summary>Solution</summary>

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
      maxSurge: 2
      maxUnavailable: 0
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

# Cleanup
kubectl delete deployment zero-downtime-app
```

</details>

**3. Revision History:** Explore different revisions and rollback to a specific revision number

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create deployment and make a few updates to build revision history
kubectl create deployment rev-test --image=nginx:1.20 --replicas=2
kubectl set image deployment/rev-test nginx=nginx:1.21
kubectl set image deployment/rev-test nginx=nginx:1.22

# View all revisions with details
kubectl rollout history deployment/rev-test

# View specific revision
kubectl rollout history deployment/rev-test --revision=2

# Rollback to specific revision
kubectl rollout undo deployment/rev-test --to-revision=1

# Cleanup
kubectl delete deployment rev-test
```

</details>

**4. Pause and Resume:** Pause a rollout, make multiple changes, then resume

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create deployment
kubectl create deployment pause-test --image=nginx:1.21 --replicas=3

# Pause rollout
kubectl rollout pause deployment/pause-test

# Make multiple changes (not applied yet)
kubectl set image deployment/pause-test nginx=nginx:1.22
kubectl set resources deployment/pause-test -c=nginx --limits=cpu=200m,memory=512Mi

# Resume to apply all changes at once
kubectl rollout resume deployment/pause-test

# Watch the single rollout with all changes
kubectl rollout status deployment/pause-test

# Cleanup
kubectl delete deployment pause-test
```

</details>

## Common issues

| Issue                | Solution                                                            |
| -------------------- | ------------------------------------------------------------------- |
| ImagePullBackOff     | Check image name and tag are correct                                |
| Deployment stuck     | Check events: `kubectl describe deployment <name>`                  |
| Pods not scaling     | Check node resources: `kubectl describe nodes`                      |
| Rollback not working | Verify revision exists: `kubectl rollout history deployment/<name>` |

## Key takeaways

1. **Deployments provide self-healing:** Pods are automatically replaced
2. **Scaling is dynamic:** Easy to scale up or down with one command
3. **Rolling updates minimize downtime:** Gradual replacement of Pods
4. **Rollback is instant:** Revert to any previous revision
5. **ReplicaSets are managed automatically:** One per Deployment revision
6. **Update strategy controls rollout behavior:** `maxSurge` and `maxUnavailable`

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../05-services/README.md)
