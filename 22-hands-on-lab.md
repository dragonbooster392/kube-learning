# 22 — Hands-On Lab

This lab walks you through building and deploying a complete application on Kubernetes. Every command is real — run them on your own cluster.

---

## Prerequisites

You need a running Kubernetes cluster. Pick one:

```bash
# Option 1: minikube (simplest for learning)
# Install:
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start a cluster:
minikube start --driver=docker --cpus=2 --memory=4096

# Option 2: kind (Kubernetes in Docker)
# Install:
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Start a cluster:
kind create cluster --name lab

# Option 3: k3s (lightweight, single binary)
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

```bash
# Verify your cluster is running:
kubectl cluster-info
# Kubernetes control plane is running at https://...

kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# minikube       Ready    control-plane   1m    v1.31.0
```

---

## Lab 1: Your First Pod

```bash
# Create a simple nginx pod:
kubectl run my-nginx --image=nginx:1.25

# Check it's running:
kubectl get pods
# NAME       READY   STATUS    RESTARTS   AGE
# my-nginx   1/1     Running   0          30s

# Get more details:
kubectl describe pod my-nginx

# See the pod's IP address:
kubectl get pod my-nginx -o wide

# Access nginx from inside the cluster:
kubectl exec my-nginx -- curl -s localhost:80 | head -5

# View the logs:
kubectl logs my-nginx

# Delete the pod:
kubectl delete pod my-nginx
```

---

## Lab 2: Deployment + Service

```bash
# Create a namespace for our lab:
kubectl create namespace lab
```

```yaml
# Save as lab-deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: lab
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
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: lab
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
# Apply:
kubectl apply -f lab-deployment.yaml

# Check the deployment:
kubectl get deployment -n lab
# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# web-app   3/3     3            3           30s

# Check the pods:
kubectl get pods -n lab
# NAME                       READY   STATUS    RESTARTS   AGE
# web-app-abc123-xyz         1/1     Running   0          30s
# web-app-abc123-uvw         1/1     Running   0          30s
# web-app-abc123-rst         1/1     Running   0          30s

# Check the service:
kubectl get svc -n lab
# NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
# web-service   ClusterIP   10.100.50.25   <none>        80/TCP

# Check the endpoints (pods behind the service):
kubectl get endpoints web-service -n lab

# Test the service from inside the cluster:
kubectl run test-curl --image=curlimages/curl --rm -it -n lab -- \
  curl -s http://web-service
```

---

## Lab 3: ConfigMap + Secret

```yaml
# Save as lab-config.yaml:
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: lab
data:
  WELCOME_MESSAGE: "Hello from Kubernetes!"
  APP_COLOR: "blue"
  APP_MODE: "production"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: lab
type: Opaque
stringData:
  API_KEY: "my-super-secret-key-12345"
  DB_PASSWORD: "postgres-pass-99"
```

```bash
# Apply:
kubectl apply -f lab-config.yaml

# Verify:
kubectl get configmap app-config -n lab
kubectl get secret app-secret -n lab

# View ConfigMap data:
kubectl describe configmap app-config -n lab

# View Secret data (base64 encoded):
kubectl get secret app-secret -n lab -o yaml

# Decode a secret value:
kubectl get secret app-secret -n lab -o jsonpath='{.data.API_KEY}' | base64 -d
# my-super-secret-key-12345
```

Now update the deployment to use them:

```yaml
# Save as lab-deployment-v2.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: lab
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
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secret
```

```bash
# Apply the updated deployment:
kubectl apply -f lab-deployment-v2.yaml

# Wait for rollout:
kubectl rollout status deployment/web-app -n lab

# Verify env vars inside a pod:
POD=$(kubectl get pods -n lab -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -n lab -- env | grep -E "WELCOME|APP_|API_KEY|DB_PASS"
# WELCOME_MESSAGE=Hello from Kubernetes!
# APP_COLOR=blue
# APP_MODE=production
# API_KEY=my-super-secret-key-12345
# DB_PASSWORD=postgres-pass-99
```

---

## Lab 4: Rolling Update + Rollback

```bash
# Check current image:
kubectl get deployment web-app -n lab -o jsonpath='{.spec.template.spec.containers[0].image}'
# nginx:1.25

# Update to a new version:
kubectl set image deployment/web-app nginx=nginx:1.26 -n lab

# Watch the rollout:
kubectl rollout status deployment/web-app -n lab
# Waiting for deployment "web-app" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "web-app" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "web-app" successfully rolled out

# Check the new image:
kubectl get deployment web-app -n lab -o jsonpath='{.spec.template.spec.containers[0].image}'
# nginx:1.26

# View rollout history:
kubectl rollout history deployment/web-app -n lab

# Rollback to the previous version:
kubectl rollout undo deployment/web-app -n lab

# Verify rollback:
kubectl get deployment web-app -n lab -o jsonpath='{.spec.template.spec.containers[0].image}'
# nginx:1.25
```

---

## Lab 5: Scaling

```bash
# Manual scaling:
kubectl scale deployment web-app -n lab --replicas=5

# Watch pods come up:
kubectl get pods -n lab --watch

# Scale down:
kubectl scale deployment web-app -n lab --replicas=2

# If metrics-server is installed, create an HPA:
kubectl autoscale deployment web-app -n lab \
  --min=2 --max=10 --cpu-percent=50

# Check HPA:
kubectl get hpa -n lab
```

---

## Lab 6: Storage

```yaml
# Save as lab-storage.yaml:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-test
  namespace: lab
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c"]
      args:
        - |
          echo "Written at $(date)" > /data/test.txt
          echo "Data written successfully!"
          cat /data/test.txt
          sleep 3600
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

```bash
# Apply:
kubectl apply -f lab-storage.yaml

# Check PVC is bound:
kubectl get pvc -n lab
# NAME       STATUS   VOLUME        CAPACITY   ACCESS MODES
# data-pvc   Bound    pvc-abc123    1Gi        RWO

# Check the data was written:
kubectl logs storage-test -n lab
# Data written successfully!
# Written at Mon Jan 1 12:00:00 UTC 2025

# Delete and recreate the pod (data persists!):
kubectl delete pod storage-test -n lab
kubectl apply -f lab-storage.yaml

# Check — the file is still there:
kubectl exec storage-test -n lab -- cat /data/test.txt
# Written at Mon Jan 1 12:00:00 UTC 2025
```

---

## Lab 7: Namespaces + Resource Quotas

```yaml
# Save as lab-quota.yaml:
apiVersion: v1
kind: Namespace
metadata:
  name: team-dev
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: team-dev
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
    pods: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: team-dev
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

```bash
# Apply:
kubectl apply -f lab-quota.yaml

# Check:
kubectl get resourcequota -n team-dev
kubectl describe resourcequota dev-quota -n team-dev

# Create a pod (gets default limits from LimitRange):
kubectl run test-pod --image=nginx -n team-dev

# Check — it has the default resource settings:
kubectl get pod test-pod -n team-dev -o yaml | grep -A 6 resources

# Try to exceed the quota:
kubectl create deployment big-app --image=nginx --replicas=15 -n team-dev
kubectl get pods -n team-dev
# Only 10 pods allowed!
```

---

## Lab 8: Labels and Selectors

```bash
# Create pods with different labels:
kubectl run web-prod --image=nginx -n lab --labels="app=web,env=production"
kubectl run web-dev --image=nginx -n lab --labels="app=web,env=development"
kubectl run api-prod --image=nginx -n lab --labels="app=api,env=production"
kubectl run api-dev --image=nginx -n lab --labels="app=api,env=development"

# Show all labels:
kubectl get pods -n lab --show-labels

# Filter by label:
kubectl get pods -n lab -l app=web
kubectl get pods -n lab -l env=production
kubectl get pods -n lab -l 'app=web,env=production'
kubectl get pods -n lab -l 'env in (production, staging)'

# Show specific labels as columns:
kubectl get pods -n lab -L app,env

# Add a label:
kubectl label pod web-prod -n lab tier=frontend

# Delete pods by label:
kubectl delete pods -l env=development -n lab
```

---

## Lab 9: Troubleshooting Practice

```bash
# Create a broken pod (wrong image name):
kubectl run broken-image --image=nginx:doesnotexist -n lab

# Diagnose:
kubectl get pods -n lab
# broken-image   0/1     ErrImagePull   0          10s

kubectl describe pod broken-image -n lab | tail -10
# Events show: Failed to pull image "nginx:doesnotexist"

# Fix it:
kubectl set image pod/broken-image broken-image=nginx:1.25 -n lab
# (or delete and recreate)

# Create a pod that crashes:
kubectl run crash-pod --image=busybox -n lab -- /bin/sh -c "exit 1"

# Diagnose:
kubectl get pods -n lab
# crash-pod   0/1     CrashLoopBackOff   3          1m

kubectl logs crash-pod -n lab --previous
kubectl describe pod crash-pod -n lab | grep -A 5 "Last State"

# Clean up:
kubectl delete pod broken-image crash-pod -n lab
```

---

## Lab 10: Full Application Stack

```yaml
# Save as lab-full-app.yaml:

# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: full-app-config
  namespace: lab
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Kubernetes Lab</title></head>
    <body>
      <h1>Hello from Kubernetes!</h1>
      <p>Pod: HOSTNAME_PLACEHOLDER</p>
      <p>This page is served by nginx running in a Kubernetes Pod.</p>
      <p>The Deployment has 3 replicas behind a Service.</p>
    </body>
    </html>
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: full-app
  namespace: lab
  labels:
    app: full-app
    version: v1.0.0
  annotations:
    description: "Lab application demonstrating K8s concepts"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: full-app
  template:
    metadata:
      labels:
        app: full-app
        version: v1.0.0
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: full-app-config
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: full-app-service
  namespace: lab
spec:
  selector:
    app: full-app
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

```bash
# Deploy the full stack:
kubectl apply -f lab-full-app.yaml

# Check everything:
kubectl get all -n lab -l app=full-app

# Check probes are working:
kubectl describe deployment full-app -n lab

# Get the NodePort:
NODE_PORT=$(kubectl get svc full-app-service -n lab -o jsonpath='{.spec.ports[0].nodePort}')
echo "Access at: http://localhost:$NODE_PORT"

# For minikube:
minikube service full-app-service -n lab --url

# Test with curl:
kubectl run curl-test --image=curlimages/curl --rm -it -n lab -- \
  curl -s http://full-app-service

# Scale and watch:
kubectl scale deployment full-app -n lab --replicas=5
kubectl get pods -n lab -l app=full-app --watch

# Observe the rolling update:
kubectl set image deployment/full-app nginx=nginx:1.26 -n lab
kubectl rollout status deployment/full-app -n lab
```

---

## Cleanup

```bash
# Delete the lab namespace (deletes everything in it):
kubectl delete namespace lab
kubectl delete namespace team-dev

# If using minikube:
minikube stop
minikube delete

# If using kind:
kind delete cluster --name lab
```

---

## What You Practiced

```
Lab    Concept                          Files Covered
───    ───────                          ─────────────
1      Pods, kubectl basics             04, 03
2      Deployments, Services            06, 07
3      ConfigMaps, Secrets              09
4      Rolling updates, Rollbacks       06
5      Scaling, HPA                     15
6      Persistent storage               10
7      Namespaces, Quotas               08, 15
8      Labels, Selectors                12
9      Troubleshooting                  21
10     Full stack (all together)        All files
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Hands-on practice          → Building muscle memory with kubectl
                           → Understanding how objects relate
Full stack deployment      → Real production patterns
                           → ConfigMap + Secret + Deployment + Service
Troubleshooting practice   → On-call debugging skills
                           → "Why is this pod not starting?"
Rolling updates            → Zero-downtime deployments
                           → CI/CD pipeline patterns
```

---

> **Course Complete!** You now have a solid foundation in Kubernetes.
> From here: practice on a real cluster, deploy your own applications,
> and explore the ecosystem (Helm, ArgoCD, Istio, Terraform + K8s).
