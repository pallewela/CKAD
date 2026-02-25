# Week 10: Helm, Custom Resources & Security Deep Dive

## The Concept

This week covers the highest-weight domain on the CKA exam (25%): Application Environment, Configuration and Security. You will learn how to package and deploy applications with Helm, customize manifests with Kustomize, extend Kubernetes with Custom Resource Definitions, and secure workloads with RBAC and SecurityContext. Mastery of these topics is essential for passing the exam.

**Helm: A Package Manager for Kubernetes**

Helm is a package manager for Kubernetes, analogous to apt for Ubuntu or brew for macOS. Instead of manually writing and applying dozens of YAML files, you install pre-packaged applications called charts.

A **Chart** is a package of YAML templates plus default values. Think of it as a recipe. A chart contains all the Kubernetes manifests needed to run an application (Deployments, Services, ConfigMaps, Secrets, etc.), but with placeholders that get filled in at install time. The templates use Go templating syntax to inject values.

A **Release** is a specific installation of a chart in your cluster. If you install the same chart twice with different names, you get two releases. Like a dish made from the recipe: the recipe is the chart, the dish on your plate is the release. Each release has a revision history; upgrades create new revisions, and you can roll back to any previous revision.

A **Repository** is where charts are stored and published. It is like a cookbook library. Popular repositories include Bitnami, the official Helm hub, and your own private registries. You add a repo with `helm repo add`, then run `helm repo update` to fetch the latest chart metadata before installing.

**Values** are the knobs you turn to customize a chart. Examples include replica count, image tag, resource limits, and environment variables. You can override values via `--set key=value` at the command line, via `-f values.yaml` for a file, or both. Values flow into the templates to produce the final manifests.

**Key Helm commands**: `helm install` creates a new release; `helm upgrade` updates an existing release (or installs if it does not exist); `helm rollback` reverts to a previous revision; `helm uninstall` removes a release; `helm list` shows all releases; `helm template` renders the chart to raw YAML without installing (essential for debugging and previewing); `helm show values` displays the default values for a chart.

**Kustomize: Declarative YAML Customization**

Kustomize is an alternative to Helm for customizing YAML without templating. It uses a base plus overlays pattern.

The **Base** is the original set of YAML files (Deployments, Services, etc.) that define your application in its default form. It is the source of truth for the structure of your resources.

An **Overlay** is a set of modifications applied on top of the base. For example, a staging overlay might change replica count to 3 and add a label `env=staging`. A production overlay might set 5 replicas, add resource limits, and use a different image tag. You can have many overlays for different environments.

The **kustomization.yaml** is the manifest that describes what to customize. It lists the resources (or bases), patches, labels, annotations, replicas, and other transformations. Kustomize reads this file to produce the final, customized YAML. You must list every resource you want to include in the `resources` field.

Kustomize is built into kubectl. You apply a kustomization with `kubectl apply -k <path-to-overlay>`.

**Custom Resource Definitions (CRDs)**

Kubernetes ships with built-in resources: Pods, Deployments, Services, ConfigMaps, and so on. CRDs let you define your own resource types.

A **CRD (Custom Resource Definition)** defines the schema for your custom resource. It specifies the API group, kind, version, and the structure of the spec (and optionally status) fields. In `apiextensions.k8s.io/v1`, you must provide an `openAPIV3Schema` for validation. Once you apply a CRD, the Kubernetes API server accepts instances of that type and validates them against the schema.

A **Custom Resource (CR)** is an instance of a CRD. For example, if you define a CRD called "Widget", you can create Widget objects in your cluster with `kubectl apply -f widget.yaml`. You list them with `kubectl get widgets` (using the plural from the CRD's `names.plural`).

**Operators** are controllers that watch CRs and automate tasks. For example, a Database operator might watch "Database" CRs and create StatefulSets, Services, and PersistentVolumes when you create a Database resource. Operators extend Kubernetes with domain-specific logic.

**Security: RBAC, SecurityContext, and Admission**

Security is critical for the exam. You must understand RBAC, SecurityContext, and admission controllers.

**RBAC (Role-Based Access Control)** governs who can do what in the cluster.

A **Role** defines permissions within a single namespace. It specifies which verbs (get, list, create, update, delete, watch, patch) are allowed on which resources (pods, services, deployments, etc.). For core resources (pods, services, configmaps, secrets), the `apiGroups` field is `""` (empty string), not "core". For Deployments, use `apiGroups: ["apps"]`.

A **ClusterRole** defines permissions cluster-wide or for cluster-scoped resources (nodes, PersistentVolumes, namespaces). It can also be reused across namespaces when bound via RoleBindings.

A **RoleBinding** assigns a Role to a user, group, or ServiceAccount within a namespace. It links "who" to "what permissions" in that namespace. You must specify the correct namespace in the RoleBinding metadata and in the subject.

A **ClusterRoleBinding** assigns a ClusterRole to a subject cluster-wide.

**SecurityContext** controls how containers and pods run from a security perspective.

At the **Pod level**: `runAsUser`, `runAsGroup`, `fsGroup`, `runAsNonRoot` control who the containers run as and ownership of volumes. `fsGroup` sets the group ownership of mounted volumes.

At the **Container level**: `capabilities` (add/drop Linux capabilities like NET_BIND_SERVICE), `readOnlyRootFilesystem`, and `allowPrivilegeEscalation` provide fine-grained control per container. Capabilities are container-level only, not pod-level. `readOnlyRootFilesystem: true` requires writable paths (e.g., emptyDir for /tmp) for apps that need to write; otherwise the container may crash.

**Admission Controllers** intercept API requests before they are persisted. A **ValidatingWebhookConfiguration** sends requests to an external service; if the service rejects, the request is denied. A **MutatingWebhookConfiguration** sends requests to an external service; the service can modify the object before it is stored.

---

## Beginner Tutorial

Follow these steps to build hands-on familiarity with Helm, Kustomize, CRDs, and RBAC.

1. Install Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

2. Add Bitnami repo:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update
```

3. Search for charts:

```bash
helm search repo nginx
```

4. Show chart values:

```bash
helm show values bitnami/nginx | head -50
```

5. Install a release:

```bash
helm install my-nginx bitnami/nginx --set replicaCount=2
```

6. List releases:

```bash
helm list
```

7. Upgrade:

```bash
helm upgrade my-nginx bitnami/nginx --set replicaCount=4
```

8. Rollback:

```bash
helm rollback my-nginx 1
```

9. Template (preview YAML):

```bash
helm template my-nginx bitnami/nginx --set replicaCount=2
```

10. Uninstall:

```bash
helm uninstall my-nginx
```

11. Kustomize — create a base directory with a Deployment YAML and kustomization.yaml, create a staging overlay that changes replica count.

Create the directory structure:

```bash
mkdir -p kustomize-demo/base kustomize-demo/overlays/staging
```

Create `kustomize-demo/base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
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
        image: nginx:1.21
```

Create `kustomize-demo/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
```

Create `kustomize-demo/overlays/staging/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
replicas:
- name: nginx
  count: 3
commonLabels:
  env: staging
```

12. Apply with `kubectl apply -k overlays/staging/`:

```bash
kubectl apply -k kustomize-demo/overlays/staging/
```

Verify:

```bash
kubectl get deployments -l env=staging
```

13. Create a CRD:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                color:
                  type: string
                size:
                  type: integer
  scope: Namespaced
  names:
    plural: widgets
    singular: widget
    kind: Widget
```

Save as `crd-widgets.yaml` and apply:

```bash
kubectl apply -f crd-widgets.yaml
```

14. Create a custom resource instance:

```yaml
apiVersion: example.com/v1
kind: Widget
metadata:
  name: red-widget
spec:
  color: red
  size: 10
```

Save as `widget-red.yaml` and apply:

```bash
kubectl apply -f widget-red.yaml
kubectl get widgets
```

15. Create a Role and RoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: mysa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

First create the ServiceAccount:

```bash
kubectl create serviceaccount mysa
```

Apply the Role and RoleBinding:

```bash
kubectl apply -f rbac-pod-reader.yaml
```

16. Test with `kubectl auth can-i list pods --as=system:serviceaccount:default:mysa`:

```bash
kubectl auth can-i list pods --as=system:serviceaccount:default:mysa
```

Expected output: `yes`

```bash
kubectl auth can-i delete pods --as=system:serviceaccount:default:mysa
```

Expected output: `no`

17. SecurityContext: create a Pod that cannot run as root:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
  containers:
  - name: nginx
    image: nginx:1.21
```

Apply and verify:

```bash
kubectl apply -f pod-nonroot.yaml
kubectl exec nonroot-pod -- id
```

Expected: `uid=1000 gid=3000` (not root).

---

## Hands-On Lab

### Challenge 1: Helm Lifecycle

**Scenario**: Install an nginx chart with replicas=2. Upgrade to replicas=4. Verify. Roll back to revision 1. Verify replicas=2 again.

**Success criteria**:
- Release installed with 2 replicas
- After upgrade, 4 replicas running
- After rollback, 2 replicas running

<details>
<summary>Hints</summary>

- Use `helm install`, `helm upgrade`, `helm rollback`
- `helm list` shows revisions; `helm history <release>` shows revision numbers
- Use `kubectl get pods` or `kubectl get deployment` to verify replica count
</details>

---

### Challenge 2: Kustomize Environments

**Scenario**: Create a base Deployment (nginx, 1 replica). Create two overlays: staging (3 replicas, label env=staging) and production (5 replicas, label env=prod, resource limits). Apply each and verify.

**Success criteria**:
- Base has 1 replica
- Staging overlay produces 3 replicas with `env=staging`
- Production overlay produces 5 replicas with `env=prod` and resource limits

<details>
<summary>Hints</summary>

- `kustomization.yaml` must list `resources` (or `bases`) and can use `replicas`, `commonLabels`, `patches`
- Use `kubectl apply -k overlays/staging` and `kubectl apply -k overlays/production`
- Use JSON patch or strategic merge for resource limits
</details>

---

### Challenge 3: Custom Resource

**Scenario**: Define a CRD called "Backup" with fields: database (string), schedule (string), retention (integer). Create 2 Backup resources. List them with kubectl get backups.

**Success criteria**:
- CRD applied successfully
- Two Backup CRs created
- `kubectl get backups` lists both

<details>
<summary>Hints</summary>

- CRD `names.plural` determines `kubectl get <plural>`
- Use `openAPIV3Schema` under `schema` for validation
- `scope: Namespaced` for namespace-scoped resources
</details>

---

### Challenge 4: RBAC Lockdown

**Scenario**: Create a ServiceAccount `deployer`. Grant it permissions to create/update/delete Deployments and Services only (not Pods, Secrets, etc.). Verify with `kubectl auth can-i`.

**Success criteria**:
- `kubectl auth can-i create deployments --as=system:serviceaccount:default:deployer` returns `yes`
- `kubectl auth can-i delete pods --as=system:serviceaccount:default:deployer` returns `no`

<details>
<summary>Hints</summary>

- Deployments and Services are in `apiGroups: ["apps"]` and `apiGroups: [""]` respectively
- Use `verbs: ["create", "update", "delete", "patch", "get", "list", "watch"]` for full CRUD
- RoleBinding must reference the correct namespace for the ServiceAccount
</details>

---

### Challenge 5: Security Hardening

**Scenario**: Deploy a Pod that: runs as user 1000, drops ALL capabilities, adds only NET_BIND_SERVICE, has readOnlyRootFilesystem, and disallows privilege escalation. Prove it works.

**Success criteria**:
- Pod runs successfully
- `kubectl exec` shows `uid=1000`
- Capabilities and filesystem settings applied (pod does not crash; nginx may need writable paths for /tmp)

<details>
<summary>Hints</summary>

- Use `securityContext` at pod and container level
- `capabilities.drop: ["ALL"]` and `capabilities.add: ["NET_BIND_SERVICE"]`
- `readOnlyRootFilesystem: true` may require an `emptyDir` mount at `/tmp` or `/var/cache/nginx` for nginx
- Use `allowPrivilegeEscalation: false`
</details>

---

## Weekly Speed Drill

Complete these 10 tasks under time pressure to build exam-ready speed:

1. Install a Helm chart
2. List Helm releases
3. Upgrade a release with --set
4. Rollback to previous revision
5. Use `helm template` to render YAML
6. Create a Role allowing list/get on pods
7. Create a RoleBinding for a ServiceAccount
8. Check `kubectl auth can-i delete pods --as=system:serviceaccount:default:mysa`
9. Apply a Kustomize overlay with `kubectl apply -k`
10. Create a CRD and a custom resource

---

## Exam Pitfalls

- **Helm**: forgetting `helm repo update` before install (stale chart versions)
- **Helm**: not knowing `helm template` for previewing generated YAML
- **RBAC**: apiGroups for core resources is "" (empty string), not "core"
- **RBAC**: forgetting to specify the correct namespace in RoleBinding
- **RBAC**: mixing up Role (namespaced) vs ClusterRole (cluster-wide)
- **Kustomize**: forgetting to list resources in kustomization.yaml
- **CRDs**: forgetting openAPIV3Schema validation (required in v1)
- **SecurityContext**: capabilities are container-level, not pod-level
- **SecurityContext**: readOnlyRootFilesystem requires writable paths via emptyDir/tmpfs
- Not testing RBAC with `kubectl auth can-i` before moving on

---

## Solution Key

### Challenge 1: Helm Lifecycle

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-nginx bitnami/nginx --set replicaCount=2
kubectl get deployment -l app.kubernetes.io/name=nginx
# Verify replicas: 2

helm upgrade my-nginx bitnami/nginx --set replicaCount=4
kubectl get deployment -l app.kubernetes.io/name=nginx
# Verify replicas: 4

helm rollback my-nginx 1
kubectl get deployment -l app.kubernetes.io/name=nginx
# Verify replicas: 2
```

---

### Challenge 2: Kustomize Environments

**base/deployment.yaml**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
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
        image: nginx:1.21
```

**base/kustomization.yaml**:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
```

**overlays/staging/kustomization.yaml**:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
replicas:
- name: nginx
  count: 3
commonLabels:
  env: staging
```

**overlays/production/kustomization.yaml**:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
replicas:
- name: nginx
  count: 5
commonLabels:
  env: prod
patches:
- target:
    kind: Deployment
    name: nginx
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/resources
      value:
        limits:
          memory: "256Mi"
          cpu: "200m"
        requests:
          memory: "128Mi"
          cpu: "100m"
```

```bash
kubectl apply -k overlays/staging/
kubectl get deployment -l env=staging
# 3 replicas

kubectl apply -k overlays/production/
kubectl get deployment -l env=prod
# 5 replicas, check resources
```

---

### Challenge 3: Custom Resource

**crd-backup.yaml**:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                database:
                  type: string
                schedule:
                  type: string
                retention:
                  type: integer
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    kind: Backup
```

**backup1.yaml**:

```yaml
apiVersion: example.com/v1
kind: Backup
metadata:
  name: db-daily
spec:
  database: postgres
  schedule: "0 2 * * *"
  retention: 7
```

**backup2.yaml**:

```yaml
apiVersion: example.com/v1
kind: Backup
metadata:
  name: db-weekly
spec:
  database: mysql
  schedule: "0 3 * * 0"
  retention: 30
```

```bash
kubectl apply -f crd-backup.yaml
kubectl apply -f backup1.yaml
kubectl apply -f backup2.yaml
kubectl get backups
```

Expected output: two Backup resources listed.

---

### Challenge 4: RBAC Lockdown

**rbac-deployer.yaml**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: default
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "update", "delete", "patch", "get", "list", "watch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["create", "update", "delete", "patch", "get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: deployer
  namespace: default
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create serviceaccount deployer
kubectl apply -f rbac-deployer.yaml
kubectl auth can-i create deployments --as=system:serviceaccount:default:deployer
# yes
kubectl auth can-i delete pods --as=system:serviceaccount:default:deployer
# no
```

---

### Challenge 5: Security Hardening

**pod-hardened.yaml**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    runAsNonRoot: true
  containers:
  - name: nginx
    image: nginx:1.21
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

```bash
kubectl apply -f pod-hardened.yaml
kubectl wait --for=condition=Ready pod/hardened-pod --timeout=60s
kubectl exec hardened-pod -- id
# uid=1000 gid=1000
kubectl exec hardened-pod -- cat /proc/self/status | grep Cap
# CapEff and CapBnd show NET_BIND_SERVICE only
```

Note: Some nginx images may need additional writable paths. If the pod fails, add more emptyDir mounts for `/var/run` or switch to a minimal image like `busybox` for verification.
