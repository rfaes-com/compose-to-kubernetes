# Lab

**Duration:** 15 minutes

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
    <details class="hint" markdown="1">
    <summary>Hint</summary>

    Use the `-o wide` flag:

    ```bash
    kubectl get pod redis-test -o wide
    ```

    Or extract just the IP:

    ```bash
    kubectl get pod redis-test -o jsonpath='{.status.podIP}'
    ```

    </details>

4. Delete the Pod

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. Create a Pod named redis-test
kubectl run redis-test --image=redis:latest

# Output:
# pod/redis-test created

# 2. Verify the Pod is running
kubectl get pods
# or watch until Running
kubectl get pods -w

# Output:
# NAME         READY   STATUS    RESTARTS   AGE
# redis-test   1/1     Running   0          10s

# 3. Get the Pod's IP address
kubectl get pod redis-test -o wide

# Output includes:
# NAME         READY   STATUS    RESTARTS   AGE   IP           NODE
# redis-test   1/1     Running   0          20s   10.244.1.2   workshop-worker

# Alternative: Extract just the IP
kubectl get pod redis-test -o jsonpath='{.status.podIP}'
# Output: 10.244.1.2

# 4. Delete the Pod
kubectl delete pod redis-test

# Output:
# pod "redis-test" deleted
```

</details>

### Task 2: Create Pods Declaratively

1. Create a YAML manifest for a Pod named `nginx-web` with:
      - Image: `nginx:latest`
      - Label: `app=web`, `tier=frontend`
      - Container port: 80

    <details class="hint" markdown="1">
    <summary>Hint</summary>

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

2. Apply the manifest
3. Verify the Pod was created with the correct labels
4. Add a new label `environment=dev` to the running Pod

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    Use the `label` command:

    ```bash
    kubectl label pod <pod-name> <key>=<value>
    ```

    </details>

5. View the updated labels

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. Create a YAML manifest
cat > nginx-web.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-web
  labels:
    app: web
    tier: frontend
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
EOF

# 2. Apply the manifest
kubectl apply -f nginx-web.yaml

# Output:
# pod/nginx-web created

# 3. Verify the Pod with labels
kubectl get pod nginx-web --show-labels

# Output:
# NAME        READY   STATUS    RESTARTS   AGE   LABELS
# nginx-web   1/1     Running   0          5s    app=web,tier=frontend

# 4. Add a new label
kubectl label pod nginx-web environment=dev

# Output:
# pod/nginx-web labeled

# 5. View updated labels
kubectl get pod nginx-web --show-labels

# Output:
# NAME        READY   STATUS    RESTARTS   AGE   LABELS
# nginx-web   1/1     Running   0          20s   app=web,environment=dev,tier=frontend
```

</details>

### Task 3: Multi-Container Pod

1. Apply the multi-container Pod example: `kubectl apply -f /workspaces/compose-to-kubernetes/docs/part-1/03-pods/examples/multi-container-pod.yaml`
2. Wait for the Pod to be running. Wait with a command, not just in front of the keyboard
3. Check logs from the `nginx` container

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    For multi-container Pods, specify the container:

    ```bash
    kubectl logs <pod-name> -c <container-name>
    ```

    </details>

4. Check logs from the `log-collector` container (it will be empty initially)
5. Generate some traffic to nginx:
   - Port forward: `kubectl port-forward pod/web-with-logging 8080:80 &`
   - Make a request: `curl http://localhost:8080`
   - Stop port forwarding: `kill %1`
6. Check the log-collector logs again (you should see access log entries)
7. Exec into the nginx container and list the mounted volume: `kubectl exec web-with-logging -c nginx -- ls -la /var/log/nginx`

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. Apply the multi-container Pod
kubectl apply -f /workspaces/compose-to-kubernetes/docs/part-1/03-pods/examples/multi-container-pod.yaml

# Output:
# pod/web-with-logging created

# 2. Wait for Pod to be running
kubectl wait --for=condition=Ready pod/web-with-logging --timeout=60s

# Output:
# pod/web-with-logging condition met

# 3. Check logs from nginx container
kubectl logs web-with-logging -c nginx

# Output might be empty or show nginx startup messages

# 4. Check logs from log-collector
kubectl logs web-with-logging -c log-collector

# Output:
# (empty - no traffic yet)

# 5. Generate traffic
kubectl port-forward pod/web-with-logging 8080:80 &
# Output: Forwarding from 127.0.0.1:8080 -> 80

curl http://localhost:8080
# Output: HTML from nginx

# Stop port forwarding
kill %1

# 6. Check log-collector logs again
kubectl logs web-with-logging -c log-collector

# Output now shows:
# 127.0.0.1 - - [11/Mar/2024:10:30:45 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.68.0"

# 7. Exec into nginx and list volume
kubectl exec web-with-logging -c nginx -- ls -la /var/log/nginx

# Output:
# total 8
# drwxrwxrwx 2 root root   60 Mar 11 10:30 .
# drwxr-xr-x 1 root root   20 Mar 11 10:29 ..
# -rw-r--r-- 1 root root  190 Mar 11 10:30 access.log
# -rw-r--r-- 1 root root  537 Mar 11 10:29 error.log
```

</details>

### Task 4: Pod Inspection

1. List all Pods with labels displayed: `kubectl get pods --show-labels`
2. Filter Pods by label: Get only Pods with `app=web`

    <details class="hint" markdown="1">
    <summary>Hint</summary>

    Use the `-l` or `--selector` flag:

    ```bash
    kubectl get pods -l app=web
    ```

    </details>

3. Describe the `nginx-web` Pod and examine:
   - Pod IP
   - Node it's running on
   - Events
4. Get the Pod definition in YAML format

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. List all Pods with labels
kubectl get pods --show-labels

# Output:
# NAME               READY   STATUS    RESTARTS   AGE   LABELS
# nginx-web          1/1     Running   0          2m    app=web,environment=dev,tier=frontend
# web-with-logging   2/2     Running   0          1m    app=web-demo

# 2. Filter Pods by label
kubectl get pods -l app=web

# Output:
# NAME        READY   STATUS    RESTARTS   AGE
# nginx-web   1/1     Running   0          2m

# 3. Describe the nginx-web Pod
kubectl describe pod nginx-web

# Output includes:
# Name:             nginx-web
# Namespace:        default
# Labels:           app=web
#                   environment=dev
#                   tier=frontend
# Status:           Running
# IP:               10.244.1.3
# Node:             workshop-worker/172.18.0.3
# Events:
#   Type    Reason     Age   From               Message
#   ----    ------     ----  ----               -------
#   Normal  Scheduled  2m    default-scheduler  Successfully assigned default/nginx-web to workshop-worker
#   Normal  Pulling    2m    kubelet            Pulling image "nginx:latest"
#   Normal  Pulled     2m    kubelet            Successfully pulled image
#   Normal  Created    2m    kubelet            Created container nginx
#   Normal  Started    2m    kubelet            Started container nginx

# 4. Get Pod in YAML format
kubectl get pod nginx-web -o yaml
```

</details>

### Task 5: Cleanup

1. Delete the `nginx-web` Pod
2. Delete the `web-with-logging` Pod
3. Verify all Pods are deleted

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# 1. Delete nginx-web Pod
kubectl delete pod nginx-web

# Output:
# pod "nginx-web" deleted

# 2. Delete web-with-logging Pod
kubectl delete pod web-with-logging

# Output:
# pod "web-with-logging" deleted

# 3. Verify all Pods are deleted
kubectl get pods

# Output:
# No resources found in default namespace.
```

</details>

## Validation

After completing all tasks, run:

```bash
# Should show no Pods
kubectl get pods

# If you see any test Pods still running, delete them
kubectl delete pod --all
```

## Bonus Challenges

**1. Init Containers:** Create a Pod with an init container that runs before the main container

<details class="solution" markdown="1">
<summary>Solution</summary>

```yaml
# pod-with-init.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: init-setup
      image: busybox:latest
      command: ['sh', '-c', 'echo "Setup complete" > /work-dir/setup.txt']
      volumeMounts:
        - name: workdir
          mountPath: /work-dir
  containers:
    - name: main-app
      image: busybox:latest
      command: ['sh', '-c', 'cat /work-dir/setup.txt && sleep 3600']
      volumeMounts:
        - name: workdir
          mountPath: /work-dir
  volumes:
    - name: workdir
      emptyDir: {}
```

```bash
# Apply
kubectl apply -f pod-with-init.yaml

# Watch the init container run first
kubectl get pods -w

# Check logs of init container
kubectl logs init-demo -c init-setup

# Check logs of main container
kubectl logs init-demo -c main-app
# Output: Setup complete

# Cleanup
kubectl delete pod init-demo
```

</details>

**2. Resource Limits:** Create a Pod with CPU and memory limits, then check the QoS class assigned to it

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Apply the resource-limited Pod
kubectl apply -f /workspaces/compose-to-kubernetes/docs/part-1/03-pods/examples/pod-with-resources.yaml

# Check the QoS class
kubectl get pod resource-limited-pod -o jsonpath='{.status.qosClass}'
# Output: Burstable (because requests != limits)

# For Guaranteed QoS, requests must equal limits
```

</details>

**3. Pod with Command:** Create a Pod that runs a custom command instead of the default image entrypoint

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create Pod with custom command
kubectl run custom-cmd --image=busybox:latest -- sh -c "echo Hello World && sleep 3600"

# Check logs
kubectl logs custom-cmd
# Output: Hello World

# Cleanup
kubectl delete pod custom-cmd
```

</details>

**4. Labels and Selectors:** Create 5 Pods with different combinations of labels, then practice filtering them

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create Pods with different labels
kubectl run pod1 --image=nginx --labels="app=web,env=dev,tier=frontend"
kubectl run pod2 --image=nginx --labels="app=web,env=prod,tier=frontend"
kubectl run pod3 --image=nginx --labels="app=api,env=dev,tier=backend"
kubectl run pod4 --image=nginx --labels="app=api,env=prod,tier=backend"
kubectl run pod5 --image=nginx --labels="app=db,env=prod,tier=data"

# Filter by single label
kubectl get pods -l app=web

# Filter by multiple labels (AND)
kubectl get pods -l app=web,env=prod

# Filter by label value in set
kubectl get pods -l 'env in (dev,prod)'

# Exclude by label
kubectl get pods -l 'env!=prod'

# Cleanup
kubectl delete pod pod1 pod2 pod3 pod4 pod5
```

</details>

## Common mistakes

### Using /bin/bash in busybox

```bash
# Wrong (busybox doesn't have bash)
kubectl exec -it test-pod -- /bin/bash

# Correct
kubectl exec -it test-pod -- /bin/sh
```

### Wrong container name in multi-container Pod

```bash
# Wrong
kubectl logs web-with-logging -c web
# Error: container "web" not found

# Correct
kubectl logs web-with-logging -c nginx
```

### YAML indentation errors

```yaml
# Wrong
spec:
  containers:
  - name: nginx
  image: nginx  # Wrong indentation

# Correct
spec:
  containers:
    - name: nginx
      image: nginx
```

## Key takeaways

1. **Imperative vs Declarative:** `kubectl run` is fast for testing; YAML manifests are for reproducibility
2. **Multi-container Pods share network and volumes:** Containers can communicate via localhost and share mounted filesystems
3. **Labels are essential:** Used for organization, selection, and Service/Deployment targeting
4. **Pod status progression:** Pending → Running → Succeeded/Failed
5. **Container logs are separate:** In multi-container Pods, specify `-c <container-name>`

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../04-deployments/README.md)
