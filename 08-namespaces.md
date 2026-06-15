# 08 — Namespaces

## What Is a Namespace

A Namespace is like a **folder** in your cluster. It groups resources together and keeps them separate from other groups.

```
Without namespaces (everything in one bucket):
  Pod: nginx
  Pod: nginx         ← ERROR! Name conflict!
  Pod: my-app
  Pod: redis
  ConfigMap: app-config
  ConfigMap: app-config  ← ERROR! Name conflict!

With namespaces (organized into folders):
  Namespace: development
    Pod: nginx        ✓
    Pod: my-app       ✓
    ConfigMap: app-config ✓
    
  Namespace: production
    Pod: nginx        ✓ (same name, different namespace — no conflict)
    Pod: my-app       ✓
    ConfigMap: app-config ✓
```

---

## Default Namespaces

Every cluster starts with these namespaces:

```bash
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   30d    ← where your stuff goes if you don't specify
# kube-system       Active   30d    ← Kubernetes internal components
# kube-public       Active   30d    ← publicly accessible data
# kube-node-lease   Active   30d    ← node heartbeat tracking
```

```
default:
  Where everything goes if you don't specify a namespace.
  kubectl get pods = kubectl get pods -n default

kube-system:
  Kubernetes components live here:
    CoreDNS (DNS server)
    kube-proxy (network rules)
    metrics-server
    cloud controller
  ⚠️ Don't deploy your apps here.

kube-public:
  Contains cluster info readable by anyone.
  Rarely used directly.

kube-node-lease:
  Node heartbeat objects.
  Used internally by K8s.
```

---

## Creating and Using Namespaces

```bash
# Create a namespace:
kubectl create namespace development
kubectl create namespace production
kubectl create namespace staging

# Or with YAML:
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

```bash
kubectl apply -f namespace.yaml
```

### Working With Namespaces

```bash
# List resources in a specific namespace:
kubectl get pods -n development
kubectl get services -n production
kubectl get all -n staging

# List resources in ALL namespaces:
kubectl get pods -A
kubectl get pods --all-namespaces

# Create a resource in a specific namespace:
kubectl run nginx --image=nginx -n development
kubectl apply -f deployment.yaml -n production

# Or specify namespace in the YAML:
```

```yaml
metadata:
  name: my-app
  namespace: production     # ← specify namespace here
```

### Set Default Namespace

```bash
# Tired of typing -n every time? Set a default:
kubectl config set-context --current --namespace=development

# Now all commands default to "development":
kubectl get pods                  # shows pods in development
kubectl get pods -n production    # still works for other namespaces

# Check current default:
kubectl config view --minify | grep namespace
```

---

## Cross-Namespace Communication

Pods in different namespaces CAN talk to each other via DNS.

```
Same namespace:
  curl http://my-service:80                    ✓ (short name)

Different namespace:
  curl http://my-service.production:80         ✓ (add namespace)
  curl http://my-service.production.svc.cluster.local:80  ✓ (full FQDN)
```

```
┌─────────── Namespace: development ─────────────┐
│                                                  │
│  my-app Pod                                      │
│    curl http://db-service:5432         ✓ local  │
│    curl http://api.production:8080     ✓ cross  │
│                                                  │
└──────────────────────────────────────────────────┘

┌─────────── Namespace: production ──────────────┐
│                                                  │
│  api Pod (api.production)                        │
│  db Pod  (db-service.production)                 │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

## When to Use Namespaces

```
Common patterns:

By environment:
  development    → dev team experiments
  staging        → pre-production testing
  production     → live traffic

By team:
  team-frontend  → frontend team's services
  team-backend   → backend team's services
  team-data      → data team's pipelines

By application:
  app-web        → web application
  app-api        → API services
  app-workers    → background workers

Small team / small cluster:
  Just use "default" — namespaces add complexity you may not need.

Large team / large cluster:
  Namespaces are essential for organization, isolation, and RBAC.
```

---

## Resource Quotas — Limit Namespace Resources

Prevent one namespace from consuming all cluster resources.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"           # total CPU requests can't exceed 4 cores
    requests.memory: 8Gi        # total memory requests can't exceed 8 GB
    limits.cpu: "8"             # total CPU limits can't exceed 8 cores
    limits.memory: 16Gi         # total memory limits can't exceed 16 GB
    pods: "20"                  # max 20 pods
    services: "10"              # max 10 services
    persistentvolumeclaims: "5" # max 5 PVCs
```

```bash
# Check quota usage:
kubectl describe resourcequota dev-quota -n development
# Name:                   dev-quota
# Resource                Used    Hard
# --------                ----    ----
# limits.cpu              2       8
# limits.memory           4Gi     16Gi
# pods                    8       20
# requests.cpu            1       4
# requests.memory         2Gi     8Gi
```

---

## LimitRange — Default Limits for Pods

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
    - default:                      # default limits (if not specified)
        memory: 256Mi
        cpu: 500m
      defaultRequest:               # default requests (if not specified)
        memory: 128Mi
        cpu: 100m
      max:                          # maximum allowed
        memory: 1Gi
        cpu: "2"
      min:                          # minimum required
        memory: 64Mi
        cpu: 50m
      type: Container
```

```
This means:
  Any container in "development" without resource specs gets:
    requests: 128Mi memory, 100m CPU
    limits:   256Mi memory, 500m CPU
  
  No container can request more than 1Gi memory or 2 CPUs.
  No container can request less than 64Mi memory or 50m CPU.
```

---

## Deleting a Namespace

```bash
# ⚠️ WARNING: This deletes the namespace AND EVERYTHING IN IT
kubectl delete namespace development
# All pods, services, deployments, configmaps, secrets — GONE

# This is irreversible! Be very careful.
```

---

## What's Namespaced and What's Not

```
Namespaced resources (exist inside a namespace):
  Pods, Deployments, Services, ConfigMaps, Secrets,
  ReplicaSets, Ingress, Roles, RoleBindings, PVCs

Cluster-scoped resources (not in any namespace):
  Nodes, PersistentVolumes, Namespaces themselves,
  ClusterRoles, ClusterRoleBindings, StorageClasses

# Check which resources are namespaced:
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Namespaces                 → Multi-tenant clusters
                           → Environment isolation (dev/staging/prod)
                           → Team isolation
Resource quotas            → Cost control
                           → Prevent runaway resource usage
                           → Fair sharing in shared clusters
LimitRanges                → Default resource limits
                           → Prevent "forgot to set limits" issues
Cross-namespace DNS        → Microservice communication
                           → Shared services (monitoring, logging)
RBAC + namespaces          → "Team A can only deploy to namespace-a"
                           → Security isolation
```

---

> **Next:** 09 — ConfigMaps & Secrets → Managing configuration without rebuilding images.
