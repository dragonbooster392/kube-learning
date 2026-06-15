# 01 — Why Kubernetes

## The Problem: You Have Containers. Now What?

You learned Docker. You can build images, run containers, set up networks, mount volumes. You can run one container on one server. Maybe five containers with Docker Compose.

But what happens when your application grows?

```
Small app:
  1 web server container
  1 database container
  1 server
  → Docker Compose handles this fine

Medium app:
  5 web server containers
  2 API containers
  1 database container
  1 cache container
  2 servers
  → Docker Compose on each server? Who decides which container goes where?

Large app:
  50 web server containers
  20 API containers
  5 database containers
  10 worker containers
  3 cache containers
  20 servers
  → You need something to MANAGE all of this automatically
```

**Kubernetes is that something.**

---

## What Kubernetes Actually Does

Kubernetes (often written **K8s** — the 8 stands for the 8 letters between K and s) is a **container orchestrator**. It manages containers across multiple servers.

Think of it like this:

```
Without Kubernetes (you do everything manually):
  "Deploy 5 copies of the web app"
  → You SSH into server 1, run docker run...
  → You SSH into server 2, run docker run...
  → You SSH into server 3, run docker run...
  → Server 2 crashes. You notice hours later.
  → You SSH into server 4, run docker run... to replace it.
  → The web app needs updating. You SSH into every server...

With Kubernetes (you tell it what you want, it figures out how):
  "I want 5 copies of the web app running"
  → Kubernetes picks the best servers
  → Kubernetes starts 5 containers
  → Server 2 crashes
  → Kubernetes AUTOMATICALLY starts a replacement on server 4
  → You update the image
  → Kubernetes rolls out the update one container at a time, zero downtime
```

---

## The Key Idea: Desired State

This is the **most important concept** in Kubernetes. Everything else builds on this.

```
You tell Kubernetes:  "I want 5 copies of nginx running"
                      (This is the DESIRED STATE)

Kubernetes says:      "OK. I'll make it happen."
                      (It creates 5 containers)

Something breaks:     Server crashes. Now only 4 copies are running.
                      (CURRENT STATE ≠ DESIRED STATE)

Kubernetes notices:   "You wanted 5, but only 4 are running.
                       I'll start another one."
                      (It reconciles — brings current state back to desired state)
```

```
You declare WHAT you want    →  Kubernetes figures out HOW to do it

This is called "declarative" — you declare the end result.
The opposite is "imperative" — you tell it every step to take.

Imperative (Docker):
  docker run nginx
  docker run nginx
  docker run nginx
  docker run nginx
  docker run nginx
  # If one dies, YOU notice and restart it

Declarative (Kubernetes):
  "replicas: 5"
  # Kubernetes keeps 5 running, forever
```

---

## What Problems Does Kubernetes Solve?

### 1. Scheduling — Where Should Containers Run?

```
You have 10 servers:
  Server 1: 16 GB RAM, 8 CPUs (80% used)
  Server 2: 32 GB RAM, 16 CPUs (20% used)
  Server 3: 16 GB RAM, 8 CPUs (50% used)
  ...

You need to run a container that needs 4 GB RAM and 2 CPUs.
Which server should it go on?

Without K8s: You decide manually (or guess)
With K8s:    Kubernetes picks Server 2 (most resources available)
             This is called "scheduling"
```

### 2. Self-Healing — What If Something Crashes?

```
Container crashes      → Kubernetes restarts it
Server dies            → Kubernetes moves containers to healthy servers
Health check fails     → Kubernetes replaces the unhealthy container
Out of memory          → Kubernetes restarts the container

You don't wake up at 3 AM. Kubernetes handles it.
```

### 3. Scaling — Handle More Traffic

```
Normal traffic:   5 copies of your app
Black Friday:     Kubernetes scales to 50 copies automatically
Traffic drops:    Kubernetes scales back to 5

This is called "horizontal auto-scaling"
```

### 4. Rolling Updates — Deploy Without Downtime

```
Current version: v1.0 (5 copies running)
New version: v2.0

Kubernetes rolls out the update:
  Step 1: Start 1 copy of v2.0, wait until healthy
  Step 2: Stop 1 copy of v1.0
  Step 3: Start another v2.0, stop another v1.0
  Step 4: Continue until all 5 are v2.0

At no point are zero copies running = zero downtime

If v2.0 is broken:
  Kubernetes can roll BACK to v1.0 automatically
```

### 5. Service Discovery — How Do Containers Find Each Other?

```
Without K8s:
  Web app needs to talk to the API.
  API is on... which server? Which port?
  The API restarted and got a new IP.
  Now the web app can't find it.

With K8s:
  Web app talks to "api-service" (a name, not an IP)
  Kubernetes keeps track of where the API is running
  If the API moves, Kubernetes updates the routing automatically
  The web app never notices
```

### 6. Configuration Management

```
Without K8s:
  Database password is hardcoded in the container
  To change it, rebuild the image and redeploy

With K8s:
  Database password is stored in a "Secret"
  Container reads it at startup
  Change the password → restart the container → new password
  No image rebuild needed
```

---

## Kubernetes vs Docker Compose

```
Feature              Docker Compose         Kubernetes
─────────           ──────────────          ──────────
Scope               1 server               Multiple servers (cluster)
Self-healing        restart: always         Full self-healing + rescheduling
Scaling             Manual (replicas: N)    Auto-scaling based on load
Updates             Stop and restart        Rolling updates, zero downtime
Networking          Simple bridge           Advanced (DNS, load balancing)
Storage             Volume mounts           Persistent volumes, dynamic provisioning
Secrets             .env files              Encrypted secrets, RBAC
Health checks       Basic                   Liveness, readiness, startup probes
Rollback            Manual                  Automatic with version history
When to use         Development, small apps Production, any scale
```

```
Docker Compose is for:
  ✓ Local development
  ✓ Small apps on one server
  ✓ Quick prototyping

Kubernetes is for:
  ✓ Production workloads
  ✓ Multiple servers (clusters)
  ✓ Auto-scaling, self-healing
  ✓ Zero-downtime deployments
  ✓ Team collaboration with RBAC
```

---

## Kubernetes vs Docker Swarm

```
Docker Swarm:
  ✓ Built into Docker
  ✓ Simpler to set up
  ✓ Good for small clusters
  ✗ Limited features
  ✗ Smaller community
  ✗ Docker Inc. deprioritized it

Kubernetes:
  ✓ Industry standard
  ✓ Massive ecosystem
  ✓ Every cloud provider supports it (EKS, AKS, GKE)
  ✓ Most features, most flexibility
  ✗ Steeper learning curve
  ✗ More complex to set up from scratch

Winner: Kubernetes (by a huge margin in adoption)
  90%+ of container orchestration in production uses K8s
```

---

## A Brief History

```
2003-2004: Google builds "Borg" — internal container orchestrator
            Manages billions of containers across Google's data centers

2013:      Docker makes containers accessible to everyone

2014:      Google open-sources Kubernetes
           Based on 15+ years of experience running Borg
           Written in Go

2015:      Kubernetes v1.0 released
           Google donates it to the Cloud Native Computing Foundation (CNCF)

2017-now:  Kubernetes becomes THE standard
           Every major cloud provider offers managed K8s:
             AWS → EKS (Elastic Kubernetes Service)
             Azure → AKS (Azure Kubernetes Service)
             Google → GKE (Google Kubernetes Engine)
           The "K8s won" era
```

---

## The Kubernetes Ecosystem

```
Kubernetes is the center of a huge ecosystem:

Kubernetes Core:
  Pods, Deployments, Services, Volumes, RBAC

Package Management:
  Helm — "apt-get for Kubernetes"

Service Mesh:
  Istio, Linkerd — advanced networking (traffic management, mTLS)

Monitoring:
  Prometheus + Grafana — metrics and dashboards
  
Logging:
  EFK (Elasticsearch + Fluentd + Kibana)
  Loki + Grafana

CI/CD:
  ArgoCD, Flux — GitOps (deploy from Git automatically)
  
Secrets:
  Vault (HashiCorp) — advanced secret management

Networking:
  Calico, Cilium, Flannel — CNI plugins (Container Network Interface)

Storage:
  Rook-Ceph, Longhorn — distributed storage

All of these plug INTO Kubernetes. K8s is the platform everything runs on.
```

---

## What You Need to Know Before Kubernetes

```
You need to understand:

From the Linux course:
  ✓ Processes and how they work
  ✓ Networking (IP, DNS, ports, firewalls)
  ✓ Storage (filesystems, mounts)
  ✓ systemd (services, unit files)
  ✓ SSH (remote access)
  ✓ Logs (journalctl, /var/log)

From the Docker course:
  ✓ What containers are (namespaces, cgroups)
  ✓ Images and layers
  ✓ Dockerfile
  ✓ Container networking
  ✓ Volumes
  ✓ Docker Compose

If you completed the Linux and Docker courses, you're ready.
```

---

## Key Terminology (Preview)

Don't memorize these now — we'll explain each one in detail. This is just a preview.

```
Cluster        A set of servers running Kubernetes
Node           One server in the cluster
Pod            The smallest deployable unit (wraps 1+ containers)
Deployment     Manages Pods (scaling, updates, rollbacks)
Service        Stable network endpoint for Pods
Namespace      Logical grouping (like folders)
ConfigMap      Configuration data (non-sensitive)
Secret         Sensitive data (passwords, keys)
Volume         Storage attached to a Pod
Ingress        HTTP routing (like a reverse proxy)
kubectl        The CLI tool to interact with Kubernetes
YAML           The language you write Kubernetes configs in
```

---

## How This Course Is Organized

```
Foundations (files 01-03):
  01: Why Kubernetes (you are here)
  02: Architecture (how a cluster is built)
  03: kubectl & YAML (your tools)

Core Objects (files 04-11):
  04: Pods
  05: ReplicaSets
  06: Deployments
  07: Services
  08: Namespaces
  09: ConfigMaps & Secrets
  10: Storage
  11: Ingress

Organization (file 12):
  12: Labels, Selectors & Annotations

Workload Types (file 13):
  13: DaemonSets, StatefulSets & Jobs

Reliability (files 14-15):
  14: Health Checks (Probes)
  15: Resource Management

Security & Scheduling (files 16-17):
  16: RBAC & Security
  17: Scheduling

Advanced (files 18-20):
  18: Networking Deep Dive
  19: Helm
  20: Monitoring & Logging

Operations (files 21-22):
  21: Troubleshooting
  22: Hands-On Lab
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Container orchestration    → Every production deployment in 2026
Desired state              → GitOps, infrastructure as code
Self-healing               → High availability, SRE practices
Scaling                    → Handle traffic growth
Rolling updates            → CI/CD pipelines
Service discovery          → Microservices architecture
K8s ecosystem              → Your entire DevOps toolkit
```

---

> **Next:** 02 — Kubernetes Architecture → Control plane, worker nodes, and every component explained.
