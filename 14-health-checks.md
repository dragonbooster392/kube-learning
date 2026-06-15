# 14 — Health Checks (Probes)

## The Problem: "Running" Doesn't Mean "Working"

```
Your Pod shows STATUS: Running. But is it actually healthy?

  Scenario 1: App started but hasn't loaded its config yet
              → Pod is Running, but NOT ready to serve traffic
              
  Scenario 2: App hit a deadlock (stuck, not responding)
              → Pod is Running, but NOT working
              
  Scenario 3: App takes 2 minutes to start (loading ML model)
              → Kubernetes thinks it's broken and keeps restarting it

Without health checks, Kubernetes has no way to know
if your app is truly healthy. It can only see "process running."
```

Kubernetes uses **probes** to check the health of your containers.

---

## Three Types of Probes

```
Liveness Probe:    "Is the container alive?"
                   If it fails → Kubernetes RESTARTS the container.
                   Catches: deadlocks, infinite loops, hung processes.

Readiness Probe:   "Is the container ready to receive traffic?"
                   If it fails → Kubernetes STOPS sending traffic to it.
                   Catches: app still starting, dependency unavailable.

Startup Probe:     "Has the container finished starting up?"
                   If it fails → Kubernetes RESTARTS the container.
                   Catches: slow-starting apps.
                   While active, liveness and readiness probes are disabled.
```

```
Probe Timeline:

  Container starts
  │
  ├── Startup Probe checks ──────── (is it done starting?)
  │     ✗ fail → restart
  │     ✓ pass → startup probe stops, other probes begin
  │
  ├── Liveness Probe checks ─────── (is it still alive?)
  │     ✗ fail → restart container
  │     ✓ pass → continue
  │
  └── Readiness Probe checks ────── (can it handle requests?)
        ✗ fail → remove from Service endpoints (no traffic)
        ✓ pass → add to Service endpoints (receives traffic)
```

---

## Probe Methods

### HTTP GET Probe

```yaml
livenessProbe:
  httpGet:
    path: /healthz                  # endpoint to check
    port: 8080                      # port to check
  initialDelaySeconds: 10           # wait 10s before first check
  periodSeconds: 15                 # check every 15s
  timeoutSeconds: 3                 # timeout after 3s
  failureThreshold: 3               # fail 3 times → take action
  successThreshold: 1               # succeed 1 time → mark healthy
```

```
Kubernetes sends:  GET http://pod-ip:8080/healthz
Expected:          HTTP 200-399 = healthy
                   Anything else = unhealthy

This is the MOST COMMON probe type for web applications.
```

### TCP Socket Probe

```yaml
livenessProbe:
  tcpSocket:
    port: 5432                      # try to connect to this port
  initialDelaySeconds: 15
  periodSeconds: 20
```

```
Kubernetes tries to open a TCP connection to port 5432.
If it connects → healthy.
If it can't connect → unhealthy.

Good for: databases, Redis, or any TCP service without HTTP.
```

### Exec Probe

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy                # run this command inside the container
  initialDelaySeconds: 5
  periodSeconds: 10
```

```
Kubernetes runs the command inside the container.
Exit code 0 → healthy.
Any other exit code → unhealthy.

Good for: custom health checks, checking file existence, running scripts.
```

---

## Liveness Probe — Is It Alive?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
    - name: app
      image: my-app:1.0
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10        # give app 10s to start
        periodSeconds: 15              # check every 15 seconds
        failureThreshold: 3            # 3 failures → restart
```

```
What happens:
  t=0s:   Container starts
  t=10s:  First liveness check → GET /healthz
  t=25s:  Second check → GET /healthz
  t=40s:  Third check → GET /healthz

  If /healthz returns 200: all good, continue
  If /healthz returns 500 three times in a row:
    Kubernetes kills the container and restarts it
    
  You'll see in:
    kubectl describe pod web-app
    Events:
      Liveness probe failed: HTTP 500
      Container web-app restarted
```

---

## Readiness Probe — Is It Ready?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
    - name: app
      image: my-app:1.0
      ports:
        - containerPort: 8080
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 3
```

```
Readiness probe does NOT restart the container.
It controls whether the Pod receives traffic from Services.

Pod starts:
  readiness check fails → Pod removed from Service endpoints
                          (Service stops sending traffic to this Pod)
  
App finishes loading:
  readiness check passes → Pod added to Service endpoints
                           (Service starts sending traffic)

App's database goes down:
  readiness check fails → Pod removed from endpoints again
                          (traffic goes to other healthy Pods)
  
Database comes back:
  readiness check passes → Pod added back to endpoints
```

---

## Startup Probe — For Slow-Starting Apps

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: legacy-app
spec:
  containers:
    - name: app
      image: legacy-app:1.0
      ports:
        - containerPort: 8080
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30          # try 30 times
        periodSeconds: 10             # every 10 seconds
        # Total: 30 × 10 = 300 seconds (5 minutes) to start

      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 15
        failureThreshold: 3

      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 10
```

```
Without startup probe:
  App needs 2 minutes to start.
  Liveness probe starts at 10 seconds.
  Liveness fails → Kubernetes restarts container.
  App tries to start again → fails again → restart loop.

With startup probe:
  Startup probe: "I'll check for up to 5 minutes"
  Once startup probe passes:
    "App is started! Now liveness and readiness probes, take over."
  Startup probe is disabled after first success.
```

---

## Using All Three Probes Together

```yaml
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
        - name: app
          image: my-app:1.0
          ports:
            - containerPort: 8080

          # Phase 1: Is the app done starting?
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

          # Phase 2: Is the app still alive? (after startup)
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 15
            failureThreshold: 3

          # Phase 3: Can the app handle traffic?
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
```

---

## What Your Health Endpoints Should Do

```python
# /healthz — Liveness (is the process alive?)
@app.route('/healthz')
def health():
    return {'status': 'ok'}, 200
    # Keep it simple. Just "I'm alive."
    # Don't check external dependencies here.
    # If the database is down, you don't want to restart YOUR app.

# /ready — Readiness (can I handle traffic?)
@app.route('/ready')
def ready():
    # Check dependencies:
    if not database.is_connected():
        return {'status': 'not ready', 'reason': 'database down'}, 503
    if not cache.is_connected():
        return {'status': 'not ready', 'reason': 'cache down'}, 503
    return {'status': 'ready'}, 200
```

```
Rule of thumb:
  Liveness:   "Is my process working?" (check internal state only)
  Readiness:  "Can I serve users?"     (check dependencies too)

Common mistake:
  Checking the database in the liveness probe.
  Database goes down → liveness fails → container restarts.
  But restarting won't fix the database!
  The app would be fine once the database recovers.
  Use readiness for dependency checks instead.
```

---

## Probe Configuration Reference

```
Parameter              Default  Description
─────────              ───────  ───────────
initialDelaySeconds    0        Wait before first probe
periodSeconds          10       How often to probe
timeoutSeconds         1        Timeout for each probe
failureThreshold       3        Failures before action
successThreshold       1        Successes to be healthy (readiness only, liveness is always 1)
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Liveness probes            → Auto-recovery from deadlocks and hangs
                           → No more "restart the pod manually"
Readiness probes           → Zero-downtime deployments
                           → During rolling update, new pod must pass readiness
                             before old pod is terminated
                           → Graceful handling of dependency outages
Startup probes             → Slow-starting apps (Java, ML models)
                           → Prevents restart loops
Health endpoints           → Standard in every production app
                           → /healthz and /ready are industry conventions
                           → Load balancers and service meshes use these too
```

---

> **Next:** 15 — Resource Management → CPU/memory requests, limits, and autoscaling.
