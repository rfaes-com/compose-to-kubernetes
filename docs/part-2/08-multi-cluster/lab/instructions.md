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

### Task 2: Practice Context-Safe Commands

Inspect both clusters without relying on the current context.

**Requirements:**

- Get nodes from `kind-c2k-primary`
- Get nodes from `kind-c2k-secondary`
- Create aliases or shell variables for both contexts
- Confirm the current context before applying anything

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

### Task 4: Deploy to the Primary Cluster

Deploy the shared application to the primary cluster.

**Requirements:**

- Apply the namespace, Deployment, and Service to `kind-c2k-primary`
- Wait for the Deployment to become available
- Port-forward the primary Service to local port 8080
- Confirm the response includes `primary`

### Task 5: Deploy to the Secondary Cluster

Deploy the application to the secondary cluster with a different response.

**Requirements:**

- Apply the same namespace, Deployment, and Service to `kind-c2k-secondary`
- Patch or configure the secondary Deployment response to include `secondary`
- Wait for the Deployment to become available
- Port-forward the secondary Service to local port 8081
- Confirm the response includes `secondary`

### Task 6: Compare Cluster State

Compare the same workload across both clusters.

**Requirements:**

- List Pods in `multi-cluster-demo` on both clusters
- List Services in `multi-cluster-demo` on both clusters
- Compare Deployment replica counts
- Scale the primary Deployment to 3 replicas
- Confirm the secondary cluster remains at 2 replicas

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

## Hints

<details>
<summary>Hint for Task 1: Creating kind clusters</summary>

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

<details>
<summary>Hint for Task 2: Using explicit contexts</summary>

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

<details>
<summary>Hint for Task 3: Making the response cluster-specific</summary>

Use nginx and write the response from an environment variable:

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

<details>
<summary>Hint for Task 4: Applying to primary</summary>

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

<details>
<summary>Hint for Task 5: Patching secondary</summary>

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

<details>
<summary>Hint for Task 6: Comparing clusters</summary>

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

<details>
<summary>Hint for Task 7: Simulating failover</summary>

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

<details>
<summary>Hint for Task 8: Fleet directory structure</summary>

Create directories:

```bash
mkdir -p fleet-demo/apps/base fleet-demo/apps/primary fleet-demo/apps/secondary
mkdir -p fleet-demo/clusters/primary fleet-demo/clusters/secondary
```

The README can describe:

- `apps/base`: common manifests
- `apps/primary`: primary-specific patches
- `apps/secondary`: secondary-specific patches
- `clusters/primary`: primary cluster sync entrypoint
- `clusters/secondary`: secondary cluster sync entrypoint

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

- Two kind clusters exist
- The same app runs in both clusters
- Primary and secondary responses are distinguishable
- Scaling one cluster does not change the other
- Secondary can serve traffic while primary has no endpoints
- A GitOps-style fleet directory exists

## Cleanup

```bash
kind delete cluster --name c2k-primary
kind delete cluster --name c2k-secondary
kubectl config use-context "$ORIGINAL_CONTEXT"
rm -rf fleet-demo
```

## Check Your Understanding

1. Why should multi-cluster commands use explicit contexts?
2. What changes between active/active and active/passive designs?
3. Why does a ClusterIP Service not automatically work across clusters?
4. How would GitOps reduce drift between clusters?
5. What would you add for production failover beyond this lab?

## Bonus Challenges

- Create Kustomize overlays for primary and secondary instead of patching live.
- Add a `cluster` label to the Deployment template in each cluster.
- Install Metrics Server in both clusters and compare resource usage.
- Sketch how DNS or a global load balancer would route users during failover.
