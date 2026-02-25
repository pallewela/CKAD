---
title: Home
layout: default
permalink: /
---

# CKAD 12-Week Study Plan

A structured, hands-on path to the **Certified Kubernetes Application Developer (CKAD)** exam, aligned to the CKAD Curriculum v1.34. Each week includes concepts, tutorials, labs, speed drills, pitfalls, and solution keys.

---

## Start Here

| Document | Description |
|----------|-------------|
| [**Syllabus**](syllabus/) | Master 12-week plan: phases, weekly topics, 40% theory / 60% hands-on, practical challenges, and speed-run checklist. |
| [**Setup**](setup/) | Local cluster setup (kind or minikube), kubectl install, and building/running a Go app in the cluster. |

Do **Setup** first so you have a working cluster, then follow the syllabus week by week.

---

## Weekly Study Material

Each week has a page under **Weeks** (see the sidebar) with:

- **The Concept** — Jargon-free explanation of that week's Kubernetes objects
- **Beginner Tutorial** — Step-by-step walkthrough with commands
- **Hands-On Lab** — 3–5 performance-based challenges
- **Weekly Speed Drill** — 15-minute imperative-command practice
- **Exam Pitfalls** — Common mistakes and how to avoid them
- **Solution Key** — Exact `kubectl` commands and YAML for all labs

### Phase 1 — Foundations (Weeks 1–3)

| Week | Focus |
|------|--------|
| [Week 1](weeks/01-containers-and-docker/) | Containers & Docker (images, Dockerfile, multi-stage builds) |
| [Week 2](weeks/02-kubernetes-architecture/) | Kubernetes architecture, cluster components, Pods, namespaces |
| [Week 3](weeks/03-pods-and-multi-container/) | Pods in depth, init containers, sidecar/ambassador/adapter patterns |

### Phase 2 — Core Objects (Weeks 4–7)

| Week | Focus |
|------|--------|
| [Week 4](weeks/04-configmaps-secrets-security/) | ConfigMaps, Secrets, SecurityContext, ServiceAccounts, quotas |
| [Week 5](weeks/05-deployments-jobs-cronjobs/) | Deployments, rolling updates, rollback, Jobs, CronJobs |
| [Week 6](weeks/06-services-and-networking/) | Services (ClusterIP, NodePort), DNS, Endpoints, port-forward |
| [Week 7](weeks/07-ingress-and-network-policies/) | Ingress, Ingress Controller, NetworkPolicies |

### Phase 3 — Advanced & Security (Weeks 8–10)

| Week | Focus |
|------|--------|
| [Week 8](weeks/08-persistent-storage/) | Volumes, PV/PVC, StorageClass, StatefulSet |
| [Week 9](weeks/09-observability-and-debugging/) | Observability: probes, logs, debugging CrashLoopBackOff |
| [Week 10](weeks/10-helm-crds-security/) | Helm, Kustomize, CRDs, RBAC, SecurityContext |

### Phase 4 — Exam Readiness (Weeks 11–12)

| Week | Focus |
|------|--------|
| [Week 11](weeks/11-timed-practice/) | Timed practice, weak-spot drills, imperative workflow |
| [Week 12](weeks/12-final-mock-exams/) | Final mock exams, exam-day prep, full question simulations |

---

## Exam Domains (CKAD v1.34)

| Domain | Weight | Weeks |
|--------|--------|-------|
| Application Design and Build | 20% | 2–5 |
| Application Deployment | 20% | 5–7 |
| Application Observability and Maintenance | 15% | 8–9 |
| Application Environment, Configuration and Security | 25% | 4, 8–10 |
| Services and Networking | 20% | 6–7 |

---

**Good luck.** The exam rewards practice; type more, read less.
