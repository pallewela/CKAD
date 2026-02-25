# CKAD 12-Week Study Plan

A structured, hands-on path to the **Certified Kubernetes Application Developer (CKAD)** exam, aligned to the CKAD Curriculum v1.34. Each week includes concepts, tutorials, labs, speed drills, pitfalls, and solution keys.

---

## Start Here

| Document | Description |
|----------|-------------|
| **[Syllabus.md](Syllabus.md)** | Master 12-week plan: phases, weekly topics, 40% theory / 60% hands-on, practical challenges, and speed-run checklist. |
| **[Setup.md](Setup.md)** | Local cluster setup (kind or minikube), kubectl install, and building/running a Go app in the cluster. |

Do **Setup.md** first so you have a working cluster, then follow the syllabus week by week.

---

## Weekly Study Material

Each week has a folder under `weeks/` with a **README.md** containing:

- **The Concept** — Jargon-free explanation of that week’s Kubernetes objects
- **Beginner Tutorial** — Step-by-step walkthrough with commands
- **Hands-On Lab** — 3–5 performance-based challenges
- **Weekly Speed Drill** — 15-minute imperative-command practice
- **Exam Pitfalls** — Common mistakes and how to avoid them
- **Solution Key** — Exact `kubectl` commands and YAML for all labs

### Phase 1 — Foundations (Weeks 1–3)

| Week | Focus | Link |
|------|--------|------|
| 1 | Containers & Docker (images, Dockerfile, multi-stage builds) | [weeks/01-containers-and-docker/README.md](weeks/01-containers-and-docker/README.md) |
| 2 | Kubernetes architecture, cluster components, Pods, namespaces | [weeks/02-kubernetes-architecture/README.md](weeks/02-kubernetes-architecture/README.md) |
| 3 | Pods in depth, init containers, sidecar/ambassador/adapter patterns | [weeks/03-pods-and-multi-container/README.md](weeks/03-pods-and-multi-container/README.md) |

### Phase 2 — Core Objects (Weeks 4–7)

| Week | Focus | Link |
|------|--------|------|
| 4 | ConfigMaps, Secrets, SecurityContext, ServiceAccounts, quotas | [weeks/04-configmaps-secrets-security/README.md](weeks/04-configmaps-secrets-security/README.md) |
| 5 | Deployments, rolling updates, rollback, Jobs, CronJobs | [weeks/05-deployments-jobs-cronjobs/README.md](weeks/05-deployments-jobs-cronjobs/README.md) |
| 6 | Services (ClusterIP, NodePort), DNS, Endpoints, port-forward | [weeks/06-services-and-networking/README.md](weeks/06-services-and-networking/README.md) |
| 7 | Ingress, Ingress Controller, NetworkPolicies | [weeks/07-ingress-and-network-policies/README.md](weeks/07-ingress-and-network-policies/README.md) |

### Phase 3 — Advanced & Security (Weeks 8–10)

| Week | Focus | Link |
|------|--------|------|
| 8 | Volumes, PV/PVC, StorageClass, StatefulSet | [weeks/08-persistent-storage/README.md](weeks/08-persistent-storage/README.md) |
| 9 | Observability: probes, logs, debugging CrashLoopBackOff | [weeks/09-observability-and-debugging/README.md](weeks/09-observability-and-debugging/README.md) |
| 10 | Helm, Kustomize, CRDs, RBAC, SecurityContext | [weeks/10-helm-crds-security/README.md](weeks/10-helm-crds-security/README.md) |

### Phase 4 — Exam Readiness (Weeks 11–12)

| Week | Focus | Link |
|------|--------|------|
| 11 | Timed practice, weak-spot drills, imperative workflow | [weeks/11-timed-practice/README.md](weeks/11-timed-practice/README.md) |
| 12 | Final mock exams, exam-day prep, full question simulations | [weeks/12-final-mock-exams/README.md](weeks/12-final-mock-exams/README.md) |

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

## Quick Links

- [Syllabus](Syllabus.md) — Full 12-week plan and hour breakdown
- [Setup](Setup.md) — kind/minikube, kubectl, Go app in cluster
- [Week 1](weeks/01-containers-and-docker/README.md) — Start here after setup

Good luck. The exam rewards practice; type more, read less.
