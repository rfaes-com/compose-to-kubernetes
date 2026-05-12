# Part 2: Advanced Kubernetes Topics

Duration: 4 hours

## Prerequisites

Before starting Part 2, you should have completed Part 1 and be comfortable with:

- Creating and managing Pods, Deployments, and Services
- Working with ConfigMaps and Secrets
- Understanding Kubernetes storage (PVs, PVCs, StatefulSets)
- Using kubectl and k9s effectively
- Writing and organizing Kubernetes manifests

## Overview

Part 2 builds on the fundamentals from Part 1 and introduces advanced topics needed for production Kubernetes deployments.

## Sections

### 01. Ingress Controllers (45 minutes)

Learn how to expose HTTP/HTTPS services externally with path-based and host-based routing.

**Topics:**

- What is Ingress and why use it
- Installing NGINX Ingress Controller
- Path-based and host-based routing
- TLS/SSL certificate management
- Ingress vs Services

**Lab:** Deploy multi-service application with single Ingress

### 02. Helm - Package Management (50 minutes)

Package, template, and manage Kubernetes applications with Helm charts.

**Topics:**

- Helm architecture and concepts
- Installing and using Helm charts
- Creating custom charts
- Templating with values
- Chart repositories
- Helm vs raw manifests

**Lab:** Create and deploy a custom Helm chart

### 03. GitOps with Flux (50 minutes)

Implement continuous deployment using GitOps methodology.

**Topics:**

- GitOps principles and workflow
- Flux architecture (source, kustomize, helm controllers)
- Repository structure for GitOps
- Automated deployments from Git
- Handling secrets in GitOps
- Notifications and alerts

**Lab:** Set up automated deployment pipeline with Flux

### 04. Monitoring and Logging (45 minutes)

Observe and troubleshoot applications with Prometheus and Loki.

**Topics:**

- Prometheus for metrics
- Grafana for visualization
- ServiceMonitors and alerts
- Loki for log aggregation
- Distributed tracing basics
- Best practices for observability

**Lab:** Deploy monitoring stack and create custom dashboards

### 05. Advanced Deployment Strategies (40 minutes)

Implement sophisticated deployment patterns for zero-downtime updates.

**Topics:**

- Rolling updates (advanced)
- Blue/Green deployments
- Canary deployments
- A/B testing
- Progressive delivery
- Rollback strategies

**Lab:** Implement canary deployment with traffic splitting

### 06. Autoscaling (45 minutes)

Automatically scale applications and clusters based on demand.

**Topics:**

- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)
- Cluster Autoscaler
- KEDA (event-driven autoscaling)
- Custom metrics for scaling
- Scaling strategies and limits

**Lab:** Configure HPA with custom metrics

### 07. Security and RBAC (50 minutes)

Secure your cluster with role-based access control and security policies.

**Topics:**

- RBAC (Roles, RoleBindings, ClusterRoles)
- ServiceAccounts
- Pod Security Standards
- Network Policies (advanced)
- Secret management (external secrets)
- Security scanning and policies

**Lab:** Implement multi-tenant namespace isolation with RBAC

### 08. Multi-Cluster Management (45 minutes)

Manage applications across multiple Kubernetes clusters.

**Topics:**

- Why multi-cluster?
- Cluster federation concepts
- Cross-cluster service discovery
- Multi-cluster networking
- Disaster recovery strategies
- Tools: kubefed, Cilium Cluster Mesh

**Lab:** Deploy application across two clusters with failover

## Environment Requirements

The Part 2 labs require:

- Kind cluster from Part 1 (or fresh multi-node cluster)
- At least 8GB RAM available
- Internet access for pulling additional images
- The workshop container with all tools pre-installed

Some advanced labs (multi-cluster) benefit from:

- Two separate kind clusters
- Additional resources (12GB+ RAM recommended)

## Learning Outcomes

After completing Part 2, you will be able to:

- Expose applications externally with Ingress controllers
- Package and deploy applications using Helm
- Implement GitOps workflows with Flux
- Monitor applications with Prometheus and Grafana
- Implement advanced deployment strategies (canary, blue/green)
- Configure autoscaling based on metrics
- Secure clusters with RBAC and network policies
- Manage applications across multiple clusters

## Additional Resources

See the `resources/` directory for:

- Helm cheatsheet
- Flux CLI reference
- Prometheus query guide
- RBAC examples
- Production readiness checklist

## Next Steps

Start with [Section 01: Ingress Controllers](01-ingress/README.md)
