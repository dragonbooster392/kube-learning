# 17 — Scheduling

## How the Scheduler Works

When you create a Pod, the **kube-scheduler** decides which node to place it on.

```
Scheduling steps:

  1. FILTERING:  Remove nodes that CAN'T run the Pod
     - Not enough CPU or memory?  → excluded
     - Node has a taint the Pod can't tolerate?  → excluded
     - Pod needs specific node label?  → only matching nodes
     
  2. SCORING:  Rank remaining nodes
     - Which node has the most free resources?  → higher score
     - Which node already has the image cached?  → higher score
     - Which node balances the spread best?  → higher score
     
  3. BINDING:  Assign Pod to the highest-scoring node
```

```
You can influence scheduling in several ways:
  - nodeSelector:     "Put me on a node with this label"
  - Node Affinity:    "Prefer nodes with these labels" (flexible)
  - Taints/Tolerations: "This node rejects pods unless they tolerate it"
  - Pod Affinity:     "Put me near (or far from) other specific pods"
```

---

## nodeSelector — Simple Node Selection

The simplest way to control where a Pod runs.

```bash
# First, label your nodes:
kubectl label node worker-1 disk=ssd
kubectl label node worker-2 disk=hdd
kubectl label node worker-3 disk=ssd

# Check labels:
kubectl get nodes --show-labels
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fast-app
spec:
  nodeSelector:
    disk: ssd                      # only run on nodes with disk=ssd
  containers:
    - name: app
      image: my-app:1.0
```

```
This Pod will ONLY be scheduled on worker-1 or worker-3 (disk=ssd).
If no node matches → Pod stays in Pending state.
```

---

## Node Affinity — Flexible Node Selection

Node affinity is like nodeSelector but more powerful.

### Required (must match)

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disk
                operator: In
                values:
                  - ssd
                  - nvme
```

```
"Schedule ONLY on nodes where disk is ssd OR nvme"
If no matching node exists → Pod stays Pending.
```

### Preferred (try to match)

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80                    # priority (1-100)
          preference:
            matchExpressions:
              - key: disk
                operator: In
                values:
                  - ssd
        - weight: 20
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values:
                  - us-east-1a
```

```
"PREFER nodes with SSD disk (weight 80) and zone us-east-1a (weight 20).
But if no matching node is available, schedule anywhere."

Preferred is a soft requirement — the Pod still gets scheduled.
```

### Operators

```
In:           value is in the list
NotIn:        value is NOT in the list
Exists:       key exists (any value)
DoesNotExist: key does NOT exist
Gt:           value is greater than
Lt:           value is less than
```

---

## Taints and Tolerations

Taints are on **nodes** — they repel Pods.
Tolerations are on **Pods** — they allow Pods to be scheduled on tainted nodes.

```
Think of it like a "No Parking" sign:

  Taint (on node):       "No parking here!" (repels all pods)
  Toleration (on pod):   "I have a parking permit" (can park here)
  
  Only Pods with the right toleration can be scheduled on a tainted node.
```

### Adding Taints to Nodes

```bash
# Taint a node:
kubectl taint nodes worker-3 gpu=true:NoSchedule
#                   node      key=value:effect

# Effects:
#   NoSchedule:       New pods won't be scheduled (existing pods stay)
#   PreferNoSchedule: Try to avoid scheduling here (soft)
#   NoExecute:        Evict existing pods AND prevent new ones

# Remove a taint (minus sign at the end):
kubectl taint nodes worker-3 gpu=true:NoSchedule-

# View taints:
kubectl describe node worker-3 | grep Taint
```

### Adding Tolerations to Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: app
      image: gpu-app:1.0
```

```
worker-3 has taint:   gpu=true:NoSchedule
gpu-app has toleration: gpu=true:NoSchedule

Result: gpu-app CAN be scheduled on worker-3.
        Regular pods (without toleration) CANNOT.
```

### Common Taint Use Cases

```
Dedicated GPU nodes:
  Taint:      kubectl taint nodes gpu-node-1 gpu=true:NoSchedule
  Toleration: Only GPU workloads tolerate this taint
  Result:     GPU nodes are reserved for GPU workloads

Maintenance:
  Taint:      kubectl taint nodes worker-1 maintenance=true:NoExecute
  Result:     All pods are evicted from worker-1 (for maintenance)

Control plane nodes:
  Already tainted:  node-role.kubernetes.io/control-plane:NoSchedule
  Result:           User pods don't run on control plane nodes
```

---

## Pod Affinity and Anti-Affinity

Control whether Pods should be **near** or **far from** other Pods.

### Pod Affinity — "Schedule Near These Pods"

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: cache                # near pods with app=cache
          topologyKey: kubernetes.io/hostname  # "same node"
```

```
"Schedule this Pod on the same NODE as pods with app=cache"

Use case: Put your app on the same node as its cache (low latency).
```

### Pod Anti-Affinity — "Schedule Away From These Pods"

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: web                 # away from other app=web pods
          topologyKey: kubernetes.io/hostname  # "different nodes"
```

```
"Don't schedule this Pod on the same NODE as other pods with app=web"

Use case: Spread replicas across different nodes for high availability.
If one node goes down, not all replicas are lost.
```

### Topology Keys

```
kubernetes.io/hostname:             same/different node
topology.kubernetes.io/zone:        same/different availability zone
topology.kubernetes.io/region:      same/different region

Example: spread web pods across availability zones:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: topology.kubernetes.io/zone
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
      # Prefer SSD nodes
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              preference:
                matchExpressions:
                  - key: disk
                    operator: In
                    values: ["ssd"]
        # Spread across nodes
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: web
                topologyKey: kubernetes.io/hostname

      # Tolerate spot instances
      tolerations:
        - key: "spot"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"

      containers:
        - name: app
          image: my-app:1.0
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
```

```
This Pod:
  1. PREFERS SSD nodes (but will accept any)
  2. SPREADS replicas across different nodes
  3. CAN run on spot-instance nodes (has toleration)
  4. Reserves 250m CPU and 256Mi memory (scheduler considers this)
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
nodeSelector               → Simple GPU/SSD/region targeting
Node Affinity              → Cloud zone placement for compliance
                           → Dedicated node pools
Taints/Tolerations         → Reserving nodes for specific workloads
                           → Node maintenance without downtime
                           → Spot/preemptible instance management
Pod Anti-Affinity          → High availability (spread across nodes/zones)
                           → Database replicas on different nodes
Pod Affinity               → Co-locating related services (app + cache)
                           → Reducing network latency
```

---

> **Next:** 18 — Networking Deep Dive → How Pods talk to each other.
