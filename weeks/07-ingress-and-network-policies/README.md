# Week 7: Ingress & Network Policies

## The Concept

**Ingress** is a Kubernetes resource that manages external HTTP and HTTPS access to Services inside your cluster. Instead of exposing each Service with its own LoadBalancer or NodePort, you define routing rules in one place. Think of Ingress as a smart router that reads incoming URLs (host and path) and sends traffic to the correct backend Service. For example, requests to `api.example.com` might go to your API Service, while `www.example.com` goes to your web frontend.

An **Ingress Controller** is the software that actually implements these routing rules. Kubernetes does not include an Ingress Controller by default. You must install one in your cluster. Popular options include NGINX Ingress Controller, Traefik, and HAProxy. The Ingress resource is just a specification; the controller watches for Ingress objects and configures the underlying load balancer or reverse proxy accordingly.

**Ingress rules** define how traffic is routed. Host-based routing sends traffic based on the `Host` header: `foo.example.com` goes to Service A, `bar.example.com` to Service B. Path-based routing uses the URL path: `/api` might route to Service B, `/web` to Service C. You can combine both: `api.example.com/users` and `api.example.com/products` could route to different Services on the same host.

**TLS termination** means the Ingress Controller handles HTTPS decryption. Clients connect over HTTPS; the controller terminates TLS using a certificate and forwards plain HTTP to the backend Services. You store the certificate and private key in a Kubernetes Secret of type `kubernetes.io/tls` and reference it in the Ingress spec.

**IngressClass** is a resource that specifies which Ingress Controller handles which Ingress objects. When you have multiple controllers (e.g., NGINX and Traefik), you use `ingressClassName` in your Ingress to bind it to the correct one. Most clusters have a default IngressClass named `nginx`.

**NetworkPolicy** is a firewall for Pods. It controls which Pods can communicate with which other Pods (and with external endpoints). Unlike Ingress (the HTTP routing object), NetworkPolicy uses the term "ingress" to mean incoming traffic to a Pod, and "egress" for outgoing traffic. This naming overlap is confusing but important to distinguish.

A NetworkPolicy has a **podSelector** that defines which Pods the policy applies to. An empty selector `{}` means all Pods in the namespace. The policy then defines **ingress rules** (who can send traffic to those Pods) and **egress rules** (where those Pods can send traffic). Rules use label selectors to specify allowed peers.

**Default deny** is a common pattern: create a NetworkPolicy with `podSelector: {}` and `policyTypes: [Ingress, Egress]` but no rules. This blocks all traffic. You then add allow policies for specific traffic. Note: NetworkPolicies are additive. If no policy selects a Pod, all traffic is allowed. Once a policy selects a Pod, only explicitly allowed traffic is permitted.

A critical caveat: **NetworkPolicies only work if your cluster has a CNI (Container Network Interface) plugin that supports them.** Calico and Cilium support NetworkPolicy. The default CNI in kind is kindnet, which does not support NetworkPolicy. For testing, you must install Calico or another compatible CNI.

## Beginner Tutorial

### 1. Install NGINX Ingress Controller

For kind:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod -l app.kubernetes.io/component=controller --timeout=120s
```

For minikube:

```bash
minikube addons enable ingress
```

### 2. Deploy two apps

Create `app1` (nginx) and `app2` (httpbin):

```bash
kubectl create deployment app1 --image=nginx
kubectl create deployment app2 --image=kennethreitz/httpbin
```

### 3. Expose both as ClusterIP Services

```bash
kubectl expose deployment app1 --port=80
kubectl expose deployment app2 --port=80
```

### 4. Create an Ingress with path-based routing

Save as `ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: demo.local
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

### 5. Test the Ingress

Get the Ingress controller address. For kind, use the node port or port-forward:

```bash
kubectl get svc -n ingress-nginx
# Or port-forward: kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

Test:

```bash
curl -H "Host: demo.local" http://localhost:8080/app1
curl -H "Host: demo.local" http://localhost:8080/app2/get
```

### 6. Create a default-deny NetworkPolicy

Save as `deny-all.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Apply:

```bash
kubectl apply -f deny-all.yaml
```

Note: This blocks all traffic. On kind with default kindnet, it will have no effect until you install Calico.

### 7. Create an allow policy for specific pods

Example: allow traffic from Pods with label `role=frontend` to Pods with label `role=backend`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
```

### 8. Test connectivity

From an allowed Pod (with `role=frontend`), connectivity to `role=backend` should work. From a Pod without the label, it should be blocked (when using a compatible CNI).

## Hands-On Lab

### Challenge 1: Path-Based Routing

Deploy three microservices: `users`, `products`, and `orders`. Create one Ingress that routes `/users`, `/products`, and `/orders` to the correct Services. All three should be reachable via a single host (e.g., `shop.local`).

**Success criteria:** `curl -H "Host: shop.local" http://<ingress>/users`, `/products`, and `/orders` each return content from the correct service.

<details>
<summary>Hints</summary>

- Use three Deployments and three ClusterIP Services.
- Use `pathType: Prefix` for each path.
- The `rewrite-target` annotation may be needed if your apps serve at `/` but you route at `/users`, etc.
</details>

### Challenge 2: Secure the Database

Create namespace `secure-ns`. Deploy a database Pod (e.g., postgres or a simple TCP echo) and a web Pod. Write a NetworkPolicy: default deny all traffic, then allow only the web Pod to reach the db Pod on port 5432. Verify with a temporary Pod: connectivity from an authorized Pod works; from an unauthorized Pod it is blocked.

**Success criteria:** Web Pod can connect to db on 5432. Unauthorized Pod cannot.

<details>
<summary>Hints</summary>

- Use `podSelector: {}` with no rules for default deny.
- Add a second NetworkPolicy with `podSelector` for the db, allowing ingress from the web Pod's labels on port 5432.
- Remember to allow DNS egress if Pods need to resolve names.
- Use `kubectl run` to create a test Pod and `kubectl exec` to run `nc` or `curl` for testing.
</details>

### Challenge 3: Egress Control

Create a NetworkPolicy that allows Pods in a namespace to connect to DNS (UDP port 53) and one specific external service (e.g., `api.github.com` on port 443), but blocks all other outbound traffic.

**Success criteria:** Pods can resolve DNS and reach the allowed external host. Other outbound traffic is blocked.

<details>
<summary>Hints</summary>

- Default deny egress with `podSelector: {}` and `policyTypes: [Egress]`.
- Allow egress to `kube-system` namespace for CoreDNS (or use `namespaceSelector` and `podSelector` for kube-dns).
- Allow egress to `0.0.0.0/0` (or the specific IP) on port 443 for the external service.
- Use `ipBlock` for external IPs.
</details>

### Challenge 4: TLS Ingress

Create a self-signed TLS certificate, store it as a Kubernetes Secret of type `kubernetes.io/tls`, and configure an Ingress to use HTTPS with that certificate.

**Success criteria:** `curl -k https://secure.local` returns a response without certificate errors (aside from self-signed warning).

<details>
<summary>Hints</summary>

- Use `openssl req -x509 -nodes -days 365 -newkey rsa:2048` to generate cert and key.
- Create Secret: `kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key`
- In Ingress spec, add `tls:` section with `hosts` and `secretName`.
</details>

## Weekly Speed Drill

Complete these 10 tasks as quickly as possible:

1. Install an Ingress Controller (NGINX) in your cluster.
2. Create an Ingress for host `test.local` routing to service `web` on port 80.
3. List all Ingresses in the cluster.
4. Describe an Ingress to see its rules and backends.
5. Create a default-deny ingress NetworkPolicy (blocks all incoming traffic to Pods in the namespace).
6. Create a NetworkPolicy allowing ingress traffic from Pods with label `role=frontend`.
7. Create a default-deny egress NetworkPolicy (blocks all outgoing traffic).
8. Add an egress rule to allow DNS (UDP port 53) to the kube-system namespace.
9. Test connectivity with a temporary busybox Pod: `kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh`.
10. Delete all NetworkPolicies in a namespace.

## Exam Pitfalls

- **Forgetting to install an Ingress Controller.** Ingress objects are just specifications. Without a controller, they do nothing. Always install one (e.g., NGINX) before creating Ingress resources.

- **Confusing Ingress "ingress" with NetworkPolicy "ingress."** Ingress is the Kubernetes object for HTTP routing. In NetworkPolicy, "ingress" means incoming traffic direction. They are unrelated concepts with the same word.

- **NetworkPolicies are additive, not subtractive.** If no NetworkPolicy selects a Pod, all traffic is allowed. Once a policy selects a Pod, only explicitly allowed traffic is permitted. You cannot "subtract" from a default allow.

- **Forgetting DNS egress in NetworkPolicy.** When you default-deny egress, Pods cannot resolve DNS names unless you explicitly allow egress to CoreDNS (typically in `kube-system` on UDP port 53).

- **pathType must be specified.** Valid values are `Prefix`, `Exact`, or `ImplementationSpecific`. Omitting it can cause validation errors.

- **Not specifying ingressClassName.** If multiple IngressClasses exist, your Ingress might not bind to any controller. Always set `ingressClassName` to match your controller.

- **Writing NetworkPolicy podSelector: {}** applies to ALL Pods in the namespace. Use this for default-deny; use specific labels when you want to target a subset.

- **Forgetting that NetworkPolicy requires a compatible CNI.** Flannel and kindnet do not support NetworkPolicy. Use Calico or Cilium for testing.

## Solution Key

### Challenge 1: Path-Based Routing

```yaml
# users-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users
spec:
  replicas: 1
  selector:
    matchLabels:
      app: users
  template:
    metadata:
      labels:
        app: users
    spec:
      containers:
      - name: users
        image: nginx
---
# products-deploy.yaml - same structure with app: products
# orders-deploy.yaml - same structure with app: orders
```

```bash
kubectl create deployment users --image=nginx
kubectl create deployment products --image=nginx
kubectl create deployment orders --image=nginx
kubectl expose deployment users --port=80
kubectl expose deployment products --port=80
kubectl expose deployment orders --port=80
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: shop.local
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: users
            port:
              number: 80
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: products
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: orders
            port:
              number: 80
```

```bash
curl -H "Host: shop.local" http://localhost:8080/users
curl -H "Host: shop.local" http://localhost:8080/products
curl -H "Host: shop.local" http://localhost:8080/orders
```

### Challenge 2: Secure the Database

```bash
kubectl create namespace secure-ns
kubectl create deployment db --image=postgres:15-alpine -n secure-ns --env="POSTGRES_PASSWORD=test"
kubectl create deployment web --image=nginx -n secure-ns
kubectl expose deployment db -n secure-ns --port=5432
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-db
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 5432
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-egress-to-db
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
```

Test from web (authorized):

```bash
kubectl exec -n secure-ns deploy/web -- nc -zv db 5432
# Connection to db 5432 port [tcp/postgresql] succeeded!
```

Test from unauthorized Pod:

```bash
kubectl run test-unauth --image=busybox -n secure-ns --restart=Never -- sleep 3600
kubectl exec -n secure-ns test-unauth -- nc -zv db 5432
# timeout or connection refused (blocked)
```

### Challenge 3: Egress Control

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-github
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
```

Note: The second policy must allow both DNS and HTTPS. Alternatively, use a single policy with multiple egress rules. Verify: `kubectl run test --image=busybox --rm -it --restart=Never -- nslookup api.github.com` and `wget -qO- https://api.github.com` should work; `wget http://example.com` should fail.

### Challenge 4: TLS Ingress

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=secure.local"
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.local
    secretName: tls-secret
  rules:
  - host: secure.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

```bash
curl -k -H "Host: secure.local" https://localhost:443/
# -k skips certificate verification (needed for self-signed)
```
