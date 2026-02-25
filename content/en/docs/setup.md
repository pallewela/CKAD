---
title: Setup
linkTitle: Setup
weight: 20
---

This guide walks you through setting up a local Kubernetes cluster on your machine.
By the end, you will have a working cluster and will deploy a Go application into it.

**Prerequisites:** You need Docker installed on your system. If you don't have it yet,
install Docker Engine (Linux) or Docker Desktop (macOS / Windows) first:
https://docs.docker.com/engine/install/

---

## Table of Contents

1. [Choose Your Tool: kind vs minikube](#1-choose-your-tool-kind-vs-minikube)
2. [Option A: Install kind (Recommended)](#2-option-a-install-kind-recommended)
3. [Option B: Install minikube](#3-option-b-install-minikube)
4. [Install kubectl](#4-install-kubectl)
5. [Verify Your Cluster](#5-verify-your-cluster)
6. [Build a Go App and Run It in Your Cluster](#6-build-a-go-app-and-run-it-in-your-cluster)
7. [Cleanup](#7-cleanup)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Choose Your Tool: kind vs minikube

Both tools create a Kubernetes cluster on your laptop. Here is how they compare:

| Feature | kind | minikube |
|---------|------|----------|
| **Full name** | **K**ubernetes **IN** **D**ocker | Mini Kubernetes |
| **How it works** | Runs cluster nodes as Docker containers | Runs a VM or Docker container |
| **Speed** | Very fast (seconds to start) | Moderate (30–60 seconds) |
| **Multi-node** | Yes, easy to configure | Yes, but heavier |
| **Resource usage** | Low | Medium |
| **Best for** | CI/CD, quick testing, Go developers | Learning, add-ons (dashboard, metrics) |
| **Install complexity** | Just a single binary | Single binary + optional drivers |

**Recommendation:** If you are a Go developer and want something lightweight, go with
**kind**. If you want built-in add-ons like the Kubernetes dashboard and metrics-server
out of the box, pick **minikube**.

---

## 2. Option A: Install kind (Recommended)

### What is kind?

**kind** stands for "Kubernetes IN Docker." It creates a Kubernetes cluster where each
node is a Docker container. It is fast, lightweight, and widely used for testing.

### Step 1: Install the kind binary

**Linux (amd64):**

```bash
# Download the kind binary
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64

# Make it executable
chmod +x ./kind

# Move it to a directory in your PATH so you can run it from anywhere
sudo mv ./kind /usr/local/bin/kind

# Verify it works
kind version
```

**Linux (arm64):**

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

**macOS (Homebrew):**

```bash
brew install kind
```

**If you have Go installed** (since you're a Go developer):

```bash
go install sigs.k8s.io/kind@latest
```

This places the `kind` binary in your `$GOPATH/bin` (or `$HOME/go/bin`). Make sure
that directory is in your `PATH`.

### Step 2: Create a cluster

```bash
# Create a single-node cluster named "ckad"
kind create cluster --name ckad
```

You should see output ending with:

```
Set kubectl context to "kind-ckad"
You can now use your cluster with:

kubectl cluster-info --context kind-ckad
```

### Step 3 (Optional): Multi-node cluster

For more realistic practice, create a cluster with one control plane and two workers.
Save this as `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Then run:

```bash
kind create cluster --name ckad-multi --config kind-config.yaml
```

### Loading images into kind

kind runs inside Docker, so it cannot pull images from your local Docker daemon by
default. You need to explicitly load them:

```bash
# Build your image locally
docker build -t myapp:v1 .

# Load it into the kind cluster
kind load docker-image myapp:v1 --name ckad
```

---

## 3. Option B: Install minikube

### What is minikube?

**minikube** creates a single-node (or multi-node) Kubernetes cluster on your local
machine. It can use Docker, VirtualBox, or other drivers to run the cluster.

### Step 1: Install minikube

**Linux (amd64):**

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

minikube version
```

**Linux (arm64):**

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-arm64
sudo install minikube-linux-arm64 /usr/local/bin/minikube
rm minikube-linux-arm64

minikube version
```

**macOS (Homebrew):**

```bash
brew install minikube
```

### Step 2: Start a cluster

```bash
# Start minikube using Docker as the driver (recommended)
minikube start --driver=docker --cpus=2 --memory=4096

# If you want to name your profile (like kind's --name)
minikube start --profile ckad --driver=docker --cpus=2 --memory=4096
```

### Step 3: Enable useful add-ons

minikube has built-in add-ons that are handy for CKAD practice:

```bash
# Metrics server (needed for `kubectl top`)
minikube addons enable metrics-server

# Ingress controller (needed for Ingress exercises)
minikube addons enable ingress

# Dashboard (visual overview of your cluster)
minikube addons enable dashboard
```

### Loading images into minikube

```bash
# Point your Docker CLI to minikube's Docker daemon
eval $(minikube docker-env)

# Now any `docker build` runs inside minikube — no loading needed
docker build -t myapp:v1 .
```

> After running `eval $(minikube docker-env)`, your `docker` commands talk to
> minikube's Docker, not your host Docker. To switch back, run `eval $(minikube docker-env -u)`.

---

## 4. Install kubectl

**kubectl** (pronounced "cube-control" or "cube-cuddle") is the command-line tool you
use to interact with any Kubernetes cluster. The exam runs entirely through kubectl.

### Check if you already have it

```bash
kubectl version --client
```

If this prints a version, you are set. If not, install it:

### Linux

```bash
# Download the latest stable version
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make it executable and move to PATH
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

kubectl version --client
```

### macOS (Homebrew)

```bash
brew install kubectl
```

### Enable shell autocompletion

This lets you press **Tab** to autocomplete resource names, saving significant time:

**Bash:**

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

**Zsh:**

```bash
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'alias k=kubectl' >> ~/.zshrc
source ~/.zshrc
```

---

## 5. Verify Your Cluster

Run these commands to confirm everything is working:

```bash
# Check cluster info
kubectl cluster-info

# List nodes — you should see one or more nodes in "Ready" status
kubectl get nodes

# Run a quick test Pod
kubectl run test --image=nginx --restart=Never

# Wait a few seconds, then check it is running
kubectl get pods

# Clean up the test Pod
kubectl delete pod test
```

If all commands succeed, your local lab is ready for CKAD practice.

---

## 6. Build a Go App and Run It in Your Cluster

Since you are a Go developer, let's build a real application, containerize it, and
deploy it to your local cluster. This will be the app you use throughout the syllabus.

### Step 1: Create the Go application

Create a directory for your app:

```bash
mkdir -p ~/ckad-app && cd ~/ckad-app
```

Initialize a Go module:

```bash
go mod init ckad-app
```

Create `main.go`:

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"os"
)

func main() {
	port := os.Getenv("APP_PORT")
	if port == "" {
		port = "8080"
	}

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{
			"status":  "ok",
			"message": "Hello from CKAD app!",
		})
	})

	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("healthy"))
	})

	http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("ready"))
	})

	log.Printf("Starting server on :%s", port)
	log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

This app has three endpoints:
- `/` — returns a JSON status message.
- `/healthz` — for Kubernetes liveness probes (checks if the app is alive).
- `/ready` — for Kubernetes readiness probes (checks if the app is ready for traffic).

### Step 2: Write the Dockerfile (multi-stage build)

A **multi-stage build** uses one container to compile the code and a second, tiny
container to run it. This keeps your final image small.

Create a `Dockerfile` in the same directory:

```dockerfile
# --- Stage 1: Build the Go binary ---
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod ./
RUN go mod download
COPY main.go ./
# CGO_ENABLED=0 produces a static binary with no C dependencies.
# This lets us run it in a "scratch" (empty) container.
RUN CGO_ENABLED=0 GOOS=linux go build -o /ckad-app .

# --- Stage 2: Run in a minimal container ---
FROM scratch
COPY --from=builder /ckad-app /ckad-app
EXPOSE 8080
ENTRYPOINT ["/ckad-app"]
```

**Why `scratch`?** The `scratch` image is completely empty — no shell, no OS, nothing.
Your Go binary is statically compiled, so it does not need anything else. The result is
an image that is typically 6–15 MB.

### Step 3: Build the Docker image

```bash
docker build -t ckad-app:v1 .
```

Verify the image size:

```bash
docker images ckad-app
# You should see something like:
# REPOSITORY   TAG   IMAGE ID       CREATED          SIZE
# ckad-app     v1    abc123def456   5 seconds ago    12MB
```

### Step 4: Test locally (optional)

```bash
docker run -p 8080:8080 ckad-app:v1

# In another terminal:
curl http://localhost:8080
# {"message":"Hello from CKAD app!","status":"ok"}

curl http://localhost:8080/healthz
# healthy
```

Press `Ctrl+C` to stop the container.

### Step 5: Load the image into your cluster

**For kind:**

```bash
kind load docker-image ckad-app:v1 --name ckad
```

**For minikube:**

```bash
# If you already ran `eval $(minikube docker-env)` before building, skip this.
# Otherwise, rebuild inside minikube's Docker:
eval $(minikube docker-env)
docker build -t ckad-app:v1 .
```

### Step 6: Deploy to Kubernetes

Create a file called `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ckad-app
  labels:
    app: ckad-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ckad-app
  template:
    metadata:
      labels:
        app: ckad-app
    spec:
      containers:
        - name: ckad-app
          image: ckad-app:v1
          # "Never" tells Kubernetes to use the local image instead of
          # trying to pull from a remote registry.
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
```

Apply it:

```bash
kubectl apply -f deployment.yaml
```

### Step 7: Expose with a Service

```bash
kubectl expose deployment ckad-app --port=80 --target-port=8080 --type=NodePort
```

### Step 8: Access your app

**For kind:**

```bash
# Find the NodePort
kubectl get svc ckad-app

# Use port-forward (simplest method with kind)
kubectl port-forward svc/ckad-app 8080:80

# In another terminal:
curl http://localhost:8080
```

**For minikube:**

```bash
minikube service ckad-app
# This opens your browser or prints the URL to access the app.
```

You now have a Go application running in a local Kubernetes cluster with health checks.
This is the foundation you will build on throughout the 12-week syllabus.

---

## 7. Cleanup

When you are done practicing, you can delete the cluster to free up resources:

**kind:**

```bash
kind delete cluster --name ckad
```

**minikube:**

```bash
minikube stop      # pause the cluster (keeps state)
minikube delete    # destroy the cluster entirely
```

You can always recreate the cluster with the commands from Section 2 or 3.

---

## 8. Troubleshooting

### "kubectl: command not found"

Make sure kubectl is in your `PATH`:

```bash
echo $PATH
which kubectl
```

If you installed it manually, verify it is in `/usr/local/bin/`:

```bash
ls -la /usr/local/bin/kubectl
```

### "The connection to the server localhost:8080 was refused"

This means kubectl is not pointed at a cluster. Make sure your cluster is running:

```bash
# For kind
kind get clusters
docker ps  # you should see kind containers

# For minikube
minikube status
```

### "ImagePullBackOff" or "ErrImagePull"

This usually means Kubernetes is trying to pull the image from a remote registry but
cannot find it. For local images:

1. Make sure you loaded the image into the cluster (see Step 5 above).
2. Set `imagePullPolicy: Never` in your Pod/Deployment spec.

### Pods stuck in "Pending" state

Check events:

```bash
kubectl describe pod <pod-name>
```

Common causes:
- **Insufficient resources:** Your cluster does not have enough CPU or memory. Try
  increasing minikube resources: `minikube start --cpus=4 --memory=8192`
- **No nodes available:** Check `kubectl get nodes` — all nodes should be `Ready`.

### kind cluster won't start

Make sure Docker is running:

```bash
docker info
```

If Docker is not running, start it:

```bash
sudo systemctl start docker  # Linux
# Or open Docker Desktop on macOS/Windows
```


---

*Next step: Open [Syllabus](/docs/syllabus/) and start Week 1.*
