# Lab: Environment Verification

**Estimated Time:** 10 minutes

## Objectives

- Verify cluster is running correctly
- Practice basic kubectl commands
- Create and interact with a Pod
- Explore using k9s

## Prerequisites

- Workshop container running
- Inside the `/workspace` directory

## Tasks

### Task 1: Cluster Verification

1. List all nodes in the cluster
2. Verify both nodes show `STATUS: Ready`
3. Display cluster information
4. View your kubeconfig (redacted for security)

**Expected Output:**

- 2 nodes: `workshop-control-plane` and `workshop-worker`
-Both with `Ready` status
- Cluster API server address displayed

### Task 2: Create and Inspect a Pod

1. Create a Pod named `test-busybox` using the `busybox:latest` image that runs the command `sleep 3600`

2. Wait for the Pod to be in `Running` state

3. Get detailed information about the Pod (describe)

4. Execute the command `echo "Hello from inside the Pod!"` in the Pod

5. Check the Pod's logs (even though it's just sleeping, there shouldn't be much)

6. Get an interactive shell into the Pod and run:
   - `hostname` (should show Pod name)
   - `ls /`
   - `exit`

7. Delete the Pod

**Expected Outcomes:**

- Pod successfully created and running
- Commands executed successfully
- Pod cleanly deleted

### Task 3: Explore with k9s

1. Launch k9s

2. Navigate to view Pods (`:pods`)

3. Navigate to view Nodes (`:nodes`)

4. Select the worker node and describe it (press `d`)

5. Navigate to `kube-system` namespace and view system Pods:
   - Type `:pods` then press Enter
   - Type `/kube-system` to filter
   - Browse the system Pods

6. Exit k9s

### Task 4: Context and Namespaces

1. List all available contexts

2. Display the current context (should show `kind-workshop`)

3. List all namespaces in the cluster

**Expected Output:**

- Context: `kind-workshop`
- Namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`

## Validation

Run these commands to verify your work:

```bash
# Verify cluster (should show 2 nodes)
kubectl get nodes

# Verify test Pod is deleted (should show "No resources found")
kubectl get pod test-busybox

# Verify you're in the right context
kubectl config current-context
# Should output: kind-workshop
```

## Hints

<details markdown="1">
<summary>Hint for Task 2.1: Creating a Pod with a command</summary>

Use `kubectl run` with the `--command` flag or `--` to specify the command:

```bash
kubectl run <pod-name> --image=<image> -- <command> <args>
```

For sleep, the command is `sleep` and the arg is `3600`.

</details>

<details markdown="1">
<summary>Hint for Task 2.4: Executing a command in a Pod</summary>

Use `kubectl exec`:

```bash
kubectl exec <pod-name> -- <command>
```

For echo, the full command is `echo "Hello from inside the Pod!"` but you may need to handle quotes.

</details>

<details markdown="1">
<summary>Hint for Task 2.6: Interactive shell</summary>

Use `kubectl exec` with `-it` flags:

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

Note: busybox uses `/bin/sh`, not `/bin/bash`.

</details>

<details markdown="1">
<summary>Hint for Task 3: k9s namespace filtering</summary>

In k9s, you can filter by namespace by typing `/` followed by search text, or use `0-9` keys to select namespace, or type `:` followed by resource type and namespace like `:pods default`.

</details>

## Solution

After completing the lab, compare your work with the [solutions/lab-solution.md](solutions/lab-solution.md).

## Bonus Challenges

1. **Create multiple Pods:** Create 3 Pods with different names but same image. Delete them all at once.

2. **Port forwarding:** Create an nginx Pod, forward port 8080 locally to port 80 in the Pod, and curl it.

3. **Resource inspection:** Find out how much CPU and memory the `workshop-worker` node has allocated.

4. **System exploration:** Use `kubectl get all -n kube-system` to see all resources in the system namespace. What do you notice?

## Troubleshooting

| Issue                     | Solution                                                  |
| ------------------------- | --------------------------------------------------------- |
| Pod stays in `Pending`    | Check: `kubectl describe pod <name>` for events           |
| "command not found" error | Verify you're using `/bin/sh` not `/bin/bash` for busybox |
| Can't delete Pod          | Use `kubectl delete pod <name> --force --grace-period=0`  |
| k9s won't start           | Verify cluster is running: `kubectl get nodes`            |

**Time Check:** If you've completed all tasks, you should have spent about 10 minutes. Move on when ready!

**Next:** [03-pods](../../03-pods/README.md)
