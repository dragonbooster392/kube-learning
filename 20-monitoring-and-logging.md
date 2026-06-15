# 20 — Monitoring & Logging

## Why Monitoring Matters

```
Without monitoring:
  "Is the app slow?" → "I don't know"
  "How much CPU is it using?" → "No idea"
  "When did it start failing?" → "Someone noticed 2 hours ago"

With monitoring:
  "Is the app slow?" → "Response time is 450ms, up from 50ms at 2:15 PM"
  "How much CPU is it using?" → "75% of its limit, scaling event expected"
  "When did it start failing?" → "Error rate spiked at 2:15 PM, here's the graph"
```

---

## The Monitoring Stack

```
The standard Kubernetes monitoring stack:

  metrics-server  → Basic CPU/memory metrics (kubectl top)
  Prometheus      → Collects and stores detailed metrics
  Grafana         → Visualizes metrics in dashboards
  Alertmanager    → Sends alerts (email, Slack, PagerDuty)

Together: Prometheus + Grafana + Alertmanager = the "Prometheus stack"
```

---

## metrics-server — Basic Metrics

metrics-server provides basic CPU and memory usage. It's required for `kubectl top` and HPA.

```bash
# Install metrics-server:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check node resource usage:
kubectl top nodes
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# worker-1   350m         8%     2100Mi          26%
# worker-2   720m         18%    3400Mi          42%

# Check pod resource usage:
kubectl top pods
# NAME                       CPU(cores)   MEMORY(bytes)
# web-app-abc123-xyz         50m          128Mi
# api-server-def456-uvw      120m         256Mi

# Top pods sorted by CPU:
kubectl top pods --sort-by=cpu

# Top pods in all namespaces:
kubectl top pods -A
```

---

## Prometheus — Metrics Collection

Prometheus **scrapes** (pulls) metrics from your applications and Kubernetes components at regular intervals.

```
How Prometheus works:

  Your App → exposes /metrics endpoint (port 9090)
  Prometheus → scrapes /metrics every 15 seconds
  Prometheus → stores the time-series data
  Grafana → queries Prometheus and shows dashboards

  Prometheus PULLS metrics from targets.
  (Unlike some systems where apps PUSH metrics.)
```

### What Prometheus Collects

```
Kubernetes metrics (built-in):
  - Node CPU, memory, disk, network usage
  - Pod CPU, memory usage
  - Container restarts, status
  - API server request latency
  - etcd health
  - Scheduler performance

Application metrics (you add these):
  - HTTP request count
  - Request duration (latency)
  - Error rate
  - Queue length
  - Active connections
  - Business metrics (orders/minute, signups/hour)
```

### Installing Prometheus Stack with Helm

```bash
# Add the Prometheus community Helm repo:
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install the full stack (Prometheus + Grafana + Alertmanager):
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Check what was installed:
kubectl get pods -n monitoring
# NAME                                                    READY   STATUS
# monitoring-grafana-abc123-xyz                           1/1     Running
# monitoring-kube-prometheus-operator-def456-uvw          1/1     Running
# monitoring-kube-state-metrics-ghi789-rst                1/1     Running
# monitoring-prometheus-node-exporter-xxxxx               1/1     Running
# prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running
# alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running
```

### Exposing Your App Metrics

```python
# Python app with prometheus_client:
from prometheus_client import Counter, Histogram, start_http_server

# Define metrics:
REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'path', 'status'])
REQUEST_DURATION = Histogram('http_request_duration_seconds', 'HTTP request duration')

# In your request handler:
@REQUEST_DURATION.time()
def handle_request():
    # ... your code ...
    REQUEST_COUNT.labels(method='GET', path='/api/users', status='200').inc()

# Start metrics server on port 9090:
start_http_server(9090)
```

```yaml
# Tell Prometheus to scrape your app (via annotations):
apiVersion: v1
kind: Pod
metadata:
  annotations:
    prometheus.io/scrape: "true"      # Prometheus should scrape this
    prometheus.io/port: "9090"        # on this port
    prometheus.io/path: "/metrics"    # at this path
spec:
  containers:
    - name: app
      image: my-app:1.0
```

---

## Grafana — Visualization

Grafana displays Prometheus metrics as dashboards and graphs.

```bash
# Access Grafana (port-forward):
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring

# Open: http://localhost:3000
# Default credentials (from Helm chart):
#   Username: admin
#   Password: prom-operator

# Grafana comes with pre-built dashboards for:
#   - Kubernetes cluster overview
#   - Node resource usage
#   - Pod resource usage
#   - API server performance
#   - etcd health
```

```
Popular Grafana dashboards (import by ID from grafana.com):
  315:   Kubernetes cluster monitoring
  6417:  Kubernetes Pods
  1860:  Node Exporter Full
  7249:  Kubernetes Cluster (Prometheus)
```

---

## Alertmanager — Sending Alerts

```yaml
# PrometheusRule: trigger alert when pod keeps restarting
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-alerts
  namespace: monitoring
spec:
  groups:
    - name: pod-alerts
      rules:
        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
            description: "Pod {{ $labels.pod }} in {{ $labels.namespace }} has been restarting"

        - alert: HighMemoryUsage
          expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} memory > 90% of limit"
```

```
Alert flow:
  Prometheus evaluates rules → condition true for 5 minutes
    → sends alert to Alertmanager
    → Alertmanager routes to: Slack, email, PagerDuty, etc.
```

---

## Logging

Monitoring tells you WHAT is happening. Logs tell you WHY.

```
Logging levels:
  Container logs    → stdout/stderr from your app
  Node logs         → kubelet, container runtime
  Cluster logs      → API server, scheduler, controllers

kubectl logs covers container logs.
For cluster-wide log aggregation, you need a logging stack.
```

### kubectl Logs

```bash
# View pod logs:
kubectl logs my-pod

# Follow logs (live):
kubectl logs my-pod -f

# Last 100 lines:
kubectl logs my-pod --tail=100

# Logs from a specific container (multi-container pod):
kubectl logs my-pod -c sidecar

# Logs from previous container (if it crashed):
kubectl logs my-pod --previous

# Logs from all pods with a label:
kubectl logs -l app=web

# Logs with timestamps:
kubectl logs my-pod --timestamps
```

### Log Aggregation Stacks

```
EFK Stack (Elasticsearch + Fluentd + Kibana):
  Fluentd:         Runs as DaemonSet, collects logs from every node
  Elasticsearch:   Stores and indexes logs
  Kibana:          Web UI for searching and visualizing logs

PLG Stack (Promtail + Loki + Grafana):
  Promtail:        Runs as DaemonSet, collects logs
  Loki:            Stores logs (lighter than Elasticsearch)
  Grafana:         Same Grafana you use for metrics

Loki is increasingly popular because:
  - Lighter than Elasticsearch (less resources)
  - Uses same Grafana (one dashboard for metrics AND logs)
  - Labels-based (same concept as Prometheus)
```

```bash
# Install Loki stack with Helm:
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true
```

```
Log flow (Loki):
  Container → writes to stdout
    → kubelet captures to /var/log/containers/
    → Promtail (DaemonSet) reads log files
    → Promtail sends to Loki
    → Grafana queries Loki
    → You search logs in Grafana
```

---

## The Four Golden Signals

```
Google's SRE book defines four key metrics to monitor:

1. LATENCY:    How long requests take
               Metric: http_request_duration_seconds

2. TRAFFIC:    How many requests per second
               Metric: http_requests_total (rate)

3. ERRORS:     How many requests fail
               Metric: http_requests_total{status=~"5.."}

4. SATURATION: How full your resources are
               Metric: CPU usage, memory usage, disk I/O

If you monitor these four things, you'll catch most problems.
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
metrics-server             → kubectl top, HPA (autoscaling)
Prometheus                 → Industry standard for K8s monitoring
                           → Every production cluster needs this
Grafana                    → Dashboards for everyone
                           → Developers, ops, management
Alertmanager               → On-call alerting
                           → PagerDuty, Slack, email
Loki/EFK                   → Debugging production issues
                           → "Why did it fail at 3 AM?"
                           → Compliance (audit logs)
Four Golden Signals        → SRE best practices
                           → SLOs and SLAs
```

---

> **Next:** 21 — Troubleshooting → How to debug when things go wrong.
