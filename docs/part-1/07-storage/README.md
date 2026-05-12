# Persistent Storage

**Duration:** 40 minutes

## Learning Objectives

- Understand the difference between ephemeral and persistent storage
- Learn about Volumes, PersistentVolumes (PV), and PersistentVolumeClaims (PVC)
- Understand StorageClasses and dynamic provisioning
- Mount persistent storage in Pods
- Understand volume access modes and reclaim policies

## Storage in Docker Compose vs Kubernetes

### Docker Compose Volumes

```yaml
services:
  database:
    image: postgres:16-alpine
    volumes:
      - db-data:/var/lib/postgresql/data  # Named volume
      - ./config:/etc/config:ro           # Bind mount

volumes:
  db-data:  # Managed by Docker
```

### Kubernetes Volumes

In Kubernetes, storage is more explicit and flexible:

- **Volumes**: Defined at Pod level, tied to Pod lifecycle
- **PersistentVolumes (PV)**: Cluster-level storage resources
- **PersistentVolumeClaims (PVC)**: Requests for storage
- **StorageClasses**: Templates for dynamic provisioning

## Storage Concepts

### 1. Volume Types

Kubernetes supports many volume types. Common ones:

**Ephemeral Volumes** (Pod lifecycle):

- `emptyDir`: Temporary directory, deleted when Pod terminates
- `configMap`: Mount ConfigMap data
- `secret`: Mount Secret data
- `downwardAPI`: Expose Pod metadata

**Persistent Volumes** (Independent lifecycle):

- `persistentVolumeClaim`: Reference to a PVC
- `hostPath`: Mount file/directory from node (for testing only)
- Cloud-specific: `awsElasticBlockStore`, `azureDisk`, `gcePersistentDisk`
- Network storage: `nfs`, `cephfs`, `glusterfs`
- `local`: Local storage on specific nodes

### 2. emptyDir Volumes

Temporary storage that exists as long as the Pod exists.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
spec:
  containers:
  - name: app
    image: nginx:1.25-alpine
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir: {}  # Stored on node's disk
    # emptyDir:
    #   medium: Memory  # Use tmpfs (RAM)
```

**Use cases:**

- Temporary cache
- Scratch space for computations
- Sharing data between containers in same Pod

### 3. PersistentVolumes and PersistentVolumeClaims

**PersistentVolume (PV)**: Cluster-level storage resource provisioned by admin or dynamically

**PersistentVolumeClaim (PVC)**: User request for storage

```text
┌─────────────────────────────────────────┐
│            Storage Layer                │
│  (Network Storage, Cloud Disks, etc.)   │
└───────────┬─────────────────────────────┘
            │
┌───────────▼─────────────────────────────┐
│      PersistentVolume (PV)              │
│  • Storage implementation               │
│  • Admin-provisioned or dynamic         │
│  • Cluster-wide resource                │
└───────────┬─────────────────────────────┘
            │ Bound
┌───────────▼─────────────────────────────┐
│   PersistentVolumeClaim (PVC)           │
│  • Storage request by user              │
│  • Specifies size and access mode       │
│  • Namespace-scoped                     │
└───────────┬─────────────────────────────┘
            │ Referenced
┌───────────▼─────────────────────────────┐
│           Pod                           │
│  • Mounts PVC as volume                 │
└─────────────────────────────────────────┘
```

### 4. Access Modes

- **ReadWriteOnce (RWO)**: Volume can be mounted read-write by single node
- **ReadOnlyMany (ROX)**: Volume can be mounted read-only by many nodes
- **ReadWriteMany (RWX)**: Volume can be mounted read-write by many nodes
- **ReadWriteOncePod (RWOP)**: Volume can be mounted read-write by single Pod

Not all volume types support all access modes.

### 5. Reclaim Policies

What happens to PV when PVC is deleted:

- **Retain**: Manual reclamation - PV remains, data preserved
- **Delete**: PV and underlying storage are deleted
- **Recycle**: (Deprecated) Basic scrub (`rm -rf /volume/*`)

### 6. StorageClasses

Define classes of storage with different properties.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

**Properties:**

- **provisioner**: Creates volumes (cloud provider, CSI driver, etc.)
- **parameters**: Provider-specific settings
- **reclaimPolicy**: What happens when PVC is deleted
- **volumeBindingMode**:
  - `Immediate`: Provision volume when PVC is created
  - `WaitForFirstConsumer`: Provision when Pod using PVC is scheduled

## In kind (Local Development)

kind doesn't have dynamic provisioning by default. You can use:

1. **hostPath volumes**: Testing only, data stored on node
2. **local-path provisioner**: Optional addon for dynamic provisioning
3. **emptyDir**: Ephemeral storage

For production-like testing, you can install the Rancher local-path provisioner:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## Example: Static Provisioning

### Step 1: Create PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-static
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data  # Path on the node (testing only)
```

### Step 2: Create PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-static
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi  # Must be <= PV capacity
```

Kubernetes automatically binds PVC to available PV.

### Step 3: Use PVC in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
      - |
        echo "Writing to persistent storage"
        echo "Data: $(date)" > /data/timestamp.txt
        cat /data/timestamp.txt
        sleep 3600
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: pvc-static
```

## Example: Dynamic Provisioning

With a StorageClass configured (like local-path in kind):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path  # References StorageClass
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-storage
spec:
  replicas: 1  # RWO: only 1 replica can mount
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["/bin/sh", "-c"]
        args:
          - |
            # Create/append to file
            echo "Pod started at: $(date)" >> /data/log.txt
            echo "Current log:"
            cat /data/log.txt
            sleep 3600
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: pvc-dynamic
```

PersistentVolume is automatically created by the StorageClass provisioner.

## StatefulSets and Storage

StatefulSets provide:

- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling

Each Pod in a StatefulSet gets its own PVC:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:  # Creates PVC for each Pod
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-path
      resources:
        requests:
          storage: 5Gi
```

This creates:

- `database-0` with PVC `data-database-0`
- `database-1` with PVC `data-database-1`
- `database-2` with PVC `data-database-2`

Each Pod gets its own persistent storage that survives Pod restarts.

## Volume Snapshots

Some storage systems support snapshots for backup/restore:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pvc-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: pvc-dynamic
```

Requires CSI driver support (not available in basic kind setup).

## Best Practices

1. **Use PVCs, not direct PVs**: Let Kubernetes handle binding
2. **Choose appropriate access modes**: Most apps need RWO
3. **Set resource requests**: Specify required storage size
4. **Use StorageClasses**: Enable dynamic provisioning
5. **Plan for backups**: Volume snapshots or external backup tools
6. **Consider StatefulSets**: For stateful applications needing persistent identities
7. **Test disaster recovery**: Verify your data survives Pod/Node failures
8. **Monitor storage usage**: Track PVC usage and available capacity
9. **Use appropriate reclaim policies**: Retain for production data
10. **Avoid hostPath in production**: Use proper storage solutions

## Storage for Different Workloads

| Workload             | Storage Type             | Access Mode | Notes                                      |
| -------------------- | ------------------------ | ----------- | ------------------------------------------ |
| Database (single)    | PVC                      | RWO         | Use StatefulSet                            |
| Database (clustered) | PVC per replica          | RWO         | StatefulSet with volumeClaimTemplates      |
| Shared files         | PVC                      | RWX         | Requires RWX-capable storage (NFS, CephFS) |
| Logs                 | emptyDir + log collector | -           | Use DaemonSet for log collection           |
| Cache                | emptyDir (Memory)        | -           | Temporary, fast access                     |
| Config files         | ConfigMap                | -           | Small configuration files                  |
| Build workspace      | emptyDir                 | -           | Temporary build artifacts                  |

## Troubleshooting Storage

```bash
# List PVs
kubectl get pv

# List PVCs
kubectl get pvc

# Describe PVC for details and events
kubectl describe pvc <pvc-name>

# Check StorageClasses
kubectl get storageclass

# Check Pod events for mount issues
kubectl describe pod <pod-name>

# View logs from provisioner
kubectl logs -n kube-system -l app=local-path-provisioner
```

Common issues:

- **PVC pending**: No available PV or StorageClass
- **Pod pending**: PVC not bound or access mode incompatible
- **Mount failed**: Permissions, path doesn't exist, or driver issue

## Docker Compose to Kubernetes: Storage

### Docker Compose

```yaml
services:
  app:
    image: myapp:latest
    volumes:
      - app-data:/var/lib/app
      - ./config:/etc/config:ro

volumes:
  app-data:
```

### Kubernetes

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yaml: |
    # configuration here
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: data
          mountPath: /var/lib/app
        - name: config
          mountPath: /etc/config
          readOnly: true
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data
      - name: config
        configMap:
          name: app-config
```

**Key differences:**

- Compose: Volumes managed automatically
- K8s: Explicit PVC creation and binding
- Compose: Bind mounts for local files
- K8s: ConfigMaps/Secrets for configuration

## Lab Exercise

See [Lab: Persistent Storage](lab/instructions.md) for hands-on practice with:

- Creating and using emptyDir volumes
- Creating PVCs and mounting in Pods
- Using dynamic provisioning with StorageClasses
- Verifying data persistence across Pod restarts
- Working with StatefulSets

## Key Takeaways

- Kubernetes separates storage provisioning (PV) from consumption (PVC)
- emptyDir is ephemeral, PVC provides persistence
- StorageClasses enable dynamic provisioning
- Access modes determine how volumes can be mounted
- StatefulSets provide per-Pod persistent storage
- Choose storage solutions based on workload requirements

## Check Your Understanding

1. What is the difference between a Volume and a PersistentVolume?
2. When would you use emptyDir vs PVC?
3. What does the ReadWriteOnce access mode mean?
4. How does a StorageClass enable dynamic provisioning?
5. Why use StatefulSet instead of Deployment for databases?
6. What happens to data in PVC when the Pod is deleted?

## Next Steps

Continue to [Resource Quotas and Namespaces](../08-namespaces/README.md) to learn about multi-tenancy and resource isolation.
