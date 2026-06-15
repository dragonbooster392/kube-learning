# 07 — Services

## The Problem: Pod IPs Are Unreliable

Pods are ephemeral. When a Pod dies and a new one is created, it gets a **new IP address**. You can't rely on Pod IPs.

```
Time 0:
  Pod nginx-abc → IP 10.244.1.5
  Your app calls: http://10.244.1.5:80  ✓ works

Time 1: Pod crashes, ReplicaSet creates replacement:
  Pod nginx-xyz → IP 10.244.2.8 (DIFFERENT IP!)
  Your app calls: http://10.244.1.5:80  ✗ FAILS (old IP is gone)

This is the problem. Pod IPs change. You need something stable.
```

**A Service gives you a stable IP and DNS name** that always routes to the right Pods, no matter how they change.

---

## What Is a Service

A Service is a stable network endpoint that routes traffic to a group of Pods.

```
Without a Service:
  App → 10.244.1.5 (pod IP) → might be gone tomorrow

With a Service:
  App → nginx-service (DNS name) → routes to current nginx Pods
        OR
  App → 10.96.0.100 (Service IP, never changes) → routes to current nginx Pods

The Service IP and DNS name NEVER change,
even when the underlying Pods are created and destroyed.
```

```
┌──────────────────────────────────────────┐
│              Service                      │
│  Name: nginx-service                      │
│  IP:   10.96.0.100 (stable, never changes)│
│  Port: 80                                 │
│                                           │
│  Routes to pods with label: app=nginx     │
│    ┌──────────┐                           │
│    │ Pod 1    │ 10.244.1.5:80            │
│    │ Pod 2    │ 10.244.2.8:80            │
│    │ Pod 3    │ 10.244.1.9:80            │
│    └──────────┘                           │
│                                           │
│  If Pod 2 dies and Pod 4 is created:      │
│    Pod 4 gets IP 10.244.3.2:80           │
│    Service automatically updates routing  │
│    Clients never notice                   │
└──────────────────────────────────────────┘
```

---

## How Services Find Pods — Labels

Services use **label selectors** to find their Pods. This is the same label system from ReplicaSets and Deployments.

```yaml
# The Deployment creates Pods with labels:
spec:
  template:
    metadata:
      labels:
        app: nginx          # ← each Pod gets this label

# The Service selects Pods with those labels:
spec:
  selector:
    app: nginx              # ← "route traffic to Pods with app=nginx"
```

```
Service selector: app=nginx
  → Finds Pod 1 (app=nginx) ✓
  → Finds Pod 2 (app=nginx) ✓
  → Finds Pod 3 (app=nginx) ✓
  → Ignores Pod 4 (app=redis) ✗

The Service continuously watches for Pods matching its selector.
New Pod with app=nginx? → added to routing
Pod with app=nginx dies? → removed from routing
Automatic. Always up to date.
```

---

## Service Types

Kubernetes has four types of Services. Each one controls **who can access your Pods**.

### ClusterIP (Default) — Internal Only

```
Access: Only from INSIDE the cluster
Use:    Communication between services

Other pods → ClusterIP Service → Your pods
Clients outside the cluster CANNOT reach this Service.
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP              # default (you can omit this line)
  selector:
    app: nginx                 # route to pods with app=nginx
  ports:
    - port: 80                 # Service port (what clients use)
      targetPort: 80           # Pod port (where the container listens)
      protocol: TCP
```

```
After creating this Service:
  Service Name:  nginx-service
  Service IP:    10.96.0.100 (assigned by K8s, stable)
  Service Port:  80

  From any pod in the cluster:
    curl http://nginx-service:80       ✓ works (DNS)
    curl http://10.96.0.100:80         ✓ works (IP)

  From outside the cluster:
    curl http://10.96.0.100:80         ✗ not reachable
```

```
When to use ClusterIP:
  ✓ Database services (only your app should access it)
  ✓ Internal APIs (backend → backend communication)
  ✓ Cache services (Redis, Memcached)
  ✓ Any service that should NOT be exposed externally
```

### NodePort — External Access Via Node IP

```
Access: From outside the cluster, via any node's IP
Use:    Development, testing, simple external access

Client → NodeIP:NodePort → Service → Pod

Kubernetes opens a port (30000-32767) on EVERY node.
Traffic to that port gets routed to the Service.
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80                 # Service port (internal)
      targetPort: 80           # Pod port
      nodePort: 30080          # Port on every node (30000-32767)
      protocol: TCP
```

```
After creating this Service:
  Service IP:    10.96.0.101:80     (internal)
  Node Port:     30080              (on every node)

  From inside the cluster:
    curl http://nginx-nodeport:80   ✓ works

  From outside the cluster:
    curl http://NODE-IP:30080       ✓ works (any node's IP!)

  If you have 3 nodes (10.12.24.141, 10.12.24.142, 10.12.24.143):
    curl http://10.12.24.141:30080  ✓
    curl http://10.12.24.142:30080  ✓  All work!
    curl http://10.12.24.143:30080  ✓
    (even if the pod is only on node 1)
```

```
When to use NodePort:
  ✓ Development and testing
  ✓ When you don't have a cloud load balancer
  ✗ Not great for production (ugly port numbers, no TLS termination)
```

### LoadBalancer — Cloud Load Balancer

```
Access: From the internet, via a cloud load balancer
Use:    Production on cloud providers (AWS, Azure, GCP)

Internet → Cloud LB → Service → Pod

Kubernetes tells the cloud provider: "Create a load balancer for me."
The cloud provider creates an actual load balancer (AWS ELB, Azure LB, etc.)
with a public IP address.
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

```bash
kubectl get service nginx-loadbalancer
# NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)
# nginx-loadbalancer   LoadBalancer   10.96.0.102   52.170.123.456   80:31234/TCP

# EXTERNAL-IP is a real public IP (or DNS name on AWS)
# Anyone on the internet can access:
#   http://52.170.123.456:80
```

```
When to use LoadBalancer:
  ✓ Production workloads that need external access
  ✓ Running on a cloud provider
  ✗ Each LoadBalancer Service creates a NEW cloud LB ($$$)
  ✗ Use Ingress instead for multiple services (file 11)
```

### ExternalName — DNS Alias

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-database
spec:
  type: ExternalName
  externalName: database.example.com
```

```
This doesn't route to pods.
It creates a DNS alias:
  my-database → database.example.com

Use case: Your app calls "my-database" and it resolves to an
external database. If you migrate the database, just update
the ExternalName. No app code changes.
```

---

## Service Comparison

```
Type           Access From        External IP    Use Case
──────         ──────────         ────────────   ────────
ClusterIP      Inside cluster     No             Internal services
NodePort       Node IP:Port       Node IPs       Dev/testing
LoadBalancer   Internet           Cloud LB IP    Production (cloud)
ExternalName   DNS alias          N/A            External services
```

```
Traffic flow:

ClusterIP:
  Pod → ClusterIP:80 → target Pod

NodePort:
  External → NodeIP:30080 → ClusterIP:80 → target Pod

LoadBalancer:
  Internet → Cloud LB:80 → NodeIP:NodePort → ClusterIP:80 → target Pod

Each type BUILDS on the previous:
  LoadBalancer includes NodePort includes ClusterIP
```

---

## Service DNS

Kubernetes has a built-in DNS server (CoreDNS). Every Service gets a DNS name automatically.

```
Service name:     nginx-service
Namespace:        default

DNS names (all resolve to the Service IP):
  nginx-service                          (within same namespace)
  nginx-service.default                  (with namespace)
  nginx-service.default.svc              (with svc)
  nginx-service.default.svc.cluster.local (fully qualified)

From a pod in the "default" namespace:
  curl http://nginx-service:80           ✓ (short name works)

From a pod in the "production" namespace:
  curl http://nginx-service.default:80   ✓ (need namespace)
```

```
This is why you NEVER hardcode IPs in Kubernetes.
  
  ✗ curl http://10.96.0.100:80    (IP might change if Service is recreated)
  ✓ curl http://nginx-service:80  (DNS name always resolves correctly)
```

---

## Endpoints — What's Behind a Service

```bash
# A Service maintains a list of Pod IPs called "Endpoints":
kubectl get endpoints nginx-service
# NAME            ENDPOINTS
# nginx-service   10.244.1.5:80,10.244.2.8:80,10.244.1.9:80

# These are the actual Pod IPs.
# When a Pod is added/removed, the Endpoints update automatically.

# describe shows full details:
kubectl describe service nginx-service
# Name:              nginx-service
# Type:              ClusterIP
# IP:                10.96.0.100
# Port:              <unset>  80/TCP
# TargetPort:        80/TCP
# Endpoints:         10.244.1.5:80,10.244.2.8:80,10.244.1.9:80
# Session Affinity:  None
```

---

## Port Mapping

```yaml
ports:
  - port: 80             # Service port (what clients connect to)
    targetPort: 8080      # Container port (where the app listens)
    protocol: TCP
```

```
Client → Service:80 → Pod:8080

This is useful when your app runs on port 8080 internally
but you want clients to connect on port 80.

You can also name ports:
  In the Pod spec:
    ports:
      - name: http
        containerPort: 8080
  
  In the Service:
    ports:
      - port: 80
        targetPort: http    # reference by name instead of number
```

### Multiple Ports

```yaml
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
    - name: metrics
      port: 9090
      targetPort: 9090
```

---

## Creating a Service

### Imperative

```bash
# Expose a deployment:
kubectl expose deployment nginx --port=80 --target-port=80 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Generate YAML:
kubectl expose deployment nginx --port=80 --type=ClusterIP \
  --dry-run=client -o yaml > service.yaml
```

### Declarative (Preferred)

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f service.yaml
```

---

## Complete Example: Deployment + Service

```yaml
# app.yaml — both in one file

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web             # ← Pods get this label
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web                 # ← selects Pods with this label
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f app.yaml
# deployment.apps/my-web-app created
# service/web-service created

# Now any pod in the cluster can reach:
#   curl http://web-service:80
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
ClusterIP                  → Internal microservice communication
                           → Backend API ↔ database
                           → Most common Service type
NodePort                   → Quick external access for testing
                           → On-premises clusters without cloud LB
LoadBalancer               → Production external access on cloud
                           → AWS ELB / Azure LB / GCP LB
Service DNS                → Service discovery (no hardcoded IPs)
                           → Microservices call each other by name
Endpoints                  → Understanding traffic flow
                           → Debugging connectivity issues
Port mapping               → Running apps on non-standard ports
                           → Legacy app migration
```

---

> **Next:** 08 — Namespaces → Organizing your cluster into logical groups.
