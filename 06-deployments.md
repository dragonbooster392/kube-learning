# 06 — Deployments

## What Is a Deployment

A Deployment is the **most common way to run applications in Kubernetes**. It manages ReplicaSets, which manage Pods. You tell it what you want, and it handles everything.

```
What a Deployment gives you (that bare Pods and ReplicaSets don't):
  ✓ Self-healing (from ReplicaSet)
  ✓ Scaling (from ReplicaSet)
  ✓ Rolling updates (change image version with zero downtime)
  ✓ Rollback (go back to a previous version)
  ✓ Pause and resume (partially roll out an update)
  ✓ Version history (track what changed)

This is what you use 90% of the time in Kubernetes.
```

---

## Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                            # Run 3 pods
  selector:
    matchLabels:
      app: nginx                         # Manage pods with this label
  template:
    metadata:
      labels:
        app: nginx                       # Pod labels (must match selector)
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "250m"
```

```
This looks almost identical to a ReplicaSet YAML.
The only difference: kind: Deployment instead of kind: ReplicaSet

That's because a Deployment wraps a ReplicaSet.
You define the same things — the Deployment passes them to the ReplicaSet it creates.
```

---

## Creating a Deployment

### Imperative (Quick)

```bash
# Create:
kubectl create deployment nginx --image=nginx:1.25 --replicas=3

# Check what was created:
kubectl get deployments
# NAME    READY   UP-TO-DATE   AVAILABLE   AGE
# nginx   3/3     3            3           30s

kubectl get replicasets
# NAME               DESIRED   CURRENT   READY   AGE
# nginx-6d4cf56db6   3         3         3       30s

kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-6d4cf56db6-abc12   1/1     Running   0          30s
# nginx-6d4cf56db6-def34   1/1     Running   0          30s
# nginx-6d4cf56db6-ghi56   1/1     Running   0          30s
```

```
Notice the naming:
  Deployment:   nginx
  ReplicaSet:   nginx-6d4cf56db6        (Deployment name + hash)
  Pod:          nginx-6d4cf56db6-abc12  (ReplicaSet name + random)

  Deployment creates → ReplicaSet creates → Pods
```

### Declarative (YAML — Preferred)

```bash
# Save YAML to file:
kubectl create deployment nginx --image=nginx:1.25 --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

# Edit as needed, then apply:
kubectl apply -f deployment.yaml
```

---

## Scaling

```bash
# Scale to 5 replicas:
kubectl scale deployment nginx --replicas=5

# Or edit the YAML and re-apply:
# replicas: 5
kubectl apply -f deployment.yaml

# Check:
kubectl get deployment nginx
# NAME    READY   UP-TO-DATE   AVAILABLE   AGE
# nginx   5/5     5            5           10m

# Scale down:
kubectl scale deployment nginx --replicas=2
```

---

## Rolling Updates — The Killer Feature

This is why Deployments exist. You can update your application with **zero downtime**.

### How Rolling Updates Work

```
Current state: 3 pods running nginx:1.25

You update the image to nginx:1.26

Step 1: Create 1 new Pod with nginx:1.26
  nginx:1.25 ✓  nginx:1.25 ✓  nginx:1.25 ✓  nginx:1.26 (starting)

Step 2: New Pod is ready. Terminate 1 old Pod.
  nginx:1.25 ✓  nginx:1.25 ✓  nginx:1.26 ✓  (nginx:1.25 terminating)

Step 3: Create another new Pod.
  nginx:1.25 ✓  nginx:1.26 ✓  nginx:1.26 (starting)

Step 4: New Pod ready. Terminate old.
  nginx:1.26 ✓  nginx:1.26 ✓  (nginx:1.25 terminating)

Step 5: Create last new Pod.
  nginx:1.26 ✓  nginx:1.26 ✓  nginx:1.26 (starting)

Step 6: Done!
  nginx:1.26 ✓  nginx:1.26 ✓  nginx:1.26 ✓

At NO point were zero pods running. Zero downtime.
```

```
What actually happens with ReplicaSets:

  Before update:
    ReplicaSet-OLD (nginx:1.25): 3 pods
    
  During update:
    ReplicaSet-OLD (nginx:1.25): scaling DOWN (3 → 2 → 1 → 0)
    ReplicaSet-NEW (nginx:1.26): scaling UP   (0 → 1 → 2 → 3)
    
  After update:
    ReplicaSet-OLD (nginx:1.25): 0 pods (kept for rollback history)
    ReplicaSet-NEW (nginx:1.26): 3 pods
```

### Triggering a Rolling Update

```bash
# Method 1: Change the image (imperative):
kubectl set image deployment/nginx nginx=nginx:1.26

# Method 2: Edit the YAML and re-apply (declarative — preferred):
# Change image: nginx:1.25 → nginx:1.26 in deployment.yaml
kubectl apply -f deployment.yaml

# Method 3: Edit live resource:
kubectl edit deployment nginx
# Opens in your editor. Change image. Save and close.

# Watch the rollout happen:
kubectl rollout status deployment/nginx
# Waiting for deployment "nginx" rollout to finish:
#   1 out of 3 new replicas have been updated...
#   2 out of 3 new replicas have been updated...
#   3 out of 3 new replicas have been updated...
# deployment "nginx" successfully rolled out
```

### Rollout Strategy

```yaml
spec:
  strategy:
    type: RollingUpdate          # default
    rollingUpdate:
      maxUnavailable: 1          # at most 1 pod can be down during update
      maxSurge: 1                # at most 1 extra pod can exist during update
```

```
maxUnavailable: 1
  During update, at least 2 of 3 pods are always available.
  (3 desired - 1 unavailable = 2 minimum)

maxSurge: 1
  During update, at most 4 pods can exist total.
  (3 desired + 1 surge = 4 maximum)

You can use numbers or percentages:
  maxUnavailable: 25%    # 25% of desired can be down
  maxSurge: 25%          # 25% extra pods allowed

For zero-downtime with 3 replicas:
  maxUnavailable: 0      # no pods go down
  maxSurge: 1            # create new before removing old
  → This guarantees 3 pods are always available
```

### Recreate Strategy (Not Rolling)

```yaml
spec:
  strategy:
    type: Recreate
```

```
Recreate: Kill ALL old pods first, then create ALL new pods.
  
  Step 1: 3 pods nginx:1.25 → all terminated
  Step 2: 0 pods running (DOWNTIME!)
  Step 3: 3 pods nginx:1.26 → all created

When to use:
  - Application can't run two versions simultaneously
  - Database migrations that require exclusive access
  - Development environments where downtime is OK
```

---

## Rollbacks

If the new version is broken, roll back to the previous version.

```bash
# View rollout history:
kubectl rollout history deployment/nginx
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# See details of a specific revision:
kubectl rollout history deployment/nginx --revision=1

# Roll back to previous version:
kubectl rollout undo deployment/nginx

# Roll back to a specific revision:
kubectl rollout undo deployment/nginx --to-revision=1

# Check status:
kubectl rollout status deployment/nginx
```

```
What happens during rollback:

  Current:   ReplicaSet-v2 (nginx:1.26): 3 pods
  Rollback:  ReplicaSet-v1 (nginx:1.25): 0 pods → scaling UP
             ReplicaSet-v2 (nginx:1.26): 3 pods → scaling DOWN

  Result:    ReplicaSet-v1 (nginx:1.25): 3 pods ✓
             ReplicaSet-v2 (nginx:1.26): 0 pods

  Kubernetes kept the old ReplicaSet around for exactly this purpose!
```

### Recording Changes

```bash
# Add a change cause (so rollout history is useful):
kubectl annotate deployment/nginx kubernetes.io/change-cause="Updated to nginx 1.26"

# Now history shows:
kubectl rollout history deployment/nginx
# REVISION  CHANGE-CAUSE
# 1         Initial deployment
# 2         Updated to nginx 1.26

# Control how many old ReplicaSets to keep:
spec:
  revisionHistoryLimit: 10    # keep last 10 revisions (default)
```

---

## Pausing and Resuming Rollouts

```bash
# Pause a rollout (make changes without triggering update):
kubectl rollout pause deployment/nginx

# Now make multiple changes:
kubectl set image deployment/nginx nginx=nginx:1.26
kubectl set resources deployment/nginx -c=nginx --limits=memory=256Mi

# Resume (applies all changes at once):
kubectl rollout resume deployment/nginx
```

---

## Complete Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
  labels:
    app: web
    version: "2.0"
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: web
        version: "2.0"
    spec:
      containers:
        - name: web
          image: my-company/web-app:2.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_HOST
              value: "db-service"
            - name: LOG_LEVEL
              value: "info"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

```
This Deployment:
  - Runs 3 copies of my-web-app:2.0
  - Uses rolling updates (max 1 down, max 1 extra)
  - Sets environment variables
  - Defines resource limits
  - Has health checks (covered in file 14)
  - Keeps 5 revisions for rollback
```

---

## Deployment Status Fields

```bash
kubectl get deployment my-web-app
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# my-web-app   3/3     3            3           10m

# READY:       3/3 — 3 of 3 desired pods are ready
# UP-TO-DATE:  3   — 3 pods have the latest template
# AVAILABLE:   3   — 3 pods are available to serve traffic

# During a rolling update, you might see:
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# my-web-app   3/3     1            3           10m
#              ^^^     ^
#              still 3 ready, but only 1 has the new version
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Deployments                → 90% of everything you deploy in K8s
Rolling updates            → Zero-downtime deployments
                           → CI/CD pipelines (Jenkins, GitLab CI, ArgoCD)
Rollbacks                  → Instant recovery from bad deployments
                           → "kubectl rollout undo" saves the day
Strategy                   → Controlling deployment risk
                           → Canary deployments (with more tools)
Revision history           → Audit trail of deployments
                           → GitOps tracks this in Git
replicas + HPA             → Auto-scaling (covered later)
                           → Handle traffic spikes automatically
```

---

> **Next:** 07 — Services → How pods communicate and how traffic reaches your app.
