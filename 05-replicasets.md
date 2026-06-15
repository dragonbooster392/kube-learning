# 05 — ReplicaSets

## The Problem: One Pod Is Not Enough

If you run a single Pod and it crashes, your application is down. Nobody can access it until someone notices and creates a new Pod.

```
1 Pod running:
  my-nginx → handling traffic ✓

Pod crashes:
  my-nginx → GONE
  Traffic → ??? → DOWN ✗
  
  You notice 30 minutes later.
  You create a new Pod manually.
  30 minutes of downtime.
```

**ReplicaSets solve this.** They make sure a specific number of identical Pods are always running.

---

## What Is a ReplicaSet

A ReplicaSet is a Kubernetes object that says: **"Always keep N copies of this Pod running."**

```
ReplicaSet: replicas: 3

  Pod 1 (running) ✓
  Pod 2 (running) ✓
  Pod 3 (running) ✓
  
  Pod 2 crashes:
  Pod 1 (running) ✓
  Pod 2 (GONE)    ✗    → ReplicaSet notices!
  Pod 3 (running) ✓
  
  ReplicaSet creates a replacement:
  Pod 1 (running) ✓
  Pod 4 (running) ✓    → NEW Pod (different name, maybe different node)
  Pod 3 (running) ✓
  
  Always 3 Pods. Self-healing. Automatic.
```

---

## How a ReplicaSet Works

```
The ReplicaSet control loop (runs continuously):

  1. Count pods that match my selector     → found 2
  2. Compare to desired replicas           → want 3
  3. 2 < 3 → create 1 more Pod
  4. Wait... now 3 pods running
  5. Go back to step 1

  What if too many pods? (someone manually created extras)
  1. Count pods that match my selector     → found 4
  2. Compare to desired replicas           → want 3
  3. 4 > 3 → delete 1 Pod
  4. Wait... now 3 pods running

  It ALWAYS maintains the exact count.
```

---

## ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3                        # How many Pods to run
  selector:                          # How to find Pods that belong to this RS
    matchLabels:
      app: nginx
  template:                          # Template for creating new Pods
    metadata:
      labels:
        app: nginx                   # MUST match selector.matchLabels
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```
Three key parts:
  1. replicas: 3              → "I want 3 copies"
  2. selector.matchLabels     → "My Pods have label app=nginx"
  3. template                 → "Here's how to create each Pod"

The selector and template labels MUST match.
  selector says: "I manage Pods with app=nginx"
  template says: "New Pods will have label app=nginx"
  If they don't match → error
```

---

## ReplicaSet in Action

```bash
# Create the ReplicaSet:
kubectl apply -f replicaset.yaml

# Check ReplicaSet status:
kubectl get replicaset
# NAME                DESIRED   CURRENT   READY   AGE
# nginx-replicaset    3         3         3       30s

# Check the Pods it created:
kubectl get pods
# NAME                       READY   STATUS    RESTARTS   AGE
# nginx-replicaset-abc12     1/1     Running   0          30s
# nginx-replicaset-def34     1/1     Running   0          30s
# nginx-replicaset-ghi56     1/1     Running   0          30s

# Notice: Pod names = ReplicaSet name + random suffix
```

### Self-Healing in Action

```bash
# Delete one Pod (simulate a crash):
kubectl delete pod nginx-replicaset-abc12

# Immediately check pods:
kubectl get pods
# NAME                       READY   STATUS    RESTARTS   AGE
# nginx-replicaset-def34     1/1     Running   0          5m
# nginx-replicaset-ghi56     1/1     Running   0          5m
# nginx-replicaset-jkl78     1/1     Running   0          3s  ← NEW!

# A new Pod was created automatically!
# Still 3 Pods running, as desired.
```

### Scaling

```bash
# Scale up to 5:
kubectl scale replicaset nginx-replicaset --replicas=5

# Check:
kubectl get pods
# 5 pods now running

# Scale down to 2:
kubectl scale replicaset nginx-replicaset --replicas=2

# Check:
kubectl get pods
# 2 pods now running (3 were terminated)
```

---

## The Selector — How ReplicaSets Find Their Pods

```
The selector is how a ReplicaSet knows which Pods belong to it.
It uses LABELS to find its Pods.

ReplicaSet selector: app=nginx
  → "I own any Pod with the label app=nginx"

If you manually create a Pod with label app=nginx:
  → The ReplicaSet counts it as one of its own!
  → If it now has 4 pods but wants 3, it will DELETE one.
```

```yaml
spec:
  selector:
    matchLabels:
      app: nginx           # Simple label matching

# Or more complex matching:
  selector:
    matchExpressions:
      - key: app
        operator: In
        values: [nginx, web]
      - key: environment
        operator: NotIn
        values: [test]
```

---

## Why You (Almost) Never Create ReplicaSets Directly

```
Here's the thing:

  ReplicaSets are great for self-healing and scaling.
  But they can't do ROLLING UPDATES.

  If you want to change the nginx image from 1.25 to 1.26:
    - ReplicaSet: You'd have to delete the old RS, create a new one.
      All Pods die at once. Downtime.
    - Deployment: Handles this automatically. Zero downtime.

  Deployments CREATE ReplicaSets for you.
  A Deployment manages ReplicaSets, which manage Pods.

  Deployment → ReplicaSet → Pods

  You create Deployments. Deployments create ReplicaSets.
  You almost NEVER create a ReplicaSet directly.
```

```
Hierarchy:

  ┌─────────────────────────────────┐
  │         Deployment              │  ← you create this
  │  (manages rolling updates)      │
  └──────────────┬──────────────────┘
                 │ creates
  ┌──────────────▼──────────────────┐
  │         ReplicaSet              │  ← Deployment creates this
  │  (maintains replica count)      │
  └──────────────┬──────────────────┘
                 │ creates
  ┌──────┐ ┌──────┐ ┌──────┐
  │ Pod  │ │ Pod  │ │ Pod  │        ← ReplicaSet creates these
  └──────┘ └──────┘ └──────┘
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Self-healing               → High availability (pods restart automatically)
                           → No manual intervention needed
Replicas                   → Load distribution across pods
                           → Horizontal scaling
Selectors + Labels         → Foundation of ALL K8s object relationships
                           → Services use the same mechanism
ReplicaSet → Deployment    → Understanding the object hierarchy
                           → Why you create Deployments, not ReplicaSets
```

---

> **Next:** 06 — Deployments → Rolling updates, rollbacks, and production-ready management.
