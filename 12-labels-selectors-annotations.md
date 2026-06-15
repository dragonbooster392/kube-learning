# 12 — Labels, Selectors & Annotations

## Labels — Tags for Your Resources

Labels are key-value pairs attached to Kubernetes objects. They are the primary way Kubernetes **organizes** and **selects** resources.

```
Think of labels like tags on items in a warehouse:

  Box A:  color=red, size=large, department=shipping
  Box B:  color=blue, size=small, department=returns
  Box C:  color=red, size=small, department=shipping

  "Give me all red boxes" → Box A, Box C
  "Give me all shipping boxes that are small" → Box C
  
  That's exactly how labels and selectors work in Kubernetes.
```

---

## Adding Labels

### In YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web                      # label 1
    environment: production       # label 2
    team: frontend                # label 3
    version: v2.1.0              # label 4
spec:
  containers:
    - name: web
      image: nginx:1.25
```

### With kubectl

```bash
# Add a label to an existing resource:
kubectl label pod web-pod tier=frontend

# Update an existing label (use --overwrite):
kubectl label pod web-pod version=v2.2.0 --overwrite

# Remove a label (use minus sign):
kubectl label pod web-pod tier-

# Add labels to multiple resources:
kubectl label pods --all environment=dev
```

---

## Label Naming Rules

```
Key rules:
  - Must be 63 characters or fewer
  - Begin and end with alphanumeric character
  - Can contain: letters, numbers, dashes, underscores, dots
  - Optional prefix: my-company.com/my-label (prefix max 253 chars)

Value rules:
  - Must be 63 characters or fewer
  - Can be empty
  - Begin and end with alphanumeric (if not empty)

Valid:
  app: web
  version: v2.1.0
  app.kubernetes.io/name: frontend
  environment: production

Invalid:
  App: web              (keys are case-sensitive, but convention is lowercase)
  my label: value       (no spaces)
  /invalid: value       (can't start with /)
```

---

## Selectors — Finding Resources by Labels

Selectors filter resources based on their labels. Selectors are used everywhere in Kubernetes.

### Equality-Based Selectors

```bash
# Get pods where app equals "web":
kubectl get pods -l app=web

# Get pods where environment is NOT production:
kubectl get pods -l environment!=production

# Multiple conditions (AND):
kubectl get pods -l app=web,environment=production
# Both conditions must be true.
```

### Set-Based Selectors

```bash
# Get pods where environment is "production" OR "staging":
kubectl get pods -l 'environment in (production, staging)'

# Get pods where environment is NOT "dev":
kubectl get pods -l 'environment notin (dev)'

# Get pods that HAVE the "version" label (any value):
kubectl get pods -l version

# Get pods that DON'T HAVE the "version" label:
kubectl get pods -l '!version'
```

---

## Where Selectors Are Used

### Services Select Pods

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web                       # select Pods with app=web
    environment: production        # AND environment=production
  ports:
    - port: 80
```

### Deployments Select Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  selector:
    matchLabels:
      app: web                     # must match template labels
  template:
    metadata:
      labels:
        app: web                   # these labels must match the selector
        version: v2.1.0
    spec:
      containers:
        - name: web
          image: nginx:1.25
```

### Network Policies Select Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web
spec:
  podSelector:
    matchLabels:
      app: web                     # applies to Pods with app=web
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api             # allow traffic FROM app=api Pods
```

```
Selectors are used by:
  - Services          (to find which Pods to send traffic to)
  - Deployments       (to manage which Pods belong to them)
  - ReplicaSets       (to count their Pods)
  - Network Policies  (to define firewall rules)
  - Node Selectors    (to schedule Pods on specific nodes)
  - Jobs              (to track their Pods)
```

---

## Recommended Labels

Kubernetes has a set of recommended labels that tools understand.

```yaml
metadata:
  labels:
    # Recommended labels (well-known):
    app.kubernetes.io/name: web-frontend
    app.kubernetes.io/instance: web-frontend-abc
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: online-store
    app.kubernetes.io/managed-by: helm
```

```
Using these recommended labels means:
  - Dashboards (like Lens, Rancher) display your apps nicely
  - Helm manages releases using these labels
  - Monitoring tools group metrics correctly
  - Other tools understand your app structure
```

---

## Annotations — Non-Identifying Metadata

Annotations are also key-value pairs, but they are **not used for selecting** resources. They store non-identifying information.

```
Labels:       Used to SELECT and ORGANIZE resources
Annotations:  Used to ATTACH extra info to resources

Labels are for Kubernetes (machine-readable, indexed).
Annotations are for humans and tools (not indexed, any content).
```

### Common Annotations

```yaml
metadata:
  annotations:
    # Descriptive:
    description: "Frontend web server for the online store"
    owner: "frontend-team@company.com"
    
    # Tool-specific:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    
    # Ingress controller:
    nginx.ingress.kubernetes.io/rewrite-target: /
    
    # Deployment info:
    kubernetes.io/change-cause: "Updated image to v2.1.0"
    
    # Build info:
    build/git-commit: "abc123def456"
    build/pipeline-url: "https://ci.example.com/builds/123"
```

```
Annotation values can be much larger than label values:
  Labels:       max 63 characters for value
  Annotations:  max 256 KB total for all annotations

Annotations can contain:
  - JSON
  - URLs
  - Multi-line text
  - Any string data
```

---

## Labels vs Annotations — When to Use Which

```
Use LABELS when:
  ✓ You need to SELECT resources (kubectl get -l, Services, Deployments)
  ✓ The value is short and well-defined
  ✓ You want to group/categorize resources
  
  Examples:
    app: web
    environment: production
    team: frontend

Use ANNOTATIONS when:
  ✓ You want to attach metadata for humans or tools
  ✓ The value is long or complex
  ✓ You don't need to filter by this value
  
  Examples:
    description: "This service handles user authentication"
    build/git-commit: "abc123def456"
    prometheus.io/scrape: "true"
```

---

## Practical Labeling Strategy

```yaml
# A well-labeled Deployment:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  labels:
    app: payment-api
    environment: production
    team: payments
    tier: backend
  annotations:
    description: "Handles payment processing via Stripe"
    owner: "payments-team@company.com"
    kubernetes.io/change-cause: "Deploy v3.2.1 - fix timeout bug"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
        environment: production
        team: payments
        tier: backend
        version: v3.2.1
    spec:
      containers:
        - name: api
          image: payment-api:3.2.1
```

```
Labeling strategy:
  app:          which application (payment-api, web-frontend, user-service)
  environment:  which environment (dev, staging, production)
  team:         which team owns it (payments, frontend, platform)
  tier:         which tier (frontend, backend, database, cache)
  version:      which version (v3.2.1)
```

---

## Useful kubectl Label Commands

```bash
# Show labels in output:
kubectl get pods --show-labels

# Show specific labels as columns:
kubectl get pods -L app,environment
# NAME        READY   STATUS    APP    ENVIRONMENT
# web-abc     1/1     Running   web    production
# api-xyz     1/1     Running   api    staging

# Count pods by label:
kubectl get pods -l environment=production --no-headers | wc -l

# Delete all pods with a label:
kubectl delete pods -l environment=dev

# Get all resources with a label:
kubectl get all -l app=web
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Labels                     → Organizing hundreds of resources
                           → Every K8s query uses labels
                           → CI/CD pipelines filter by labels
Selectors                  → Services find Pods (networking)
                           → Deployments manage Pods
                           → Network Policies (security)
Annotations                → Build metadata (git commit, pipeline URL)
                           → Monitoring config (Prometheus scraping)
                           → Tool configuration (Ingress, cert-manager)
Labeling strategy          → Team ownership and accountability
                           → Cost allocation (filter by team/environment)
                           → Access control (RBAC by namespace + labels)
```

---

> **Next:** 13 — DaemonSets, StatefulSets & Jobs → Specialized workload types.
