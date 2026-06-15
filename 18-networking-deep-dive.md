# 18 — Networking Deep Dive

## Kubernetes Networking Rules

Kubernetes networking follows three fundamental rules:

```
Rule 1: Every Pod gets its own IP address.
        No need to map ports between Pods.

Rule 2: Pods can communicate with ALL other Pods
        without NAT (Network Address Translation).
        Any Pod can reach any other Pod using its IP.

Rule 3: Nodes can communicate with ALL Pods (and vice versa)
        without NAT.
```

```
This is very different from Docker:
  Docker:      containers share the host's IP, need port mapping (-p 8080:80)
  Kubernetes:  every Pod gets its own IP, no port mapping needed

  Pod A (10.244.1.5) can reach Pod B (10.244.2.8) directly.
  Doesn't matter which nodes they're on.
```

---

## CNI — Container Network Interface

Kubernetes doesn't implement networking itself. It uses a **CNI plugin** to set up the network.

```
CNI plugin responsibilities:
  1. Assign an IP address to each Pod
  2. Set up routes so Pods can talk to each other
  3. Set up routes so Pods can reach external networks

Popular CNI plugins:
  Calico:     Most popular. Network policies, BGP, eBPF support.
  Cilium:     eBPF-based. Advanced security and observability.
  Flannel:    Simplest. Good for learning. No network policies.
  Weave Net:  Easy setup. Mesh networking.
  AWS VPC CNI: Pods get real VPC IPs (AWS EKS).
```

```
How Pods on different nodes communicate:

  Node 1 (10.0.0.1)                Node 2 (10.0.0.2)
  ┌────────────────┐               ┌────────────────┐
  │ Pod A           │               │ Pod C           │
  │ 10.244.1.5      │               │ 10.244.2.8      │
  │                 │               │                 │
  │ Pod B           │               │ Pod D           │
  │ 10.244.1.6      │               │ 10.244.2.9      │
  └───────┬─────────┘               └────────┬────────┘
          │                                   │
          └──────── overlay network ──────────┘
          (VXLAN, IPIP, or direct routing)

  Pod A (10.244.1.5) → Pod C (10.244.2.8):
    1. Packet leaves Pod A
    2. CNI plugin encapsulates it (or routes it directly)
    3. Packet crosses the physical network to Node 2
    4. CNI plugin on Node 2 delivers it to Pod C
```

---

## Service Networking — How Services Work

When you create a Service, it gets a **virtual IP** (ClusterIP).

```
How does traffic get from ClusterIP to the actual Pod?
Answer: kube-proxy.

kube-proxy runs on every node and sets up rules to forward traffic.
```

### kube-proxy Modes

```
iptables mode (default):
  kube-proxy creates iptables rules on every node.
  When traffic hits a ClusterIP, iptables redirects it to a Pod.
  Random selection among healthy Pods.
  
  Pros: Simple, reliable, well-tested.
  Cons: Slow with thousands of Services (iptables rules grow linearly).

IPVS mode:
  Uses Linux IPVS (IP Virtual Server) for load balancing.
  Hash-based lookup instead of sequential iptables rules.
  
  Pros: Better performance with many Services.
  Cons: More complex, requires IPVS kernel modules.

eBPF mode (Cilium):
  Replaces kube-proxy entirely with eBPF programs.
  Traffic is redirected at the kernel level.
  
  Pros: Fastest, most efficient.
  Cons: Requires Cilium CNI and newer kernel.
```

```
Traffic flow for ClusterIP Service:

  App Pod → Service ClusterIP:80
    → kube-proxy iptables rule
    → Randomly selects a backend Pod IP
    → Traffic goes directly to Pod IP:8080
    
  The ClusterIP is virtual — no process listens on it.
  iptables intercepts the packet and redirects it.
```

---

## DNS — CoreDNS

CoreDNS is the DNS server inside every Kubernetes cluster. It lets Pods find Services by name.

```
When you create a Service named "web-service" in namespace "default":
  CoreDNS creates DNS records:
    web-service                             → ClusterIP
    web-service.default                     → ClusterIP
    web-service.default.svc                 → ClusterIP
    web-service.default.svc.cluster.local   → ClusterIP (FQDN)

From a Pod in the same namespace:
  curl http://web-service                   (just the name works)

From a Pod in a different namespace:
  curl http://web-service.default           (need namespace)
```

```
How DNS resolution works:

  Pod → "I need to reach web-service"
    → Pod's /etc/resolv.conf points to CoreDNS
    → CoreDNS looks up "web-service" → returns ClusterIP
    → Pod connects to ClusterIP
    → kube-proxy routes to a backend Pod
```

```bash
# Check CoreDNS is running:
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check DNS from inside a Pod:
kubectl run dns-test --image=busybox --rm -it -- nslookup web-service
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# Name:      web-service.default.svc.cluster.local
# Address:   10.100.50.25

# Check a Pod's DNS config:
kubectl exec my-pod -- cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
```

---

## NetworkPolicies — Firewalls for Pods

By default, every Pod can talk to every other Pod. NetworkPolicies restrict this.

```
Without NetworkPolicies:
  All Pods can communicate with all other Pods.
  This is like a network with no firewall — dangerous in production.

With NetworkPolicies:
  You define which Pods can talk to which Pods.
  Everything else is denied (default deny).
```

### Default Deny All

```yaml
# Deny ALL ingress traffic to Pods in this namespace:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}                  # applies to ALL pods in namespace
  policyTypes:
    - Ingress                      # control incoming traffic
  # No ingress rules = deny all incoming
```

```
With this policy:
  No Pod in "production" namespace can receive traffic
  (unless another NetworkPolicy explicitly allows it)
```

### Allow Specific Traffic

```yaml
# Allow traffic FROM app=web TO app=api:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api                     # this policy applies to api Pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web             # allow traffic from web Pods
      ports:
        - port: 8080               # only on port 8080
          protocol: TCP
```

```
This policy:
  ✓ web Pods can reach api Pods on port 8080
  ✗ database Pods cannot reach api Pods
  ✗ Pods from other namespaces cannot reach api Pods
  ✗ No one can reach api Pods on any other port
```

### Allow From Another Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: monitoring    # allow from namespaces labeled purpose=monitoring
          podSelector:
            matchLabels:
              app: prometheus        # and only from prometheus pods
      ports:
        - port: 9090
```

### Complete Network Policy Example

```yaml
# Three-tier app: web → api → database

# 1. Default deny all in production
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# 2. Web can receive traffic from Ingress and reach API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: web
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from: []                     # allow from anywhere (Ingress controller)
      ports:
        - port: 80
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: api
      ports:
        - port: 8080
    - to:                          # allow DNS
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
---
# 3. API can only receive from web and reach database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: web
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - port: 5432
    - to:
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
---
# 4. Database only accepts from API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: api
      ports:
        - port: 5432
```

```
Traffic allowed:
  Internet → web (port 80) → api (port 8080) → database (port 5432)

Traffic denied:
  Internet → api         ✗
  Internet → database    ✗
  web → database         ✗
  database → anything    ✗
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
CNI plugins                → Cluster setup (choosing Calico vs Cilium)
                           → Network performance and features
kube-proxy                 → Understanding Service traffic flow
                           → Troubleshooting connectivity
CoreDNS                    → Service discovery
                           → Debugging "can't reach service" issues
NetworkPolicies            → Security compliance
                           → Zero-trust networking
                           → PCI-DSS, SOC2 requirements
                           → Micro-segmentation
```

---

> **Next:** 19 — Helm → The package manager for Kubernetes.
