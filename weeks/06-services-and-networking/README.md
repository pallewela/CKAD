---
title: Week 6 — Services & Networking
permalink: /weeks/06-services-and-networking/
---

# Week 6: Services & Networking

## The Concept

Pods in Kubernetes get their own IP addresses when they start. That IP is unique and routable within the cluster. The problem: Pod IPs are ephemeral. When a Pod restarts, is rescheduled, or is replaced during a rolling update, it gets a new IP. Any client that hard-coded the old Pod IP will break. A Service solves this by providing a stable virtual IP and DNS name that always points to the correct set of Pods, regardless of which nodes they run on or how many times they restart.

A Service is a Kubernetes object that defines a logical set of Pods and a policy to access them. The Service gets a stable ClusterIP (unless it is headless) and a DNS name. Traffic sent to the Service is load-balanced across the matching Pods. The Service does not "run" anything; it is a network abstraction that sits in front of Pods.

Kubernetes offers four Service types. ClusterIP is the default. It assigns an internal IP that is only reachable from inside the cluster. Use ClusterIP for internal communication between frontends and backends, databases, caches, and any component that should not be exposed outside the cluster.

NodePort exposes the Service on a static port in the range 30000–32767 on every node. Any traffic hitting that port on any node is forwarded to the Service, which then forwards to a Pod. NodePort is useful for development and testing when you need to access a Service from your laptop or a VM without setting up a full load balancer. In production, NodePort is rarely used directly; it is often used as a building block for LoadBalancer.

LoadBalancer provisions an external load balancer when running on a cloud provider (AWS, GCP, Azure). The cloud controller creates a real load balancer and assigns an external IP. Use LoadBalancer for production workloads that need external access. On local clusters (minikube, kind, k3s), LoadBalancer Services often stay in Pending because there is no cloud provider to provision the LB.

ExternalName maps the Service to an external DNS name via a CNAME record. Use it when you want Pods to reach an external service (e.g., a database hosted outside the cluster) using a Kubernetes DNS name instead of hard-coding the external hostname.

Services find Pods using label selectors. The Service spec includes a selector such as `app: nginx`. The Service controller continuously watches for Pods whose labels match that selector. Only matching Pods receive traffic. If no Pods match, the Service has no endpoints and traffic goes nowhere. This is the most common cause of "Service not working": the selector does not match the Pod labels.

Endpoints are the list of Pod IPs and ports that a Service routes to. Kubernetes creates an Endpoints object automatically for every Service that has a selector. The Endpoints object is updated in real time as matching Pods come and go. When you run `kubectl get endpoints`, you see the actual Pod IPs. If the list is empty, no Pods match the Service selector.

Kubernetes runs an internal DNS server. Every Service gets a DNS entry: `<service-name>.<namespace>.svc.cluster.local`. From a Pod in the same namespace, you can use just `<service-name>`. From another namespace, use the full name. The cluster domain suffix is `cluster.local` by default.

Port terminology matters. The Service has a `port` (the port clients connect to on the Service) and a `targetPort` (the port on the container that receives traffic). They can differ: for example, Service port 80, targetPort 8080. For NodePort and LoadBalancer, `nodePort` is the port on the node (30000–32767).

A Headless Service is created by setting `clusterIP: None`. It has no virtual IP. DNS for a headless Service returns the individual Pod IPs (A records) instead of a single Service IP. Use headless Services for StatefulSets when clients need to reach specific Pods directly, or for service discovery patterns that require direct Pod addressing.

---

## Beginner Tutorial

### 1. Create a Deployment with 3 nginx replicas

```bash
kubectl create deployment nginx --image=nginx --replicas=3
```

### 2. Expose as ClusterIP

```bash
kubectl expose deployment nginx --port=80
```

### 3. Inspect

```bash
kubectl get svc
kubectl get endpoints
```

### 4. Test from another Pod

```bash
kubectl run tmp --image=busybox --rm -it -- wget -qO- nginx
```

### 5. Expose as NodePort

```bash
kubectl expose deployment nginx --port=80 --type=NodePort --name=nginx-np
```

### 6. Access via NodePort

For minikube:

```bash
minikube service nginx-np
```

For kind or other clusters, use port-forward to a node or:

```bash
kubectl port-forward svc/nginx-np 8080:80
```

### 7. DNS resolution

```bash
kubectl run tmp --image=busybox --rm -it -- nslookup nginx.default.svc.cluster.local
```

### 8. Create a headless Service and observe DNS A records

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

Apply and verify:

```bash
kubectl apply -f nginx-headless.yaml
kubectl run tmp --image=busybox --rm -it -- nslookup nginx-headless.default.svc.cluster.local
```

You should see multiple A records (one per Pod IP).

### 9. Port-forward for local debugging

```bash
kubectl port-forward svc/nginx 8080:80
```

Then `curl localhost:8080` in another terminal. The command runs in the foreground; stop it with Ctrl+C.

### 10. Create a Service YAML from scratch

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-manual
  namespace: default
spec:
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
```

Apply with `kubectl apply -f nginx-manual.yaml`.

---

## Hands-On Lab

### Challenge 1: Frontend-Backend Communication

**Scenario:** Deploy a frontend (nginx) and a backend (httpbin or your Go app). Create ClusterIP Services for both. Exec into the frontend Pod and curl the backend using its DNS name.

**Success criteria:** From inside the frontend Pod, `curl http://backend` (or the backend service name) returns a successful response.

<details>
<summary>Hints</summary>

- Create Deployments for frontend and backend with distinct labels (e.g., `app: frontend`, `app: backend`).
- Expose each Deployment with `kubectl expose` or create Service manifests with matching selectors.
- Use `kubectl exec -it <frontend-pod> -- sh` (or `-- /bin/sh`) to get a shell, then `curl http://backend`.
- Ensure both are in the same namespace so short DNS names work.
</details>

---

### Challenge 2: Service Discovery

**Scenario:** Create Services in two different namespaces (e.g., `ns-a` and `ns-b`). From a Pod in namespace A, reach a Service in namespace B using the full DNS name `<svc>.<ns>.svc.cluster.local`.

**Success criteria:** A Pod in `ns-a` can successfully curl a Service in `ns-b` using the FQDN.

<details>
<summary>Hints</summary>

- Create namespaces: `kubectl create namespace ns-a` and `kubectl create namespace ns-b`.
- Deploy a simple app (e.g., nginx) in `ns-b` and expose it as a Service.
- Run a temporary Pod in `ns-a`: `kubectl run tmp -n ns-a --image=busybox -it --rm -- sh`.
- From inside that Pod: `wget -qO- http://<service-name>.ns-b.svc.cluster.local`.
</details>

---

### Challenge 3: Debug a Broken Service

**Scenario:** You are given a Deployment and a Service where the labels do not match. The Service selector does not match the Pod labels. Diagnose why `kubectl get endpoints` shows no endpoints. Fix the labels so traffic flows.

**Success criteria:** After the fix, `kubectl get endpoints` shows Pod IPs, and traffic to the Service reaches the Pods.

<details>
<summary>Hints</summary>

- Inspect the Service selector: `kubectl get svc <name> -o yaml` and check the `spec.selector` field.
- Inspect the Pod labels: `kubectl get pods --show-labels` or `kubectl get pod <name> -o yaml`.
- Compare: every key-value pair in the selector must exist on the Pod.
- Fix by either updating the Service selector to match the Pod labels, or updating the Deployment's Pod template to match the Service selector.
</details>

---

### Challenge 4: Headless Service

**Scenario:** Create a StatefulSet with a headless Service. Verify that DNS returns individual Pod IPs (A records), not a single ClusterIP.

**Success criteria:** `nslookup <headless-svc>` returns multiple A records, one per StatefulSet Pod.

<details>
<summary>Hints</summary>

- Create a Service with `clusterIP: None` and a selector matching the StatefulSet Pods.
- StatefulSet Pods get stable names: `<statefulset-name>-0`, `<statefulset-name>-1`, etc.
- The headless Service DNS returns A records for each Pod. Individual Pod DNS: `<pod-name>.<headless-svc>.<ns>.svc.cluster.local`.
- Use `kubectl run tmp --image=busybox --rm -it -- nslookup <headless-svc>` to verify.
</details>

---

## Weekly Speed Drill

1. Create a deployment `web` with 2 replicas of nginx
2. Expose it as ClusterIP on port 80
3. Get the service's ClusterIP
4. Expose it as NodePort
5. Get the assigned NodePort number using jsonpath
6. Create a temporary pod and curl the service by name
7. Get endpoints for the service
8. Port-forward the service to localhost:9090
9. Delete the service but keep the deployment
10. Recreate the service using `kubectl expose`

**Commands (for self-check):**

```bash
# 1
kubectl create deployment web --image=nginx --replicas=2
# 2
kubectl expose deployment web --port=80
# 3
kubectl get svc web -o jsonpath='{.spec.clusterIP}'
# 4
kubectl expose deployment web --port=80 --type=NodePort --name=web-np
# 5
kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}'
# 6
kubectl run tmp --image=busybox --rm -it -- wget -qO- web
# 7
kubectl get endpoints web
# 8
kubectl port-forward svc/web 9090:80
# 9
kubectl delete svc web
# 10
kubectl expose deployment web --port=80
```

---

## Exam Pitfalls

- **Service selector not matching Pod labels** — The most common networking bug. If the selector does not match any Pod labels, the Service has no endpoints. Traffic goes nowhere. Always run `kubectl get ep <svc>` when debugging.

- **Confusing port vs targetPort vs nodePort** — `port` is the Service port (what clients hit). `targetPort` is the container port. `nodePort` is the host port for NodePort/LoadBalancer (30000–32767).

- **Forgetting that Services use label selectors** — Services do not reference Pods by name. They match Pods by labels. Ensure the selector in the Service spec matches the labels on the Pod template.

- **Not knowing DNS format** — Full DNS: `<svc>.<ns>.svc.cluster.local`. Short form (same namespace): `<svc>`.

- **Using LoadBalancer in local clusters** — On minikube, kind, or k3s, LoadBalancer Services stay Pending because there is no cloud provider. Use NodePort or port-forward instead.

- **Forgetting `--type=NodePort` when exposing** — `kubectl expose` defaults to ClusterIP. You must add `--type=NodePort` to get a node port.

- **Not checking endpoints when traffic does not reach Pods** — If `kubectl get ep <svc>` shows no addresses, the selector is wrong or no Pods are ready.

- **Port-forward only works while the command runs** — `kubectl port-forward` is not persistent. It runs in the foreground and stops when you terminate it. Do not rely on it for production access.

---

## Solution Key

### Challenge 1: Frontend-Backend Communication

**Deployments:**

```yaml
# frontend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
---
# backend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: httpbin
        image: kennethreitz/httpbin
        ports:
        - containerPort: 80
```

**Services:**

```bash
kubectl expose deployment frontend --port=80 --name=frontend
kubectl expose deployment backend --port=80 --name=backend
```

**Test:**

```bash
kubectl exec -it $(kubectl get pod -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- curl -s http://backend/get
```

**Expected output:** JSON response from httpbin (e.g., `{"args": {}, "headers": {...}, ...}`).

---

### Challenge 2: Service Discovery

**Setup:**

```bash
kubectl create namespace ns-a
kubectl create namespace ns-b
kubectl create deployment nginx -n ns-b --image=nginx
kubectl expose deployment nginx -n ns-b --port=80 --name=web
```

**Test from Pod in ns-a:**

```bash
kubectl run tmp -n ns-a --image=busybox --rm -it -- wget -qO- http://web.ns-b.svc.cluster.local
```

**Expected output:** HTML from nginx default page.

---

### Challenge 3: Debug a Broken Service

**Broken setup (selector mismatch):**

```yaml
# Deployment has app: nginx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
---
# Service selects app: web (does not match)
apiVersion: v1
kind: Service
metadata:
  name: broken-svc
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

**Diagnosis:**

```bash
kubectl get endpoints broken-svc
# No addresses

kubectl get svc broken-svc -o yaml
# spec.selector: app: web

kubectl get pods --show-labels
# Pods have app: nginx
```

**Fix:** Update the Service selector to match the Pod labels:

```bash
kubectl patch svc broken-svc -p '{"spec":{"selector":{"app":"nginx"}}}'
```

Or edit: `kubectl edit svc broken-svc` and change `app: web` to `app: nginx`.

**Verify:**

```bash
kubectl get endpoints broken-svc
# Shows 2 addresses
```

---

### Challenge 4: Headless Service

**StatefulSet and Headless Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx-headless
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

**Apply and verify:**

```bash
kubectl apply -f statefulset-headless.yaml
kubectl wait --for=condition=ready pod -l app=nginx --timeout=60s
kubectl run tmp --image=busybox --rm -it -- nslookup nginx-headless.default.svc.cluster.local
```

**Expected output:** Three A records, one for each Pod (e.g., `web-0`, `web-1`, `web-2` with their respective IPs).
