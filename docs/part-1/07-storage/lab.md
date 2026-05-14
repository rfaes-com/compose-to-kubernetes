# Lab

**Duration:** 30 minutes

## Objectives

- Work with ephemeral emptyDir volumes
- Create and use PersistentVolumeClaims
- Understand dynamic provisioning
- Verify data persistence across Pod restarts
- Work with Deployments and persistent storage

## Prerequisites

- Kind cluster running
- kubectl configured

## Setup: Install Local Path Provisioner (if not already installed)

```bash
# Check if you already have a default StorageClass
kubectl get storageclass

# If no StorageClass exists, install local-path-provisioner
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# Set it as default
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
```

## Tasks

### Task 1: EmptyDir Volume

Create a Pod with two containers sharing an emptyDir volume.

**Requirements:**

- Pod name: `shared-volume`
- Container 1 (writer): busybox, writes timestamp to `/cache/data.txt` every 5 seconds
- Container 2 (reader): busybox, reads from `/cache/data.txt` every 5 seconds
- Use emptyDir volume mounted at `/cache` in both containers

<details class="hint" markdown="1">
<summary>Hint</summary>

In the Pod spec, define an `emptyDir` volume and mount it in both containers:

```yaml
volumes:
- name: cache
  emptyDir: {}
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
      - while true; do echo "\$(date)" >> /cache/data.txt; sleep 5; done
    volumeMounts:
    - name: cache
      mountPath: /cache
  - name: reader
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
      - while true; do echo "Reading:"; tail -1 /cache/data.txt 2>/dev/null || echo "Waiting for data..."; sleep 5; done
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir: {}
EOF

# Output:
# pod/shared-volume created

kubectl get pod shared-volume
```

</details>

### Task 2: Verify EmptyDir Shared Storage

Check that both containers can access the shared volume and that data is lost when the Pod is deleted.

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Wait for pod to be ready
kubectl wait --for=condition=Ready pod/shared-volume --timeout=30s

# Check writer logs (last 5 lines)
kubectl logs shared-volume -c writer --tail=5

# Check reader logs (last 10 lines)
kubectl logs shared-volume -c reader --tail=10
# Reader should show the same timestamps writer is producing

# Verify file exists in both containers
kubectl exec shared-volume -c writer -- ls -lh /cache/
kubectl exec shared-volume -c reader -- wc -l /cache/data.txt

# Delete pod to show data is lost
kubectl delete pod shared-volume

# Try to access it (will fail - pod gone)
kubectl get pod shared-volume
# Output: Error from server (NotFound)
```

</details>

### Task 3: Create PersistentVolumeClaim

Create a PVC using dynamic provisioning.

**Requirements:**

- PVC name: `my-pvc`
- Storage: 1Gi
- AccessMode: ReadWriteOnce
- StorageClass: local-path (or standard)

<details class="hint" markdown="1">
<summary>Hint</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
EOF

# Output:
# persistentvolumeclaim/my-pvc created

# Check PVC status (may be Pending until bound to a Pod)
kubectl get pvc my-pvc

# Output:
# NAME     STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc   Pending                                      local-path     5s
# Status is Pending because local-path uses WaitForFirstConsumer binding mode
```

</details>

### Task 4: Use PVC in a Pod

Create a Pod that uses the PVC and writes data to it.

**Requirements:**

- Pod name: `pvc-pod`
- Image: busybox:1.36
- Mount PVC at `/data`
- Write timestamp to `/data/log.txt`

<details class="hint" markdown="1">
<summary>Hint</summary>

Reference the PVC in the Pod spec volumes:

```yaml
volumes:
- name: storage
  persistentVolumeClaim:
    claimName: my-pvc
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
      - |
        echo "Pod started at: \$(date)" >> /data/log.txt
        echo "=== Log file contents ==="
        cat /data/log.txt
        echo "Sleeping..."
        sleep 3600
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
EOF

# Wait for pod (volume provisioning may take a moment)
kubectl wait --for=condition=Ready pod/pvc-pod --timeout=60s

# Now PVC should be Bound
kubectl get pvc my-pvc
# Output:
# NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc   Bound    ...      1Gi        RWO            local-path     2m

# Check log file content
kubectl exec pvc-pod -- cat /data/log.txt
```

</details>

### Task 5: Verify Data Persistence

Delete and recreate the Pod to verify data persists.

**Requirements:**

- View the contents of `/data/log.txt` in the first Pod
- Delete the Pod
- Create a new Pod with the same PVC
- Verify the data still exists

<details class="hint" markdown="1">
<summary>Hint</summary>

When a Pod is deleted, the PVC remains intact. Create a new Pod referencing the same `claimName` to access the persisted data.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Check current log file
kubectl exec pvc-pod -- cat /data/log.txt
# Output:
# Pod started at: Tue Jan  2 12:40:15 UTC 2024

# Delete the pod (PVC remains)
kubectl delete pod pvc-pod

# PVC still exists
kubectl get pvc my-pvc
# Output:
# NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc   Bound    ...      1Gi        RWO            local-path     5m

# Create a new pod with same PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod-2
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
      - |
        echo "Second pod started at: \$(date)" >> /data/log.txt
        echo "=== Log file contents ==="
        cat /data/log.txt
        echo "Data persisted!"
        sleep 3600
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
EOF

kubectl wait --for=condition=Ready pod/pvc-pod-2 --timeout=30s
kubectl logs pvc-pod-2

# Output:
# === Log file contents ===
# Pod started at: Tue Jan  2 12:40:15 UTC 2024
# Second pod started at: Tue Jan  2 12:45:30 UTC 2024
# Data persisted!
```

</details>

### Task 6: Deployment with PVC

Create a Deployment using the same PVC.

**Requirements:**

- Deployment name: `web-with-storage`
- Image: nginx:1.25-alpine
- 1 replica (ReadWriteOnce)
- Mount PVC at `/usr/share/nginx/html`
- Create a custom index.html in the volume

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Delete pvc-pod-2 first to release the RWO volume
kubectl delete pod pvc-pod-2

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-with-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        lifecycle:
          postStart:
            exec:
              command:
                - /bin/sh
                - -c
                - |
                  if [ ! -f /usr/share/nginx/html/index.html ]; then
                    echo '<h1>Hello from PVC!</h1>' > /usr/share/nginx/html/index.html
                  fi
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: my-pvc
EOF

kubectl wait --for=condition=Available deployment/web-with-storage --timeout=60s

# Check the HTML file
kubectl exec deployment/web-with-storage -- cat /usr/share/nginx/html/index.html
```

</details>

## Verification

```bash
# List PVCs
kubectl get pvc

# List PVs (automatically created)
kubectl get pv

# Describe PVC
kubectl describe pvc my-pvc

# Check StorageClass
kubectl get storageclass
```

## Cleanup

```bash
# Delete deployment
kubectl delete deployment web-with-storage

# Delete PVC (this also deletes the PV with Delete reclaim policy)
kubectl delete pvc my-pvc

# Verify PV is gone
kubectl get pv
kubectl get pvc
```

## Bonus Challenges

**1. RWO Limitation:** Try to create 2 replicas using the same RWO PVC and observe what happens

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rwo-test
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
EOF

# Try 2 replicas with the same PVC
kubectl create deployment rwo-test --image=nginx:1.25-alpine --replicas=2
kubectl set volume deployment/rwo-test --add --name=storage \
  --type=persistentVolumeClaim --claim-name=rwo-test --mount-path=/data

# Check pods
kubectl get pods -l app=rwo-test
# Only one Pod will run; the second stays Pending because
# ReadWriteOnce can only be mounted by one node at a time

# Cleanup
kubectl delete deployment rwo-test
kubectl delete pvc rwo-test
```

</details>

**2. Data Persistence Through Rolling Update:** Verify data persists across a Deployment rolling update

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
# Create PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rolling-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
EOF

# Create deployment with v1
kubectl create deployment rolling-test --image=nginx:1.24-alpine
kubectl set volume deployment/rolling-test --add --name=data \
  --type=persistentVolumeClaim --claim-name=rolling-pvc --mount-path=/data

# Write data
kubectl wait --for=condition=Available deployment/rolling-test --timeout=60s
kubectl exec deployment/rolling-test -- sh -c 'echo "v1 data" > /data/version.txt'

# Update to v2
kubectl set image deployment/rolling-test nginx=nginx:1.25-alpine
kubectl rollout status deployment/rolling-test

# Check data persisted through the update
kubectl exec deployment/rolling-test -- cat /data/version.txt

# Output:
# v1 data

# Cleanup
kubectl delete deployment rolling-test
kubectl delete pvc rolling-pvc
```

</details>

## Common issues

| Issue                           | Solution                                                                                   |
| ------------------------------- | ------------------------------------------------------------------------------------------ |
| PVC stuck in Pending            | No matching StorageClass or using WaitForFirstConsumer — create a Pod that uses the PVC    |
| Pod pending with FailedMount    | Check `kubectl describe pod <name>`; verify PVC is Bound and not already mounted elsewhere |
| Data not persisting             | Ensure Pod spec references `persistentVolumeClaim`, not `emptyDir`                         |
| Can't scale Deployment with PVC | ReadWriteOnce only allows one node; use `replicas: 1` or switch to ReadWriteMany           |

## Key takeaways

1. **emptyDir** volumes are ephemeral — data is deleted when the Pod is deleted
2. **PersistentVolumeClaims** provide storage that survives Pod deletion and restarts
3. **StorageClasses** enable dynamic provisioning — no need to manually create PVs
4. **ReadWriteOnce (RWO)** means only one node can mount the volume at a time
5. **PVs created by dynamic provisioning are deleted when the PVC is deleted** (Delete reclaim policy)

## Next section

Once you've reviewed the content and completed the lab, proceed to the [next section](../08-namespaces/README.md)
