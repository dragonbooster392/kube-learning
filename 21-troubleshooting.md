# 21 — Troubleshooting

## The Debugging Mindset

```
When something isn't working in Kubernetes, follow this order:

  1. What's the current state?     (kubectl get)
  2. What happened?                (kubectl describe)
  3. What is the app saying?       (kubectl logs)
  4. Can I get inside?             (kubectl exec)
  5. Is the network working?       (DNS, connectivity)
```

---

## Step 1: Check the Current State

```bash
# What's running?
kubectl get pods
# NAME                       READY   STATUS             RESTARTS   AGE
# web-app-abc123-xyz         1/1     Running            0          2h
# api-server-def456-uvw      0/1     CrashLoopBackOff   5          10m
# db-migration-ghi789        0/1     ImagePullBackOff   0          5m

# More detail:
kubectl get pods -o wide
# Shows NODE, IP, and nominated node

# All resources:
kubectl get all

# Specific namespace:
kubectl get pods -n production

# Watch (live updates):
kubectl get pods --watch
```

---

## Step 2: Describe the Resource

```bash
# Describe shows EVENTS — the most useful debugging info:
kubectl describe pod api-server-def456-uvw

# Look at the Events section at the bottom:
# Events:
#   Type     Reason     Age   Message
#   ----     ------     ----  -------
#   Normal   Scheduled  10m   Successfully assigned to worker-2
#   Normal   Pulling    10m   Pulling image "api-server:2.0"
#   Normal   Pulled     10m   Successfully pulled image
#   Normal   Created    10m   Created container
#   Normal   Started    10m   Started container
#   Warning  Unhealthy  9m    Liveness probe failed: connection refused
#   Normal   Killing    9m    Container failed liveness probe, will be restarted

# Describe works for any resource:
kubectl describe deployment web-app
kubectl describe service web-service
kubectl describe node worker-1
kubectl describe ingress my-ingress
```

---

## Step 3: Check the Logs

```bash
# Current logs:
kubectl logs api-server-def456-uvw

# Previous container logs (if it crashed and restarted):
kubectl logs api-server-def456-uvw --previous

# Follow logs live:
kubectl logs api-server-def456-uvw -f

# Multi-container pod (specify container):
kubectl logs my-pod -c sidecar

# All pods with a label:
kubectl logs -l app=api --all-containers

# Last N lines:
kubectl logs api-server-def456-uvw --tail=50

# Logs since a time:
kubectl logs api-server-def456-uvw --since=1h
```

---

## Step 4: Exec Into the Container

```bash
# Get a shell inside a running container:
kubectl exec -it web-app-abc123-xyz -- /bin/bash
# or if bash isn't available:
kubectl exec -it web-app-abc123-xyz -- /bin/sh

# Run a single command:
kubectl exec web-app-abc123-xyz -- cat /etc/config/app.yaml
kubectl exec web-app-abc123-xyz -- env
kubectl exec web-app-abc123-xyz -- curl localhost:8080/healthz

# For multi-container pods:
kubectl exec -it my-pod -c sidecar -- /bin/sh
```

---

## Common Pod Problems

### ImagePullBackOff / ErrImagePull

```
Symptom:  Pod stuck in ImagePullBackOff
Meaning:  Kubernetes can't pull the container image

Check:
  kubectl describe pod <pod-name>
  # Events show: "Failed to pull image: rpc error: code = NotFound"

Common causes:
  1. Wrong image name:     image: ngimx:1.25  (typo!)
  2. Wrong tag:            image: nginx:99.0   (doesn't exist)
  3. Private registry:     need imagePullSecrets
  4. Registry down:        Docker Hub rate limiting

Fix:
  # Check the image name and tag:
  kubectl get pod <pod> -o jsonpath='{.spec.containers[0].image}'
  
  # For private registries:
  kubectl create secret docker-registry my-registry \
    --docker-server=registry.example.com \
    --docker-username=user \
    --docker-password=pass
```

### CrashLoopBackOff

```
Symptom:  Pod keeps restarting (CrashLoopBackOff)
Meaning:  Container starts, crashes, restarts, crashes, restarts...

Check:
  kubectl logs <pod-name>                # see why it crashed
  kubectl logs <pod-name> --previous     # see previous crash logs

Common causes:
  1. Application error:     app crashes on startup (missing config, bad code)
  2. Missing config:        ConfigMap or Secret not found
  3. Wrong command:         entrypoint or command is wrong
  4. Missing dependency:    database not available yet
  5. OOMKilled:             container ran out of memory

Fix:
  # Check exit code:
  kubectl describe pod <pod> | grep "Exit Code"
  # Exit Code 1:   application error (check logs)
  # Exit Code 137:  OOMKilled (increase memory limit)
  # Exit Code 0:    process exited normally (wrong for a server)
```

### Pending

```
Symptom:  Pod stuck in Pending state
Meaning:  Scheduler can't find a node to place the Pod

Check:
  kubectl describe pod <pod-name>
  # Events show the reason

Common causes:
  1. Not enough resources:
     "0/3 nodes are available: 3 Insufficient cpu"
     Fix: reduce requests, add nodes, or delete unused pods

  2. Node selector doesn't match:
     "0/3 nodes are available: 3 node(s) didn't match node selector"
     Fix: check nodeSelector labels match actual node labels

  3. Taints:
     "0/3 nodes are available: 3 node(s) had taints not tolerated"
     Fix: add tolerations or remove taints

  4. PVC not bound:
     "persistentvolumeclaim 'my-pvc' not found"
     Fix: create the PVC or check StorageClass
```

### OOMKilled

```
Symptom:  Container killed with OOMKilled
Meaning:  Container used more memory than its limit

Check:
  kubectl describe pod <pod-name>
  # Last State:  Terminated
  #   Reason:    OOMKilled
  #   Exit Code: 137

Fix:
  Increase memory limit:
    resources:
      limits:
        memory: "512Mi"   # was 256Mi, increase it

  Or fix the memory leak in your application.
  Check: kubectl top pod <pod-name>  to see current usage
```

---

## Debugging Services

```bash
# Service not routing traffic?

# 1. Check the Service exists:
kubectl get svc web-service
# NAME          TYPE        CLUSTER-IP     PORT(S)
# web-service   ClusterIP   10.100.50.25   80/TCP

# 2. Check endpoints (are there Pods behind the Service?):
kubectl get endpoints web-service
# NAME          ENDPOINTS                           AGE
# web-service   10.244.1.5:8080,10.244.2.8:8080     2h

# If ENDPOINTS is empty → selector doesn't match any Pods!
# Check labels match:
kubectl get pods --show-labels
kubectl describe svc web-service  # look at Selector

# 3. Test from inside the cluster:
kubectl run test --image=busybox --rm -it -- sh
  wget -qO- http://web-service
  nslookup web-service

# 4. Check port mapping:
# Service port 80 → targetPort 8080
# Is your app actually listening on 8080?
kubectl exec <pod> -- curl localhost:8080
```

---

## Debugging DNS

```bash
# DNS not resolving?

# 1. Check CoreDNS is running:
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Test DNS from a pod:
kubectl run dns-test --image=busybox --rm -it -- nslookup web-service
# If this fails → CoreDNS problem

# 3. Check the pod's DNS config:
kubectl exec <pod> -- cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local

# 4. Try the full DNS name:
kubectl run dns-test --image=busybox --rm -it -- \
  nslookup web-service.default.svc.cluster.local
```

---

## Debugging Networking

```bash
# Can't connect between pods?

# 1. Get Pod IPs:
kubectl get pods -o wide

# 2. Test connectivity from one pod to another:
kubectl exec pod-a -- curl http://10.244.2.8:8080

# 3. Check NetworkPolicies:
kubectl get networkpolicies -n <namespace>
# Network policies might be blocking traffic!

# 4. Check the CNI plugin:
kubectl get pods -n kube-system
# Are Calico/Cilium/Flannel pods running?
```

---

## Debugging Nodes

```bash
# Node not ready?

kubectl get nodes
# NAME       STATUS     ROLES    AGE   VERSION
# worker-1   Ready      <none>   30d   v1.31.0
# worker-2   NotReady   <none>   30d   v1.31.0

kubectl describe node worker-2
# Look for:
#   Conditions:
#     MemoryPressure:   True  (node running out of memory)
#     DiskPressure:     True  (node running out of disk)
#     PIDPressure:      True  (too many processes)
#     Ready:            False (kubelet not healthy)

# Check kubelet on the node:
ssh worker-2
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago"
```

---

## Quick Reference: Debugging Commands

```bash
# State:
kubectl get pods/deploy/svc/nodes
kubectl get all
kubectl get events --sort-by=.lastTimestamp

# Details:
kubectl describe pod/deploy/svc/node <name>

# Logs:
kubectl logs <pod> [--previous] [--tail=N] [-f]

# Inside container:
kubectl exec -it <pod> -- /bin/sh

# Resource usage:
kubectl top pods
kubectl top nodes

# Network:
kubectl run test --image=busybox --rm -it -- sh
  wget -qO- http://<service>
  nslookup <service>

# YAML output (see full spec):
kubectl get pod <name> -o yaml

# Events (cluster-wide):
kubectl get events -A --sort-by=.lastTimestamp
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
kubectl describe           → First thing to check, always
                           → Events show exactly what went wrong
kubectl logs               → Application-level debugging
                           → "Why did my code crash?"
kubectl exec               → Live debugging inside containers
                           → Checking config, environment, connectivity
Service debugging          → "Why can't my app reach the database?"
                           → Endpoint mismatches, selector problems
Node debugging             → Infrastructure issues
                           → Capacity planning
Common patterns            → ImagePullBackOff, CrashLoopBackOff, Pending
                           → You'll see these every week in production
```

---

> **Next:** 22 — Hands-On Lab → Build and deploy a real application on Kubernetes.
