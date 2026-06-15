# 15 — Resource Management

## The Problem: Noisy Neighbors

```
A cluster has limited CPU and memory.
If one app uses all of it, other apps starve.

Node: 4 CPU, 8 GB RAM
  App A: uses 3.5 CPU (runaway process)
  App B: gets 0.5 CPU (starving, barely responding)
  App C: can't even start (no resources left)

This is the "noisy neighbor" problem.
Kubernetes solves it with resource requests and limits.
```

---

## Requests vs Limits

```
Request:  "I need at LEAST this much"
          Used for SCHEDULING — the scheduler finds a node with enough room.

Limit:    "I must NEVER use more than this"
          Used for ENFORCEMENT — the container is restricted.
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
    - name: app
      image: my-app:1.0
      resources:
        requests:
          cpu: "250m"              # request 250 millicores (0.25 CPU)
          memory: "128Mi"          # request 128 MiB
        limits:
          cpu: "500m"              # max 500 millicores (0.5 CPU)
          memory: "256Mi"          # max 256 MiB
```

```
What this means:
  Scheduling: "Find a node with at least 0.25 CPU and 128Mi free"
  Running:    "Container can use up to 0.5 CPU and 256Mi"
  
  If container tries to use more than 256Mi memory → KILLED (OOMKilled)
  If container tries to use more than 0.5 CPU → THROTTLED (slowed down)
```

---

## CPU Units

```
1 CPU = 1 core = 1 vCPU (cloud) = 1 hyperthread

Millicores (m):
  1000m = 1 CPU
  500m  = 0.5 CPU (half a core)
  250m  = 0.25 CPU (quarter of a core)
  100m  = 0.1 CPU

You can also use decimals:
  0.5   = 500m
  0.1   = 100m

Common values:
  Small app:   100m - 250m request, 500m limit
  Medium app:  250m - 500m request, 1000m limit
  Large app:   1000m+ request, 2000m+ limit
```

```
CPU is compressible:
  If a container hits its CPU limit, it gets THROTTLED (slowed down).
  It still runs, just slower.
  The container is NOT killed.
```

---

## Memory Units

```
Binary units (power of 2):
  Ki = kibibyte (1024 bytes)
  Mi = mebibyte (1024 Ki)
  Gi = gibibyte (1024 Mi)
  Ti = tebibyte (1024 Gi)

Decimal units (power of 10):
  k = kilobyte (1000 bytes)
  M = megabyte (1000 k)
  G = gigabyte (1000 M)

Use binary units (Mi, Gi) — they match how computers actually work.

Common values:
  Small app:   64Mi - 128Mi request, 256Mi limit
  Medium app:  256Mi - 512Mi request, 1Gi limit
  Large app:   1Gi+ request, 2Gi+ limit
```

```
Memory is NOT compressible:
  If a container exceeds its memory limit → KILLED (OOMKilled)
  Kubernetes restarts it (based on restartPolicy)
  
  This is why memory limits are important:
    Too low  → container keeps getting OOMKilled
    Too high → wasting cluster resources
```

---

## What Happens Without Requests and Limits

```
No requests:
  Scheduler doesn't reserve any resources for the Pod.
  Pod can be placed on any node.
  But it might not get the resources it needs.

No limits:
  Container can use ALL available resources on the node.
  It can starve other containers.
  This is the "noisy neighbor" problem.

Best practice: ALWAYS set both requests and limits.
```

---

## QoS Classes

Kubernetes assigns a Quality of Service class to each Pod based on its resource settings.

```
Guaranteed:
  Every container has requests AND limits.
  Requests EQUAL limits.
  Last to be evicted when the node runs low on resources.
  
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "500m"          ← same as request
      memory: "256Mi"      ← same as request

Burstable:
  At least one container has requests OR limits set.
  Requests are LESS than limits.
  Evicted after BestEffort pods, before Guaranteed.
  
  resources:
    requests:
      cpu: "250m"
      memory: "128Mi"
    limits:
      cpu: "500m"          ← higher than request
      memory: "256Mi"      ← higher than request

BestEffort:
  No requests or limits set at all.
  First to be evicted when the node is under pressure.
  
  resources: {}            ← nothing set
```

```
Eviction order when a node runs out of memory:
  1. BestEffort pods → killed first
  2. Burstable pods  → killed next (using most above request)
  3. Guaranteed pods → killed last (only if absolutely necessary)

For production workloads, use Guaranteed or Burstable.
Never use BestEffort in production.
```

---

## LimitRange — Namespace Defaults

A LimitRange sets default requests/limits for a namespace, so developers don't have to remember to set them.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:                     # default limits (if not specified)
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:              # default requests (if not specified)
        cpu: "100m"
        memory: "128Mi"
      max:                         # maximum allowed
        cpu: "2"
        memory: "2Gi"
      min:                         # minimum allowed
        cpu: "50m"
        memory: "64Mi"
```

```
With this LimitRange in the "production" namespace:

  Pod with no resources set:
    → Gets default: request 100m/128Mi, limit 500m/256Mi

  Pod requesting 4 CPU:
    → REJECTED (max is 2 CPU)

  Pod requesting 10m CPU:
    → REJECTED (min is 50m CPU)
```

---

## ResourceQuota — Namespace Totals

A ResourceQuota limits the TOTAL resources a namespace can use.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-frontend
spec:
  hard:
    requests.cpu: "10"             # total CPU requests across all pods
    requests.memory: "20Gi"        # total memory requests
    limits.cpu: "20"               # total CPU limits
    limits.memory: "40Gi"          # total memory limits
    pods: "50"                     # max 50 pods in this namespace
    services: "10"                 # max 10 services
    persistentvolumeclaims: "20"   # max 20 PVCs
```

```bash
# Check quota usage:
kubectl get resourcequota -n team-frontend
# NAME         AGE   REQUEST                                           LIMIT
# team-quota   1h    requests.cpu: 4/10, requests.memory: 8Gi/20Gi    limits.cpu: 8/20
```

---

## Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of Pod replicas based on CPU, memory, or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app                    # scale this Deployment
  minReplicas: 2                     # never go below 2
  maxReplicas: 10                    # never go above 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70     # target 70% CPU usage
```

```
How HPA works:
  1. Monitors average CPU usage across all pods
  2. If average > 70% → add more pods (scale up)
  3. If average < 70% → remove pods (scale down)
  4. Never fewer than 2, never more than 10

  Low traffic:  2 pods at 30% CPU → stays at 2
  Medium:       3 pods at 65% CPU → stays at 3
  High traffic: 5 pods at 80% CPU → scales to 7 (to bring average below 70%)
  Traffic drops: 7 pods at 20% CPU → scales down to 2
```

```bash
# Create HPA with kubectl:
kubectl autoscale deployment web-app \
  --min=2 --max=10 --cpu-percent=70

# Check HPA status:
kubectl get hpa
# NAME      REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS
# web-hpa   Deployment/web-app  45%/70%   2         10        3

# metrics-server must be installed for HPA to work:
kubectl top pods                    # requires metrics-server
kubectl top nodes                   # requires metrics-server
```

---

## Complete Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: app
          image: my-app:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Requests & Limits          → Every production pod MUST have these
                           → Prevents noisy neighbors
                           → Enables proper scheduling
QoS Classes                → Understanding eviction priority
                           → Critical pods should be Guaranteed
LimitRange                 → Platform teams set namespace defaults
                           → Prevents resource abuse
ResourceQuota              → Multi-tenant clusters
                           → Team budgets / cost control
HPA                        → Auto-scaling for traffic spikes
                           → Cost optimization (scale down when idle)
                           → Combined with cluster autoscaler for full auto-scaling
```

---

> **Next:** 16 — RBAC & Security → Who can do what in Kubernetes.
