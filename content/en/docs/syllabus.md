---
title: Syllabus
linkTitle: Syllabus
weight: 10
---

# CKAD 12-Week Study Syllabus

> Based on the **CKAD Curriculum v1.34** (Kubernetes 1.34).
> Exam format: 2 hours, hands-on, command-line only, open-book (kubernetes.io docs allowed).

## How This Syllabus Works

Each week targets **12 hours** of study, split as follows:

| Activity | Time | What it means |
|----------|------|---------------|
| Theory / Concept | ~5 hours (40%) | Reading docs, watching videos, studying YAML structures |
| Hands-On Labs | ~7 hours (60%) | Typing real commands in a cluster you control |

> **Tip:** Short daily sessions (1.5–2 hours) beat one marathon weekend session.
> Set up your local cluster first — see [Setup](/docs/setup/) in this site.

---

## CKAD Exam Domains at a Glance

Before diving in, here are the five domains the exam tests and how much each is worth:

| # | Domain | Weight | Covered in |
|---|--------|--------|------------|
| 1 | Application Design and Build | 20% | Weeks 2–5 |
| 2 | Application Deployment | 20% | Weeks 5–7 |
| 3 | Application Observability and Maintenance | 15% | Weeks 8–9 |
| 4 | Application Environment, Configuration and Security | 25% | Weeks 4, 8–10 |
| 5 | Services and Networking | 20% | Weeks 6–7 |

---

## Phase 1 — Foundations (Weeks 1–3)

The goal of this phase is to become comfortable with containers and the Kubernetes
architecture before you touch any exam-level topics.

---

### Week 1: Containers & Docker Fundamentals

**What you will learn:** How applications are packaged into *containers* (lightweight,
isolated environments that share the host operating system kernel) and why Kubernetes
needs them.

#### Theory (~5 h)

- What is a container and how it differs from a virtual machine (VM)
- **OCI images** — the standard format for container images (think of an image as a
  snapshot of your application and all its dependencies)
- Writing a `Dockerfile`: `FROM`, `COPY`, `RUN`, `EXPOSE`, `ENTRYPOINT`
- Image layers and caching — why order in a Dockerfile matters
- Container registries: Docker Hub, GitHub Container Registry (GHCR)

#### Hands-On Labs (~7 h)

- [ ] Install Docker (or Podman) and run `docker run hello-world`
- [ ] Write a Dockerfile for a simple Go HTTP server (see [Setup](/docs/setup/))
- [ ] Build, tag, and push the image to a registry
- [ ] Run the container locally, map a port, and curl the endpoint
- [ ] Explore `docker exec`, `docker logs`, `docker inspect`
- [ ] Multi-stage builds: shrink your Go image from ~800 MB to ~15 MB

#### Practical Challenge — Week 1

> **Challenge:** Build a multi-stage Docker image for a Go application that responds
> with `{"status":"ok"}` on port 8080. The final image must be under 20 MB. Push it to
> Docker Hub or GHCR.

#### Speed Run Checklist — Week 1

These aliases and imperative commands will save you hundreds of keystrokes over the
next 12 weeks and during the exam. **Set them up now.**

**Shell aliases** — add to your `~/.bashrc` or `~/.zshrc`:

```bash
# --- CKAD Speed Aliases ---
alias k='kubectl'
alias kn='kubectl config set-context --current --namespace'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kga='kubectl get all'
alias kdp='kubectl describe pod'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
alias kex='kubectl explain'
alias klo='kubectl logs'
alias kpf='kubectl port-forward'

# Dry-run shortcut: generates YAML without creating anything
alias kdry='kubectl run --dry-run=client -o yaml'

# Quick edit
alias ke='kubectl edit'
```

**Imperative commands** — faster than writing YAML by hand:

```bash
# Create a Pod named "nginx" running the nginx image
k run nginx --image=nginx

# Create a Pod and immediately expose it as a Service
k run nginx --image=nginx --port=80 --expose

# Generate YAML for a Pod without creating it (great for exams)
k run mypod --image=busybox --dry-run=client -o yaml > pod.yaml

# Create a Deployment
k create deployment myapp --image=nginx --replicas=3

# Expose a Deployment as a ClusterIP Service
k expose deployment myapp --port=80 --target-port=80

# Create a ConfigMap from a literal value
k create configmap myconfig --from-literal=key1=value1

# Create a Secret
k create secret generic mysecret --from-literal=password=s3cret

# Scale a Deployment
k scale deployment myapp --replicas=5

# Delete a Pod immediately (useful in exams to save time)
k delete pod nginx --grace-period=0 --force
```

**Kubectl productivity tips:**

```bash
# Enable shell autocompletion (TAB to complete resource names)
source <(kubectl completion bash)   # or zsh
complete -o default -F __start_kubectl k

# Set a default namespace so you don't type -n every time
k config set-context --current --namespace=myns

# Quickly check which context and namespace you are in
k config current-context
k config view --minify | grep namespace
```

---

*For the full syllabus content (Weeks 2–12), see the [Syllabus](/docs/syllabus/) page or the individual [Week](/docs/weeks/) pages. The breakdown above is the outline; each week has a dedicated page under [Weeks](/docs/weeks/) with full concepts, tutorials, labs, and solution keys.*

---

## Quick Reference: Weekly Hour Breakdown

| Week | Phase | Focus | Theory | Hands-On | Total |
|------|-------|-------|--------|----------|-------|
| 1 | Foundations | Containers & Docker | 5 h | 7 h | 12 h |
| 2 | Foundations | K8s Architecture | 5 h | 7 h | 12 h |
| 3 | Foundations | Pods & Multi-Container | 5 h | 7 h | 12 h |
| 4 | Core Objects | ConfigMaps, Secrets, Security | 5 h | 7 h | 12 h |
| 5 | Core Objects | Deployments, Jobs, CronJobs | 5 h | 7 h | 12 h |
| 6 | Core Objects | Services & Networking (1) | 5 h | 7 h | 12 h |
| 7 | Core Objects | Ingress & NetworkPolicies | 5 h | 7 h | 12 h |
| 8 | Advanced | Storage & Volumes | 5 h | 7 h | 12 h |
| 9 | Advanced | Observability & Debugging | 5 h | 7 h | 12 h |
| 10 | Advanced | Helm, CRDs & Security | 5 h | 7 h | 12 h |
| 11 | Exam Prep | Timed Practice | 3 h | 9 h | 12 h |
| 12 | Exam Prep | Final Mock Exams | 2 h | 10 h | 12 h |
| | | **Total** | **55 h** | **89 h** | **144 h** |

---

## Recommended Resources

| Resource | Type | Cost |
|----------|------|------|
| [Kubernetes Official Docs](https://kubernetes.io/docs/) | Documentation | Free |
| [killer.sh CKAD Simulator](https://killer.sh/) | Mock Exam | Included with exam purchase |
| [CKAD Exercises (GitHub - dgkanatsios)](https://github.com/dgkanatsios/CKAD-exercises) | Practice | Free |
| [Kubernetes the Hard Way (Kelsey Hightower)](https://github.com/kelseyhightower/kubernetes-the-hard-way) | Deep Dive | Free |
| [CKAD Study Guide (O'Reilly)](https://www.oreilly.com/library/view/certified-kubernetes-application/9781098152857/) | Book | Paid |

---

*Good luck. The exam rewards practice, not memorization. Type more, read less.*
