# Lab Solutions: Persistent Storage

Complete solutions for the Persistent Storage lab exercises.

## Setup: Install Local Path Provisioner

```bash
# Check existing StorageClasses
kubectl get storageclass
```

**Expected output (if none exist):**

```text
No resources found
```

```bash
# Install local-path-provisioner
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

**Expected output:**

```text
namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
deployment.apps/local-path-provisioner created
storageclass.storage.k8s.io/local-path created
configmap/local-path-config created
```

```bash
# Set as default StorageClass
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**Expected output:**

```text
storageclass.storage.k8s.io/local-path patched
```

```bash
# Verify
kubectl get storageclass
```

**Expected output:**

```text
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   30s
```

## Task 1: EmptyDir Volume

```bash
# Create Pod with shared emptyDir volume
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
```

**Expected output:**

```text
pod/shared-volume created
```

```bash
# Verify pod is running
kubectl get pod shared-volume
```

**Expected output:**

```text
NAME            READY   STATUS    RESTARTS   AGE
shared-volume   2/2     Running   0          10s
```

## Task 2: Verify EmptyDir Shared Storage

```bash
# Wait for pod to be ready
kubectl wait --for=condition=Ready pod/shared-volume --timeout=30s
```

**Expected output:**

```text
pod/shared-volume condition met
```

```bash
# Check writer logs (last 5 lines)
kubectl logs shared-volume -c writer --tail=5
```

**Expected output:**

```text
Tue Jan  2 12:34:56 UTC 2024
Tue Jan  2 12:35:01 UTC 2024
Tue Jan  2 12:35:06 UTC 2024
Tue Jan  2 12:35:11 UTC 2024
Tue Jan  2 12:35:16 UTC 2024
```

```bash
# Check reader logs (last 10 lines)
kubectl logs shared-volume -c reader --tail=10
```

**Expected output:**

```text
Reading:
Tue Jan  2 12:35:06 UTC 2024
Reading:
Tue Jan  2 12:35:11 UTC 2024
Reading:
Tue Jan  2 12:35:16 UTC 2024
```

Both containers can access the same data!

```bash
# Verify file exists in both containers
kubectl exec shared-volume -c writer -- ls -lh /cache/
kubectl exec shared-volume -c reader -- wc -l /cache/data.txt
```

**Expected output:**

```text
total 4K
-rw-r--r--    1 root     root         840 Jan  2 12:35 data.txt

28 /cache/data.txt
```

```bash
# Delete pod to show data is lost
kubectl delete pod shared-volume
```

**Expected output:**

```text
pod "shared-volume" deleted
```

```bash
# Try to access it (will fail)
kubectl exec shared-volume -- cat /cache/data.txt
```

**Expected output:**

```text
Error from server (NotFound): pods "shared-volume" not found
```

The data is gone with the Pod!

## Task 3: Create PersistentVolumeClaim

```bash
# Create PVC
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

**Expected output:**

```text
persistentvolumeclaim/my-pvc created
```

```bash
# Check PVC status (may be Pending until bound to a Pod)
kubectl get pvc my-pvc
```

**Expected output:**

```text
NAME     STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Pending                                      local-path     5s
```

Status is "Pending" because local-path uses `WaitForFirstConsumer` binding mode.

```bash
# Describe PVC
kubectl describe pvc my-pvc
```

**Expected output:**

```text
Name:          my-pvc
Namespace:     default
StorageClass:  local-path
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Events:
  Type    Reason                Age   From                         Message
  ----    ------                ----  ----                         -------
  Normal  WaitForFirstConsumer  5s    persistentvolume-controller  waiting for first consumer to be created before binding
```

## Task 4: Use PVC in a Pod

```bash
# Create Pod using PVC
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
        echo "========================"
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

**Expected output:**

```text
pod/pvc-pod created
```

```bash
# Wait for pod (may take a moment for volume provisioning)
kubectl wait --for=condition=Ready pod/pvc-pod --timeout=60s
```

**Expected output:**

```text
pod/pvc-pod condition met
```

```bash
# Now PVC should be Bound
kubectl get pvc my-pvc
```

**Expected output:**

```text
NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Bound    pvc-12345678-1234-1234-1234-123456789012   1Gi        RWO            local-path     2m
```

```bash
# Check PV was automatically created
kubectl get pv
```

**Expected output:**

```text
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   AGE
pvc-12345678-1234-1234-1234-123456789012   1Gi        RWO            Delete           Bound    default/my-pvc   local-path     1m
```

```bash
# Check pod logs
kubectl logs pvc-pod
```

**Expected output:**

```text
=== Log file contents ===
Pod started at: Tue Jan  2 12:40:15 UTC 2024
========================
Sleeping...
```

```bash
# Check log file content
kubectl exec pvc-pod -- cat /data/log.txt
```

**Expected output:**

```text
Pod started at: Tue Jan  2 12:40:15 UTC 2024
```

## Task 5: Verify Data Persistence

```bash
# Check current log file
kubectl exec pvc-pod -- cat /data/log.txt
```

**Expected output:**

```text
Pod started at: Tue Jan  2 12:40:15 UTC 2024
```

```bash
# Delete the pod
kubectl delete pod pvc-pod
```

**Expected output:**

```text
pod "pvc-pod" deleted
```

```bash
# PVC still exists
kubectl get pvc my-pvc
```

**Expected output:**

```text
NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Bound    pvc-12345678-1234-1234-1234-123456789012   1Gi        RWO            local-path     5m
```

```bash
# Create a new pod with the same PVC
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
        echo "========================"
        echo "Data persisted across pod deletion!"
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

**Expected output:**

```text
pod/pvc-pod-2 created
```

```bash
# Wait for new pod
kubectl wait --for=condition=Ready pod/pvc-pod-2 --timeout=30s
```

**Expected output:**

```text
pod/pvc-pod-2 condition met
```

```bash
# Check logs - should show BOTH timestamps!
kubectl logs pvc-pod-2
```

**Expected output:**

```text
=== Log file contents ===
Pod started at: Tue Jan  2 12:40:15 UTC 2024
Second pod started at: Tue Jan  2 12:45:30 UTC 2024
========================
Data persisted across pod deletion!
```

```bash
# Verify file contains both entries
kubectl exec pvc-pod-2 -- cat /data/log.txt
```

**Expected output:**

```text
Pod started at: Tue Jan  2 12:40:15 UTC 2024
Second pod started at: Tue Jan  2 12:45:30 UTC 2024
```

Success! Data persisted across Pod deletion.

## Task 6: Deployment with PVC

```bash
# First, delete the test pod
kubectl delete pod pvc-pod-2
```

**Expected output:**

```text
pod "pvc-pod-2" deleted
```

```bash
# Create Deployment with PVC
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-with-storage
spec:
  replicas: 1  # Only 1 because ReadWriteOnce
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
                    echo '<p>Pod: '$HOSTNAME'</p>' >> /usr/share/nginx/html/index.html
                    echo '<p>Time: '$(date)'</p>' >> /usr/share/nginx/html/index.html
                  fi
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: my-pvc
EOF
```

**Expected output:**

```text
deployment.apps/web-with-storage created
```

```bash
# Wait for deployment
kubectl wait --for=condition=Available deployment/web-with-storage --timeout=60s
```

**Expected output:**

```text
deployment.apps/web-with-storage condition met
```

```bash
# Check pods
kubectl get pods -l app=web
```

**Expected output:**

```text
NAME                                READY   STATUS    RESTARTS   AGE
web-with-storage-5c8b9d7f8d-abcde   1/1     Running   0          15s
```

```bash
# Check the HTML file
kubectl exec deployment/web-with-storage -- cat /usr/share/nginx/html/index.html
```

**Expected output:**

```text
<h1>Hello from PVC!</h1>
<p>Pod: web-with-storage-5c8b9d7f8d-abcde</p>
<p>Time: Tue Jan  2 12:50:00 UTC 2024</p>
```

```bash
# Test with curl
kubectl run curl-test --image=curlimages/curl:8.5.0 --rm -it --restart=Never -- curl -s http://web-with-storage/index.html
```

**Expected output:**

```text
<h1>Hello from PVC!</h1>
<p>Pod: web-with-storage-5c8b9d7f8d-abcde</p>
<p>Time: Tue Jan  2 12:50:00 UTC 2024</p>
pod "curl-test" deleted
```

```bash
# Create service for easier access
kubectl expose deployment web-with-storage --port=80 --type=NodePort --name=web-svc

# Get service
kubectl get svc web-svc
```

**Expected output:**

```text
NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
web-svc   NodePort   10.96.123.45    <none>        80:31234/TCP   5s
```

## Verification

```bash
# List all PVCs
kubectl get pvc
```

**Expected output:**

```text
NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Bound    pvc-12345678-1234-1234-1234-123456789012   1Gi        RWO            local-path     15m
```

```bash
# List all PVs
kubectl get pv
```

**Expected output:**

```text
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   AGE
pvc-12345678-1234-1234-1234-123456789012   1Gi        RWO            Delete           Bound    default/my-pvc   local-path     15m
```

```bash
# Describe PVC
kubectl describe pvc my-pvc
```

**Expected output:**

```text
Name:          my-pvc
Namespace:     default
StorageClass:  local-path
Status:        Bound
Volume:        pvc-12345678-1234-1234-1234-123456789012
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: rancher.io/local-path
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      1Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       web-with-storage-5c8b9d7f8d-abcde
Events:        <none>
```

```bash
# Check StorageClass
kubectl get storageclass local-path -o yaml
```

## Cleanup

```bash
# Delete deployment and service
kubectl delete deployment web-with-storage
kubectl delete service web-svc

# Delete PVC (this will also delete the PV with Delete reclaim policy)
kubectl delete pvc my-pvc

# Verify PV is gone
kubectl get pv
```

**Expected output:**

```text
No resources found
```

```bash
# Verify PVC is gone
kubectl get pvc
```

**Expected output:**

```text
No resources found in default namespace.
```

## Bonus Challenges

### Challenge 1: Multiple Pods with RWO

Try to create 2 replicas using the same RWO PVC.

```bash
# Create PVC
kubectl apply -f - <<EOF
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

# Try 2 replicas
kubectl create deployment rwo-test --image=nginx:1.25-alpine --replicas=2
kubectl set volume deployment/rwo-test --add --name=storage --type=persistentVolumeClaim --claim-name=rwo-test --mount-path=/data

# Check pods
kubectl get pods -l app=rwo-test
```

**Expected behavior:**
Only one Pod will run successfully. The second Pod will be in Pending state because the volume can only be mounted by one node at a time (ReadWriteOnce).

```bash
# Cleanup
kubectl delete deployment rwo-test
kubectl delete pvc rwo-test
```

### Challenge 2: Data Persistence with Rolling Update

```bash
# Create PVC
kubectl apply -f - <<EOF
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
kubectl set volume deployment/rolling-test --add --name=data --type=persistentVolumeClaim --claim-name=rolling-pvc --mount-path=/data

# Write data
kubectl wait --for=condition=Available deployment/rolling-test --timeout=60s
kubectl exec deployment/rolling-test -- sh -c 'echo "v1 data" > /data/version.txt'

# Update to v2
kubectl set image deployment/rolling-test nginx=nginx:1.25-alpine

# Wait for rollout
kubectl rollout status deployment/rolling-test

# Check data persisted
kubectl exec deployment/rolling-test -- cat /data/version.txt
```

**Expected output:**

```text
v1 data
```

Data persists across rolling updates!

```bash
# Cleanup
kubectl delete deployment rolling-test
kubectl delete pvc rolling-pvc
```

## Key Takeaways

1. **emptyDir** volumes are ephemeral - deleted when Pod is deleted
2. **PersistentVolumeClaims** provide persistent storage that survives Pod deletion
3. **StorageClasses** enable dynamic provisioning of PersistentVolumes
4. **ReadWriteOnce (RWO)** means only one node can mount the volume (but multiple Pods on same node can)
5. Data in PVCs persists across Pod restarts, updates, and even Deployment recreations
6. PVs created by dynamic provisioning are deleted when PVC is deleted (Reclaim Policy: Delete)
7. Use **StatefulSets** for applications needing per-Pod persistent storage

## Common Issues and Solutions

**Issue:** PVC stuck in Pending

- **Cause:** No matching PV or StorageClass using WaitForFirstConsumer
- **Solution:** Create a Pod that uses the PVC, or check StorageClass configuration

**Issue:** Pod pending with "FailedMount" error

- **Cause:** Volume not available, already mounted elsewhere (RWO), or provisioning failed
- **Solution:** Check `kubectl describe pod <name>` events, verify PVC is bound, ensure no other Pod is using RWO volume

**Issue:** Data not persisting

- **Cause:** Using emptyDir instead of PVC
- **Solution:** Ensure Pod spec references PersistentVolumeClaim, not emptyDir

**Issue:** Cannot scale deployment with PVC

- **Cause:** Using ReadWriteOnce with multiple replicas on different nodes
- **Solution:** Use ReadWriteMany if supported, or keep replicas=1, or use StatefulSet with per-Pod PVCs
