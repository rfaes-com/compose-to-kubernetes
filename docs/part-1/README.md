# Part 1: Kubernetes Fundamentals

**Duration:** 4 hours (including breaks)
**Prerequisites:** Docker Compose experience, workshop environment set up

## Learning Objectives

By the end of Part 1, you will be able to:

- Understand Kubernetes architecture and core concepts
- Map Docker Compose concepts to Kubernetes equivalents
- Create and manage Pods, Deployments, and Services
- Configure applications using ConfigMaps and Secrets
- Implement persistent storage with Volumes and PersistentVolumes
- Use kubectl and k9s effectively for cluster management
- Deploy a multi-tier application on Kubernetes

## Sections

1. **[Introduction to Kubernetes](01-intro/README.md)** (30 min)
   What is Kubernetes, architecture overview, K8s vs Docker Compose
2. **[Environment Setup](02-environment/README.md)** (25 min)
   Creating your first cluster, verifying tools, exploring with k9s
3. **[Pods](03-pods/README.md)** (35 min)
   The atomic unit of Kubernetes, Pod lifecycle, multi-container Pods
4. **[Deployments & ReplicaSets](04-deployments/README.md)** (40 min)
   Declarative updates, self-healing, scaling applications
5. **[Services](05-services/README.md)** (40 min)
   Service discovery, ClusterIP, NodePort, LoadBalancer types
6. **[Configuration Management](06-config/README.md)** (35 min)
   ConfigMaps for config, Secrets for sensitive data
7. **[Storage](07-storage/README.md)** (40 min)
   Volumes, PersistentVolumes, PersistentVolumeClaims, StatefulSets overview
8. **[Namespaces](08-namespaces/README.md)** (20 min)
   Logical isolation, resource organization, quotas
9. **[kubectl & k9s](09-tools/README.md)** (30 min)
   Essential commands, troubleshooting, k9s navigation
10. **[Manifests & Best Practices](10-manifests/README.md)** (20 min)
    YAML organization, labels, annotations, standards
11. **[Final Lab](11-final-lab/README.md)** (50 min)
    Deploy a complete 3-tier application from Docker Compose to Kubernetes

## Schedule

### Suggested Timeline (4 hours with breaks)

**9:00 - 10:15** (75 min)

- Section 1: Introduction (30 min)
- Section 2: Environment Setup (25 min)
- Section 3: Pods (20 min demo + start lab)

**10:15 - 10:30** Break

**10:30 - 11:40** (70 min)

- Section 3: Pods (finish lab)
- Section 4: Deployments (40 min)
- Section 5: Services (start)

**11:40 - 11:55** Break

**11:55 - 1:00** (65 min)

- Section 5: Services (finish)
- Section 6: ConfigMaps & Secrets (35 min)
- Section 7: Storage (40 min)
- Section 8: Namespaces (20 min)
- Section 9: kubectl (20 min)
- Section 10: Manifests (10 min)
- Section 11: Final Lab (50 min - may extend past 1:00pm)

## Getting Started

### Create Your First Cluster

In the visual studio code terminal:

```bash
# Create a simple cluster (1 control-plane, 1 worker)
kind create cluster --config /workspaces/compose-to-kubernetes/setup/kind/simple.yaml

# Verify it works
kubectl get nodes
kubectl cluster-info

# Launch k9s to explore
k9s
```

You're ready to start! Head to [01-intro](01-intro/README.md) to begin.

## Learning Approach

Each section follows this structure:

```text
XX-topic/
├── README.md           # Concepts and theory
├── examples/           # Demo manifests
│   ├── example.yaml
│   └── compose.yaml    # Docker Compose comparison
└── lab/                # Hands-on exercise
    ├── instructions.md
    └── solutions/      # Don't peek too early!
```

### How to Use This Material

1. **Read the README.md** in each section
2. **Follow along with examples** - Try them in your cluster!
3. **Complete the lab** - Hands-on practice
4. **Compare with solutions** - After attempting the lab
5. **Ask questions** - Instructor or GitHub issues

## Workshop Format

- **Instructor-led presentations** with live demonstrations
- **Hands-on labs** after each major concept
- **Docker Compose → Kubernetes comparisons** throughout
- **Progressive complexity** - each section builds on previous ones
- **Real-world examples** using public container images

## Tips for Success

1. **Don't skip the labs** - Hands-on practice is how you learn
2. **Experiment!** - Your cluster is disposable. Break things and learn.
3. **Use k9s** - It provides great visual feedback while learning
4. **Save your work** - Files in `/workspace` persist after container exits
5. **Ask questions** - There are no stupid questions in learning!

## Resources

### Quick References

- [kubectl Cheatsheet](../resources/kubectl-cheatsheet.md)
- [k9s Shortcuts](../resources/k9s-shortcuts.md)
- [Compose → K8s Mapping](../resources/compose-to-k8s-mapping.md)

### Official Documentation

- [Kubernetes Docs](https://kubernetes.io/docs/)
- [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
- [API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)

### Tools

- [kind Documentation](https://kind.sigs.k8s.io/)
- [k9s Documentation](https://k9scli.io/)

## Ready?

Start with [Section 1: Introduction to Kubernetes](01-intro/README.md)

Good luck and have fun learning Kubernetes!
