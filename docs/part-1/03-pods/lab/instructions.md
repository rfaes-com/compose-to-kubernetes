# Lab: Working with Pods

**Estimated Time:** 15 minutes

## Objectives

- Create Pods using both imperative and declarative methods
- Inspect Pod status and logs
- Work with multi-container Pods
- Practice using labels for organization

## Prerequisites

- Cluster is running (`kind create cluster --config /workspaces/compose-to-kubernetes/setup/kind/simple.yaml`)
- kubectl is configured

## Tasks

### Task 1: Create Pods Imperatively

1. Create a Pod named `redis-test` using the `redis:latest` image
2. Verify the Pod is running
3. Get the Pod's IP address
4. Delete the Pod

### Task 2: Create Pods Declaratively

1. Create a YAML manifest for a Pod named `nginx-web` with:
   - Image: `nginx:latest`
   - Label: `app=web`, `tier=frontend`
   - Container port: 80
2. Apply the manifest
3. Verify the Pod was created with the correct labels
4. Add a new label `environment=dev` to the running Pod
5. View the updated labels

### Task 3: Multi-Container Pod

1. Apply the multi-container Pod example: `kubectl apply -f /workspaces/compose-to-kubernetes/part-1/03-pods/examples/multi-container-pod.yaml`
2. Wait for the Pod to be running. Wait with a command, not just in front of the keyboard
3. Check logs from the `nginx` container
4. Check logs from the `log-collector` container (it will be empty initially)
5. Generate some traffic to nginx:
   - Port forward: `kubectl port-forward pod/web-with-logging 8080:80 &`
   - Make a request: `curl http://localhost:8080`
   - Stop port forwarding: `kill %1`
6. Check the log-collector logs again (you should see access log entries)
7. Exec into the nginx container and list the mounted volume: `kubectl exec web-with-logging -c nginx -- ls -la /var/log/nginx`

### Task 4: Pod Inspection

1. List all Pods with labels displayed: `kubectl get pods --show-labels`
2. Filter Pods by label: Get only Pods with `app=web`
3. Describe the `nginx-web` Pod and examine:
   - Pod IP
   - Node it's running on
   - Events
4. Get the Pod definition in YAML format

### Task 5: Cleanup

1. Delete the `nginx-web` Pod
2. Delete the `web-with-logging` Pod
3. Verify all Pods are deleted

## Validation

After completing all tasks, run:

```bash
# Should show no Pods
kubectl get pods

# If you see any test Pods still running, delete them
kubectl delete pod --all
```

## Hints

<details markdown="1">
<summary>Hint for Task 1.3: Getting Pod IP</summary>

Use the `-o wide` flag:

```bash
kubectl get pod redis-test -o wide
```

Or extract just the IP:

```bash
kubectl get pod redis-test -o jsonpath='{.status.podIP}'
```

</details>

<details markdown="1">
<summary>Hint for Task 2.1: YAML structure</summary>

Basic Pod structure:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
  labels:
    key: value
spec:
  containers:
    - name: <container-name>
      image: <image:tag>
      ports:
        - containerPort: <port>
```

</details>

<details markdown="1">
<summary>Hint for Task 2.4: Adding labels</summary>

Use the `label` command:

```bash
kubectl label pod <pod-name> <key>=<value>
```

</details>

<details markdown="1">
<summary>Hint for Task 3.3: Checking container logs</summary>

For multi-container Pods, specify the container:

```bash
kubectl logs <pod-name> -c <container-name>
```

</details>

<details markdown="1">
<summary>Hint for Task 4.2: Filtering by label</summary>

Use the `-l` or `--selector` flag:

```bash
kubectl get pods -l app=web
```

</details>

## Bonus Challenges

1. **Init Containers:** Create a Pod with an init container that runs before the main container
2. **Resource Limits:** Create a Pod with CPU and memory limits, then check the QoS class assigned to it
3. **Pod with Command:** Create a Pod that runs a custom command instead of the default image entrypoint
4. **Labels and Selectors:** Create 5 Pods with different combinations of labels, then practice filtering them

## Common Issues

| Issue                  | Solution                                                    |
| ---------------------- | ----------------------------------------------------------- |
| Pod stuck in `Pending` | Check: `kubectl describe pod <name>` for events             |
| Can't see logs         | Ensure Pod is `Running`, check container name is correct    |
| Port forward fails     | Verify Pod is running and port is correct                   |
| YAML syntax error      | Validate with `kubectl apply --dry-run=client -f file.yaml` |

## Solution

After attempting the lab, check [solutions/lab-solution.md](solutions/lab-solution.md) for complete solutions.

**Next:** [04-deployments](../../04-deployments/README.md)
