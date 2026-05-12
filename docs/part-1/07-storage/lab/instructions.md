# Lab: Persistent Storage

**Duration:** 30 minutes

## Objectives

- Work with ephemeral emptyDir volumes
- Create and use PersistentVolumeClaims
- Understand dynamic provisioning
- Verify data persistence across Pod restarts
- Work with StatefulSets and per-Pod storage

## Prerequisites

- Kind cluster running
- kubectl configured
- Optional: local-path-provisioner installed for dynamic provisioning

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

**Hints:**

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
      - while true; do echo "Reading:"; tail -1 /cache/data.txt; sleep 5; done
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir: {}
EOF
```

### Task 2: Verify EmptyDir Shared Storage

Check that both containers can access the shared volume.

**Requirements:**

- View logs from both containers
- Verify that reader sees data written by writer
- Verify data is lost when Pod is deleted

**Hints:**

```bash
# Wait for pod to be ready
kubectl wait --for=condition=Ready pod/shared-volume --timeout=30s

# Check writer logs
kubectl logs shared-volume -c writer

# Check reader logs
kubectl logs shared-volume -c reader

# Delete pod
kubectl delete pod shared-volume

# Verify pod is gone (and data is lost)
kubectl get pod shared-volume
```

### Task 3: Create PersistentVolumeClaim

Create a PVC using dynamic provisioning.

**Requirements:**

- PVC name: `my-pvc`
- Storage: 1Gi
- AccessMode: ReadWriteOnce
- StorageClass: local-path (or standard)

**Hints:**

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
```

### Task 4: Use PVC in a Pod

Create a Pod that uses the PVC and writes data to it.

**Requirements:**

- Pod name: `pvc-pod`
- Image: busybox:1.36
- Mount PVC at `/data`
- Write timestamp to `/data/log.txt`

**Hints:**

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
```

### Task 5: Verify Data Persistence

Delete and recreate the Pod to verify data persists.

**Requirements:**

- View the contents of `/data/log.txt` in the first Pod
- Delete the Pod
- Create a new Pod with the same PVC
- Verify the data still exists

**Hints:**

```bash
# Wait for pod
kubectl wait --for=condition=Ready pod/pvc-pod --timeout=30s

# Check log file
kubectl exec pvc-pod -- cat /data/log.txt

# Delete the pod (PVC remains)
kubectl delete pod pvc-pod

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

# Check logs - should see both timestamps
kubectl wait --for=condition=Ready pod/pvc-pod-2 --timeout=30s
kubectl exec pvc-pod-2 -- cat /data/log.txt
```

### Task 6: Deployment with PVC

Create a Deployment using the same PVC.

**Requirements:**

- Deployment name: `web-with-storage`
- Image: nginx:1.25-alpine
- 1 replica (ReadWriteOnce)
- Mount PVC at `/usr/share/nginx/html`
- Create a custom index.html in the volume

**Hints:**

```bash
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
              command: ["/bin/sh", "-c", "echo '<h1>Hello from PVC!</h1>' > /usr/share/nginx/html/index.html"]
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: my-pvc
EOF
```

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

# Test the web server
kubectl exec deployment/web-with-storage -- cat /usr/share/nginx/html/index.html
```

## Cleanup

```bash
# Delete pods
kubectl delete pod pvc-pod-2

# Delete deployment
kubectl delete deployment web-with-storage

# Delete PVC (this may delete PV depending on reclaim policy)
kubectl delete pvc my-pvc

# Verify PV is deleted or released
kubectl get pv
```

## Check Your Understanding

1. What happens to emptyDir data when a Pod is deleted?
2. What happens to PVC data when the Pod using it is deleted?
3. Can multiple Pods mount the same PVC with ReadWriteOnce access mode?
4. What is the purpose of a StorageClass?
5. When would you use a StatefulSet instead of a Deployment for storage?

## Next Steps

Proceed to [Namespaces and Resource Quotas](../../08-namespaces/README.md) to learn about resource isolation.
