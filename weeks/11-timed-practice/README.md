---
title: Week 11 — Timed Practice & Weak-Spot Drills
permalink: /weeks/11-timed-practice/
---

# Week 11: Timed Practice & Weak-Spot Drills

## The Concept

The CKAD exam is a performance-based certification that tests your ability to work with Kubernetes in a time-pressured environment. Understanding the exam format and developing effective strategies is essential for success.

**Exam format:** You have 2 hours to complete approximately 15–20 performance-based questions. There are no multiple-choice items; every task requires you to use the command line to create, modify, or troubleshoot Kubernetes resources. The exam runs in an Ubuntu terminal with vim and nano available. You have access to kubernetes.io/docs only—no other external resources.

**Scoring:** You need 66% to pass. Each question carries a weight, and partial credit is possible. If you cannot fully solve a question, submitting a partial solution is better than leaving it blank.

**Time management:** With roughly 6 minutes per question on average, you must work efficiently. The most important strategy is "flag and move on": never spend more than 8 minutes on any single question. If you are stuck, flag it, move to the next, and return later if time permits.

**Context switching:** The exam provides multiple clusters. You must switch between them using `kubectl config use-context <context-name>`. Always verify you are in the correct context before answering; answering in the wrong cluster costs you points.

**The imperative-then-edit workflow:** Writing YAML from scratch is slow. Use this workflow instead: (1) Generate YAML with `kubectl run`, `kubectl create deployment`, or similar commands using `--dry-run=client -o yaml > file.yaml`; (2) Edit the file with vim to add probes, volumes, or other fields; (3) Apply with `kubectl apply -f file.yaml`. This is the most powerful technique for the exam.

**Vim survival guide:** You need basic vim to edit YAML. Navigation: h (left), j (down), k (up), l (right). Press `i` to enter insert mode, `Esc` to exit. Save and quit: `:wq`. Search: `/pattern` then `n` for next. Replace: `:%s/old/new/g`. Undo: `u`. Copy line: `yy`. Paste: `p`. Delete line: `dd`.

**Bookmarks:** Have the kubectl cheat sheet and the API reference bookmarked in kubernetes.io/docs for quick lookups.

## Beginner Tutorial

1. **Set up aliases and autocompletion (from Week 1)**  
   Ensure `k` is aliased to `kubectl` and that `source <(kubectl completion bash)` (or zsh) is in your shell config.

2. **Practice context switching**  
   Create two kind clusters: `kind create cluster --name cluster-a` and `kind create cluster --name cluster-b`. Use `kubectl config get-contexts` to list contexts, then `kubectl config use-context kind-cluster-a` and `kind-cluster-b` to switch. Verify with `kubectl cluster-info`.

3. **Practice the imperative workflow for each resource type:**
   - Pod: `k run mypod --image=nginx --dry-run=client -o yaml > pod.yaml`
   - Deployment: `k create deployment mydeploy --image=nginx --dry-run=client -o yaml > deploy.yaml`
   - Service: `k expose deployment mydeploy --port=80 --dry-run=client -o yaml > svc.yaml`
   - ConfigMap: `k create configmap mycm --from-literal=KEY=value --dry-run=client -o yaml > cm.yaml`
   - Secret: `k create secret generic mysecret --from-literal=KEY=value --dry-run=client -o yaml > secret.yaml`
   - Job: `k create job myjob --image=busybox -- echo hello --dry-run=client -o yaml > job.yaml`
   - CronJob: `k create cronjob mycron --image=busybox --schedule="*/5 * * * *" -- echo date --dry-run=client -o yaml > cronjob.yaml`
   - Ingress: `k create ingress myingress --rule="host/path=svc:port" --dry-run=client -o yaml > ingress.yaml`
   - ServiceAccount: `k create serviceaccount mysa --dry-run=client -o yaml > sa.yaml`
   - Role: `k create role myrole --verb=get,list --resource=pods --dry-run=client -o yaml > role.yaml`
   - RoleBinding: `k create rolebinding myrb --role=myrole --serviceaccount=ns:mysa --dry-run=client -o yaml > rb.yaml`

4. **Practice vim**  
   Open a Deployment YAML, add a `livenessProbe` and `readinessProbe` section under `containers`, save with `:wq`, and apply the file.

5. **Practice `kubectl explain`**  
   Use `kubectl explain pod.spec.containers` and `kubectl explain deployment.spec.template.spec.containers` to find field paths quickly instead of searching the docs.

## Hands-On Lab

### Challenge 1: "Multi-Object Deploy" (6 min)

Create namespace `exam-1`. In it: Deployment `api` (image nginx, 3 replicas), ClusterIP Service on port 80, ConfigMap `api-config` with key `LOG_LEVEL=debug` injected as env var.

**Success criteria:** All resources exist in namespace `exam-1`, Deployment has 3 ready replicas, Service targets the Deployment, Pods have `LOG_LEVEL=debug` as env var.

<details>
<summary>Hints</summary>

- Create namespace first, then use `-n exam-1` for all subsequent commands.
- Use `k create deployment api --image=nginx --replicas=3 -n exam-1`, then `k expose deployment api --port=80 -n exam-1`.
- Create ConfigMap with `k create configmap api-config --from-literal=LOG_LEVEL=debug -n exam-1`, then edit the Deployment to add `envFrom` or `env` referencing the ConfigMap.
</details>

### Challenge 2: "Troubleshooting" (6 min)

A Pod named `broken-app` is in CrashLoopBackOff. Find and fix all issues. (Provide a broken YAML with: wrong image name, bad command, missing volume mount for a required ConfigMap.)

**Success criteria:** Pod runs successfully and stays in Running state.

<details>
<summary>Hints</summary>

- Use `kubectl describe pod broken-app` and `kubectl logs broken-app` to diagnose.
- Check: image name (typo or non-existent), command (invalid syntax), volume/volumeMount for ConfigMap.
- Fix by editing the manifest and reapplying, or by deleting and recreating.
</details>

### Challenge 3: "Network Security" (6 min)

In namespace `restricted`, create a default-deny NetworkPolicy. Then allow Pods with label `role=api` to receive traffic on port 8080 from Pods with label `role=frontend` only.

**Success criteria:** Default-deny policy exists; allow policy exists with correct podSelector, ingress rules, and port.

<details>
<summary>Hints</summary>

- Default-deny: `podSelector: {}` with no ingress rules.
- Allow policy: `podSelector: matchLabels: role: api`, `ingress` with `from` matching `role=frontend`, `ports` with `port: 8080`.
- NetworkPolicy has no imperative create command; use `--dry-run=client -o yaml` from a placeholder or write minimal YAML.
</details>

### Challenge 4: "Probes and Rollout" (6 min)

Create a Deployment `webapp` with nginx, add a liveness probe (HTTP GET / every 10s), and a readiness probe (HTTP GET / every 5s). Perform a rolling update to nginx:alpine. Verify zero-downtime with `kubectl rollout status`.

**Success criteria:** Deployment has both probes; rolling update completes; `kubectl rollout status` shows success.

<details>
<summary>Hints</summary>

- Generate Deployment YAML with `k create deployment webapp --image=nginx --dry-run=client -o yaml`, edit to add probes.
- Liveness: `httpGet: path: / port: 80`, `periodSeconds: 10`.
- Readiness: same path/port, `periodSeconds: 5`.
- Update: `k set image deployment/webapp nginx=nginx:alpine` then `k rollout status deployment/webapp`.
</details>

### Challenge 5: "Helm and RBAC" (8 min)

Install any nginx Helm chart with 2 replicas. Create a ServiceAccount `viewer`. Create a Role granting get/list on pods and services. Bind it to the ServiceAccount. Verify with `kubectl auth can-i`.

**Success criteria:** Helm release installed with 2 replicas; ServiceAccount, Role, and RoleBinding exist; `kubectl auth can-i get pods --as=system:serviceaccount:<namespace>:viewer` returns yes.

<details>
<summary>Hints</summary>

- `helm repo add bitnami https://charts.bitnami.com/bitnami` (if needed), `helm install mynginx bitnami/nginx --set replicaCount=2`.
- `k create serviceaccount viewer`, `k create role viewer-role --verb=get,list --resource=pods,services`, `k create rolebinding viewer-rb --role=viewer-role --serviceaccount=<namespace>:viewer`.
- Verify: `k auth can-i get pods --as=system:serviceaccount:<namespace>:viewer -n <namespace>`.
</details>

## Weekly Speed Drill

This week's speed drill is a comprehensive 15-minute "Create Everything" challenge. Create ALL of these from memory using imperative commands (no YAML files):

1. Namespace `speed`
2. ConfigMap `app-config` with APP_ENV=test
3. Secret `app-secret` with DB_PASS=exam123
4. Deployment `web` with 2 replicas of nginx
5. Service exposing the deployment on port 80
6. ServiceAccount `runner`
7. Role `pod-reader` with get/list on pods
8. RoleBinding binding `pod-reader` to `runner`
9. Job that runs `echo done`
10. CronJob that runs `date` every 5 minutes
11. Pod with resource limits (cpu: 200m, memory: 256Mi)
12. NetworkPolicy default-deny in namespace `speed`

Target: complete all 12 in under 15 minutes.

## Exam Pitfalls

- **Running out of time on hard questions** — Skip after 8 minutes, come back later.
- **Forgetting to switch context** — Always confirm you are in the correct cluster before answering.
- **Not using imperative commands** — Writing YAML from scratch is too slow; use `--dry-run=client -o yaml`.
- **Not knowing vim basics** — You must be able to edit generated YAML; learn hjkl, i, :wq, /, %s, u, yy, p, dd.
- **Forgetting `--dry-run=client -o yaml`** — This is the most powerful exam technique; use it for every resource.
- **Not using `kubectl explain`** — Faster than searching docs for field paths and structure.
- **Leaving incomplete answers** — Partial credit exists; always submit something.
- **Not reading the question carefully** — Check namespace, cluster context, and exact field names required.

## Solution Key

### Challenge 1: Multi-Object Deploy

```bash
kubectl create namespace exam-1
kubectl create deployment api --image=nginx --replicas=3 -n exam-1
kubectl expose deployment api --port=80 -n exam-1
kubectl create configmap api-config --from-literal=LOG_LEVEL=debug -n exam-1
kubectl set env deployment/api --from=configmap/api-config -n exam-1
# Or: kubectl edit deployment api -n exam-1 and add envFrom/configMapRef
```

### Challenge 2: Troubleshooting

```bash
# Inspect
kubectl describe pod broken-app
kubectl logs broken-app

# Fix: edit the manifest (fix image name, command, add volume/volumeMount for ConfigMap)
kubectl edit pod broken-app
# Or delete and recreate with corrected YAML
kubectl delete pod broken-app
kubectl apply -f fixed-pod.yaml
```

### Challenge 3: Network Security

```bash
kubectl create namespace restricted

# Default-deny
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: restricted
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Allow role=api from role=frontend on 8080
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: restricted
spec:
  podSelector:
    matchLabels:
      role: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
EOF
```

### Challenge 4: Probes and Rollout

```bash
kubectl create deployment webapp --image=nginx --dry-run=client -o yaml > webapp.yaml
# Edit webapp.yaml to add under containers:
#   livenessProbe:
#     httpGet:
#       path: /
#       port: 80
#     periodSeconds: 10
#   readinessProbe:
#     httpGet:
#       path: /
#       port: 80
#     periodSeconds: 5
kubectl apply -f webapp.yaml
kubectl set image deployment/webapp nginx=nginx:alpine
kubectl rollout status deployment/webapp
```

### Challenge 5: Helm and RBAC

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mynginx bitnami/nginx --set replicaCount=2

NS=$(kubectl config view --minify -o jsonpath='{..namespace}')
NS=${NS:-default}

kubectl create serviceaccount viewer -n $NS
kubectl create role viewer-role --verb=get,list --resource=pods,services -n $NS
kubectl create rolebinding viewer-rb --role=viewer-role --serviceaccount=$NS:viewer -n $NS
kubectl auth can-i get pods --as=system:serviceaccount:$NS:viewer -n $NS
```

### Speed Drill (all 12 in under 15 min)

```bash
kubectl create namespace speed

kubectl create configmap app-config --from-literal=APP_ENV=test -n speed
kubectl create secret generic app-secret --from-literal=DB_PASS=exam123 -n speed

kubectl create deployment web --image=nginx --replicas=2 -n speed
kubectl expose deployment web --port=80 -n speed

kubectl create serviceaccount runner -n speed
kubectl create role pod-reader --verb=get,list --resource=pods -n speed
kubectl create rolebinding pod-reader-rb --role=pod-reader --serviceaccount=speed:runner -n speed

kubectl create job echo-job --image=busybox -- echo done -n speed
kubectl create cronjob date-cron --image=busybox --schedule="*/5 * * * *" -- date -n speed

kubectl run limited-pod --image=nginx --restart=Never --limits='cpu=200m,memory=256Mi' -n speed

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: speed
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```
