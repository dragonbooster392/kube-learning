# 11 — Ingress

## The Problem: LoadBalancer Per Service Is Expensive

```
You have 3 web services:
  api.example.com     → API Service
  app.example.com     → Web App Service
  admin.example.com   → Admin Panel Service

With LoadBalancer Services:
  Each one gets its own cloud load balancer.
  3 services = 3 load balancers = 3x the cost.
  And you can't share TLS certificates between them.

With Ingress:
  ONE load balancer handles ALL three.
  Routes traffic based on hostname or path.
  Manages TLS certificates in one place.
```

---

## What Is Ingress

Ingress is a Kubernetes object that manages **HTTP/HTTPS routing** from outside the cluster to Services inside the cluster.

```
Without Ingress:
  Internet → LoadBalancer 1 → API Service
  Internet → LoadBalancer 2 → Web Service
  Internet → LoadBalancer 3 → Admin Service
  (3 load balancers, 3 public IPs)

With Ingress:
  Internet → ONE LoadBalancer → Ingress Controller → API Service
                                                    → Web Service
                                                    → Admin Service
  (1 load balancer, 1 public IP, routing by hostname/path)
```

```
┌──────────────────────────────────────────────────────┐
│                    INGRESS                             │
│                                                       │
│  api.example.com    ──────► API Service (ClusterIP)   │
│  app.example.com    ──────► Web Service (ClusterIP)   │
│  admin.example.com  ──────► Admin Service (ClusterIP) │
│                                                       │
│  TLS termination (HTTPS)                              │
│  Rate limiting                                        │
│  Path-based routing                                   │
│  Hostname-based routing                               │
└──────────────────────────────────────────────────────┘
```

---

## Ingress Controller — Required!

The Ingress **object** is just a set of rules. You also need an **Ingress Controller** — a program that reads the rules and actually does the routing.

```
Ingress object:      "Route api.example.com to the api service"
                     (this is just a rule, stored in etcd)

Ingress Controller:  A reverse proxy (like nginx) that reads the rules
                     and configures itself to route traffic accordingly.
                     (this does the actual work)

You MUST install an Ingress Controller.
Kubernetes does NOT come with one by default.
```

### Popular Ingress Controllers

```
NGINX Ingress Controller:
  Most popular. Based on nginx.
  Works everywhere (cloud, on-prem, local).
  
Traefik:
  Automatic config discovery.
  Built-in Let's Encrypt support.
  Popular for smaller setups.

AWS ALB Ingress Controller:
  Uses AWS Application Load Balancer natively.
  Best for AWS EKS.

HAProxy Ingress:
  High performance.
  Advanced load balancing features.

Istio Gateway:
  Part of the Istio service mesh.
  Most features, most complex.
```

```bash
# Install NGINX Ingress Controller (most common):
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml

# Verify:
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS
# ingress-nginx-controller-abc123-xyz         1/1     Running
```

---

## Ingress YAML

### Host-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: api.example.com           # requests to api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service      # go to api-service
                port:
                  number: 80

    - host: app.example.com           # requests to app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service      # go to web-service
                port:
                  number: 80
```

### Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /api                 # example.com/api/*
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

          - path: /admin               # example.com/admin/*
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80

          - path: /                    # everything else
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

```
Path types:
  Prefix:  /api matches /api, /api/, /api/v1, /api/users
  Exact:   /api matches ONLY /api (not /api/ or /api/v1)

Put more specific paths first. K8s matches top to bottom.
```

---

## TLS — HTTPS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
    - hosts:
        - app.example.com
        - api.example.com
      secretName: tls-secret           # Secret with TLS cert and key
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

```bash
# Create the TLS secret:
kubectl create secret tls tls-secret \
  --cert=./tls.crt \
  --key=./tls.key
```

```
With TLS configured:
  https://app.example.com → Ingress terminates TLS → routes to web-service:80
  
  The connection from client to Ingress is HTTPS (encrypted).
  The connection from Ingress to Service is HTTP (internal).
  This is called "TLS termination" — the Ingress handles the encryption.
```

### cert-manager — Automatic TLS Certificates

```
cert-manager is a K8s add-on that automatically gets TLS certificates
from Let's Encrypt (free!) and renews them.

1. Install cert-manager
2. Create a ClusterIssuer (tells cert-manager how to get certs)
3. Add an annotation to your Ingress
4. cert-manager automatically:
   - Requests a certificate from Let's Encrypt
   - Creates a Secret with the cert
   - Renews before expiration
```

```yaml
# Ingress with cert-manager annotation:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod    # ← magic annotation
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls             # cert-manager creates this Secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

---

## Ingress Annotations

Annotations customize Ingress Controller behavior.

```yaml
metadata:
  annotations:
    # NGINX-specific:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    
    # Rate limiting:
    nginx.ingress.kubernetes.io/limit-connections: "10"
    
    # CORS:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
```

---

## Default Backend

```yaml
# Handle requests that don't match any rule:
spec:
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
    # ... your rules
```

```
If someone visits a hostname/path that doesn't match any rule,
the default backend handles it (usually a 404 page).
```

---

## Complete Example

```yaml
# All the parts together:

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
  type: ClusterIP                      # ClusterIP is enough with Ingress
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx              # which Ingress Controller to use
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

```
Traffic flow:
  User → https://app.example.com
    → DNS resolves to Ingress Controller's LoadBalancer IP
    → Ingress Controller terminates TLS
    → Routes to web-service (ClusterIP)
    → web-service load-balances to one of 3 web-app Pods
    → Pod responds
    → Response flows back through the same path
```

---

## Checking Ingress Status

```bash
# List ingresses:
kubectl get ingress
# NAME          CLASS   HOSTS             ADDRESS         PORTS     AGE
# web-ingress   nginx   app.example.com   52.170.123.4    80, 443   5m

# Describe:
kubectl describe ingress web-ingress
# Shows rules, backends, and any errors
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Ingress                    → HTTP routing for all web traffic
                           → Single entry point for the cluster
Host-based routing         → Multiple apps on one cluster
                           → Microservices with different domains
Path-based routing         → API versioning (/v1, /v2)
                           → Frontend + backend on same domain
TLS                        → HTTPS everywhere (required!)
                           → cert-manager for auto-renewal
Annotations                → Fine-tuning (rate limits, CORS, timeouts)
Cost                       → 1 LoadBalancer vs N LoadBalancers
                           → Significant cost savings on cloud
```

---

> **Next:** 12 — Labels, Selectors & Annotations → How Kubernetes organizes everything.
