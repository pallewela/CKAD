# Week 12: Final Mock Exams & Exam Day Preparation

## The Concept

The CKAD exam is a performance-based assessment delivered through PSI Bridge. Understanding the logistics and environment before exam day eliminates surprises and lets you focus entirely on the tasks. This week is about simulating exam conditions, reinforcing high-value skills, and building the confidence to perform under pressure.

**Exam day logistics.** Arrive early for check-in. You will need a government-issued photo ID that matches your registration. The proctor will verify your identity and workspace. Expect a clean desk policy: no phones, notes, books, or second monitors. Your workspace must be clear except for your computer, keyboard, mouse, and a transparent water bottle. The webcam and screen share must stay on for the entire exam. Any violation can result in exam invalidation.

**Allowed resources.** During the exam you may use only two sites: kubernetes.io/docs and kubernetes.io/blog. No other tabs, search engines, or external documentation. Bookmark these URLs before the exam. The PSI Bridge environment provides an Ubuntu terminal and a browser; you will work with one monitor. Familiarize yourself with this setup beforehand. Practice with a single terminal and a single docs tab open to simulate the real constraints.

**Scoring and results.** You need 66% to pass. Results are typically available within 24 hours via email. If you fail, one free retake is included with your exam purchase. Use the gap between attempts to focus on weak domains and practice more. Do not rush to retake; analyze your score report and drill the areas where you lost points.

**Mental preparation.** Sleep well the night before. Use the bathroom before starting. Stay calm; partial credit is awarded. If a task feels impossible, move on and return later. Confidence comes from practice, so complete mock exams under timed conditions.

**killer.sh** is the best mock exam simulator. Two free sessions come with your exam purchase. Use them to experience the real environment: same terminal, same browser constraints, similar question style. Run through killer.sh at least once before your exam date. Treat each session as a dress rehearsal: time yourself, switch contexts correctly, and use only the allowed documentation. The questions are harder than the real exam, so if you can complete killer.sh tasks, you are well prepared.

**Final review strategy.** Focus on the highest-weight domains: Config & Security (25%), Design & Build (20%), Deployment (20%), and Networking (20%). These four areas account for 85% of the exam. Master the imperative commands for Pods, Deployments, Services, ConfigMaps, Secrets, Jobs, RBAC, and NetworkPolicies. Allocate your remaining study time proportionally to these weights rather than spreading effort evenly across all topics.

**The 80/20 rule.** Roughly 80% of exam tasks use 20% of kubectl commands. Get fluent with: `kubectl run`, `kubectl create`, `kubectl expose`, `kubectl get`, `kubectl describe`, `kubectl edit`, `kubectl config use-context`, and `kubectl explain`. Use `--dry-run=client -o yaml` to generate manifests instead of writing YAML from memory. Speed comes from muscle memory: run these commands hundreds of times before exam day so your fingers know the patterns without thinking.

## Beginner Tutorial

This is NOT a standard tutorial — it is a structured exam simulation guide:

1. **Set up your environment exactly like the exam:**
   - Open ONE terminal window
   - Open ONE browser tab with kubernetes.io/docs
   - Set up aliases and autocompletion
   - Close all other apps

2. **Context switching practice:**
   ```bash
   kubectl config use-context cluster1
   kubectl config current-context
   ```

3. **The 20-task speed run.** Create each of these resources as fast as possible using imperative commands. Time yourself. Target: under 40 minutes total.

   For each task, provide the exact imperative command:

   1. Pod running nginx
   2. Pod with labels app=web,tier=frontend
   3. Deployment with 3 replicas
   4. ClusterIP Service
   5. NodePort Service
   6. ConfigMap with 3 keys
   7. Secret with username/password
   8. Job that runs `echo done`
   9. CronJob every 10 minutes
   10. Pod with resource limits
   11. Pod with liveness probe
   12. Pod with readiness probe
   13. Multi-container Pod (generate + edit)
   14. Namespace with ResourceQuota
   15. ServiceAccount
   16. Role with get/list on pods
   17. RoleBinding
   18. NetworkPolicy default-deny
   19. PVC requesting 1Gi
   20. Ingress routing to a service

<details>
<summary>Speed run command reference</summary>

1. `kubectl run nginx --image=nginx --port=80`
2. `kubectl run web --image=nginx --labels="app=web,tier=frontend"`
3. `kubectl create deployment myapp --image=nginx --replicas=3`
4. `kubectl expose deployment myapp --port=80 --target-port=80 --type=ClusterIP`
5. `kubectl expose deployment myapp --port=80 --target-port=80 --type=NodePort`
6. `kubectl create configmap myconfig --from-literal=key1=val1 --from-literal=key2=val2 --from-literal=key3=val3`
7. `kubectl create secret generic mysecret --from-literal=username=admin --from-literal=password=secret`
8. `kubectl create job myjob --image=busybox -- echo done`
9. `kubectl create cronjob mycron --image=busybox --schedule="*/10 * * * *" -- echo done`
10. `kubectl run limits-pod --image=nginx --restart=Never --requests='memory=64Mi' --limits='memory=128Mi'`
11. `kubectl run liveness-pod --image=nginx --restart=Never` then `kubectl edit pod liveness-pod` to add livenessProbe (httpGet :80)
12. `kubectl run readiness-pod --image=nginx --restart=Never` then `kubectl edit pod readiness-pod` to add readinessProbe (httpGet :80)
13. `kubectl run multi --image=nginx --restart=Never --dry-run=client -o yaml > multi.yaml` then add second container and `kubectl apply -f multi.yaml`
14. `kubectl create namespace quota-ns` then `kubectl create resourcequota myquota --hard=cpu=1,memory=1Gi,pods=2 -n quota-ns`
15. `kubectl create serviceaccount mysa`
16. `kubectl create role myrole --verb=get,list --resource=pods`
17. `kubectl create rolebinding mybinding --role=myrole --serviceaccount=default:mysa`
18. Create NetworkPolicy YAML (podSelector: {}, policyTypes: [Ingress,Egress], no rules) and `kubectl apply -f -`
19. `kubectl create pvc mypvc --storageclass=standard -o yaml --dry-run=client` then add resources.requests.storage: 1Gi and apply
20. `kubectl create ingress myingress --rule="myhost.com/=mysvc:80"`

</details>

4. **Review your weak spots based on the speed run**

## Hands-On Lab

5 full exam-style questions (each simulating a real CKAD question with context, task, and weight):

### Question 1 (Weight: 7%) — "Application Deployment"

**Context:** Switch to cluster `ckad-cluster1`.

**Task:** Create a Deployment named `cache` in namespace `db` using image `redis:7-alpine` with 2 replicas. Set memory request to 64Mi and limit to 128Mi. Expose it as a ClusterIP Service named `cache-svc` on port 6379.

<details>
<summary>Hints</summary>

- Create the namespace first if it does not exist: `kubectl create namespace db`
- Use `kubectl create deployment` with `--replicas`, `--image`, and `-n db`
- Add resource requests/limits via `kubectl set resources` or edit the deployment
- Expose with `kubectl expose deployment cache --name=cache-svc --port=6379 --target-port=6379 -n db`
</details>

### Question 2 (Weight: 7%) — "Configuration"

**Context:** Switch to cluster `ckad-cluster1`.

**Task:** In namespace `app-team`, create a ConfigMap `app-settings` with keys: THEME=dark, CACHE_TTL=300. Create a Secret `app-creds` with key DB_PASSWORD=exampass2026. Deploy a Pod `config-pod` using image `busybox` (command: sleep 3600) that mounts ConfigMap values as env vars and the Secret as a volume at `/etc/creds`.

<details>
<summary>Hints</summary>

- ConfigMap: `kubectl create configmap app-settings --from-literal=THEME=dark --from-literal=CACHE_TTL=300 -n app-team`
- Secret: `kubectl create secret generic app-creds --from-literal=DB_PASSWORD=exampass2026 -n app-team`
- Pod needs envFrom or env for ConfigMap, and volumeMount for Secret volume
- Use `kubectl run` with `--dry-run=client -o yaml` then add the volume and env sections
</details>

### Question 3 (Weight: 8%) — "Troubleshooting"

**Context:** Switch to cluster `ckad-cluster2`.

**Task:** A Deployment `web-app` in namespace `production` has Pods in CrashLoopBackOff. Investigate and fix the issue. The application should be accessible via its Service. (Provide broken YAML: wrong port in probe, missing configmap reference, incorrect image tag.)

<details>
<summary>Hints</summary>

- Use `kubectl describe pod` and `kubectl logs` to identify the failure
- Check probe ports match the container's listening port
- Verify ConfigMap/Secret references exist and are correct
- Ensure image tag is valid and pullable
- After fixing, verify Pods are Running and Service endpoints are populated
</details>

### Question 4 (Weight: 8%) — "Networking & Security"

**Context:** Switch to cluster `ckad-cluster2`.

**Task:** In namespace `secure`, create a NetworkPolicy `api-policy` that:
- Applies to Pods with label `app=api`
- Allows ingress only from Pods with label `app=frontend` on port 8080
- Allows egress only to Pods with label `app=database` on port 5432 and DNS (UDP 53)

Also create a Role `api-role` allowing get/list/watch on deployments and services. Bind it to ServiceAccount `api-sa`.

<details>
<summary>Hints</summary>

- NetworkPolicy uses podSelector, ingress rules with from/podSelector, egress rules with to/podSelector and ports
- DNS egress: add rule for namespaceSelector {} and port 53/UDP
- Role: `kubectl create role api-role --verb=get,list,watch --resource=deployments,services -n secure`
- RoleBinding: `kubectl create rolebinding` linking api-role to api-sa in namespace secure
- Create ServiceAccount api-sa if it does not exist
</details>

### Question 5 (Weight: 7%) — "Application Design"

**Context:** Switch to cluster `ckad-cluster1`.

**Task:** Create a Pod `data-processor` with an init container and a main container:
- Init container (busybox): downloads a config file by writing `{"mode":"production"}` to `/config/settings.json` on a shared volume
- Main container (busybox): reads the config and runs `cat /config/settings.json && sleep 3600`

Add a liveness probe that execs `cat /config/settings.json` every 15 seconds.

<details>
<summary>Hints</summary>

- Use a shared emptyDir volume mounted at /config in both containers
- Init container command: `sh -c 'echo '{"mode":"production"}' > /config/settings.json'`
- Main container command: `sh -c 'cat /config/settings.json && sleep 3600'`
- Liveness probe: exec with command `["cat","/config/settings.json"]`, periodSeconds: 15
- Generate base with `kubectl run` then edit to add init container and probe
</details>

## Weekly Speed Drill

Final comprehensive drill — 15 minutes, 15 tasks. These are the most commonly tested imperative commands:

1. `k run nginx --image=nginx --port=80 --labels="app=web"`
2. `k create deploy myapp --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml`
3. `k expose deploy myapp --port=80 --target-port=80 --type=ClusterIP`
4. `k create cm myconfig --from-literal=key1=val1 --from-literal=key2=val2`
5. `k create secret generic mysecret --from-literal=pass=secret123`
6. `k create job myjob --image=busybox -- echo hello`
7. `k create cronjob mycron --image=busybox --schedule="*/5 * * * *" -- date`
8. `k create sa mysa`
9. `k create role myrole --verb=get,list --resource=pods`
10. `k create rolebinding mybinding --role=myrole --serviceaccount=default:mysa`
11. `k create ingress myingress --rule="myhost.com/=mysvc:80"`
12. `k run tmp --image=busybox --rm -it -- wget -qO- http://myapp`
13. `k auth can-i list pods --as=system:serviceaccount:default:mysa`
14. `k config set-context --current --namespace=mynamespace`
15. `k get pods -A --sort-by=.metadata.creationTimestamp`

## Exam Pitfalls

- Not doing a warm-up before the exam (set up aliases first thing)
- Spending too long on one question (skip at 6-8 min mark)
- Forgetting to switch cluster context (answers in wrong cluster score 0)
- Not using `kubectl explain` (faster than searching kubernetes.io)
- Writing YAML from memory instead of using `--dry-run=client -o yaml`
- Not verifying your answer (always `kubectl get` after applying)
- Panicking when you see an unfamiliar topic (partial credit exists)
- Not reading the FULL question (namespace, context, specific requirements)

## Solution Key

### Question 1 — Application Deployment

```bash
kubectl config use-context ckad-cluster1
kubectl create namespace db --dry-run=client -o yaml | kubectl apply -f -
kubectl create deployment cache -n db --image=redis:7-alpine --replicas=2
kubectl set resources deployment cache -n db --requests=memory=64Mi --limits=memory=128Mi
kubectl expose deployment cache -n db --name=cache-svc --port=6379 --target-port=6379
```

Verification:
```bash
kubectl get deployment cache -n db
kubectl get svc cache-svc -n db
kubectl get pods -n db -l app=cache
```

### Question 2 — Configuration

```bash
kubectl config use-context ckad-cluster1
kubectl create namespace app-team --dry-run=client -o yaml | kubectl apply -f -
kubectl create configmap app-settings -n app-team --from-literal=THEME=dark --from-literal=CACHE_TTL=300
kubectl create secret generic app-creds -n app-team --from-literal=DB_PASSWORD=exampass2026
```

Pod manifest (create via `kubectl run config-pod -n app-team --image=busybox --dry-run=client -o yaml` then add):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
  namespace: app-team
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    envFrom:
    - configMapRef:
        name: app-settings
    volumeMounts:
    - name: creds
      mountPath: /etc/creds
      readOnly: true
  volumes:
  - name: creds
    secret:
      secretName: app-creds
```

Apply and verify:
```bash
kubectl apply -f config-pod.yaml
kubectl get pod config-pod -n app-team
kubectl exec config-pod -n app-team -- env | grep -E "THEME|CACHE"
kubectl exec config-pod -n app-team -- cat /etc/creds/DB_PASSWORD
```

### Question 3 — Troubleshooting

```bash
kubectl config use-context ckad-cluster2
kubectl get pods -n production -l app=web-app
kubectl describe pod <pod-name> -n production
kubectl logs <pod-name> -n production
kubectl logs <pod-name> -n production -c <container-name> --previous
```

Common fixes:
- Probe port: ensure liveness/readiness probe `port` or `httpGet.port` matches container port
- ConfigMap: add `envFrom` or `env` with `configMapKeyRef` if referenced
- Image: use valid tag (e.g., fix typo or use `:latest` if appropriate)

After editing deployment:
```bash
kubectl rollout restart deployment web-app -n production
kubectl get pods -n production -w
kubectl get endpoints -n production
```

### Question 4 — Networking & Security

```bash
kubectl config use-context ckad-cluster2
kubectl create namespace secure --dry-run=client -o yaml | kubectl apply -f -
kubectl create serviceaccount api-sa -n secure
```

NetworkPolicy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: secure
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

```bash
kubectl create role api-role -n secure --verb=get,list,watch --resource=deployments,services
kubectl create rolebinding api-binding -n secure --role=api-role --serviceaccount=secure:api-sa
```

Verification:
```bash
kubectl get networkpolicy api-policy -n secure
kubectl get role api-role -n secure
kubectl get rolebinding api-binding -n secure
kubectl auth can-i get deployments -n secure --as=system:serviceaccount:secure:api-sa
```

### Question 5 — Application Design

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  initContainers:
  - name: init-config
    image: busybox
    command:
    - sh
    - -c
    - 'echo ''{"mode":"production"}'' > /config/settings.json'
    volumeMounts:
    - name: config
      mountPath: /config
  containers:
  - name: main
    image: busybox
    command:
    - sh
    - -c
    - cat /config/settings.json && sleep 3600
    volumeMounts:
    - name: config
      mountPath: /config
    livenessProbe:
      exec:
        command:
        - cat
        - /config/settings.json
      periodSeconds: 15
  volumes:
  - name: config
    emptyDir: {}
```

```bash
kubectl apply -f data-processor.yaml
kubectl get pod data-processor
kubectl logs data-processor -c main
```
