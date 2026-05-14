# Lab: Multi-Cluster Management

**Duration:** 25 minutes

## Objectives

- Create and manage two kind clusters
- Switch safely between Kubernetes contexts
- Deploy the same application to multiple clusters
- Customize per-cluster configuration
- Simulate active/passive failover
- Compare resource state across clusters

## Prerequisites

- kind installed
- kubectl configured and working
- At least 12GB RAM recommended for two clusters
- Basic understanding of Kubernetes contexts

## Tasks

### Task 1: Create Two Clusters

Create two local kind clusters to represent separate environments.

**Requirements:**

- Primary cluster name: `c2k-primary`
- Secondary cluster name: `c2k-secondary`
- Confirm both clusters appear in `kind get clusters`
- Confirm both contexts appear in `kubectl config get-contexts`
- Store your original context name so you can switch back later

<details class="hint" markdown="1">
<summary>Hint</summary>

Record your current context:

```bash
ORIGINAL_CONTEXT=$(kubectl config current-context)
echo $ORIGINAL_CONTEXT
```

Create the clusters:

```bash
kind create cluster --name c2k-primary
kind create cluster --name c2k-secondary
```

List clusters and contexts:

```bash
kind get clusters
kubectl config get-contexts
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
ORIGINAL_CONTEXT=$(kubectl config current-context)
echo $ORIGINAL_CONTEXT

kind create cluster --name c2k-primary
kind create cluster --name c2k-secondary

kind get clusters
kubectl config get-contexts
```

**Expected output:**

```text
c2k-primary
c2k-secondary

CURRENT   NAME                 CLUSTER              AUTHINFO             NAMESPACE
          kind-c2k-primary     kind-c2k-primary     kind-c2k-primary
*         kind-c2k-secondary   kind-c2k-secondary   kind-c2k-secondary
```

</details>

### Task 2: Practice Context-Safe Commands

Inspect both clusters without relying on the current context.

**Requirements:**

- Get nodes from `kind-c2k-primary`
- Get nodes from `kind-c2k-secondary`
- Create aliases or shell variables for both contexts
- Confirm the current context before applying anything

<details class="hint" markdown="1">
<summary>Hint</summary>

Set context variables:

```bash
PRIMARY=kind-c2k-primary
SECONDARY=kind-c2k-secondary
```

Use them explicitly:

```bash
kubectl --context=$PRIMARY get nodes
kubectl --context=$SECONDARY get nodes
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
PRIMARY=kind-c2k-primary
SECONDARY=kind-c2k-secondary

kubectl --context=$PRIMARY get nodes
kubectl --context=$SECONDARY get nodes
kubectl config current-context
```

**Expected output:** Each cluster shows one node with `Ready` status. Using `--context` means you never accidentally apply to the wrong cluster.

</details>

### Task 3: Create a Shared Application Manifest

Create a small application that can run unchanged in both clusters.

**Requirements:**

- Namespace: `multi-cluster-demo`
- Deployment name: `hello`
- Image: `nginx:1.25-alpine`
- Label: `app=hello`
- Replicas: 2
- Service name: `hello`
- Service type: `ClusterIP`
- Service port: 80

<details class="hint" markdown="1">
<summary>Hint</summary>

Use nginx with a shell command that writes `index.html` from an environment variable before starting nginx. This allows patching the response per cluster without changing the base manifest:

```yaml
env:
- name: CLUSTER_NAME
  value: primary
command: ["/bin/sh", "-c"]
args:
- |
  echo "hello from ${CLUSTER_NAME}" > /usr/share/nginx/html/index.html
  nginx -g 'daemon off;'
```

For the shared manifest, start with `primary`; patch it for secondary later.

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

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

</details>

### Task 4: Deploy to the Primary Cluster

Deploy the shared application to the primary cluster.

**Requirements:**

- Apply the namespace, Deployment, and Service to `kind-c2k-primary`
- Wait for the Deployment to become available
- Port-forward the primary Service to local port 8080
- Confirm the response includes `primary`

<details class="hint" markdown="1">
<summary>Hint</summary>

Apply with an explicit context:

```bash
kubectl --context=$PRIMARY apply -f hello-namespace.yaml
kubectl --context=$PRIMARY apply -f hello-deployment.yaml
kubectl --context=$PRIMARY apply -f hello-service.yaml
```

Wait and test:

```bash
kubectl --context=$PRIMARY wait --for=condition=Available deployment/hello \
  -n multi-cluster-demo --timeout=90s
kubectl --context=$PRIMARY port-forward -n multi-cluster-demo svc/hello 8080:80
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

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

</details>

### Task 5: Deploy to the Secondary Cluster

Deploy the application to the secondary cluster with a different response.

**Requirements:**

- Apply the same namespace, Deployment, and Service to `kind-c2k-secondary`
- Patch or configure the secondary Deployment response to include `secondary`
- Wait for the Deployment to become available
- Port-forward the secondary Service to local port 8081
- Confirm the response includes `secondary`

<details class="hint" markdown="1">
<summary>Hint</summary>

Apply the same files to secondary, then patch the environment variable:

```bash
kubectl --context=$SECONDARY set env deployment/hello \
  -n multi-cluster-demo CLUSTER_NAME=secondary
```

Restart the Deployment so nginx rewrites the page:

```bash
kubectl --context=$SECONDARY rollout restart deployment/hello -n multi-cluster-demo
kubectl --context=$SECONDARY rollout status deployment/hello -n multi-cluster-demo
```

Port-forward to a different local port:

```bash
kubectl --context=$SECONDARY port-forward -n multi-cluster-demo svc/hello 8081:80
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

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

</details>

### Task 6: Compare Cluster State

Compare the same workload across both clusters.

**Requirements:**

- List Pods in `multi-cluster-demo` on both clusters
- List Services in `multi-cluster-demo` on both clusters
- Compare Deployment replica counts
- Scale the primary Deployment to 3 replicas
- Confirm the secondary cluster remains at 2 replicas

<details class="hint" markdown="1">
<summary>Hint</summary>

Run the same commands against both contexts:

```bash
kubectl --context=$PRIMARY get pods,svc,deploy -n multi-cluster-demo
kubectl --context=$SECONDARY get pods,svc,deploy -n multi-cluster-demo
```

Scale only primary:

```bash
kubectl --context=$PRIMARY scale deployment hello -n multi-cluster-demo --replicas=3
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl --context=$PRIMARY get pods,svc,deploy -n multi-cluster-demo
kubectl --context=$SECONDARY get pods,svc,deploy -n multi-cluster-demo

kubectl --context=$PRIMARY scale deployment hello -n multi-cluster-demo --replicas=3
kubectl --context=$PRIMARY get deployment hello -n multi-cluster-demo
kubectl --context=$SECONDARY get deployment hello -n multi-cluster-demo
```

**Expected result:** Primary has 3 desired replicas. Secondary remains at 2 desired replicas. Each cluster is fully independent.

</details>

### Task 7: Simulate Failover

Simulate moving traffic from primary to secondary.

**Requirements:**

- Treat local port 8080 as primary traffic
- Treat local port 8081 as secondary traffic
- Scale primary `hello` to 0 replicas to simulate failure
- Confirm primary Service has no available endpoints
- Confirm secondary still serves traffic
- Scale secondary to 3 replicas to handle failover
- Restore primary to 2 replicas

<details class="hint" markdown="1">
<summary>Hint</summary>

Scale primary down:

```bash
kubectl --context=$PRIMARY scale deployment hello -n multi-cluster-demo --replicas=0
kubectl --context=$PRIMARY get endpoints hello -n multi-cluster-demo
```

Scale secondary up:

```bash
kubectl --context=$SECONDARY scale deployment hello -n multi-cluster-demo --replicas=3
```

Restore primary:

```bash
kubectl --context=$PRIMARY scale deployment hello -n multi-cluster-demo --replicas=2
```

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

```bash
kubectl --context=$PRIMARY scale deployment hello -n multi-cluster-demo --replicas=0
kubectl --context=$PRIMARY get endpoints hello -n multi-cluster-demo
```

Primary endpoints should be empty (no active Pods).

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

</details>

### Task 8: Document a GitOps Layout

Create a local directory structure that could manage both clusters from Git.

**Requirements:**

- Directory: `fleet-demo`
- Base app path: `fleet-demo/apps/base`
- Primary overlay path: `fleet-demo/apps/primary`
- Secondary overlay path: `fleet-demo/apps/secondary`
- Cluster config path: `fleet-demo/clusters/primary`
- Cluster config path: `fleet-demo/clusters/secondary`
- Add a short `README.md` explaining which path belongs to which cluster

<details class="hint" markdown="1">
<summary>Hint</summary>

Create directories:

```bash
mkdir -p fleet-demo/apps/base fleet-demo/apps/primary fleet-demo/apps/secondary
mkdir -p fleet-demo/clusters/primary fleet-demo/clusters/secondary
```

The README can describe:

- `apps/base`: common manifests shared by every cluster
- `apps/primary`: primary-specific patches
- `apps/secondary`: secondary-specific patches
- `clusters/primary`: primary cluster sync entrypoint (e.g., Flux Kustomization)
- `clusters/secondary`: secondary cluster sync entrypoint

</details>

<details class="solution" markdown="1">
<summary>Solution</summary>

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
- `clusters/primary`: primary cluster sync entrypoint (Flux Kustomization)
- `clusters/secondary`: secondary cluster sync entrypoint (Flux Kustomization)
```

Verify:

```bash
find fleet-demo -maxdepth 3 -type d | sort
```

</details>

## Verification

Check your work:

```bash
kind get clusters
kubectl --context=kind-c2k-primary get deploy,svc,endpoints -n multi-cluster-demo
kubectl --context=kind-c2k-secondary get deploy,svc,endpoints -n multi-cluster-demo
find fleet-demo -maxdepth 3 -type d | sort
```

Expected outcomes:

- Both clusters exist and are reachable independently
- Primary and secondary deployments return different responses
- Scaling one cluster does not affect the other
- `fleet-demo` directory structure exists locally

## Cleanup

```bash
kind delete cluster --name c2k-primary
kind delete cluster --name c2k-secondary
kubectl config use-context "$ORIGINAL_CONTEXT"
rm -rf fleet-demo
```

## Key takeaways

1. **`--context` flag** is the safest way to target a specific cluster without changing the active context globally
2. **Kustomize overlays** customize base manifests per cluster without duplicating YAML
3. **GitOps controllers** reconcile cluster state from Git continuously, making multi-cluster management auditable and repeatable
4. **Context discipline** prevents accidental changes to the wrong cluster — always verify with `kubectl config current-context`
5. **Active/passive failover** requires independent clusters with synchronized configuration and a traffic-switching mechanism
6. **Multiple kubeconfigs** can be merged or referenced via `KUBECONFIG` to manage many clusters from a single terminal

## Next section

Congratulations on completing Part 2! Explore the [Resources](../../resources/production-readiness-checklist.md) section for production readiness guidance.
