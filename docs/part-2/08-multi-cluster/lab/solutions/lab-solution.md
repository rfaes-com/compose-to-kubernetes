# Lab Solutions: Multi-Cluster Management

Complete solutions for the Multi-Cluster Management lab exercises.

## Task 1: Create Two Clusters

```bash
ORIGINAL_CONTEXT=$(kubectl config current-context)
echo $ORIGINAL_CONTEXT

kind create cluster --name c2k-primary
kind create cluster --name c2k-secondary

kind get clusters
kubectl config get-contexts
```

## Task 2: Practice Context-Safe Commands

```bash
PRIMARY=kind-c2k-primary
SECONDARY=kind-c2k-secondary

kubectl --context=$PRIMARY get nodes
kubectl --context=$SECONDARY get nodes
kubectl config current-context
```

## Task 3: Create a Shared Application Manifest

Create `hello-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: multi-cluster-demo
```

Create `hello-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: multi-cluster-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        env:
        - name: CLUSTER_NAME
          value: primary
        ports:
        - containerPort: 80
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo "hello from ${CLUSTER_NAME}" > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
```

Create `hello-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: multi-cluster-demo
spec:
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 80
```

## Task 4: Deploy to the Primary Cluster

```bash
kubectl --context=$PRIMARY apply -f hello-namespace.yaml
kubectl --context=$PRIMARY apply -f hello-deployment.yaml
kubectl --context=$PRIMARY apply -f hello-service.yaml

kubectl --context=$PRIMARY wait --for=condition=Available deployment/hello \
  -n multi-cluster-demo --timeout=90s

kubectl --context=$PRIMARY port-forward -n multi-cluster-demo svc/hello 8080:80
```

In another terminal:

```bash
curl http://localhost:8080
```

**Expected output:**

```text
hello from primary
```

## Task 5: Deploy to the Secondary Cluster

```bash
kubectl --context=$SECONDARY apply -f hello-namespace.yaml
kubectl --context=$SECONDARY apply -f hello-deployment.yaml
kubectl --context=$SECONDARY apply -f hello-service.yaml

kubectl --context=$SECONDARY set env deployment/hello \
  -n multi-cluster-demo CLUSTER_NAME=secondary

kubectl --context=$SECONDARY rollout restart deployment/hello -n multi-cluster-demo
kubectl --context=$SECONDARY rollout status deployment/hello -n multi-cluster-demo

kubectl --context=$SECONDARY port-forward -n multi-cluster-demo svc/hello 8081:80
```

In another terminal:

```bash
curl http://localhost:8081
```

**Expected output:**

```text
hello from secondary
```

## Task 6: Compare Cluster State

```bash
kubectl --context=$PRIMARY get pods,svc,deploy -n multi-cluster-demo
kubectl --context=$SECONDARY get pods,svc,deploy -n multi-cluster-demo

kubectl --context=$PRIMARY scale deployment hello -n multi-cluster-demo --replicas=3
kubectl --context=$PRIMARY get deployment hello -n multi-cluster-demo
kubectl --context=$SECONDARY get deployment hello -n multi-cluster-demo
```

**Expected result:**

```text
Primary has 3 desired replicas. Secondary remains at 2 desired replicas.
```

## Task 7: Simulate Failover

```bash
kubectl --context=$PRIMARY scale deployment hello -n multi-cluster-demo --replicas=0
kubectl --context=$PRIMARY get endpoints hello -n multi-cluster-demo
```

Primary endpoints should be empty.

```bash
curl http://localhost:8081
kubectl --context=$SECONDARY scale deployment hello -n multi-cluster-demo --replicas=3
kubectl --context=$SECONDARY rollout status deployment/hello -n multi-cluster-demo
```

Restore primary:

```bash
kubectl --context=$PRIMARY scale deployment hello -n multi-cluster-demo --replicas=2
kubectl --context=$PRIMARY rollout status deployment/hello -n multi-cluster-demo
```

## Task 8: Document a GitOps Layout

```bash
mkdir -p fleet-demo/apps/base fleet-demo/apps/primary fleet-demo/apps/secondary
mkdir -p fleet-demo/clusters/primary fleet-demo/clusters/secondary
```

Create `fleet-demo/README.md`:

```markdown
# Fleet Demo

This directory shows one way to organize multi-cluster GitOps configuration.

- `apps/base`: common application manifests shared by every cluster
- `apps/primary`: primary cluster patches and overrides
- `apps/secondary`: secondary cluster patches and overrides
- `clusters/primary`: primary cluster sync entrypoint
- `clusters/secondary`: secondary cluster sync entrypoint
```

Verify:

```bash
find fleet-demo -maxdepth 3 -type d | sort
```

## Verification

```bash
kind get clusters
kubectl --context=kind-c2k-primary get deploy,svc,endpoints -n multi-cluster-demo
kubectl --context=kind-c2k-secondary get deploy,svc,endpoints -n multi-cluster-demo
find fleet-demo -maxdepth 3 -type d | sort
```

## Cleanup

```bash
kind delete cluster --name c2k-primary
kind delete cluster --name c2k-secondary
kubectl config use-context "$ORIGINAL_CONTEXT"
rm -rf fleet-demo
```
