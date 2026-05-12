# Compose to Kubernetes Workshop

A hands-on workshop for developers transitioning from Docker Compose to Kubernetes.
Designed for developers familiar with `docker-compose.yml`, this workshop provides a
practical path to understanding Kubernetes fundamentals and advanced operations.

## Workshop Overview

|                   |                                                 |
| ----------------- | ----------------------------------------------- |
| **Duration**      | 8 hours total (two 4-hour sessions)             |
| **Format**        | Instructor-led with hands-on labs               |
| **Prerequisites** | Docker Compose experience, basic terminal usage |

## Getting Started

1. Clone the repository and open it in VS Code
2. Click **Reopen in Container** when prompted
3. All tools are pre-installed and ready

## Workshop Content

- [Part 1: Kubernetes Fundamentals](part-1/README.md) — Pods, Deployments, Services, ConfigMaps, Secrets, Storage, Namespaces, and a final 3-tier application lab
- [Part 2: Advanced Operations](part-2/README.md) — Ingress, Helm, GitOps with Flux, Monitoring, Advanced Deployment Strategies, Autoscaling, Security, and Multi-Cluster

## What's Included

- Complete workshop materials with theory, examples, and hands-on labs
- Presentation slides for [Part 1](part-1/slides.md) and [Part 2](part-2/slides.md)
- Workshop environment with all tools pre-installed (kubectl, kind, k9s, Helm, Flux)
- Kind cluster configurations (simple, multi-node, HA)
- Lab exercises with step-by-step solutions
- Cheatsheets: kubectl, k9s, and compose-to-k8s mapping

## Learning Approach

Each section follows the same pattern:

- **Docker Compose → Kubernetes mapping** to anchor new concepts
- **Live demonstration** of the concept
- **Hands-on lab** to practice (~45% of total workshop time)
- **Real-world examples** using public container images
