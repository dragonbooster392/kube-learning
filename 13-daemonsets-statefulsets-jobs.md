# 13 — DaemonSets, StatefulSets & Jobs

Beyond Deployments, Kubernetes has specialized controllers for different workload patterns.

---

## DaemonSets — Run on Every Node

A DaemonSet ensures that **one copy of a Pod runs on every node** in the cluster (or a subset of nodes).

```
Deployment: "I want 3 replicas" → K8s picks 3 nodes
DaemonSet:  "I want one on EVERY node" → K8s puts one on each

Cluster with 5 nodes + DaemonSet:
  Node 1: [daemon-pod]
  Node 2: [daemon-pod]
  Node 3: [daemon-pod]
  Node 4: [daemon-pod]
  Node 5: [daemon-pod]

Add a 6th node:
  Node 6: [daemon-pod]  ← automatically added!

Remove Node 3:
  daemon-pod on Node 3 ← automatically removed
```

### When to Use DaemonSets

```
DaemonSets are for things that need to run on EVERY node:
  - Log collectors      (Fluentd, Filebeat — collect logs from each node)
  - Monitoring agents   (Prometheus node-exporter — collect node metrics)
  - Storage daemons     (Ceph, GlusterFS — provide distributed storage)
  - Network plugins     (Calico, Cilium — CNI networking)
  - Security agents     (Falco — runtime security monitoring)
```

### DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  labels:
    app: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
        - name: fluentd
          image: fluentd:v1.16
          volumeMounts:
            - name: varlog
              mountPath: /var/log        # access node's logs
      volumes:
        - name: varlog
          hostPath:
            path: /var/log               # mount node's /var/log
```

```
Notice: The YAML looks almost like a Deployment, but:
  - Kind is DaemonSet
  - No "replicas" field (it runs on every node automatically)
```

```bash
# Check DaemonSet:
kubectl get daemonset
# NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
# log-collector   5         5         5       5            5
# (5 nodes = 5 pods)

# Run on specific nodes only (using node selector):
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd                 # only run on nodes labeled disk=ssd
```

---

## StatefulSets — Ordered, Stable Identity

A StatefulSet is for **stateful applications** that need:
- Stable network identity (predictable hostnames)
- Ordered startup and shutdown
- Persistent storage per replica

```
Deployment Pods:                StatefulSet Pods:
  web-abc123                      web-0
  web-xyz789                      web-1
  web-def456                      web-2
  (random names)                  (ordered, predictable)

Deployment:   Pods are interchangeable (cattle)
StatefulSet:  Each Pod is unique (pets)
```

### Why StatefulSets Exist

```
Databases need:
  1. Stable hostname: "I'm always db-0, connect to me at db-0.db-service"
  2. Ordered startup: db-0 starts first (primary), then db-1 (replica)
  3. Own storage: each replica has its OWN persistent volume
  4. Ordered shutdown: db-2 stops first, then db-1, then db-0

Deployments can't do any of this. StatefulSets can.
```

### StatefulSet YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless       # required: headless service name
  replicas: 3
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
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:                # each Pod gets its OWN PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

```yaml
# Headless Service (required for StatefulSet):
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None                      # headless = no cluster IP
  selector:
    app: postgres
  ports:
    - port: 5432
```

### How StatefulSet Naming Works

```
StatefulSet name: postgres
Replicas: 3

Pod names (stable, predictable):
  postgres-0
  postgres-1
  postgres-2

DNS names (via headless service):
  postgres-0.postgres-headless.default.svc.cluster.local
  postgres-1.postgres-headless.default.svc.cluster.local
  postgres-2.postgres-headless.default.svc.cluster.local

Each Pod gets its own PVC:
  data-postgres-0   (10Gi)
  data-postgres-1   (10Gi)
  data-postgres-2   (10Gi)
```

```
Startup order:
  postgres-0 starts first and becomes Ready
  postgres-1 starts ONLY after postgres-0 is Ready
  postgres-2 starts ONLY after postgres-1 is Ready

Shutdown order (reverse):
  postgres-2 stops first
  postgres-1 stops after postgres-2 is terminated
  postgres-0 stops last

If postgres-1 crashes:
  Only postgres-1 is restarted (same name, same PVC, same data)
```

### When to Use StatefulSets

```
StatefulSet:
  ✓ Databases (PostgreSQL, MySQL, MongoDB)
  ✓ Message queues (Kafka, RabbitMQ)
  ✓ Distributed systems (Elasticsearch, Zookeeper)
  ✓ Any app that needs stable identity or ordered deployment

Deployment (use for everything else):
  ✓ Web servers
  ✓ APIs
  ✓ Workers
  ✓ Any stateless application
```

---

## Jobs — Run to Completion

A Job creates Pods that run a task and then **exit**. Unlike Deployments (which run forever), Jobs are for one-time or batch tasks.

```
Deployment: "Keep 3 pods running forever"
Job:        "Run this task once and stop"
```

### Job YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: database-migration
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: my-app:1.0
          command: ["python", "manage.py", "migrate"]
      restartPolicy: Never              # don't restart on success
  backoffLimit: 3                       # retry up to 3 times on failure
```

```
Job lifecycle:
  1. Job creates a Pod
  2. Pod runs the command
  3. Command exits with code 0 (success)
  4. Job marks as Complete
  5. Pod stays around (for log inspection) until you delete it

If the command fails:
  Pod is recreated (up to backoffLimit times)
  If all retries fail, Job marks as Failed
```

### Parallel Jobs

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-process
spec:
  completions: 10                # total tasks to complete
  parallelism: 3                 # run 3 at a time
  template:
    spec:
      containers:
        - name: worker
          image: worker:1.0
      restartPolicy: Never
```

```
This runs 10 tasks, 3 at a time:
  Batch 1: pod-1, pod-2, pod-3  (running)
  Batch 2: pod-4, pod-5, pod-6  (after first 3 complete)
  Batch 3: pod-7, pod-8, pod-9
  Batch 4: pod-10
  Total completions: 10 ✓
```

```bash
# Check Job status:
kubectl get jobs
# NAME                 COMPLETIONS   DURATION   AGE
# database-migration   1/1           15s        2m
# batch-process        7/10          45s        1m

# See the Pods created by a Job:
kubectl get pods -l job-name=database-migration
```

---

## CronJobs — Scheduled Jobs

A CronJob creates Jobs on a schedule, like Linux cron.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"                # 2:00 AM every day
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: backup-tool:1.0
              command: ["./backup.sh"]
          restartPolicy: Never
  successfulJobsHistoryLimit: 3        # keep last 3 successful jobs
  failedJobsHistoryLimit: 1            # keep last 1 failed job
```

### Cron Schedule Format

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6, 0=Sunday)
│ │ │ │ │
* * * * *

Examples:
  "0 2 * * *"        Every day at 2:00 AM
  "*/15 * * * *"     Every 15 minutes
  "0 0 * * 0"        Every Sunday at midnight
  "0 9 1 * *"        First day of every month at 9:00 AM
  "30 8 * * 1-5"     Weekdays at 8:30 AM
```

```bash
# Check CronJobs:
kubectl get cronjobs
# NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE
# nightly-backup   0 2 * * *     False     0        2h

# Manually trigger a CronJob now:
kubectl create job manual-backup --from=cronjob/nightly-backup
```

---

## Comparison Table

```
                 Deployment     DaemonSet       StatefulSet     Job          CronJob
─────────────    ──────────     ─────────       ───────────     ───          ───────
Purpose          Stateless      One per node    Stateful        One-time     Scheduled
                 apps                           apps            task         tasks

Replicas         You choose     Auto (=nodes)   You choose      completions  Per schedule
Pod Names        Random         Random          Ordered (0,1,2) Random       Random
Stable Storage   No             No              Yes (per pod)   No           No
Startup Order    Parallel       Parallel        Sequential      Parallel*    Parallel*
Runs Forever     Yes            Yes             Yes             No           No
Example          Web server     Log collector   Database        Migration    Backup
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
DaemonSets                 → Logging (EFK/Loki) on every node
                           → Monitoring (node-exporter) on every node
                           → Security scanning on every node
StatefulSets               → Databases in Kubernetes
                           → Kafka, Elasticsearch, Redis clusters
                           → Any app needing stable identity
Jobs                       → Database migrations in CI/CD
                           → Data processing, ETL pipelines
                           → One-time setup tasks
CronJobs                   → Automated backups
                           → Report generation
                           → Cleanup tasks (delete old data)
                           → Certificate rotation
```

---

> **Next:** 14 — Health Checks → Liveness, readiness, and startup probes.
