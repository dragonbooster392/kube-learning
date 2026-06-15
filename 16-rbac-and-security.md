# 16 — RBAC & Security

## The Problem: Everyone Has Admin Access

```
Without access control:
  Every developer can:
    - Delete any pod in any namespace
    - Read all secrets (passwords, API keys)
    - Modify production deployments
    - Access the control plane

This is dangerous. You need:
  - Developers: read/write in their namespace only
  - CI/CD: deploy to specific namespaces
  - Monitoring: read-only access to metrics
  - Admins: full access
```

---

## RBAC — Role-Based Access Control

RBAC controls **who** can do **what** on **which resources**.

```
RBAC has four objects:

  Role:              "What actions are allowed?" (in ONE namespace)
  ClusterRole:       "What actions are allowed?" (cluster-wide)
  RoleBinding:       "Who gets this Role?" (in ONE namespace)
  ClusterRoleBinding:"Who gets this ClusterRole?" (cluster-wide)
```

```
The pattern:
  1. Create a Role (what permissions)
  2. Create a RoleBinding (who gets those permissions)

  Role says:          "Can get, list, and watch Pods"
  RoleBinding says:   "Give that Role to user john"
  Result:             john can get, list, and watch Pods
```

---

## Roles — Namespace-Scoped Permissions

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
  - apiGroups: [""]              # "" = core API group (pods, services, etc.)
    resources: ["pods"]          # which resources
    verbs: ["get", "list", "watch"]  # what actions
```

```
API verbs:
  get:     Read a single resource (kubectl get pod my-pod)
  list:    List all resources (kubectl get pods)
  watch:   Watch for changes (kubectl get pods --watch)
  create:  Create a resource (kubectl apply -f pod.yaml)
  update:  Update a resource (kubectl edit pod)
  patch:   Partially update a resource
  delete:  Delete a resource (kubectl delete pod my-pod)

API groups:
  ""                     Core (pods, services, configmaps, secrets, namespaces)
  "apps"                 Deployments, StatefulSets, DaemonSets, ReplicaSets
  "batch"                Jobs, CronJobs
  "networking.k8s.io"    Ingress, NetworkPolicies
  "rbac.authorization.k8s.io"  Roles, RoleBindings
```

### More Role Examples

```yaml
# Developer role: manage deployments and services
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: [""]
    resources: ["pods/log"]         # can view pod logs
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]          # can only READ secrets, not create/delete
    verbs: ["get", "list"]
```

---

## RoleBindings — Assigning Roles to Users

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: john-pod-reader
  namespace: development
subjects:
  - kind: User
    name: john                     # the user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader                 # the role to assign
  apiGroup: rbac.authorization.k8s.io
```

```
This gives user "john" the "pod-reader" Role in the "development" namespace.
John can now get, list, and watch Pods — but only in "development".
```

### Binding to Groups

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: development
subjects:
  - kind: Group
    name: dev-team                 # all users in "dev-team" group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

---

## ClusterRoles and ClusterRoleBindings

For permissions across ALL namespaces or for cluster-scoped resources.

```yaml
# ClusterRole: can read pods in ALL namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-reader         # no namespace (cluster-wide)
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["nodes"]           # nodes are cluster-scoped
    verbs: ["get", "list"]
```

```yaml
# ClusterRoleBinding: give it to the monitoring service
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-binding
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: cluster-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```
Role + RoleBinding             = permissions in ONE namespace
ClusterRole + ClusterRoleBinding = permissions in ALL namespaces

You can also bind a ClusterRole with a RoleBinding:
  This gives the ClusterRole's permissions but only in ONE namespace.
  Useful for reusing a ClusterRole across many namespaces.
```

---

## ServiceAccounts — Identity for Pods

Users are for humans. ServiceAccounts are for **applications running in Pods**.

```yaml
# Create a ServiceAccount:
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
```

```yaml
# Use it in a Pod:
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app-sa       # use this ServiceAccount
  containers:
    - name: app
      image: my-app:1.0
```

```
Every Pod runs as a ServiceAccount.
If you don't specify one, it uses "default" (which has minimal permissions).

The Pod gets a token mounted at:
  /var/run/secrets/kubernetes.io/serviceaccount/token

Your app can use this token to call the Kubernetes API.
```

### ServiceAccount + RBAC Example

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployer-sa
  namespace: production
---
# Role: can manage deployments
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "create", "update", "patch"]
---
# RoleBinding: give the role to the ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: deployer-sa
    namespace: production
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

```
Now a CI/CD pipeline Pod using deployer-sa can:
  ✓ Create and update Deployments in production
  ✗ Cannot delete Deployments
  ✗ Cannot access Secrets
  ✗ Cannot do anything in other namespaces
```

---

## SecurityContext — Container-Level Security

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:                      # Pod-level settings
    runAsNonRoot: true                 # container must not run as root
    runAsUser: 1000                    # run as user ID 1000
    fsGroup: 2000                      # files created are group 2000
  containers:
    - name: app
      image: my-app:1.0
      securityContext:                 # Container-level settings
        allowPrivilegeEscalation: false  # can't gain more privileges
        readOnlyRootFilesystem: true    # can't write to container filesystem
        capabilities:
          drop:
            - ALL                      # remove all Linux capabilities
```

```
Why this matters:
  - Running as root inside a container is risky
  - If an attacker breaks into the container, they're root
  - readOnlyRootFilesystem prevents malware from being written
  - Dropping capabilities removes unnecessary Linux permissions

Best practice: always run as non-root with minimal capabilities.
```

---

## Checking RBAC

```bash
# "Can I do this?" (check your own permissions):
kubectl auth can-i create deployments
# yes

kubectl auth can-i delete pods --namespace production
# no

# "Can this user do this?" (admin check):
kubectl auth can-i create deployments --as john
# no

kubectl auth can-i get pods --as system:serviceaccount:default:my-app-sa
# yes

# List all roles in a namespace:
kubectl get roles -n production

# List all role bindings:
kubectl get rolebindings -n production

# Describe a role (see permissions):
kubectl describe role developer -n production
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
RBAC                       → Least-privilege access for everyone
                           → Developers get namespace access, not cluster admin
Roles/RoleBindings         → Per-namespace permissions
                           → Team isolation
ClusterRoles               → Monitoring, logging, cluster-level operations
ServiceAccounts            → CI/CD pipelines need K8s API access
                           → Each app gets its own identity
SecurityContext             → Container hardening
                           → Required by security policies
                           → CIS benchmarks check for these
```

---

> **Next:** 17 — Scheduling → How Kubernetes decides which node runs your Pod.
