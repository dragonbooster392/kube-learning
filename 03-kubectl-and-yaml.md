# 03 — kubectl & YAML

## What Is kubectl

kubectl (pronounced "kube-control", "kube-C-T-L", or "kube-cuddle" — nobody agrees) is the **command-line tool** you use to talk to Kubernetes.

```
Every interaction with Kubernetes goes through the API server.
kubectl is how YOU talk to the API server.

You ──► kubectl ──► API Server ──► Cluster

Without kubectl, you can't:
  - Create pods
  - Deploy applications
  - Check what's running
  - Debug problems
  - Anything at all

kubectl is to Kubernetes what docker is to Docker.
```

---

## Installing kubectl

```bash
# Linux:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify:
kubectl version --client
# Client Version: v1.32.x
```

---

## kubeconfig — How kubectl Knows Which Cluster

```bash
# kubectl reads its config from:
~/.kube/config

# This file tells kubectl:
#   1. Which cluster to connect to (server URL)
#   2. Who you are (credentials)
#   3. Which namespace to use by default
```

```yaml
# ~/.kube/config (simplified)
apiVersion: v1
kind: Config

clusters:                          # WHICH cluster
- cluster:
    server: https://10.12.24.141:6443
    certificate-authority-data: LS0t...
  name: my-cluster

users:                             # WHO you are
- name: admin
  user:
    client-certificate-data: LS0t...
    client-key-data: LS0t...

contexts:                          # Cluster + User + Namespace
- context:
    cluster: my-cluster
    user: admin
    namespace: default
  name: my-context

current-context: my-context        # Which context to use
```

```bash
# View current config:
kubectl config view

# Which cluster am I connected to?
kubectl config current-context

# Switch context (if you have multiple clusters):
kubectl config use-context production-cluster

# List all contexts:
kubectl config get-contexts
```

---

## kubectl Basics — Your First Commands

### Cluster Info

```bash
# Is the cluster running?
kubectl cluster-info
# Kubernetes control plane is running at https://10.12.24.141:6443
# CoreDNS is running at https://10.12.24.141:6443/api/v1/...

# What nodes are in the cluster?
kubectl get nodes
# NAME       STATUS   ROLES           AGE    VERSION
# node-1     Ready    control-plane   30d    v1.32.0
# node-2     Ready    <none>          30d    v1.32.0
# node-3     Ready    <none>          30d    v1.32.0

# Detailed node info:
kubectl describe node node-1
```

### The Core Pattern: kubectl [verb] [resource]

```bash
# The basic pattern:
kubectl get <resource>           # List resources
kubectl describe <resource>      # Detailed info about a resource
kubectl create <resource>        # Create a resource
kubectl apply -f <file>          # Create or update from a YAML file
kubectl delete <resource>        # Delete a resource
kubectl edit <resource>          # Edit a running resource
kubectl logs <pod>               # View pod logs
kubectl exec <pod> -- <command>  # Run a command in a pod
```

### get — List Things

```bash
# List pods:
kubectl get pods
kubectl get pods -o wide              # more columns (node, IP)
kubectl get pods -A                   # ALL namespaces
kubectl get pods -n kube-system       # specific namespace

# List other resources:
kubectl get deployments
kubectl get services
kubectl get nodes
kubectl get namespaces
kubectl get configmaps
kubectl get secrets
kubectl get all                       # pods + services + deployments

# Output formats:
kubectl get pods -o yaml              # full YAML output
kubectl get pods -o json              # JSON output
kubectl get pods -o name              # just the names
kubectl get pods -o wide              # extra columns
```

### describe — Detailed Info

```bash
# Describe a specific pod:
kubectl describe pod nginx-abc123

# This shows EVERYTHING about the pod:
#   - Name, namespace, node, IP
#   - Labels and annotations
#   - Containers (image, ports, environment)
#   - Conditions (Ready, Initialized, etc.)
#   - Events (what happened to this pod)

# Events are at the bottom — most useful for debugging:
# Events:
#   Type     Reason     Age   Message
#   ----     ------     ---   -------
#   Normal   Scheduled  5m    Successfully assigned to node-2
#   Normal   Pulled     5m    Container image "nginx" already present
#   Normal   Created    5m    Created container nginx
#   Normal   Started    5m    Started container nginx

# If a pod fails, the events will tell you why:
#   Warning  Failed     2m    Error: ImagePullBackOff
#   Warning  Failed     1m    Error: CrashLoopBackOff
```

### create and delete — Imperative Commands

```bash
# Quick way to create resources (imperative — for testing):
kubectl create deployment nginx --image=nginx
kubectl create deployment nginx --image=nginx --replicas=3

# Expose a deployment as a service:
kubectl expose deployment nginx --port=80 --type=ClusterIP

# Delete:
kubectl delete pod nginx-abc123
kubectl delete deployment nginx
kubectl delete service nginx

# Delete everything in a namespace:
kubectl delete all --all -n my-namespace
```

### apply — The Declarative Way (Preferred)

```bash
# Create or update from a YAML file:
kubectl apply -f deployment.yaml

# Apply all YAML files in a directory:
kubectl apply -f ./manifests/

# Apply from a URL:
kubectl apply -f https://example.com/deployment.yaml

# This is the PREFERRED way to manage Kubernetes.
# You write YAML files, commit them to Git, and apply them.
# This is called "GitOps" — Git is the source of truth.
```

### logs — View Container Output

```bash
# View pod logs:
kubectl logs nginx-abc123

# Follow logs (like tail -f):
kubectl logs -f nginx-abc123

# Last N lines:
kubectl logs --tail=50 nginx-abc123

# Logs from a specific container (multi-container pod):
kubectl logs nginx-abc123 -c sidecar

# Previous container's logs (if it restarted):
kubectl logs nginx-abc123 --previous
```

### exec — Run Commands Inside a Pod

```bash
# Run a command in a pod:
kubectl exec nginx-abc123 -- ls /etc/nginx/

# Open an interactive shell:
kubectl exec -it nginx-abc123 -- /bin/bash
kubectl exec -it nginx-abc123 -- /bin/sh     # if bash isn't available

# This is like: docker exec -it <container> /bin/bash
# Useful for debugging:
kubectl exec -it nginx-abc123 -- curl localhost:80
kubectl exec -it nginx-abc123 -- cat /etc/nginx/nginx.conf
kubectl exec -it nginx-abc123 -- env    # see environment variables
```

---

## YAML — The Language of Kubernetes

Every Kubernetes resource is defined in YAML. You **must** understand YAML to use Kubernetes.

### YAML Basics

```yaml
# YAML is like JSON but more readable.
# Indentation matters (use spaces, NEVER tabs).
# Comments start with #

# Key-value pairs:
name: nginx
replicas: 3
enabled: true

# Nested objects (indent with 2 spaces):
metadata:
  name: my-app
  namespace: default
  labels:
    app: nginx
    version: "1.0"

# Lists (dash + space):
ports:
  - 80
  - 443
  - 8080

# List of objects:
containers:
  - name: web
    image: nginx:1.25
    ports:
      - containerPort: 80
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "echo hello"]
```

### Common YAML Mistakes

```yaml
# WRONG: tabs instead of spaces
name:	nginx          # ← TAB character. YAML rejects this.

# WRONG: inconsistent indentation
metadata:
  name: my-app
   namespace: default  # ← 3 spaces instead of 2. YAML fails.

# WRONG: missing space after colon
name:nginx             # ← needs a space: name: nginx

# WRONG: number interpreted as string
replicas: "3"          # ← this is a string "3", not the number 3
replicas: 3            # ← this is correct (number)

# TRICKY: strings that look like other types
version: 1.0           # ← YAML sees this as number 1.0
version: "1.0"         # ← this is the string "1.0" (usually what you want)

name: true             # ← YAML sees boolean true
name: "true"           # ← string "true"
```

---

## The Four Required Fields

Every Kubernetes YAML file has the same four top-level fields:

```yaml
apiVersion: apps/v1         # 1. Which API version
kind: Deployment             # 2. What type of resource
metadata:                    # 3. Data about the resource
  name: my-app
  namespace: default
spec:                        # 4. The specification (what you want)
  replicas: 3
  # ... rest of the spec
```

```
1. apiVersion — Which version of the Kubernetes API to use
   v1              → Core resources (Pod, Service, ConfigMap, Secret)
   apps/v1         → Deployments, ReplicaSets, DaemonSets, StatefulSets
   batch/v1        → Jobs, CronJobs
   networking.k8s.io/v1 → Ingress, NetworkPolicy
   rbac.authorization.k8s.io/v1 → Roles, ClusterRoles

2. kind — What type of resource
   Pod, Deployment, Service, ConfigMap, Secret, Ingress, etc.

3. metadata — Information ABOUT the resource
   name:        The name (required)
   namespace:   Which namespace (default: "default")
   labels:      Key-value pairs for organizing
   annotations: Key-value pairs for metadata

4. spec — What you WANT (the desired state)
   Different for each kind of resource
   This is where all the configuration goes
```

### Example: A Complete Pod YAML

```yaml
apiVersion: v1                    # Core API (Pods are core)
kind: Pod                         # This is a Pod
metadata:
  name: my-nginx                  # Pod name
  namespace: default              # In the default namespace
  labels:                         # Labels for organizing
    app: nginx
    environment: dev
spec:
  containers:                     # What containers to run
    - name: nginx                 # Container name
      image: nginx:1.25           # Docker image
      ports:
        - containerPort: 80       # Expose port 80
```

### Example: A Complete Deployment YAML

```yaml
apiVersion: apps/v1               # apps API (Deployments)
kind: Deployment                   # This is a Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                      # Run 3 copies
  selector:                        # How to find pods this Deployment manages
    matchLabels:
      app: nginx
  template:                        # Template for creating Pods
    metadata:
      labels:
        app: nginx                 # Must match selector above
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```
Notice the structure:
  Deployment
    ├── metadata (about the Deployment itself)
    ├── spec
    │   ├── replicas: 3
    │   ├── selector (how to find pods)
    │   └── template (the Pod template)
    │       ├── metadata (about the Pods)
    │       └── spec (the Pod spec — containers, volumes, etc.)
```

---

## Imperative vs Declarative

```
Imperative (commands — quick testing):
  kubectl create deployment nginx --image=nginx --replicas=3
  kubectl scale deployment nginx --replicas=5
  kubectl set image deployment nginx nginx=nginx:1.26
  
  ✓ Quick for testing
  ✗ No record of what you did
  ✗ Can't version control
  ✗ Can't reproduce

Declarative (YAML files — production):
  Write deployment.yaml with replicas: 3
  kubectl apply -f deployment.yaml
  Edit file: replicas: 5
  kubectl apply -f deployment.yaml
  Commit to Git
  
  ✓ Version controlled
  ✓ Reproducible
  ✓ Reviewable (pull requests)
  ✓ GitOps ready
  ✗ More verbose

Rule: Use imperative for learning and quick tests.
      Use declarative (YAML + apply) for everything else.
```

---

## Generating YAML (Don't Write From Scratch)

You don't have to memorize YAML structures. Generate them:

```bash
# Generate a Deployment YAML without creating it:
kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml

# Output:
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
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
      - image: nginx
        name: nginx

# Save it to a file:
kubectl create deployment nginx --image=nginx --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

# Then edit and apply:
vim deployment.yaml
kubectl apply -f deployment.yaml
```

```bash
# Generate other resources:
kubectl create service clusterip my-service --tcp=80:80 \
  --dry-run=client -o yaml > service.yaml

kubectl create configmap my-config --from-literal=key=value \
  --dry-run=client -o yaml > configmap.yaml

kubectl create secret generic my-secret --from-literal=password=mysecret \
  --dry-run=client -o yaml > secret.yaml

# --dry-run=client  → don't actually create, just generate
# -o yaml           → output as YAML
```

---

## Useful kubectl Tricks

### Aliases

```bash
# kubectl is a lot to type. Use an alias:
alias k=kubectl

# Now:
k get pods
k get nodes
k describe pod nginx-abc123

# Add to ~/.bashrc to make it permanent:
echo 'alias k=kubectl' >> ~/.bashrc
```

### Tab Completion

```bash
# Enable bash completion:
source <(kubectl completion bash)

# Make permanent:
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# Now you can tab-complete:
kubectl get pod<TAB>     → shows available pods
kubectl desc<TAB>        → completes to "describe"
```

### Watching Resources

```bash
# Watch pods in real-time (updates every 2 seconds):
kubectl get pods -w

# Or use the watch command:
watch kubectl get pods

# Watch with wide output:
watch kubectl get pods -o wide
```

### Getting Help

```bash
# Explain a resource (shows all fields):
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy

# This is your built-in documentation.
# Better than searching the internet.
```

### Multiple Resources in One File

```yaml
# You can put multiple resources in one YAML file
# Separate them with ---

apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  key: value
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  # ... rest of deployment
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  # ... rest of service
```

---

## kubectl Cheat Sheet

```
Cluster:
  kubectl cluster-info                   Cluster status
  kubectl get nodes                      List nodes
  kubectl get nodes -o wide              Nodes with IPs

Pods:
  kubectl get pods                       List pods
  kubectl get pods -A                    All namespaces
  kubectl get pods -o wide               Show node & IP
  kubectl describe pod <name>            Pod details
  kubectl logs <pod>                     Pod logs
  kubectl logs -f <pod>                  Follow logs
  kubectl exec -it <pod> -- /bin/sh      Shell into pod
  kubectl delete pod <name>              Delete pod

Deployments:
  kubectl get deployments                List deployments
  kubectl describe deployment <name>     Details
  kubectl scale deployment <name> --replicas=5
  kubectl rollout status deployment <name>
  kubectl rollout undo deployment <name>

Services:
  kubectl get services                   List services
  kubectl describe service <name>        Details

Apply/Create:
  kubectl apply -f file.yaml             Create/update
  kubectl delete -f file.yaml            Delete
  kubectl create -f file.yaml            Create only

Generate:
  kubectl create deployment X --image=Y --dry-run=client -o yaml

Debug:
  kubectl describe <resource> <name>     Check Events section
  kubectl logs <pod>                     Container output
  kubectl logs <pod> --previous          Previous crash logs
  kubectl get events                     Cluster events
  kubectl top pods                       CPU/memory usage
  kubectl top nodes                      Node resource usage
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
kubectl                    → Your primary K8s tool (daily use)
                           → CI/CD pipelines use kubectl to deploy
YAML                       → Every K8s config is YAML
                           → Stored in Git (GitOps)
                           → Reviewed in pull requests
kubeconfig                 → Manage multiple clusters
                           → Dev / staging / production contexts
--dry-run + -o yaml        → Generate manifests quickly
                           → CKA/CKAD exam essential skill
Declarative (apply)        → GitOps (ArgoCD, Flux)
                           → Infrastructure as Code
kubectl explain             → Self-documentation
                           → Faster than searching docs
```

---

> **Next:** 04 — Pods → The smallest deployable unit in Kubernetes.
