# 02 — Kubernetes Architecture

## The Big Picture

A Kubernetes cluster is a group of servers working together. Some servers make decisions (**control plane**), and some servers run your containers (**worker nodes**).

```
                    ┌─────────────────────────────────────────┐
                    │           KUBERNETES CLUSTER             │
                    │                                          │
   You ──kubectl──► │  ┌──────────────────────────────────┐   │
                    │  │        CONTROL PLANE              │   │
                    │  │  (the brain — makes decisions)    │   │
                    │  │                                    │   │
                    │  │  API Server                        │   │
                    │  │  etcd                              │   │
                    │  │  Scheduler                         │   │
                    │  │  Controller Manager                │   │
                    │  └──────────────┬───────────────────┘   │
                    │                 │                         │
                    │        "Run these containers"            │
                    │                 │                         │
                    │  ┌──────────────▼───────────────────┐   │
                    │  │         WORKER NODES              │   │
                    │  │  (the muscles — run containers)   │   │
                    │  │                                    │   │
                    │  │  Node 1: [Pod] [Pod] [Pod]        │   │
                    │  │  Node 2: [Pod] [Pod]              │   │
                    │  │  Node 3: [Pod] [Pod] [Pod] [Pod]  │   │
                    │  └──────────────────────────────────┘   │
                    └─────────────────────────────────────────┘
```

---

## Control Plane — The Brain

The control plane runs on one or more servers (usually 3 for high availability). It never runs your application containers — its only job is to **manage the cluster**.

### API Server (kube-apiserver)

```
What it does:
  The FRONT DOOR to Kubernetes.
  Every request goes through the API server — no exceptions.

Who talks to it:
  kubectl (your CLI tool)     → "Create a deployment"
  Worker nodes (kubelet)      → "Reporting status of my pods"
  Other control plane parts   → "I need to schedule a pod"
  CI/CD pipelines             → "Deploy the new version"
  Monitoring tools            → "What's the cluster status?"

How it works:
  1. Receives a request (e.g., "create 3 nginx pods")
  2. Authenticates: "Are you allowed to do this?"
  3. Validates: "Is this a valid request?"
  4. Stores the request in etcd
  5. Other components watch etcd and take action

Think of it as: The receptionist at a hospital.
  Every request goes through them first.
  They check your ID, validate your request, and route it.
```

```
┌──────────────────────────────────────────────────────┐
│                    API SERVER                          │
│                                                       │
│  kubectl ──────────► Authentication                   │
│  kubelet ──────────► Authorization                    │
│  CI/CD   ──────────► Admission Control                │
│  Dashboard ────────► Validation                       │
│                      │                                │
│                      ▼                                │
│                   Store in etcd                       │
└──────────────────────────────────────────────────────┘

Everything goes through the API server.
There is NO other way to interact with Kubernetes.
```

### etcd — The Database

```
What it does:
  Stores ALL cluster data. Every piece of information Kubernetes knows.

What it stores:
  - What pods exist
  - What nodes are available
  - What deployments are configured
  - What secrets are stored
  - What the desired state is
  - What the current state is

Key facts:
  - It's a key-value store (like a dictionary/hash map)
  - It's distributed (runs on multiple servers for reliability)
  - It uses the Raft consensus algorithm (majority must agree)
  - ONLY the API server talks to etcd directly
  - If etcd is lost, the cluster is lost → ALWAYS back it up

Think of it as: The hospital's medical records system.
  Every patient record, every appointment, every prescription.
  If it's destroyed, no one knows anything.
```

```
etcd stores data like:
  /registry/pods/default/nginx-abc123
  /registry/deployments/default/my-app
  /registry/services/default/my-service
  /registry/secrets/default/db-password
  /registry/nodes/worker-1

It's a FLAT key-value store, not a relational database.
```

### Scheduler (kube-scheduler)

```
What it does:
  Decides WHICH NODE a new pod should run on.

How it works:
  1. Watches for new pods that have no node assigned
  2. Looks at each node's available resources (CPU, RAM)
  3. Checks constraints (node selectors, affinity, taints)
  4. Picks the best node
  5. Tells the API server: "Put this pod on Node 2"

It does NOT start the container. It only makes the decision.
The kubelet on the chosen node does the actual starting.

Think of it as: The hospital's bed manager.
  A new patient arrives.
  The bed manager checks which rooms are available,
  which rooms have the right equipment,
  and assigns the patient to a room.
  But the bed manager doesn't treat the patient — the nurse does.
```

```
New pod created → Scheduler evaluates:

  Node 1: 16 GB RAM, 8 CPU (12 GB used)    → 4 GB free ✗ (not enough)
  Node 2: 32 GB RAM, 16 CPU (8 GB used)    → 24 GB free ✓ (best fit)
  Node 3: 16 GB RAM, 8 CPU (6 GB used)     → 10 GB free ✓
  Node 4: tainted "database-only"           → ✗ (taint blocks it)

  Decision: Node 2 (most resources available)
```

### Controller Manager (kube-controller-manager)

```
What it does:
  Runs "controllers" — programs that watch the cluster state and
  make changes to move CURRENT STATE toward DESIRED STATE.

Controllers inside the controller manager:
  Deployment Controller:
    "You wanted 5 replicas. Only 4 exist. I'll create 1 more."
  
  ReplicaSet Controller:
    "You wanted 3 replicas. 1 crashed. I'll create a replacement."
  
  Node Controller:
    "Node 3 hasn't reported in 5 minutes. Marking it as unhealthy."
  
  Job Controller:
    "This batch job finished. Marking it as complete."
  
  Service Account Controller:
    "New namespace created. Creating a default service account."
  
  Endpoint Controller:
    "Pod IPs changed. Updating the service endpoints."

Think of it as: The hospital's operations team.
  They don't treat patients, but they make sure:
  - Enough nurses are on shift (ReplicaSet)
  - If a nurse calls in sick, a replacement is found (self-healing)
  - The schedule is maintained (desired state)
```

```
The control loop (runs continuously):

  ┌──────────────────────────────────┐
  │                                  │
  │    Observe current state         │
  │           │                      │
  │           ▼                      │
  │    Compare to desired state      │
  │           │                      │
  │           ▼                      │
  │    Take action if different      │
  │           │                      │
  │           ▼                      │
  │    (repeat forever)              │
  │                                  │
  └──────────────────────────────────┘

  Desired: 5 replicas
  Current: 4 replicas (one crashed)
  Action:  Create 1 more → now 5 replicas ✓
```

### Cloud Controller Manager (optional)

```
What it does:
  Talks to your cloud provider (AWS, Azure, GCP).

Examples:
  You create a LoadBalancer Service
  → Cloud Controller creates an actual AWS ELB / Azure LB
  
  A node disappears
  → Cloud Controller checks if the VM still exists in the cloud
  
  You request a PersistentVolume
  → Cloud Controller creates an EBS volume / Azure Disk

Only exists when running K8s on a cloud provider.
Not present in local clusters (minikube, kind).
```

---

## Worker Nodes — The Muscles

Worker nodes are the servers that actually **run your containers**. Each worker node runs three components.

### kubelet

```
What it does:
  The "agent" on every worker node.
  Takes instructions from the API server and runs containers.

How it works:
  1. Watches the API server for pods assigned to THIS node
  2. Tells the container runtime to start/stop containers
  3. Reports pod status back to the API server
  4. Runs health checks (liveness/readiness probes)
  5. Mounts volumes

Think of it as: The nurse on each hospital floor.
  Receives orders from the doctor (API server).
  Administers treatment (starts containers).
  Reports patient status (pod status).
  Calls for help if something goes wrong.

Key fact: kubelet does NOT run in a container.
  It runs as a systemd service directly on the node.
  systemctl status kubelet
```

```
API Server: "Run nginx on Node 2"
     │
     ▼
kubelet on Node 2:
  1. Receives the instruction
  2. Tells containerd: "Pull nginx image and start a container"
  3. containerd → runc → container starts
  4. kubelet monitors the container
  5. Reports to API server: "Pod is Running"
```

### kube-proxy

```
What it does:
  Handles networking rules on each node.
  Makes "Services" work — routes traffic to the right pods.

How it works:
  When you create a Service (e.g., my-service on port 80):
  kube-proxy creates iptables/IPVS rules on EVERY node:
    "Traffic to my-service:80 → forward to Pod IPs"

  If a pod dies and a new one starts (with a new IP):
  kube-proxy updates the rules automatically.

Think of it as: The hospital's internal phone system.
  You dial "surgery department" (Service name).
  The phone system routes your call to an available surgeon (Pod).
  If that surgeon is busy, it routes to another one (load balancing).
  If a surgeon leaves, the system removes their extension automatically.
```

```
Without kube-proxy:
  "I need to talk to nginx... which IP? There are 5 copies
   on different nodes with different IPs..."

With kube-proxy:
  "I'll just call nginx-service:80"
  → kube-proxy routes to one of the 5 nginx pods automatically
```

### Container Runtime

```
What it does:
  Actually runs containers. kubelet tells it what to do.

The standard runtime stack:
  kubelet → CRI → containerd → runc → container

Options:
  containerd:  Industry standard (used by Docker, K8s default)
  CRI-O:       Lightweight, built specifically for Kubernetes
  Docker:      Removed in K8s v1.24 (kubelet can't talk to Docker directly)
               Docker images still work (they're OCI standard)

You already know this from the Docker course:
  Docker Engine = Docker CLI + dockerd + containerd + runc
  Kubernetes    = kubectl + kubelet + containerd + runc
  They share the same container runtime underneath!
```

---

## How All the Parts Work Together

Let's trace what happens when you run: `kubectl create deployment nginx --replicas=3`

```
Step 1: kubectl → API Server
  kubectl sends a request: "Create a Deployment with 3 nginx replicas"
  API Server: Authenticates, validates, stores in etcd

Step 2: Controller Manager notices
  Deployment Controller: "New Deployment! I need to create a ReplicaSet."
  → Creates a ReplicaSet with replicas: 3
  → Stored in etcd via API Server

Step 3: Controller Manager again
  ReplicaSet Controller: "New ReplicaSet! I need to create 3 Pods."
  → Creates 3 Pod objects (but they have no node assigned yet)
  → Stored in etcd via API Server

Step 4: Scheduler notices
  "3 new pods with no node! Let me assign them."
  Pod 1 → Node 1 (has most free resources)
  Pod 2 → Node 2
  Pod 3 → Node 1 (still has room)
  → Updates pod assignments in etcd via API Server

Step 5: kubelet on Node 1 notices
  "2 pods assigned to me! Let me start them."
  → Tells containerd to pull nginx image and start 2 containers
  → Reports: "Pods are Running"

Step 6: kubelet on Node 2 notices
  "1 pod assigned to me!"
  → Starts 1 container
  → Reports: "Pod is Running"

Step 7: kube-proxy on ALL nodes
  "New pods exist. Updating network rules."
  → Configures iptables/IPVS so traffic reaches these pods

Result: 3 nginx containers running across 2 nodes ✓
```

```
Timeline:

  You                API Server    etcd    Controller    Scheduler    kubelet
   │                    │           │          │            │           │
   │─── create ────────►│           │          │            │           │
   │                    │──store───►│          │            │           │
   │                    │           │          │            │           │
   │                    │◄─watch────│──────────│            │           │
   │                    │           │     create ReplicaSet │           │
   │                    │──store───►│          │            │           │
   │                    │           │     create 3 Pods     │           │
   │                    │──store───►│          │            │           │
   │                    │           │          │            │           │
   │                    │◄─watch────│──────────│────────────│           │
   │                    │           │          │    assign nodes        │
   │                    │──store───►│          │            │           │
   │                    │           │          │            │           │
   │                    │◄─watch────│──────────│────────────│───────────│
   │                    │           │          │            │   start pods
   │                    │◄─status───│──────────│────────────│───────────│
   │                    │──store───►│          │            │           │
   │                    │           │          │            │           │
   │◄── "created" ─────│           │          │            │           │
```

---

## Control Plane vs Worker Node — Summary

```
┌─────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                         │
│                                                          │
│  ┌─────────────┐  ┌───────┐  ┌───────────┐  ┌────────┐ │
│  │ API Server  │  │ etcd  │  │ Scheduler │  │Contrlr │ │
│  │             │  │       │  │           │  │Manager │ │
│  │ Front door  │  │ The   │  │ Picks     │  │ Watches│ │
│  │ to K8s      │  │ data- │  │ which     │  │ & fixes│ │
│  │             │  │ base  │  │ node      │  │ state  │ │
│  └─────────────┘  └───────┘  └───────────┘  └────────┘ │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    WORKER NODE (each one)                 │
│                                                          │
│  ┌──────────┐  ┌────────────┐  ┌──────────────────────┐ │
│  │ kubelet  │  │ kube-proxy │  │ Container Runtime    │ │
│  │          │  │            │  │                      │ │
│  │ Runs     │  │ Network    │  │ containerd + runc    │ │
│  │ pods     │  │ routing    │  │ Actually runs        │ │
│  │          │  │            │  │ containers           │ │
│  └──────────┘  └────────────┘  └──────────────────────┘ │
│                                                          │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                   │
│  │ Pod  │ │ Pod  │ │ Pod  │ │ Pod  │  ← your apps      │
│  └──────┘ └──────┘ └──────┘ └──────┘                   │
└─────────────────────────────────────────────────────────┘
```

---

## How Many Nodes Do You Need?

```
Development / Learning:
  1 node that acts as BOTH control plane and worker
  → minikube, kind, k3s, Docker Desktop
  → Perfect for learning

Small production:
  3 control plane nodes (for high availability)
  3 worker nodes
  → Can handle moderate workloads

Medium production:
  3 control plane nodes
  10-50 worker nodes
  → Most companies

Large production:
  3-5 control plane nodes
  100-5000 worker nodes
  → Large companies, cloud providers
  → K8s supports up to 5000 nodes per cluster
```

---

## Managed Kubernetes (Cloud)

```
Running K8s yourself:
  You manage: Control plane + worker nodes + upgrades + etcd backups
  Complexity: Very high
  When: On-premises, special requirements

Managed Kubernetes (most common):
  Cloud provider manages: Control plane, etcd, upgrades
  You manage: Worker nodes, your applications
  
  AWS EKS:    Amazon Elastic Kubernetes Service
  Azure AKS:  Azure Kubernetes Service
  GCP GKE:    Google Kubernetes Engine
  
  Cost: $0.10/hour for the control plane (EKS)
        Worker nodes charged as regular VMs

In managed K8s:
  You NEVER see the control plane servers
  You NEVER SSH into them
  You NEVER worry about etcd backups
  The cloud provider handles all of that
  
  You just worry about:
  - Your application code
  - Your Kubernetes manifests (YAML files)
  - Your worker node configuration
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
API Server                 → Every CI/CD pipeline talks to it
                           → kubectl is your primary tool
etcd                       → Backup and disaster recovery
                           → Cluster state management
Scheduler                  → Resource planning, node sizing
                           → Cost optimization (right-size nodes)
Controller Manager         → Understanding self-healing
                           → Custom controllers (operators)
kubelet                    → Node troubleshooting
                           → Understanding pod lifecycle
kube-proxy                 → Service networking
                           → Understanding traffic flow
Managed K8s                → EKS/AKS/GKE — what most teams use
                           → Terraform to provision clusters
```

---

> **Next:** 03 — kubectl & YAML → Your tools for interacting with Kubernetes.
