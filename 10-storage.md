# 10 — Storage

## The Problem: Containers Lose Data

When a container restarts, everything written to its filesystem is **gone**. This is because the container filesystem is ephemeral (temporary).

```
Container starts:
  Writes to /data/myfile.txt ✓

Container crashes and restarts:
  /data/myfile.txt → GONE ✗

This is fine for stateless apps (web servers, APIs).
This is NOT fine for databases, file uploads, or any persistent data.
```

---

## Volumes — Attach Storage to Pods

A Volume is storage that is attached to a Pod. It outlives the **container** (survives container restarts within the same Pod) but not necessarily the Pod itself.

### emptyDir — Temporary Shared Storage

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hello > /data/message.txt && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: busybox
      command: ["sh", "-c", "cat /data/message.txt && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
  volumes:
    - name: shared-data
      emptyDir: {}                    # empty directory, created when Pod starts
```

```
emptyDir:
  Created: when Pod is assigned to a node
  Deleted: when Pod is removed from the node
  Use:     sharing files between containers in the same Pod
  NOT for: data that must survive Pod deletion
```

### hostPath — Node's Filesystem

```yaml
volumes:
  - name: host-data
    hostPath:
      path: /var/log              # path on the node
      type: Directory
```

```
hostPath mounts a directory from the NODE's filesystem into the Pod.

⚠️ Dangerous:
  - Ties your Pod to a specific node
  - If Pod moves to another node, the data is different
  - Security risk (Pod can access node files)
  
Use only for: DaemonSets that need node-level access (log collectors, monitoring agents)
```

---

## The Storage Problem at Scale

```
emptyDir:  Data lost when Pod dies
hostPath:  Data stuck on one node

But what if you need:
  - Data that survives Pod deletion?
  - Data that can be accessed from any node?
  - Data that is replicated for reliability?

You need PERSISTENT storage.
```

---

## PersistentVolume (PV) — The Storage Resource

A PersistentVolume is a piece of storage in the cluster, provisioned by an admin or dynamically by a StorageClass.

```
Think of it like a hard drive:
  The hard drive exists independently of any computer.
  You can plug it into different computers.
  It keeps its data even when unplugged.

  PersistentVolume = the hard drive
  Pod = the computer
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi                      # 10 GB
  accessModes:
    - ReadWriteOnce                    # can be mounted read-write by one node
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:                            # where the data lives (for local testing)
    path: /mnt/data
```

### Access Modes

```
ReadWriteOnce (RWO):
  Can be mounted as read-write by ONE node.
  Most common. Used by databases, most applications.
  
ReadOnlyMany (ROX):
  Can be mounted as read-only by MANY nodes.
  Used for shared config files, static content.

ReadWriteMany (RWX):
  Can be mounted as read-write by MANY nodes.
  Requires special storage (NFS, CephFS, EFS).
  Used for shared upload directories.
```

### Reclaim Policies

```
Retain:   When PVC is deleted, PV keeps its data (manual cleanup)
Delete:   When PVC is deleted, PV and its data are deleted
Recycle:  (deprecated) Wipe the data and make PV available again

Retain is safest — you won't accidentally lose data.
Delete is common in cloud (dynamically provisioned storage).
```

---

## PersistentVolumeClaim (PVC) — Requesting Storage

A PVC is a **request for storage**. It's how Pods ask for a PersistentVolume.

```
PV  = the hard drive (provisioned by admin)
PVC = "I need a 10GB hard drive with read-write access" (requested by user)

Kubernetes matches the PVC to a suitable PV.
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi                    # request 5 GB
  storageClassName: standard
```

```
K8s finds a PV that matches:
  - At least 5 GB
  - Supports ReadWriteOnce
  - StorageClass matches

PV (10Gi) ← bound to → PVC (5Gi)
Status: Bound ✓
```

### Using a PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: data
          mountPath: /app/data        # where to mount in the container
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc             # reference the PVC
```

```
The flow:
  1. Admin creates a PV (or StorageClass does it automatically)
  2. Developer creates a PVC requesting storage
  3. K8s binds the PVC to a matching PV
  4. Developer references the PVC in their Pod
  5. Pod starts with the volume mounted

  Pod → PVC → PV → actual disk
```

---

## StorageClasses — Dynamic Provisioning

Manually creating PVs for every request is tedious. StorageClasses automate this.

```
Without StorageClass (manual):
  Admin creates PV (10GB, SSD)
  Developer creates PVC (requests 10GB)
  K8s binds them

With StorageClass (automatic):
  Admin creates a StorageClass (describes how to create storage)
  Developer creates PVC (requests 10GB, references StorageClass)
  StorageClass AUTOMATICALLY creates a PV of the right size
  K8s binds them

No admin intervention needed for each request.
```

```yaml
# StorageClass definition
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs    # or other cloud provisioner
parameters:
  type: gp3                           # AWS EBS volume type
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```yaml
# PVC referencing the StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd           # use the fast-ssd class
```

```
What happens:
  1. PVC requests 20Gi with StorageClass "fast-ssd"
  2. StorageClass tells AWS to create a 20GB gp3 EBS volume
  3. PV is automatically created and bound to the PVC
  4. When Pod references the PVC, the EBS volume is attached to the node

Common cloud StorageClasses:
  AWS:    gp3 (SSD), io2 (high IOPS), sc1 (cold HDD)
  Azure:  managed-premium (SSD), managed-standard (HDD)
  GCP:    pd-ssd (SSD), pd-standard (HDD)
```

---

## Complete Example: Database with Persistent Storage

```yaml
# storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1                          # databases usually run 1 replica
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

```
This creates:
  1. A PVC requesting 10GB storage
  2. A Deployment running PostgreSQL
  3. PostgreSQL data stored on the PV (survives pod restarts)
  4. A Service so other pods can reach it at postgres-service:5432
```

---

## Checking Storage Status

```bash
# List PersistentVolumes:
kubectl get pv
# NAME     CAPACITY   ACCESS MODES   RECLAIM   STATUS   STORAGECLASS
# pv-001   10Gi       RWO            Retain    Bound    standard

# List PersistentVolumeClaims:
kubectl get pvc
# NAME           STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
# postgres-pvc   Bound    pv-001   10Gi       RWO            standard

# Status meanings:
#   Available:  PV is free, no PVC bound
#   Bound:      PV is bound to a PVC
#   Released:   PVC was deleted, PV data still exists (Retain policy)
#   Failed:     Automatic reclamation failed

# List StorageClasses:
kubectl get storageclass
kubectl get sc                # shorthand
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
emptyDir                   → Temp space for sidecar containers
                           → Cache, scratch space
PV/PVC                     → Database storage (PostgreSQL, MySQL, MongoDB)
                           → File uploads
                           → Any stateful workload
StorageClasses             → Cloud storage automation
                           → AWS EBS, Azure Disk, GCP PD
                           → Terraform provisions StorageClasses
Dynamic provisioning       → Self-service storage for developers
                           → No admin bottleneck
Access modes               → Understanding what storage supports
                           → RWX needed for shared file systems
```

---

> **Next:** 11 — Ingress → HTTP routing, TLS termination, and exposing multiple services.
