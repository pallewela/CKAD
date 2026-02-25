---
title: Week 9 — Observability, Probes & Debugging
permalink: /weeks/09-observability-and-debugging/
---

# Week 9: Observability, Probes & Debugging

## The Concept

Observability is the practice of understanding what is happening inside your system from the outside. In Kubernetes, you cannot log into a container the way you log into a physical server. You must rely on probes, logs, events, and metrics to diagnose problems. This domain carries significant weight on the CKA exam: 15% for Observability plus overlap with the 25% Security and Configuration domain. Mastery of these tools is essential.

**Why observability matters in the exam:** You WILL be asked to debug broken Pods, fix probes, and read logs. The exam presents you with broken workloads. A Pod may be in CrashLoopBackOff, ImagePullBackOff, or Pending. A Service may not be routing traffic. You must use `kubectl describe`, `kubectl logs`, and `kubectl get events` to find the root cause and fix it. There is no guessing: the Events section and logs contain the answers. Knowing the debugging playbook by heart saves time under exam pressure.

**Terms you must know:** Endpoints are the list of Pod IPs a Service routes traffic to. CrashLoopBackOff means the container starts, crashes, and Kubernetes backs off before restarting. ImagePullBackOff means the image cannot be pulled (wrong name, private registry without credentials, or network failure). Pending means the Pod cannot be scheduled (often due to insufficient CPU or memory). CreateContainerConfigError means a volume, ConfigMap, or Secret referenced in the Pod spec does not exist.

---

### Three Types of Probes

Kubernetes uses probes to determine container health. Each probe type answers a different question and triggers a different action.

**Liveness Probe** — "Is the container still alive?" If the liveness probe fails, Kubernetes RESTARTS the container. Use it for deadlock detection: an application that is running but no longer responding. A web server that has stopped accepting connections, a process that has hung on a mutex, or a thread pool that is exhausted. The liveness probe gives Kubernetes permission to kill and restart the container when it believes the process is stuck. Be careful: if the probe is too aggressive (short timeout, low failure threshold) or fires before the app is ready (low initialDelaySeconds), you can create a restart loop where the container is killed before it finishes starting.

**Readiness Probe** — "Is the container ready to receive traffic?" If the readiness probe fails, the Pod is removed from the Service's Endpoints list. It is no longer a target for load balancer traffic. The container is NOT restarted. Use it for slow startup (the app needs time to load config or connect to a database) or temporary unavailability (the app is overloaded and should not receive new requests). When the probe succeeds again, the Pod is added back to Endpoints. Readiness is the correct choice when you want to stop sending traffic without restarting the process.

**Startup Probe** — "Has the container finished starting?" The startup probe disables liveness and readiness checks until it succeeds for the first time. Use it for slow-starting legacy applications: a Java app that takes 60 seconds to initialize, a database that needs time to recover. Without a startup probe, the liveness probe would fire during startup, fail (because the app is not ready yet), and restart the container repeatedly. The startup probe gives the container a grace period. Once it succeeds, liveness and readiness take over. You can set a high failureThreshold so that `failureThreshold * periodSeconds` equals your expected startup time (e.g., 30 failures at 10 seconds = 300 seconds of startup time).

---

### Probe Mechanisms

Probes can check health in four ways:

**httpGet** — Sends an HTTP GET request to a path and port. The probe succeeds if the response status code is between 200 and 399 (inclusive). Use it for HTTP services. Example: `path: /healthz`, `port: 8080`.

**tcpSocket** — Checks if a TCP port is open. The probe succeeds if a connection can be established. Use it when the app does not expose an HTTP endpoint but does listen on a port (e.g., a database).

**exec** — Runs a command inside the container. The probe succeeds if the command exits with code 0. Non-zero exit means unhealthy. Use it when HTTP or TCP checks are not suitable. Example: `command: ["cat", "/tmp/healthy"]`.

**grpc** — Sends a gRPC health check. Available in Kubernetes 1.24+. Use it for gRPC services.

---

### Probe Parameters Explained in Detail

**initialDelaySeconds** — How long to wait after the container starts before the first probe runs. If your app needs 10 seconds to open a port, set this to at least 10. Too low and the first probe fails, triggering a restart (liveness) or removal from Endpoints (readiness). This is the most common cause of probe-induced restart loops.

**periodSeconds** — How often to run the probe after the first success. Default is 10. A value of 5 means the probe runs every 5 seconds.

**timeoutSeconds** — How long to wait for a response. Default is 1. If the probe takes longer, it is counted as a failure. For slow endpoints, increase this.

**failureThreshold** — How many consecutive failures before the probe is considered failed. For liveness, the default is 3: three failures in a row trigger a restart. For readiness, three failures remove the Pod from Endpoints. Setting failureThreshold to 1 is aggressive: a single slow response or transient network blip can trigger action.

**successThreshold** — How many successes are required to be considered recovered. For liveness and startup, the default is 1: one success means the probe has passed. For readiness, the default is 1: one success adds the Pod back to Endpoints. You can increase this if you want multiple consecutive successes before considering the Pod recovered.

---

### Container Logging

**`kubectl logs <pod>`** — Streams logs from the main container. If the Pod has multiple containers, you must specify which one with `-c <container>`.

**`kubectl logs <pod> --previous`** — Logs from the LAST crashed container. Critical for debugging CrashLoopBackOff: the current container may not have started yet, so its logs are empty. The previous container's logs show why it crashed.

**`kubectl logs <pod> -c <container>`** — Logs from a specific container in a multi-container Pod.

**`kubectl logs <pod> -f`** — Follows logs in real time (like `tail -f`).

**`kubectl logs <pod> --tail=100`** — Last 100 lines.

**`kubectl logs <pod> --since=1h`** — Logs from the last hour.

---

### Debugging Workflow (The Exam Playbook)

1. **`kubectl get pods`** — Check status. CrashLoopBackOff, ImagePullBackOff, Pending, ErrImagePull, CreateContainerConfigError, etc. Each status points to a category of problem.

2. **`kubectl describe pod <name>`** — Inspect the Pod spec. The Events section at the bottom shows what happened: "Failed to pull image", "Back-off restarting failed container", "0/3 nodes are available: insufficient memory", "MountVolume.SetUp failed for volume". Most answers are here.

3. **`kubectl logs <pod>`** and **`kubectl logs <pod> --previous`** — Application output. Errors, stack traces, connection refused. For CrashLoopBackOff, always use `--previous` first.

4. **`kubectl get events --sort-by=.metadata.creationTimestamp`** — Cluster-wide events. Often reveals scheduling issues, node problems, or resource constraints.

5. **`kubectl exec -it <pod> -- /bin/sh`** — Get a shell inside the container. Test connectivity with wget or curl, check files, verify config. Use when logs are insufficient.

---

### Monitoring

**`kubectl top pods`** — CPU and memory usage per Pod. Requires metrics-server.

**`kubectl top nodes`** — CPU and memory usage per node. Requires metrics-server.

**`kubectl top pods --sort-by=cpu`** — Sort Pods by CPU usage.

**`kubectl top pods --sort-by=memory`** — Sort Pods by memory usage.

**metrics-server** — A cluster add-on that aggregates resource usage from the kubelet. Without it, `kubectl top` returns "Metrics API not available". Install it in clusters that do not have it (e.g., minikube, kind).

---

### API Deprecations

Kubernetes deprecates and removes API versions over time. Use `kubectl api-resources` to see current API versions. Use `kubectl explain` to inspect resource schemas. Check the Kubernetes deprecation guide for removed APIs and migration paths. On the exam, manifests may use deprecated apiVersion; you may need to update them to the current version.

---

## Beginner Tutorial

Follow these steps to build hands-on familiarity with probes, logging, and debugging.

### 1. Deploy a Go app (or nginx) with an HTTP liveness probe

Use an app that serves on port 8080 with a `/healthz` endpoint. The hashicorp/http-echo image returns 200 on any path. Create a Deployment with a liveness probe:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: probe-demo
  template:
    metadata:
      labels:
        app: probe-demo
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-listen=:8080", "-text=ok"]
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
```

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: probe-demo
  template:
    metadata:
      labels:
        app: probe-demo
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-listen=:8080", "-text=ok"]
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
EOF

kubectl get pods -l app=probe-demo
kubectl describe pod -l app=probe-demo
```

### 2. Add a readiness probe

Add a readiness probe to the same Deployment. Use `/ready` (http-echo returns 200 on any path):

```yaml
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
          failureThreshold: 2
```

```bash
kubectl patch deployment probe-demo --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/readinessProbe", "value": {"httpGet": {"path": "/ready", "port": 8080}, "initialDelaySeconds": 3, "periodSeconds": 5, "failureThreshold": 2}}]'
```

### 3. Add a startup probe for slow startup

Add a startup probe. It disables liveness and readiness until it succeeds:

```yaml
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 2
          failureThreshold: 15
```

```bash
kubectl patch deployment probe-demo --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/startupProbe", "value": {"httpGet": {"path": "/healthz", "port": 8080}, "initialDelaySeconds": 0, "periodSeconds": 2, "failureThreshold": 15}}]'
```

### 4. Break the liveness endpoint and watch the container restart

Change the liveness probe to a path that returns 404 (e.g., `/nonexistent`). The probe fails, and after `failureThreshold` failures, Kubernetes restarts the container:

```bash
kubectl patch deployment probe-demo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe/httpGet/path", "value": "/nonexistent"}]'

kubectl get pods -l app=probe-demo -w
```

Observe: the Pod enters a restart loop. `kubectl describe pod` shows "Liveness probe failed" in Events. Restore the probe:

```bash
kubectl patch deployment probe-demo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe/httpGet/path", "value": "/healthz"}]'
```

### 5. Break the readiness endpoint — Pod stays running but is removed from Endpoints

Create a Service for the Deployment:

```bash
kubectl expose deployment probe-demo --port=8080
```

Check Endpoints:

```bash
kubectl get endpoints probe-demo
```

Change the readiness probe to a failing path:

```bash
kubectl patch deployment probe-demo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/path", "value": "/nonexistent"}]'
```

Wait for the new Pod to roll out. Check Endpoints again:

```bash
kubectl get endpoints probe-demo
```

The Endpoints list is empty (or has no addresses). The Pod is still Running, but it is not receiving traffic. Restore the readiness probe:

```bash
kubectl patch deployment probe-demo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/path", "value": "/ready"}]'
```

### 6. Use kubectl describe pod to see probe failure events

```bash
kubectl describe pod -l app=probe-demo
```

Scroll to the Events section. You will see entries like "Liveness probe failed" or "Readiness probe failed" when probes fail.

### 7. Debug a CrashLoopBackOff scenario

Create a broken Deployment that crashes immediately:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash-demo
  template:
    metadata:
      labels:
        app: crash-demo
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sh", "-c", "echo 'About to crash' && exit 1"]
```

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash-demo
  template:
    metadata:
      labels:
        app: crash-demo
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sh", "-c", "echo 'About to crash' && exit 1"]
EOF

kubectl get pods -l app=crash-demo
# Status: CrashLoopBackOff

kubectl logs -l app=crash-demo --previous
# Output: About to crash

kubectl describe pod -l app=crash-demo
# Events: Back-off restarting failed container
```

Fix by changing the command to keep the container running:

```bash
kubectl patch deployment crash-demo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/command", "value": ["sh", "-c", "echo ready && sleep 3600"]}]'
```

### 8. Debug an ImagePullBackOff scenario

Create a Deployment with a non-existent image:

```bash
kubectl run bad-image --image=doesnotexist:latest --restart=Never
kubectl get pods
# Status: ImagePullBackOff or ErrImagePull

kubectl describe pod bad-image
# Events: Failed to pull image "doesnotexist:latest": rpc error: code = NotFound ...
```

Fix by using a valid image:

```bash
kubectl delete pod bad-image
kubectl run good-image --image=nginx:alpine --restart=Never
```

### 9. Debug a Pod stuck in Pending (insufficient resources)

Create a Pod that requests more CPU than the cluster has:

```bash
kubectl run hungry-pod --image=nginx:alpine --restart=Never --requests='cpu=999'
kubectl get pods
# Status: Pending

kubectl describe pod hungry-pod
# Events: 0/3 nodes are available: insufficient cpu
```

Fix by reducing the request:

```bash
kubectl delete pod hungry-pod
kubectl run normal-pod --image=nginx:alpine --restart=Never --requests='cpu=100m'
```

### 10. Install metrics-server and use kubectl top

On minikube:

```bash
minikube addons enable metrics-server
```

On a generic cluster:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Wait for metrics-server to be ready, then:

```bash
kubectl top nodes
kubectl top pods
kubectl top pods --sort-by=cpu
```

### 11. Use kubectl logs --previous to see crash logs

From the CrashLoopBackOff scenario in step 7:

```bash
kubectl logs -l app=crash-demo --previous
```

The current container may have no logs (it crashed before producing any). The `--previous` flag retrieves logs from the last terminated container.

### 12. Use kubectl get events sorted by timestamp

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Events show scheduling decisions, probe failures, and resource errors across the cluster.

---

## Hands-On Lab

### Challenge 1: "Probe Configuration"

**Scenario:** Deploy a Go app (or nginx with custom endpoints) with all three probes: startup, liveness, and readiness. The startup probe should allow 30 seconds for initialization. The liveness probe should check `/healthz` every 10 seconds. The readiness probe should check `/ready` every 5 seconds. Verify all three are working via `kubectl describe`.

**Success criteria:** The Pod runs. `kubectl describe pod` shows startup, liveness, and readiness probes with the specified parameters. All probes report success.

<details>
<summary>Hints</summary>

- Use nginx with a custom ConfigMap that defines locations for `/healthz` and `/ready` returning 200. Or use an image like `hashicorp/http-echo` that serves on a configurable port.
- For nginx: create a config file with `location /healthz { return 200; }` and `location /ready { return 200; }`, mount it via ConfigMap.
- Startup probe: set `failureThreshold` so that `failureThreshold * periodSeconds >= 30` (e.g., 15 failures at 2 seconds).
- Liveness: `periodSeconds: 10`, path `/healthz`.
- Readiness: `periodSeconds: 5`, path `/ready`.
- Run `kubectl describe pod <name>` and inspect the Conditions and Events sections.
</details>

---

### Challenge 2: "Fix CrashLoopBackOff"

**Scenario:** Apply the pre-built broken deployment below. The student must use ONLY kubectl commands to diagnose and fix ALL issues without being told what they are.

**Broken deployment YAML** (save as `broken-app.yaml` and apply it; do not create the ConfigMap beforehand):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        command: ["false"]
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      volumes:
      - name: config
        configMap:
          name: broken-app-config
```

Three issues to find and fix: (1) wrong command that crashes immediately, (2) missing ConfigMap `broken-app-config`, (3) incorrect containerPort (nginx listens on 80).

**Success criteria:** The Deployment runs with 1/1 Ready Pod. No CrashLoopBackOff, no CreateContainerConfigError.

<details>
<summary>Hints</summary>

- Run `kubectl get pods` and `kubectl describe pod` to see Events.
- Check for "configmaps 'broken-app-config' not found" or similar.
- Use `kubectl logs <pod> --previous` if the container crashes.
- Create the missing ConfigMap, or remove the volume mount if the default nginx config is sufficient.
- Fix the containerPort if the app listens on 80.
- Ensure the volume mount path and ConfigMap keys are correct.
</details>

---

### Challenge 3: "Readiness Gate"

**Scenario:** Deploy an application with a readiness probe. Create a Service pointing to it. Demonstrate that when the readiness probe fails, the Pod IP is removed from the Endpoints list. Fix the readiness probe and verify the Pod IP returns to Endpoints.

**Success criteria:** Initially, `kubectl get endpoints` shows the Pod IP. After breaking the readiness probe, Endpoints is empty. After fixing the probe, the Pod IP returns to Endpoints.

<details>
<summary>Hints</summary>

- Deploy nginx with a readiness probe on `/`.
- Create a Service with a selector matching the Pod labels.
- Use `kubectl get endpoints <service-name>` to see the list of Pod IPs.
- Patch the deployment to change the readiness probe path to `/nonexistent` (returns 404).
- Wait for the new Pod to roll out. Check Endpoints again — it should be empty.
- Restore the readiness probe path to `/`.
- Verify the Pod IP returns to Endpoints.
</details>

---

### Challenge 4: "Log Investigation"

**Scenario:** Deploy a multi-container Pod with two containers: `logger` (writes structured logs and runs indefinitely) and `crasher` (crashes intermittently after printing an error message). Using only kubectl, find which container is crashing, get the crash logs using `--previous`, determine the root cause from the logs, and fix the issue.

**Success criteria:** Identify the crashing container. Retrieve the crash logs. Determine the root cause (e.g., missing env var, wrong command, file not found). Fix the Pod so both containers run successfully.

<details>
<summary>Hints</summary>

- Create a Pod with two containers. Use `restartPolicy: Always` (default).
- Logger: `busybox` with `command: ["sh", "-c", "while true; do echo 'log entry'; sleep 5; done"]`.
- Crasher: `busybox` with `command: ["sh", "-c", "echo 'ERROR: config file not found at /etc/config/app.conf'; exit 1"]`.
- Use `kubectl get pods` to see restart count; `kubectl describe pod` to see which container failed.
- Use `kubectl logs <pod> -c crasher --previous` to get crash logs.
- Fix by adding a ConfigMap with the config file and mounting it, or by changing the command to not require the file.
</details>

---

### Challenge 5: "Resource Monitoring"

**Scenario:** Install metrics-server (if not already installed). Deploy a Pod that consumes high CPU (e.g., a tight loop). Use `kubectl top` to identify the resource hog. Set appropriate resource limits to constrain it.

**Success criteria:** metrics-server is running. `kubectl top pods` shows the high-CPU Pod. The Pod has resource limits set. The Pod is throttled or remains within limits.

<details>
<summary>Hints</summary>

- Install metrics-server: `minikube addons enable metrics-server` or apply the metrics-server manifest.
- Deploy a CPU-heavy Pod: `busybox` with `command: ["sh", "-c", "while true; do :; done"]` or use `stress` image.
- Run `kubectl top pods` and `kubectl top pods --sort-by=cpu` to find the consumer.
- Add `resources.limits.cpu: "500m"` to the Pod spec.
- Verify with `kubectl describe pod` that limits are set. The container may be throttled when it exceeds the limit.
</details>

---

## Weekly Speed Drill

Complete these 10 tasks as fast as you can.

1. Get pod status and restart count: `kubectl get pods` (RESTARTS column shows restart count)
2. Describe a pod and find the Events section: `kubectl describe pod <name>`
3. Get logs from a specific container: `kubectl logs <pod> -c <container>`
4. Get previous container logs: `kubectl logs <pod> --previous`
5. Stream logs with -f flag: `kubectl logs <pod> -f`
6. Get all events sorted by timestamp: `kubectl get events --sort-by=.metadata.creationTimestamp`
7. Add an HTTP liveness probe to an existing deployment (using kubectl edit or patch)
8. Check endpoints for a service: `kubectl get endpoints <service>`
9. Run `kubectl top pods` sorted by CPU: `kubectl top pods --sort-by=cpu`
10. Exec into a pod and test if a service is reachable (wget/curl): `kubectl exec -it <pod> -- wget -qO- http://<service>:<port>` or `kubectl exec -it <pod> -- curl -s http://<service>:<port>`

---

## Exam Pitfalls

- **Setting initialDelaySeconds too low** — The probe fires before the app is ready, causing a restart loop. Set it to at least the time your app needs to start listening.

- **Using liveness when readiness is needed** — Liveness restarts the container. Readiness only removes the Pod from the load balancer. If the app is temporarily overloaded, use readiness so it can recover without a restart.

- **Forgetting `--previous` when debugging CrashLoopBackOff** — The current container may not have started yet; its logs are empty. The previous container's logs show why it crashed.

- **Not checking the Events section of `kubectl describe`** — Most answers are there: "Failed to pull image", "Liveness probe failed", "0/3 nodes available: insufficient memory".

- **Wrong probe port** — The probe checks port 80 but the app listens on 8080. The probe fails. Match the port to the container's actual listen port.

- **Confusing exec probe exit code** — Exit code 0 means healthy. Non-zero means unhealthy. Do not invert this.

- **Not installing metrics-server** — `kubectl top` returns "Metrics API not available" without it. Know how to install it.

- **Forgetting that startup probe disables liveness/readiness until it succeeds** — Liveness and readiness do not run until the startup probe succeeds once. Plan failureThreshold and periodSeconds accordingly.

- **Using failureThreshold=1** — Too aggressive. One slow response or transient failure kills the container (liveness) or removes it from Endpoints (readiness). Use at least 2 or 3.

- **Not using `-c <container>` for multi-container pod logs** — `kubectl logs <pod>` fails or returns the wrong container's logs when the Pod has multiple containers. Always specify `-c` when there is more than one container.

---

## Solution Key

### Challenge 1: Probe Configuration

**ConfigMap for nginx with /healthz and /ready:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-probe-config
data:
  default.conf: |
    server {
        listen 80;
        location / {
            return 200 'ok';
        }
        location /healthz {
            return 200 'healthy';
        }
        location /ready {
            return 200 'ready';
        }
    }
```

**Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: probe-lab
  template:
    metadata:
      labels:
        app: probe-lab
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
        startupProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 2
          failureThreshold: 15
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
          failureThreshold: 2
      volumes:
      - name: config
        configMap:
          name: nginx-probe-config
```

**Commands:**

```bash
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
kubectl wait --for=condition=Ready pod -l app=probe-lab --timeout=60s
kubectl describe pod -l app=probe-lab
```

**Verification:** In `kubectl describe pod`, check Conditions (Ready: True) and the probe configuration. Startup allows up to 30 seconds (15 * 2). Liveness checks every 10 seconds. Readiness checks every 5 seconds.

---

### Challenge 2: Fix CrashLoopBackOff

**Diagnosis:**

```bash
kubectl apply -f broken-app.yaml
kubectl get pods -l app=broken-app
# CreateContainerConfigError (ConfigMap missing) or CrashLoopBackOff (after ConfigMap exists)

kubectl describe pod -l app=broken-app
# Events: configmap "broken-app-config" not found
# Or: Back-off restarting failed container

kubectl logs -l app=broken-app --previous
# (empty or minimal - command "false" exits immediately)
```

**Issues:**
1. ConfigMap `broken-app-config` does not exist — Pod cannot start (CreateContainerConfigError).
2. Command is `["false"]` — container exits immediately with code 1 (CrashLoopBackOff once ConfigMap exists).
3. containerPort is 8080 — nginx listens on 80.

**Fix — Create ConfigMap, fix command, fix port:**

```bash
# Create minimal nginx config for the ConfigMap
cat <<'NGINX' > nginx.conf
events {}
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    server {
        listen 80;
        location / { root /usr/share/nginx/html; index index.html; }
    }
}
NGINX
kubectl create configmap broken-app-config --from-file=nginx.conf

# Fix command and port
kubectl patch deployment broken-app --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/command", "value": ["nginx", "-g", "daemon off;"]},
  {"op": "replace", "path": "/spec/template/spec/containers/0/ports/0/containerPort", "value": 80}
]'

kubectl rollout status deployment/broken-app
kubectl get pods -l app=broken-app
# 1/1 Ready
```

**Alternative — Remove volume and use default nginx:**

```bash
kubectl patch deployment broken-app --type='json' -p='[
  {"op": "remove", "path": "/spec/template/spec/containers/0/volumeMounts"},
  {"op": "remove", "path": "/spec/template/spec/volumes"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/command", "value": ["nginx", "-g", "daemon off;"]},
  {"op": "replace", "path": "/spec/template/spec/containers/0/ports/0/containerPort", "value": 80}
]'
```

---

### Challenge 3: Readiness Gate

**Deploy app and Service:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: readiness-demo
  template:
    metadata:
      labels:
        app: readiness-demo
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-demo
spec:
  selector:
    app: readiness-demo
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f deployment.yaml
kubectl wait --for=condition=Ready pod -l app=readiness-demo --timeout=60s
kubectl get endpoints readiness-demo
# Should show one address
```

**Break readiness:**

```bash
kubectl patch deployment readiness-demo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/path", "value": "/nonexistent"}]'
kubectl rollout status deployment/readiness-demo
kubectl get endpoints readiness-demo
# Endpoints list should be empty (no addresses)
```

**Fix readiness:**

```bash
kubectl patch deployment readiness-demo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/path", "value": "/"}]'
kubectl rollout status deployment/readiness-demo
kubectl get endpoints readiness-demo
# Pod IP should return to Endpoints
```

---

### Challenge 4: Log Investigation

**Broken Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-lab
spec:
  containers:
  - name: logger
    image: busybox
    command: ["sh", "-c", "while true; do echo 'log entry'; sleep 5; done"]
  - name: crasher
    image: busybox
    command: ["sh", "-c", "echo 'ERROR: config file not found at /etc/config/app.conf'; exit 1"]
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
# log-lab: 0/2 Ready, one container restarting

kubectl describe pod log-lab
# Events: Back-off restarting failed container "crasher"

kubectl logs log-lab -c crasher --previous
# ERROR: config file not found at /etc/config/app.conf
```

**Fix:** Create ConfigMap and mount it, or change crasher to not require the file:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.conf: "enabled=true"
---
apiVersion: v1
kind: Pod
metadata:
  name: log-lab-fixed
spec:
  containers:
  - name: logger
    image: busybox
    command: ["sh", "-c", "while true; do echo 'log entry'; sleep 5; done"]
  - name: crasher
    image: busybox
    command: ["sh", "-c", "cat /etc/config/app.conf && echo 'Config found' && sleep 3600"]
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
```

```bash
kubectl apply -f fixed-pod.yaml
kubectl wait --for=condition=Ready pod/log-lab-fixed --timeout=60s
kubectl logs log-lab-fixed -c crasher
# enabled=true
# Config found
```

---

### Challenge 5: Resource Monitoring

**Install metrics-server (minikube):**

```bash
minikube addons enable metrics-server
```

**Install metrics-server (generic):**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl wait --for=condition=Ready pod -l k8s-app=metrics-server -n kube-system --timeout=120s
```

**Deploy CPU-heavy Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-hog
spec:
  containers:
  - name: stress
    image: busybox
    command: ["sh", "-c", "while true; do :; done"]
```

```bash
kubectl apply -f cpu-hog.yaml
sleep 30
kubectl top pods
kubectl top pods --sort-by=cpu
# cpu-hog should show high CPU
```

**Add resource limits:**

```bash
kubectl patch pod cpu-hog --type='json' -p='[{"op": "add", "path": "/spec/containers/0/resources", "value": {"limits": {"cpu": "500m"}}}]'
```

Note: Patching a Pod's spec may not work for some fields. If so, delete and recreate:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-hog-limited
spec:
  containers:
  - name: stress
    image: busybox
    command: ["sh", "-c", "while true; do :; done"]
    resources:
      limits:
        cpu: "500m"
```

```bash
kubectl delete pod cpu-hog
kubectl apply -f cpu-hog-limited.yaml
kubectl describe pod cpu-hog-limited
# Limits: cpu: 500m
kubectl top pods
# CPU should be capped or throttled
```
