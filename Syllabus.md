---
title: Syllabus
permalink: /syllabus/
nav_order: 2
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
> Set up your local cluster first — see `Setup.md` in this repo.

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
- [ ] Write a Dockerfile for a simple Go HTTP server (see `Setup.md`)
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

### Week 2: Kubernetes Architecture & Core Concepts

**What you will learn:** The building blocks of a Kubernetes cluster — what each
component does, how they communicate, and the mental model you need before creating
any resource.

**Key terms:**
- **Cluster** — a set of machines (called *nodes*) that run your containerized apps.
- **Control Plane** — the "brain" of the cluster (API Server, Scheduler, etcd, Controller Manager).
- **Worker Node** — a machine that actually runs your containers via *kubelet* and a container runtime.
- **Pod** — the smallest deployable unit in Kubernetes; wraps one or more containers.
- **Namespace** — a virtual partition inside a cluster used to organize resources.

#### Theory (~5 h)

- Control Plane deep dive: API Server, etcd, Scheduler, Controller Manager
- Worker Node components: kubelet, kube-proxy, container runtime
- The Kubernetes API & declarative model ("desired state vs. actual state")
- YAML anatomy: `apiVersion`, `kind`, `metadata`, `spec`
- Namespaces and why they matter

#### Hands-On Labs (~7 h)

- [ ] Set up a local cluster with `kind` or `minikube` (see `Setup.md`)
- [ ] Run `kubectl cluster-info` and `kubectl get nodes`
- [ ] Create your first Pod from a YAML file
- [ ] Explore `kubectl get`, `describe`, `logs`, `exec`
- [ ] Create and switch between Namespaces
- [ ] Use `kubectl explain pod.spec` to self-discover fields
- [ ] Practice `kubectl api-resources` to see all available object types

#### Practical Challenge — Week 2

> **Challenge:** Write a YAML manifest that creates a Pod named `inspector` running
> `busybox`. The Pod should run the command `sleep 3600` so it stays alive. Exec into
> it, install `wget`, and fetch the Kubernetes API server's `/version` endpoint from
> inside the Pod.

---

### Week 3: Pods In Depth & Multi-Container Patterns

**What you will learn:** Everything about Pods — lifecycle, restart policies, init
containers, and the three multi-container patterns you will see on the exam.

**Key terms:**
- **Init Container** — a container that runs *before* the main app container starts; used for setup tasks.
- **Sidecar** — an extra container that runs *alongside* the main container (e.g., a log shipper).
- **Ambassador** — a proxy container that simplifies connecting to external services.
- **Adapter** — a container that transforms or normalizes the output of the main container.

#### Theory (~5 h)

- Pod lifecycle: Pending → Running → Succeeded / Failed
- Restart policies: `Always`, `OnFailure`, `Never`
- Init containers: use cases and ordering guarantees
- Multi-container patterns: Sidecar, Ambassador, Adapter
- Resource requests and limits (CPU and memory) — introduction
- `kubectl explain` as your in-exam documentation tool

#### Hands-On Labs (~7 h)

- [ ] Create a Pod with an init container that writes a file to a shared volume
- [ ] Build a sidecar pattern: main container writes logs, sidecar tails them
- [ ] Experiment with restart policies — make a container crash and observe behaviour
- [ ] Set resource requests and limits, then exceed the memory limit to see OOMKill
- [ ] Use `kubectl top pods` (requires metrics-server) to observe resource usage
- [ ] Practice generating Pod YAML with `--dry-run=client -o yaml` and editing it

#### Practical Challenge — Week 3

> **Challenge:** Create a Pod with two containers:
> 1. An `nginx` container serving a custom `index.html`.
> 2. A sidecar `busybox` container that writes the current date into
>    `/usr/share/nginx/html/index.html` every 5 seconds via a shared `emptyDir` volume.
> Verify the page updates by curling the nginx container.

---

## Phase 2 — Core Objects (Weeks 4–7)

Now you start working with the resources that make up the bulk of the exam.

---

### Week 4: Configuration — ConfigMaps, Secrets & Environment Variables

**What you will learn:** How to separate configuration from code in Kubernetes using
ConfigMaps (non-sensitive data) and Secrets (sensitive data like passwords and tokens).

**Key terms:**
- **ConfigMap** — a Kubernetes object that stores non-confidential key-value pairs.
- **Secret** — like a ConfigMap, but intended for sensitive data; base64-encoded (not encrypted by default).
- **Environment Variable** — a variable passed to a container at startup.
- **Volume Mount** — exposing a ConfigMap or Secret as files inside a container.

#### Theory (~5 h)

- Creating ConfigMaps from literals, files, and directories
- Creating Secrets (generic, TLS, docker-registry)
- Injecting config via environment variables vs. volume mounts
- SecurityContext basics: `runAsUser`, `runAsNonRoot`, `readOnlyRootFilesystem`
- ServiceAccounts: what they are and how Pods use them for API access
- Resource requirements: `requests` vs. `limits`, LimitRanges, ResourceQuotas

#### Hands-On Labs (~7 h)

- [ ] Create a ConfigMap and mount it as a file in a Pod
- [ ] Create a Secret and expose it as environment variables
- [ ] Update a ConfigMap and observe how mounted files change (but env vars do not)
- [ ] Configure a Pod with a SecurityContext (`runAsUser: 1000`)
- [ ] Create a ServiceAccount and assign it to a Pod
- [ ] Set ResourceQuotas on a namespace and try to exceed them
- [ ] Practice imperative creation: `k create configmap` / `k create secret generic`

#### Practical Challenge — Week 4

> **Challenge:** Deploy a Pod running PostgreSQL. Inject the database name via a
> ConfigMap environment variable and the password via a Secret. Verify the database
> starts correctly by exec-ing in and running `psql`.

---

### Week 5: Workload Resources — Deployments, Jobs & CronJobs

**What you will learn:** How Kubernetes manages groups of Pods, keeps them running,
scales them, and runs batch work.

**Key terms:**
- **ReplicaSet** — ensures a specified number of identical Pods are running (usually managed by a Deployment).
- **Deployment** — the standard way to run stateless applications; manages ReplicaSets and enables rolling updates.
- **Rolling Update** — replacing Pods gradually so there is zero downtime.
- **Job** — runs a Pod to completion (for one-off tasks).
- **CronJob** — runs a Job on a schedule (like Unix cron).

#### Theory (~5 h)

- Deployments: create, scale, update, rollback
- Rolling update strategy: `maxSurge`, `maxUnavailable`
- Rollback with `kubectl rollout undo`
- Jobs: `completions`, `parallelism`, `backoffLimit`, `activeDeadlineSeconds`
- CronJobs: schedule syntax, `concurrencyPolicy`, `startingDeadlineSeconds`
- DaemonSets: one Pod per node (exam may reference them)

#### Hands-On Labs (~7 h)

- [ ] Create a Deployment with 3 replicas and observe the ReplicaSet
- [ ] Perform a rolling update by changing the image version
- [ ] Roll back to a previous version with `kubectl rollout undo`
- [ ] Use `kubectl rollout status` and `kubectl rollout history`
- [ ] Create a Job that computes pi to 2000 digits
- [ ] Create a CronJob that runs every minute and writes the date to a log
- [ ] Delete a Deployment and see what happens to its Pods and ReplicaSet

#### Practical Challenge — Week 5

> **Challenge:** Create a Deployment for your Go application (from Week 1). Perform a
> rolling update to a "v2" image. Then intentionally push a broken "v3" image and
> roll back to "v2" using `kubectl rollout undo`.

---

### Week 6: Services & Networking (Part 1)

**What you will learn:** How Pods communicate with each other and with the outside
world through Services.

**Key terms:**
- **Service** — a stable network endpoint that routes traffic to a set of Pods.
- **ClusterIP** — the default Service type; accessible only *inside* the cluster.
- **NodePort** — exposes the Service on a static port on each Node (accessible from outside).
- **LoadBalancer** — provisions an external load balancer (cloud environments).
- **Endpoint** — the list of Pod IPs that a Service routes to.
- **DNS** — Kubernetes runs an internal DNS so Pods can reach Services by name
  (e.g., `my-svc.my-namespace.svc.cluster.local`).

#### Theory (~5 h)

- Service types: ClusterIP, NodePort, LoadBalancer, ExternalName
- How selectors match Pods to Services
- Kubernetes DNS: how `<service>.<namespace>.svc.cluster.local` works
- Port mappings: `port`, `targetPort`, `nodePort`
- Headless Services (`clusterIP: None`)
- `kubectl port-forward` for local debugging

#### Hands-On Labs (~7 h)

- [ ] Create a Deployment + ClusterIP Service and curl it from another Pod
- [ ] Inspect Endpoints with `kubectl get endpoints`
- [ ] Create a NodePort Service and access it from your host machine
- [ ] Use DNS resolution: run `nslookup my-svc.default.svc.cluster.local` from a Pod
- [ ] Experiment with `port-forward` to access a Service from localhost
- [ ] Create a headless Service and observe DNS A records returning Pod IPs
- [ ] Practice imperative: `k expose deployment myapp --port=80 --type=NodePort`

#### Practical Challenge — Week 6

> **Challenge:** Deploy two applications — a frontend (nginx) and a backend (your Go
> app). The frontend must reach the backend via the Kubernetes DNS name. Verify
> connectivity by exec-ing into the frontend Pod and curling the backend Service.

---

### Week 7: Services & Networking (Part 2) — Ingress & Network Policies

**What you will learn:** How to expose HTTP applications to the outside world with
Ingress, and how to control which Pods can talk to each other using NetworkPolicies.

**Key terms:**
- **Ingress** — a Kubernetes object that manages external HTTP/HTTPS access to
  Services, typically via host/path rules.
- **Ingress Controller** — the actual software (e.g., NGINX Ingress Controller) that
  reads Ingress objects and configures routing. Kubernetes does not include one by default.
- **NetworkPolicy** — a firewall rule that controls traffic *into* (ingress) and *out of*
  (egress) Pods based on labels, namespaces, or IP blocks.

#### Theory (~5 h)

- Ingress resources: host-based and path-based routing
- Ingress Controller installation (NGINX)
- TLS termination with Ingress
- NetworkPolicy basics: `podSelector`, `ingress`, `egress`
- Default deny policies
- Combining namespace selectors and pod selectors

#### Hands-On Labs (~7 h)

- [ ] Install an Ingress Controller in your local cluster
- [ ] Create an Ingress that routes `/app1` to one Service and `/app2` to another
- [ ] Create a default-deny NetworkPolicy in a namespace
- [ ] Allow traffic only from specific Pods using label selectors
- [ ] Create an egress NetworkPolicy that blocks external traffic
- [ ] Test policies by running `curl` and `wget` from different Pods
- [ ] Practice writing NetworkPolicy YAML from scratch (common exam task)

#### Practical Challenge — Week 7

> **Challenge:** Create a namespace `secure-ns`. Deploy a database Pod and a web Pod.
> Write a NetworkPolicy that:
> 1. Denies all ingress traffic by default.
> 2. Allows the web Pod to reach the database on port 5432.
> 3. Blocks all other Pods from reaching the database.
> Verify by curling from an unauthorized Pod (should fail) and from the web Pod
> (should succeed).

---

## Phase 3 — Advanced & Security (Weeks 8–10)

These topics carry heavy exam weight (25% for Config & Security alone). Master them.

---

### Week 8: Persistent Storage & Volumes

**What you will learn:** How Kubernetes handles data that needs to survive Pod
restarts — from simple in-Pod volumes to dynamically provisioned cloud disks.

**Key terms:**
- **Volume** — storage attached to a Pod (lives as long as the Pod).
- **emptyDir** — a temporary volume created when a Pod starts, deleted when the Pod is removed.
- **PersistentVolume (PV)** — a piece of storage in the cluster provisioned by an admin or dynamically.
- **PersistentVolumeClaim (PVC)** — a request for storage by a user/Pod.
- **StorageClass** — defines *how* storage is provisioned (e.g., SSD vs. HDD, cloud provider).

#### Theory (~5 h)

- Volume types: `emptyDir`, `hostPath`, `configMap`, `secret`, `persistentVolumeClaim`
- PV / PVC lifecycle: Provisioning → Binding → Using → Reclaiming
- Access modes: `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`
- Reclaim policies: `Retain`, `Delete`
- StorageClasses and dynamic provisioning
- StatefulSets: when ordering and stable network identity matter

#### Hands-On Labs (~7 h)

- [ ] Create an `emptyDir` volume shared between two containers
- [ ] Create a PV and PVC manually, mount it in a Pod, write data, delete the Pod,
      and verify data persists in a new Pod
- [ ] Use a StorageClass for dynamic provisioning (kind/minikube has a default one)
- [ ] Resize a PVC (if the StorageClass allows it)
- [ ] Deploy a StatefulSet with volumeClaimTemplates
- [ ] Observe what happens when a PVC is deleted but the PV reclaim policy is `Retain`
- [ ] Practice writing PV/PVC YAML from memory

#### Practical Challenge — Week 8

> **Challenge:** Deploy a MySQL StatefulSet with a PVC. Insert some data. Delete the
> Pod (not the PVC). Let the StatefulSet recreate it. Verify the data survived.

---

### Week 9: Observability, Probes & Debugging

**What you will learn:** How to monitor application health, gather logs, and debug
failing Pods — skills tested directly on the exam.

**Key terms:**
- **Liveness Probe** — checks if a container is still running; if it fails, Kubernetes
  restarts the container.
- **Readiness Probe** — checks if a container is ready to serve traffic; if it fails,
  the Pod is removed from Service endpoints.
- **Startup Probe** — gives slow-starting containers time to initialize before liveness
  checks kick in.

#### Theory (~5 h)

- Probe types: HTTP GET, TCP Socket, Exec command, gRPC
- Probe parameters: `initialDelaySeconds`, `periodSeconds`, `failureThreshold`
- Container logging: `kubectl logs`, `kubectl logs --previous`
- Debugging: `kubectl describe`, events, `kubectl get events --sort-by=.metadata.creationTimestamp`
- `kubectl top` for CPU/memory metrics (requires metrics-server)
- API deprecations: how to check and migrate (`kubectl convert`, docs)

#### Hands-On Labs (~7 h)

- [ ] Add a liveness probe (HTTP GET) to your Go app Deployment
- [ ] Add a readiness probe and observe Endpoints being added/removed
- [ ] Create a startup probe for a slow-starting application
- [ ] Intentionally break a probe and watch Kubernetes restart the container
- [ ] Debug a CrashLoopBackOff Pod: check events, logs, describe
- [ ] Use `kubectl logs -f` to stream logs in real time
- [ ] Install metrics-server and run `kubectl top pods` / `kubectl top nodes`

#### Practical Challenge — Week 9

> **Challenge:** Deploy your Go application with all three probes configured:
> 1. A startup probe that checks `/healthz` every 2 seconds (10 failure threshold).
> 2. A liveness probe that checks `/healthz` every 5 seconds.
> 3. A readiness probe that checks `/ready` every 3 seconds.
> Break the readiness endpoint and verify the Pod is removed from the Service's
> Endpoints list but is not restarted.

---

### Week 10: Helm, CRDs & Security Deep Dive

**What you will learn:** Package management with Helm, extending Kubernetes with
Custom Resource Definitions, and the security topics the exam focuses on.

**Key terms:**
- **Helm** — a package manager for Kubernetes; packages are called *charts*.
- **Chart** — a collection of YAML templates and values that describe a set of Kubernetes resources.
- **Release** — a running instance of a chart in a cluster.
- **Custom Resource Definition (CRD)** — lets you create your own Kubernetes object types.
- **Operator** — a controller that watches CRDs and automates complex application management.
- **RBAC** — Role-Based Access Control; controls who can do what in the cluster.
- **Admission Controller** — a gatekeeper that intercepts API requests before they are persisted.

#### Theory (~5 h)

- Helm basics: `helm install`, `helm upgrade`, `helm rollback`, `helm uninstall`
- Helm values: overriding defaults with `--set` and `-f values.yaml`
- `helm template` and `helm show values`
- CRDs: defining, creating, and using custom resources
- RBAC: Roles, ClusterRoles, RoleBindings, ClusterRoleBindings
- Admission controllers: what they are (ValidatingWebhookConfiguration, MutatingWebhookConfiguration)
- SecurityContexts at Pod and container level
- Kustomize: `kustomization.yaml`, overlays, patches

#### Hands-On Labs (~7 h)

- [ ] Install Helm and add the Bitnami chart repository
- [ ] Deploy an application with `helm install` and customize values
- [ ] Upgrade a Helm release and roll it back
- [ ] Use `helm template` to preview generated YAML
- [ ] Create a CRD and a custom resource instance
- [ ] Create a Role and RoleBinding that grants read-only access to Pods
- [ ] Apply Kustomize overlays to modify a base set of manifests
- [ ] Set a SecurityContext that prevents a container from running as root

#### Practical Challenge — Week 10

> **Challenge:** Using Helm, deploy an NGINX chart with custom values (replica count
> = 3, custom port). Then upgrade the release to change the replica count to 5. Roll
> back to the previous version. Finally, create a Kustomize overlay that patches the
> Deployment to add a label `env: staging`.

---

## Phase 4 — Speed & Exam Readiness (Weeks 11–12)

Your goal now is *speed* and *accuracy* under pressure. You should be able to solve
most tasks from memory with minimal doc lookups.

---

### Week 11: Timed Practice & Weak-Spot Drills

**What you will learn:** How to work under exam conditions — 2 hours, ~15–20
questions, no IDE, only `vim`/`nano` and the Kubernetes docs.

#### Theory (~3 h, reduced)

- Review any domains where you scored poorly on practice tests
- Re-read the official CKAD exam FAQ and environment rules
- Memorize `kubectl explain` paths for commonly tested objects
- Review the allowed bookmarks: https://kubernetes.io/docs and https://kubernetes.io/blog

#### Hands-On Labs (~9 h, increased)

- [ ] Complete a full CKAD mock exam (killer.sh — one free retake comes with the exam purchase)
- [ ] Time yourself: aim for < 6 minutes per question on average
- [ ] Practice the "skip and return" strategy: flag hard questions and come back
- [ ] Drill your weakest domain (likely NetworkPolicies or RBAC) with 10 rapid-fire exercises
- [ ] Practice vim basics: search (`/pattern`), replace (`:%s/old/new/g`), copy/paste
- [ ] Work entirely with imperative commands + `--dry-run=client -o yaml` pipeline:
  1. Generate YAML imperatively
  2. Edit with vim
  3. Apply with `kubectl apply -f`
- [ ] Create all common resources from memory without looking at docs

#### Practical Challenge — Week 11

> **Challenge:** Set a 15-minute timer. Complete the following without consulting
> any documentation:
> 1. Create a Deployment `web` with 3 replicas of `nginx:1.25`.
> 2. Expose it as a NodePort Service.
> 3. Create an Ingress routing `web.local/` to the Service.
> 4. Add a liveness probe (HTTP GET on `/`) to the Deployment.
> 5. Create a NetworkPolicy allowing ingress only from Pods with label `role=frontend`.

---

### Week 12: Final Mock Exams & Confidence Building

**What you will learn:** Simulate exam day. Eliminate last-minute gaps.

#### Theory (~2 h, minimal)

- Read through the list of `kubectl` cheat sheet one final time
- Review any questions you got wrong on mock exams
- Mental preparation: sleep well, know your exam check-in process

#### Hands-On Labs (~10 h, maximum)

- [ ] Take a second killer.sh mock exam (or another provider's mock)
- [ ] Redo any question you got wrong and understand *why*
- [ ] Do a "speed run" of 20 common tasks in under 40 minutes:
  - Pod, Deployment, Service, Ingress, ConfigMap, Secret, PVC, Job, CronJob,
    NetworkPolicy, ServiceAccount, Role, RoleBinding, LimitRange, ResourceQuota,
    Probes, Multi-container Pod, Helm install, Kustomize overlay, CRD
- [ ] Practice context switching: `kubectl config use-context` (the exam gives you
      multiple clusters)
- [ ] Final review of imperative commands and aliases

#### Practical Challenge — Week 12

> **Challenge:** Full 2-hour mock exam simulation. Set up a quiet environment, use
> only a terminal and the Kubernetes docs. Grade yourself. If you score above 66%
> (the passing mark), you are ready. If not, identify the top 3 gaps and drill them
> for another day.

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
