---
title: Week 3 — Pods & Multi-Container Patterns
permalink: /weeks/03-pods-and-multi-container/
---

# Week 3: Pods In Depth & Multi-Container Patterns

## The Concept

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers that share the same network namespace and storage. Understanding how Pods behave over time, how to run multiple containers together, and how to control resources is essential for the CKAD exam.

**Pod lifecycle.** When you create a Pod, it goes through distinct phases. **Pending** means the Pod has been accepted by the cluster but one or more containers have not yet started. Common causes: the scheduler has not yet placed it on a node, or the node is pulling the container image. **Running** means the Pod is bound to a node and at least one container is running. **Succeeded** means all containers terminated successfully (exit code 0). **Failed** means at least one container failed (non-zero exit). You can observe these transitions with `kubectl get pod -w`.

**Restart policies** control what happens when a container exits. **Always** (the default) restarts the container whenever it stops, regardless of exit code. Use this for long-running services like web servers. **OnFailure** restarts only when the container exits with a non-zero code. Use this for batch jobs that may fail and need retries. **Never** never restarts. Use this for one-off tasks that should run once and stop; the Pod will show **Completed** when the container exits successfully.

**Init containers** run before the main application containers. They run in sequence: each init container must complete successfully before the next starts. Use them to wait for dependencies (e.g., a database to be reachable), download configuration, or prepare shared storage. If an init container fails, the Pod stays in Init state and the main containers never start. Init containers are defined under `spec.initContainers` at the same indentation level as `spec.containers`.

**Multi-container patterns** are common on the CKAD exam. A **Sidecar** runs alongside the main container. Examples: a log shipper that tails application logs and sends them to a central system, or a monitoring agent that scrapes metrics. The sidecar shares the Pod's network and volumes with the main container. An **Ambassador** acts as a proxy to external services. The main container talks to localhost; the ambassador forwards traffic to the real backend. This simplifies configuration when the backend URL changes. An **Adapter** transforms output from the main container. For example, the main app writes logs in a custom format; the adapter reformats them to JSON or another standard before they are shipped.

**Shared resources.** Containers in a Pod share the same network namespace. They can reach each other via **localhost** and the same port space. They also share volumes. **emptyDir** is a temporary volume created when the Pod starts and deleted when the Pod is removed. It is ideal for sharing data between containers in the same Pod (e.g., an init container writes a config file, the main container reads it).

**Resource requests and limits.** A **request** is the guaranteed minimum the scheduler reserves for the container. A **limit** is the maximum the container can use. The scheduler uses requests to decide placement; limits prevent a container from consuming more than allowed. CPU is specified in cores: `100m` means 0.1 CPU (100 millicores). Memory uses `Mi` (mebibytes) or `Gi` (gibibytes): `128Mi` is 128 mebibytes. If a container exceeds its memory limit, it is **OOMKilled** (Out Of Memory killed) and restarted according to the restart policy.

---

## Beginner Tutorial

Follow these steps in order. Run each command and observe the output.

### 1. Create a Pod and observe lifecycle with `kubectl get pod -w`

Create `simple-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
    - name: nginx
      image: nginx:alpine
```

Apply and watch:

```bash
kubectl apply -f simple-pod.yaml
kubectl get pod lifecycle-demo -w
```

Observe the transition from Pending to Running. Press Ctrl+C to stop watching.

### 2. Pod with `restartPolicy: Never` — observe Completed status

Create `never-restart-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: one-shot
spec:
  restartPolicy: Never
  containers:
    - name: busybox
      image: busybox:1.36
      command: ["sh", "-c", "echo done && sleep 2"]
```

Apply and watch:

```bash
kubectl apply -f never-restart-pod.yaml
kubectl get pod one-shot -w
```

The Pod will show **Completed** when the container exits successfully. It will not restart.

### 3. Pod with init container that writes to shared emptyDir, main reads it

Create `init-emptyDir-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-shared
spec:
  containers:
    - name: main
      image: busybox:1.36
      command: ["sh", "-c", "cat /shared/config.txt && sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  initContainers:
    - name: init
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Hello from init' > /shared/config.txt"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

Apply and verify:

```bash
kubectl apply -f init-emptyDir-pod.yaml
kubectl logs init-shared -c main
```

Expected output: `Hello from init`.

### 4. Sidecar pattern: nginx + busybox writing date to shared volume every 5s

Create `sidecar-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: shared
          mountPath: /usr/share/nginx/html
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date > /shared/date.txt; sleep 5; done"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

Note: nginx serves from `/usr/share/nginx/html`. The sidecar writes to `/shared`. To share the same directory, use a single mount path. Revised version:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: shared
          mountPath: /usr/share/nginx/html
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date > /usr/share/nginx/html/date.txt; sleep 5; done"]
      volumeMounts:
        - name: shared
          mountPath: /usr/share/nginx/html
  volumes:
    - name: shared
      emptyDir: {}
```

Apply and test:

```bash
kubectl apply -f sidecar-pod.yaml
kubectl wait --for=condition=Ready pod/sidecar-demo --timeout=60s
kubectl exec sidecar-demo -c sidecar -- cat /usr/share/nginx/html/date.txt
```

### 5. Set resource requests and limits

Create `resources-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resources-demo
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      resources:
        requests:
          memory: "64Mi"
          cpu: "100m"
        limits:
          memory: "128Mi"
          cpu: "200m"
```

Apply:

```bash
kubectl apply -f resources-pod.yaml
kubectl describe pod resources-demo
```

Check the Limits and Requests section in the output.

### 6. Exceed memory limit to see OOMKilled

Create `oom-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
    - name: memory-eater
      image: busybox:1.36
      command: ["sh", "-c", "sleep 2 && x=; while true; do x=$x$x; done"]
      resources:
        limits:
          memory: "32Mi"
```

Apply and watch:

```bash
kubectl apply -f oom-pod.yaml
kubectl get pod oom-demo -w
```

The container will be OOMKilled. Check `kubectl describe pod oom-demo` for the OOMKilled reason.

### 7. Use `kubectl explain pod.spec.initContainers`

```bash
kubectl explain pod.spec.initContainers
```

This shows the schema for init containers. Drill down:

```bash
kubectl explain pod.spec.initContainers.image
kubectl explain pod.spec.initContainers.command
```

---

## Hands-On Lab

Five performance-based challenges. Complete each and verify the success criteria.

### Challenge 1: Init Container Gate

**Scenario:** Create a Pod with an init container that waits for a Service named `mydb` to be resolvable via `nslookup`. The main container runs `busybox` with `sleep 3600`. The Pod should stay in Init until you create the `mydb` Service. Once the Service exists, the init container succeeds and the main container starts.

**Success criteria:** Pod is Init:0/1 until `mydb` Service exists. After creating the Service, Pod becomes Running.

<details>
<summary>Hints</summary>

- Use an init container with `command: ["sh", "-c", "until nslookup mydb; do sleep 2; done"]`.
- The Service can be a simple ClusterIP with no selector (headless) or with a selector pointing to any Pod.
- Create the Pod first, observe Init state. Then create the Service in the same namespace.
</details>

### Challenge 2: Sidecar Log Shipper

**Scenario:** Create a Pod with two containers. The main container writes logs to `/var/log/app.log` (e.g., a loop that appends a line every 3 seconds). The sidecar container tails `/var/log/app.log` and outputs to stdout so you can view the logs with `kubectl logs <pod> -c sidecar`.

**Success criteria:** Main container writes to the shared file. `kubectl logs <pod> -c sidecar` shows the tailed content.

<details>
<summary>Hints</summary>

- Use an emptyDir volume mounted at `/var/log` in both containers.
- Main: `while true; do echo "$(date) log line" >> /var/log/app.log; sleep 3; done`
- Sidecar: `tail -f /var/log/app.log`
- Use `-c sidecar` to get logs from the sidecar container.
</details>

### Challenge 3: Fix a CrashLoopBackOff

**Scenario:** A Pod named `broken` is in CrashLoopBackOff. Diagnose the cause (bad command, wrong image, or similar) and fix it so the Pod runs successfully. The Pod should run `nginx` and stay Running.

**Success criteria:** Pod status is Running. No restarts after the fix.

<details>
<summary>Hints</summary>

- Use `kubectl describe pod broken` to see events and last state.
- Use `kubectl logs broken --previous` to see logs from the crashed container.
- Common causes: wrong image name, command that exits immediately, missing required env vars.
</details>

### Challenge 4: Resource Limits

**Scenario:** Create a Pod that requests 64Mi memory and has a limit of 128Mi. Configure the container to allocate 200Mi (e.g., using a memory-eating script). Observe OOMKilled. Then fix the Pod so it stays within the limit and runs successfully.

**Success criteria:** First run: Pod is OOMKilled. After fix: Pod runs without OOMKilled.

<details>
<summary>Hints</summary>

- Use `resources.requests.memory: "64Mi"` and `resources.limits.memory: "128Mi"`.
- To exceed: use a script that allocates memory (e.g., `dd` or a loop that grows a variable).
- Fix: either increase the limit to 256Mi or change the app to use less memory.
</details>

### Challenge 5: Adapter Pattern

**Scenario:** Create a Pod with a main container that outputs custom log lines in the format `LEVEL: message` (e.g., `INFO: request received`). An adapter container reads these logs, reformats them to JSON (e.g., `{"level":"INFO","message":"request received"}`), and outputs to stdout. View the adapter output with `kubectl logs`.

**Success criteria:** Main container produces custom format. Adapter outputs valid JSON to stdout. `kubectl logs <pod> -c adapter` shows JSON lines.

<details>
<summary>Hints</summary>

- Main writes to a shared file (emptyDir). Adapter reads and transforms.
- Use `awk` or `sed` in the adapter to parse and reformat, or a simple script.
- Example: `echo "INFO: request received" >> /shared/app.log` (main); adapter tails and transforms.
</details>

---

## Weekly Speed Drill

Set a 15-minute timer. Complete these 10 tasks as fast as possible.

1. Create a busybox Pod with `sleep 3600`
2. Create a 2-container Pod (nginx + busybox) sharing an emptyDir
3. Get logs from the specific container `busybox` using `-c busybox`
4. Generate multi-container YAML via dry-run, then add a second container
5. Set resource requests on a Pod
6. Create a Pod with `restartPolicy: OnFailure`
7. Check the restart count of a Pod
8. Delete and recreate a Pod
9. View Pod events only
10. Get Pods sorted by creation timestamp

**Commands:**

```bash
# 1
kubectl run sleepy --image=busybox:1.36 --restart=Never -- sleep 3600

# 2
# Create pod-multi.yaml with nginx + busybox sharing emptyDir, then:
kubectl apply -f pod-multi.yaml

# 3
kubectl logs <pod-name> -c busybox

# 4
kubectl run multi --image=nginx --dry-run=client -o yaml > multi.yaml
# Edit multi.yaml to add second container under spec.containers, then:
kubectl apply -f multi.yaml

# 5
# Add to pod spec: resources: { requests: { memory: "64Mi", cpu: "100m" } }
kubectl apply -f pod.yaml

# 6
kubectl run onfail --image=busybox --restart=OnFailure -- sh -c "exit 1"

# 7
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].restartCount}'

# 8
kubectl delete pod <pod-name>
kubectl apply -f pod.yaml

# 9
kubectl get events --field-selector involvedObject.name=<pod-name>

# 10
kubectl get pods --sort-by=.metadata.creationTimestamp
```

---

## Exam Pitfalls

Common mistakes that cost points on the CKAD exam:

**1. Forgetting `-c <name>` for multi-container Pods**

When a Pod has multiple containers, `kubectl logs` and `kubectl exec` default to the first container. You must specify `-c <container-name>` to target a specific container. Without it, you may view or exec into the wrong one.

**2. Init container ordering**

Init containers run sequentially. Each must complete before the next starts. If you need init container B to run after A, list them in order under `initContainers`. Do not assume parallel execution.

**3. Wrong restartPolicy for Jobs**

Jobs use Pods with `restartPolicy: OnFailure` or `Never`. Using `Always` for a Job's Pod template is invalid; Jobs require that the Pod does not restart indefinitely on success.

**4. Confusing emptyDir with PersistentVolume**

**emptyDir** is ephemeral: it is created when the Pod starts and deleted when the Pod is removed. Data does not persist across Pod restarts. A **PersistentVolume** (PV) is for long-term storage. Use emptyDir for temporary sharing between containers in the same Pod.

**5. Memory limits too low (OOMKilled)**

If you set a memory limit below what the application needs, the container will be OOMKilled repeatedly. Check `kubectl describe pod` for the OOMKilled reason. Increase the limit or reduce memory usage.

**6. Not knowing containers share localhost**

Containers in a Pod share the same network namespace. They can reach each other via `localhost`. If the main container listens on port 8080, a sidecar can connect to `localhost:8080`. Do not use the Pod IP for inter-container communication within the same Pod.

**7. Init container must succeed first**

If any init container fails, the main containers never start. The Pod stays in Init state. Ensure init container commands exit with 0 on success. Use `until` loops or retry logic for dependency checks.

**8. initContainers indentation level**

`initContainers` is a sibling of `containers`, not a child. Both live under `spec`:

```yaml
spec:
  initContainers:
    - name: init
      image: busybox
  containers:
    - name: main
      image: nginx
```

Wrong: placing `initContainers` inside `containers` or at the wrong indentation causes validation errors.

---

## Solution Key

### Challenge 1: Init Container Gate

**Pod YAML (`init-gate-pod.yaml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-gate
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ["sh", "-c", "until nslookup mydb; do echo waiting for mydb; sleep 2; done"]
  containers:
    - name: main
      image: busybox:1.36
      command: ["sleep", "3600"]
```

**Service YAML (`mydb-svc.yaml`):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
    - port: 3306
      targetPort: 3306
  selector:
    app: mydb
```

**Commands:**

```bash
kubectl apply -f init-gate-pod.yaml
kubectl get pod init-gate
# STATUS: Init:0/1

# Create a placeholder Pod for the Service (or use a Service without selector)
kubectl run mydb --image=mysql:8 --restart=Never
kubectl apply -f mydb-svc.yaml

# Or create Service without selector (headless-style) - need Endpoints manually
# Simpler: create any Pod with label app=mydb, then create Service with that selector
kubectl run mydb --image=busybox --restart=Never --labels=app=mydb -- sleep 3600
kubectl expose pod mydb --name mydb --port=3306

kubectl get pod init-gate
# STATUS: Running (after init succeeds)
```

**Expected output:** Pod transitions from Init:0/1 to Running after `mydb` Service exists.

---

### Challenge 2: Sidecar Log Shipper

**Pod YAML (`sidecar-logs-pod.yaml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-shipper
spec:
  containers:
    - name: main
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"$(date) log line\" >> /var/log/app.log; sleep 3; done"]
      volumeMounts:
        - name: logs
          mountPath: /var/log
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /var/log/app.log"]
      volumeMounts:
        - name: logs
          mountPath: /var/log
  volumes:
    - name: logs
      emptyDir: {}
```

**Commands:**

```bash
kubectl apply -f sidecar-logs-pod.yaml
kubectl wait --for=condition=Ready pod/log-shipper --timeout=60s
kubectl logs log-shipper -c sidecar
```

**Expected output:** Lines like `Wed Feb 25 12:00:00 UTC 2025 log line` streaming from the sidecar.

---

### Challenge 3: Fix a CrashLoopBackOff

**Diagnosis:**

```bash
kubectl describe pod broken
kubectl logs broken --previous
```

**Common fixes:** Wrong image (e.g., `ngnix` instead of `nginx`), command that exits (e.g., `echo done` instead of keeping process running). For nginx, ensure no invalid command overrides the default.

**Fixed Pod YAML (`broken-fixed.yaml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken
spec:
  containers:
    - name: nginx
      image: nginx:alpine
```

**Commands:**

```bash
kubectl delete pod broken --force --grace-period=0
kubectl apply -f broken-fixed.yaml
kubectl get pod broken
```

**Expected output:** STATUS = Running, RESTARTS = 0.

---

### Challenge 4: Resource Limits

**OOM Pod YAML (`oom-pod.yaml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 5 && x=; while true; do x=$x$x; done"]
      resources:
        requests:
          memory: "64Mi"
        limits:
          memory: "128Mi"
```

**Commands (observe OOMKilled):**

```bash
kubectl apply -f oom-pod.yaml
kubectl get pod resource-demo -w
# Observe OOMKilled
kubectl describe pod resource-demo
```

**Fixed Pod YAML (`resource-fixed.yaml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      resources:
        requests:
          memory: "64Mi"
        limits:
          memory: "128Mi"
```

**Commands:**

```bash
kubectl delete pod resource-demo
kubectl apply -f resource-fixed.yaml
kubectl get pod resource-demo
```

**Expected output:** STATUS = Running, no OOMKilled.

---

### Challenge 5: Adapter Pattern

**Pod YAML (`adapter-pod.yaml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter-demo
spec:
  containers:
    - name: main
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          while true; do
            echo "INFO: request received"
            echo "ERROR: connection failed"
            sleep 2
          done >> /shared/app.log
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: adapter
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          while true; do
            if [ -f /shared/app.log ]; then
              tail -f /shared/app.log | while read line; do
                level=$(echo "$line" | cut -d: -f1)
                msg=$(echo "$line" | cut -d: -f2-)
                echo "{\"level\":\"$level\",\"message\":\"$msg\"}"
              done
            fi
            sleep 1
          done
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

**Simpler adapter using awk (`adapter-pod-simple.yaml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter-demo
spec:
  containers:
    - name: main
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          while true; do
            echo "INFO: request received"
            sleep 2
          done >> /shared/app.log
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: adapter
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          sleep 3
          tail -f /shared/app.log | while read line; do
            level=$(echo "$line" | cut -d: -f1 | tr -d ' ')
            msg=$(echo "$line" | cut -d: -f2- | sed 's/^ //')
            echo "{\"level\":\"$level\",\"message\":\"$msg\"}"
          done
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

**Commands:**

```bash
kubectl apply -f adapter-pod-simple.yaml
kubectl wait --for=condition=Ready pod/adapter-demo --timeout=60s
kubectl logs adapter-demo -c adapter
```

**Expected output:** JSON lines like `{"level":"INFO","message":" request received"}`.

---

*Next: Week 4 — ConfigMaps, Secrets, and Security.*
