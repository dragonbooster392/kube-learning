# 04 — Pods

## What Is a Pod

A Pod is the **smallest thing you can create in Kubernetes**. It wraps one or more containers.

```
In Docker:    you run containers
In K8s:       you run Pods (which contain containers)

You NEVER create a bare container in Kubernetes.
You always create a Pod, and the Pod contains your container(s).
```

```
Think of it like this:

Docker container = a person
Kubernetes Pod   = a person sitting in a chair

The Pod is the wrapper. The container is inside it.
You can't have a person in a Kubernetes cluster without a chair.
```

---

## Why Pods? Why Not Just Containers?

```
A Pod adds things around the container that Kubernetes needs:

  1. An IP address (every Pod gets its own IP)
  2. Shared storage (volumes)
  3. Network namespace (containers in a Pod share localhost)
  4. Lifecycle management (restart policies, probes)
  5. Labels (for organizing and selecting)

A bare container doesn't have these.
A Pod is the "Kubernetes wrapper" that makes a container manageable.
```

---

## Single-Container Pods (The Common Case)

99% of the time, a Pod has exactly **one container**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

```
This creates:
  1 Pod named "my-nginx"
  containing 1 container running nginx:1.25
  exposing port 80

That's it. One Pod, one container. The most common pattern.
```

---

## Multi-Container Pods (The Special Case)

Sometimes you need two containers that are **tightly coupled** — they must run on the same node, share the same network, and access the same files.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-sidecar
spec:
  containers:
    - name: web
      image: nginx:1.25
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
    - name: log-shipper
      image: fluentd:v1.16
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
  volumes:
    - name: shared-logs
      emptyDir: {}
```

```
This creates 1 Pod with 2 containers:
  Container 1 (web):        nginx writes logs to /var/log/nginx/
  Container 2 (log-shipper): fluentd reads logs from /var/log/nginx/

They share the same volume (shared-logs).
They share the same network (can talk via localhost).
They're ALWAYS scheduled on the same node.
```

### Multi-Container Patterns

```
Sidecar:
  Main container: your application
  Sidecar: helper (log shipper, config reloader, proxy)
  Example: nginx + fluentd (logs), app + envoy (proxy)

Ambassador:
  Main container: your application
  Ambassador: proxy to external services
  Example: app + redis-proxy (handles connection pooling)

Adapter:
  Main container: your application  
  Adapter: transforms output to a standard format
  Example: app (custom metrics) + prometheus-adapter (converts to Prometheus format)
```

```
Rule: Only use multi-container Pods when containers are TIGHTLY COUPLED.
      If they can run independently, put them in SEPARATE Pods.

  ✗ Web server + database → separate Pods (they're independent)
  ✓ Web server + log shipper → same Pod (log shipper needs web server's files)
```

---

## Pod Networking

```
Every Pod gets its own IP address.

Pod "my-nginx" gets IP: 10.244.1.5
Pod "my-api"   gets IP: 10.244.2.8

They can talk to each other directly:
  From my-api:  curl 10.244.1.5:80  → reaches my-nginx

Containers WITHIN the same Pod share the same IP:
  Pod "web-with-sidecar" has IP: 10.244.1.10
  Container "web" is at:       10.244.1.10:80
  Container "log-shipper" is at: 10.244.1.10 (same IP)
  They talk to each other via: localhost
    From log-shipper: curl localhost:80 → reaches web
```

```
┌─────── Pod (10.244.1.10) ────────┐
│                                   │
│  ┌──────────┐   ┌──────────────┐ │
│  │   web    │   │ log-shipper  │ │
│  │  :80     │   │              │ │
│  └──────────┘   └──────────────┘ │
│                                   │
│  Shared network namespace         │
│  Shared volumes                   │
│  Same node                        │
└───────────────────────────────────┘

Container-to-container within a Pod: localhost
Pod-to-Pod: use the Pod IP
```

---

## Pod Lifecycle

```
Pod states (what you see in kubectl get pods):

Pending     → Pod accepted but not yet running
             (image being pulled, waiting for node, etc.)

Running     → Pod is on a node, at least one container is running

Succeeded   → All containers exited with code 0 (batch jobs)

Failed      → All containers stopped, at least one exited with error

Unknown     → Pod status can't be determined (node communication lost)
```

```
Lifecycle of a Pod:

  Created (Pending)
       │
       ▼
  Scheduled to a node (still Pending)
       │
       ▼
  Image pulled (still Pending)
       │
       ▼
  Container started (Running)
       │
       ▼
  Container exits:
    Exit code 0  → Succeeded
    Exit code ≠ 0 → Failed or CrashLoopBackOff (if restarting)
```

### Restart Policies

```yaml
spec:
  restartPolicy: Always      # default — always restart
  restartPolicy: OnFailure   # restart only if exit code ≠ 0
  restartPolicy: Never       # never restart
```

```
Always:     For web servers, APIs — things that should run forever
OnFailure:  For batch jobs — restart if they crash, not if they succeed
Never:      For one-shot tasks — run once and done
```

---

## Creating Pods

### Imperative (Quick Testing)

```bash
# Create a pod:
kubectl run my-nginx --image=nginx:1.25

# Create a pod and expose a port:
kubectl run my-nginx --image=nginx:1.25 --port=80

# Create a pod with labels:
kubectl run my-nginx --image=nginx:1.25 --labels="app=nginx,env=dev"

# Generate YAML without creating:
kubectl run my-nginx --image=nginx:1.25 --dry-run=client -o yaml
```

### Declarative (YAML File)

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
  labels:
    app: nginx
    environment: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

```bash
# Create:
kubectl apply -f pod.yaml

# Check:
kubectl get pods
kubectl describe pod my-nginx
kubectl logs my-nginx

# Delete:
kubectl delete -f pod.yaml
# or:
kubectl delete pod my-nginx
```

---

## Inspecting Pods

```bash
# List pods:
kubectl get pods
# NAME       READY   STATUS    RESTARTS   AGE
# my-nginx   1/1     Running   0          5m

# READY column: 1/1 means 1 of 1 containers are ready
# For multi-container: 2/2 means both containers are ready

# Wide output (shows node and IP):
kubectl get pods -o wide
# NAME       READY  STATUS   RESTARTS  AGE  IP           NODE
# my-nginx   1/1    Running  0         5m   10.244.1.5   node-2

# Describe (detailed info + events):
kubectl describe pod my-nginx

# View logs:
kubectl logs my-nginx
kubectl logs my-nginx -f          # follow (live)
kubectl logs my-nginx --previous  # previous container (if restarted)

# Shell into a pod:
kubectl exec -it my-nginx -- /bin/bash
kubectl exec -it my-nginx -- /bin/sh

# Run a one-off command:
kubectl exec my-nginx -- cat /etc/nginx/nginx.conf
kubectl exec my-nginx -- curl -s localhost:80
```

---

## Pod Status Problems

### ImagePullBackOff

```
kubectl get pods
# NAME       READY   STATUS             RESTARTS   AGE
# my-app     0/1     ImagePullBackOff   0          2m

Meaning: Kubernetes can't pull the container image.

Causes:
  1. Image name is wrong (typo)
  2. Image doesn't exist in the registry
  3. Registry requires authentication (imagePullSecrets needed)
  4. No internet access on the node

Fix: Check the image name. Check registry access.
  kubectl describe pod my-app    # look at Events section
```

### CrashLoopBackOff

```
kubectl get pods
# NAME       READY   STATUS             RESTARTS   AGE
# my-app     0/1     CrashLoopBackOff   5          3m

Meaning: The container starts, crashes, and Kubernetes keeps restarting it.
  Each restart waits longer (10s, 20s, 40s, 80s, up to 5 minutes).

Causes:
  1. Application error (bug, missing config, bad command)
  2. Missing environment variables or config
  3. Port conflict
  4. Wrong entrypoint/command

Fix: Check the logs for the error:
  kubectl logs my-app
  kubectl logs my-app --previous
```

### Pending

```
kubectl get pods
# NAME       READY   STATUS    RESTARTS   AGE
# my-app     0/1     Pending   0          5m

Meaning: Pod can't be scheduled to any node.

Causes:
  1. Not enough CPU or memory on any node
  2. No node matches the pod's node selector
  3. Pod has a taint that no node tolerates
  4. PersistentVolumeClaim not bound

Fix: Check Events:
  kubectl describe pod my-app
  # Events:
  # Warning  FailedScheduling  Insufficient memory
```

---

## Important: Pods Are Ephemeral

```
Pods are TEMPORARY. They are born and they die.

A Pod that dies is NEVER brought back.
  A NEW Pod is created to replace it (with a new name and new IP).

This is why you should NOT:
  ✗ Store data inside a Pod (use Volumes)
  ✗ Hardcode Pod IPs (use Services)
  ✗ Create Pods directly (use Deployments)

Pod dies → new Pod → new name → new IP
            my-nginx-abc123  →  my-nginx-xyz789
            10.244.1.5       →  10.244.2.8
```

```
This is the #1 concept beginners struggle with:
  
  "But I created a Pod and it disappeared!"
  → Yes. Pods die. That's normal.
  
  "Why did my Pod get a new IP?"
  → It's a new Pod, not the same one.
  
  "How do I keep my Pod running?"
  → Don't manage Pods directly. Use a Deployment.
     Deployments create and manage Pods for you.
     If a Pod dies, the Deployment creates a replacement.
```

---

## Pods vs Deployments — When to Use What

```
Direct Pod creation:
  kubectl run nginx --image=nginx
  kubectl apply -f pod.yaml
  
  ✓ Quick testing
  ✓ Learning
  ✗ No self-healing (if it dies, it's gone)
  ✗ No scaling
  ✗ No rolling updates

Deployment (wraps Pods):
  kubectl create deployment nginx --image=nginx
  kubectl apply -f deployment.yaml
  
  ✓ Self-healing (recreates crashed Pods)
  ✓ Scaling (replicas: 5)
  ✓ Rolling updates
  ✓ Rollback
  ✓ Production use

Rule: ALWAYS use Deployments in production.
      Only create bare Pods for quick tests or learning.
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Pods                       → The fundamental unit of everything in K8s
                           → Every application runs inside a Pod
Pod IP                     → Container networking
                           → Service discovery builds on Pod IPs
Multi-container Pods       → Sidecar pattern (Istio, Envoy)
                           → Log collection (Fluentd, Filebeat)
                           → Service mesh proxies
Pod lifecycle              → Understanding crash recovery
                           → Designing resilient applications
Ephemeral Pods             → Stateless application design
                           → Why you need Volumes and Services
CrashLoopBackOff           → The error you'll see most often
                           → First debugging skill to learn
kubectl exec/logs          → Daily debugging tools
```

---

> **Next:** 05 — ReplicaSets → Running multiple copies of a Pod for reliability.
