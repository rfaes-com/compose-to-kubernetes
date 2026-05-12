# Lab: Working with Deployments

**Estimated Time:** 20 minutes

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

2. Verify the Deployment is created

3. List the ReplicaSet(s) created by the Deployment

4. List the Pods created and note their names

5. Describe the Deployment and examine the events

### Task 2: Test Self-Healing

1. Watch the Pods: `kubectl get pods -l app=web-app -w`

2. In another terminal, delete one of the Pods

3. Observe in the watch window:
   - Pod terminates
   - New Pod is immediately created
   - Replica count is maintained

4. Stop watching (Ctrl+C)

### Task 3: Scale the Deployment

1. Scale the Deployment to 5 replicas

2. Watch the new Pods being created

3. Scale back down to 2 replicas

4. Observe Pods being terminated

5. Verify final state has 2 Pods running

### Task 4: Rolling Update

1. Update the image to `nginx:1.22`

2. Watch the rollout status

3. Check the rollout history

4. List ReplicaSets - you should see two:
   - Old ReplicaSet with 0 Pods
   - New ReplicaSet with 2 Pods

5. Verify all Pods are running the new image: `kubectl describe pods -l app=web-app | grep Image:`

### Task 5: Rollback

1. Intentionally break the deployment by updating to a non-existent image: `nginx:broken`

2. Watch the rollout - it will hang because the image doesn't exist

3. Check Pod status - new Pods will be in `ImagePullBackOff`

4. Rollback to the previous version

5. Verify the Pods are running correctly again with `nginx:1.22`

### Task 6: Cleanup

1. Delete the Deployment

2. Verify that the ReplicaSet and all Pods are also deleted

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

## Hints

<details>
<summary>Hint for Task 1.1: Creating Deployment</summary>

Imperative method:

```bash
kubectl create deployment <name> --image=<image> --replicas=<count>
```

Or declarative with YAML (recommended).

</details>

<details>
<summary>Hint for Task 3.1: Scaling</summary>

```bash
kubectl scale deployment <name> --replicas=<count>
```

</details>

<details>
<summary>Hint for Task 4.1: Updating image</summary>

```bash
kubectl set image deployment/<name> <container-name>=<new-image>
```

For nginx container named `nginx`:

```bash
kubectl set image deployment/web-app nginx=nginx:1.22
```

</details>

<details>
<summary>Hint for Task 5.4: Rollback</summary>

```bash
kubectl rollout undo deployment/<name>
```

</details>

## Bonus Challenges

1. **Declarative Updates:** Create a YAML file for the Deployment, modify the replicas and image, then apply
2. **Custom Strategy:** Create a Deployment with `maxSurge: 2` and `maxUnavailable: 0` (zero downtime guarantee)
3. **Revision History:** Explore different revisions and rollback to a specific revision number
4. **Pause and Resume:** Pause a rollout, make multiple changes, then resume

## Common Issues

| Issue                | Solution                                                            |
| -------------------- | ------------------------------------------------------------------- |
| ImagePullBackOff     | Check image name and tag are correct                                |
| Deployment stuck     | Check events: `kubectl describe deployment <name>`                  |
| Pods not scaling     | Check node resources: `kubectl describe nodes`                      |
| Rollback not working | Verify revision exists: `kubectl rollout history deployment/<name>` |

## Solution

Complete solution available in [solutions/lab-solution.md](solutions/lab-solution.md).

**Next:** [05-services](../../05-services/README.md)
