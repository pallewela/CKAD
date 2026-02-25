# Week 5: Deployments, Jobs & CronJobs

## The Concept

Kubernetes provides several workload controllers to run different types of applications. Understanding when and how to use each is essential for the exam.

**ReplicaSet** ensures that a specified number of identical Pods are running at all times. If a Pod fails or is deleted, the ReplicaSet creates a replacement. You rarely create a ReplicaSet directly; Deployments manage ReplicaSets for you.

**Deployment** is the standard way to run stateless applications in Kubernetes. A Deployment creates and manages ReplicaSets, which in turn manage Pods. Deployments add two critical capabilities: rolling updates and rollbacks. When you change the container image or other Pod template fields, the Deployment creates a new ReplicaSet and gradually shifts traffic from the old one to the new one.

**Rolling Update** is the default update strategy. Two fields control the pace: `maxSurge` (how many extra Pods can exist during the update beyond the desired count) and `maxUnavailable` (how many Pods can be unavailable during the update). The default for both is 25%. For example, with 4 replicas and 25% each, Kubernetes might create 1 new Pod and allow 1 old Pod to be unavailable at a time, ensuring continuous availability.

**Rollback** lets you revert to a previous version. The command `kubectl rollout undo` switches back to the previous ReplicaSet. Each update creates a new revision, and you can undo to a specific revision with `--to-revision=N`.

**Job** runs one or more Pods until they complete successfully. Unlike Deployments, Jobs are for batch or one-off tasks. Key fields: `completions` (how many successful runs are needed), `parallelism` (how many Pods run at once), `backoffLimit` (retries on failure, default 6), and `activeDeadlineSeconds` (maximum time for the Job). Jobs require `restartPolicy: Never` or `OnFailure`; `Always` is invalid.

**CronJob** schedules Jobs to run at specified times using cron syntax. The format is "minute hour day-of-month month day-of-week" (e.g., `*/5 * * * *` means every 5 minutes). Important fields: `concurrencyPolicy` (Allow, Forbid, or Replace for overlapping runs), `startingDeadlineSeconds` (tolerance for missed schedules), and `successfulJobsHistoryLimit` (how many completed Jobs to retain).

**DaemonSet** runs exactly one Pod on every node (or every node matching a selector). Use it for node-level agents like log collectors or monitoring. It is mentioned here for awareness; the exam may reference it.

---

## Beginner Tutorial

### 1. Create Deployment

```bash
kubectl create deployment nginx-deploy --image=nginx:1.24 --replicas=3
```

### 2. Inspect

```bash
kubectl get deployment
kubectl get rs
kubectl get pods
```

### 3. Observe the ReplicaSet

The ReplicaSet name has a hash suffix (e.g., `nginx-deploy-7d4f8b9c6`). The Deployment owns this ReplicaSet.

### 4. Perform rolling update

```bash
kubectl set image deployment/nginx-deploy nginx=nginx:1.25
```

### 5. Watch rollout

```bash
kubectl rollout status deployment/nginx-deploy
```

### 6. Check history

```bash
kubectl rollout history deployment/nginx-deploy
```

### 7. Roll back

```bash
kubectl rollout undo deployment/nginx-deploy
```

### 8. Roll back to specific revision

```bash
kubectl rollout undo deployment/nginx-deploy --to-revision=1
```

### 9. Scale

```bash
kubectl scale deployment nginx-deploy --replicas=5
```

### 10. Create a Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
spec:
  completions: 1
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

Apply with `kubectl apply -f pi-job.yaml`.

### 11. Create a CronJob that runs every minute

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: every-minute
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["echo", "hello from cron"]
          restartPolicy: OnFailure
```

Apply with `kubectl apply -f cronjob.yaml`.

### 12. Check job status, logs, and cleanup

```bash
kubectl get jobs
kubectl get cronjobs
kubectl logs job/pi-job
kubectl logs job/<cronjob-generated-job-name>
kubectl delete job pi-job
kubectl delete cronjob every-minute
```

---

## Hands-On Lab

### Challenge 1: Rolling Update & Rollback

**Scenario:** Deploy nginx:1.24 with 4 replicas. Update to nginx:1.25 and verify zero-downtime. Then intentionally deploy a bad image (nginx:broken), observe the failure, and roll back to a working version.

**Success criteria:** Deployment runs 4 replicas of nginx:1.24, updates cleanly to 1.25, fails on broken image, and rollback restores working Pods.

<details>
<summary>Hints</summary>

- Use `kubectl create deployment` with `--replicas=4` and `--image=nginx:1.24`
- Use `kubectl set image` for updates
- Use `kubectl rollout status` to watch progress
- Use `kubectl rollout undo` after the broken image fails
</details>

### Challenge 2: Batch Processing

**Scenario:** Create a Job with `completions=5` and `parallelism=2` that processes items. Verify all 5 complete successfully.

**Success criteria:** Job runs 5 completions with at most 2 Pods running at once; all 5 complete.

<details>
<summary>Hints</summary>

- Set `completions: 5` and `parallelism: 2` in the Job spec
- Use `restartPolicy: Never` or `OnFailure`
- Check `kubectl get jobs` and `kubectl get pods` to verify completion count
</details>

### Challenge 3: Scheduled Task

**Scenario:** Create a CronJob that writes hostname and date to a log every 2 minutes. Verify at least 2 jobs ran. Set `successfulJobsHistoryLimit=3`.

**Success criteria:** CronJob runs every 2 minutes; at least 2 Jobs complete; history limit is 3.

<details>
<summary>Hints</summary>

- Schedule: `*/2 * * * *` (every 2 minutes)
- Use a container that writes to stdout (e.g., `echo $(hostname) $(date)`)
- Add `successfulJobsHistoryLimit: 3` to the CronJob spec
- Wait 4+ minutes and check `kubectl get jobs` for spawned Jobs
</details>

### Challenge 4: Deployment Strategy

**Scenario:** Create a Deployment with `maxSurge=1` and `maxUnavailable=0` to ensure no downtime. Update the image and watch that old Pods only terminate after new ones are Ready.

**Success criteria:** During update, no Pod is terminated until its replacement is Ready; maxSurge and maxUnavailable are set correctly.

<details>
<summary>Hints</summary>

- Add a `strategy` block under `spec` with `type: RollingUpdate`
- Set `maxSurge: 1` and `maxUnavailable: 0`
- With 0 unavailable, Kubernetes must bring new Pods up before removing old ones
- Use `kubectl rollout status -w` to watch
</details>

---

## Weekly Speed Drill

Complete these 10 tasks as quickly as possible:

1. Create a deployment with 3 replicas
2. Scale it to 5
3. Update the image
4. Check rollout status
5. View rollout history
6. Undo last rollout
7. Create a Job that runs `echo hello`
8. Create a CronJob for every 5 minutes
9. Suspend a CronJob
10. Delete all completed jobs

---

## Exam Pitfalls

- **`--record` is deprecated:** Use `kubectl annotate` instead for tracking changes in rollout history.

- **Deployments own ReplicaSets:** Do not delete ReplicaSets directly; the Deployment will recreate them or you may orphan Pods.

- **Job restartPolicy:** Must be `Never` or `OnFailure`. `Always` is invalid for Jobs and CronJobs.

- **CronJob schedule syntax:** Minutes come first (minute hour day month weekday). Common mistake: using seconds (Kubernetes cron has no seconds field).

- **backoffLimit on Jobs:** Default is 6. If you need fewer retries, set it explicitly to avoid wasting time on excessive retries.

- **`--replicas` when creating deployment imperatively:** Forgetting `--replicas` defaults to 1; specify it if you need more.

- **Check rollout history before undo:** `kubectl rollout undo` goes to the previous revision. Verify with `kubectl rollout history` that you are undoing to the correct version.

- **CronJob concurrencyPolicy:** Default is `Allow`, so overlapping jobs can run. Use `Forbid` or `Replace` if you need to prevent or replace overlapping runs.

---

## Solution Key

### Challenge 1: Rolling Update & Rollback

```bash
# Create deployment
kubectl create deployment nginx-deploy --image=nginx:1.24 --replicas=4

# Verify
kubectl get deployment nginx-deploy
kubectl get pods -l app=nginx-deploy

# Update to 1.25
kubectl set image deployment/nginx-deploy nginx=nginx:1.25
kubectl rollout status deployment/nginx-deploy

# Intentionally break
kubectl set image deployment/nginx-deploy nginx=nginx:broken
kubectl rollout status deployment/nginx-deploy  # will show failure

# Roll back
kubectl rollout undo deployment/nginx-deploy
kubectl rollout status deployment/nginx-deploy
```

### Challenge 2: Batch Processing

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
      - name: processor
        image: busybox
        command: ["sh", "-c", "echo Processing item; sleep 2"]
      restartPolicy: Never
  backoffLimit: 4
```

```bash
kubectl apply -f batch-job.yaml
kubectl get jobs
kubectl get pods
# Wait for COMPLETIONS to show 5/5
```

### Challenge 3: Scheduled Task

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-writer
spec:
  schedule: "*/2 * * * *"
  successfulJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: logger
            image: busybox
            command: ["sh", "-c", "echo $(hostname) $(date)"]
          restartPolicy: OnFailure
```

```bash
kubectl apply -f log-writer-cronjob.yaml
# Wait 4+ minutes
kubectl get cronjobs
kubectl get jobs
kubectl logs job/<job-name-from-cronjob>
```

### Challenge 4: Deployment Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero-downtime-deploy
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: zero-downtime
  template:
    metadata:
      labels:
        app: zero-downtime
    spec:
      containers:
      - name: app
        image: nginx:1.24
```

```bash
kubectl apply -f deployment-strategy.yaml
kubectl set image deployment/zero-downtime-deploy app=nginx:1.25
kubectl rollout status deployment/zero-downtime-deploy -w
# Observe: new Pods become Ready before old ones terminate
```
