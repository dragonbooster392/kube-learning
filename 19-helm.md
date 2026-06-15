# 19 — Helm

## The Problem: Too Many YAML Files

```
A simple web app in Kubernetes needs:
  - Deployment
  - Service
  - Ingress
  - ConfigMap
  - Secret
  - HPA
  - ServiceAccount
  - Role + RoleBinding
  - NetworkPolicy

That's 9+ YAML files for ONE application.
Now multiply by dev, staging, and production environments.
And you need different values for each environment.

Managing this manually is painful.
```

---

## What Is Helm

Helm is the **package manager for Kubernetes** — like apt for Ubuntu or pip for Python.

```
apt install nginx         → installs nginx on Linux
helm install my-app ./    → installs your app on Kubernetes

A Helm "package" is called a CHART.
A chart bundles all the YAML files your app needs.
```

```
Helm terminology:
  Chart:    A package of Kubernetes YAML templates
  Values:   Configuration that customizes the chart
  Release:  An installed instance of a chart
  Repository: A place where charts are stored
```

---

## Installing Helm

```bash
# Linux:
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS:
brew install helm

# Verify:
helm version
# version.BuildInfo{Version:"v3.16.x", ...}
```

---

## Using Existing Charts

### Add a Repository

```bash
# Add the Bitnami chart repository:
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repo index:
helm repo update

# Search for charts:
helm search repo nginx
# NAME                    CHART VERSION   APP VERSION   DESCRIPTION
# bitnami/nginx           18.2.1          1.27.0        NGINX web server
# bitnami/nginx-ingress   11.3.0          1.11.0        NGINX Ingress Controller
```

### Install a Chart

```bash
# Install nginx:
helm install my-nginx bitnami/nginx
#             ↑ release name   ↑ chart name

# What this does:
# 1. Downloads the bitnami/nginx chart
# 2. Renders all YAML templates with default values
# 3. Applies them to your Kubernetes cluster
# 4. Creates a "release" named "my-nginx"

# Check what was installed:
helm list
# NAME       NAMESPACE   REVISION   STATUS     CHART          APP VERSION
# my-nginx   default     1          deployed   nginx-18.2.1   1.27.0

# See the Kubernetes resources created:
kubectl get all -l app.kubernetes.io/instance=my-nginx
```

### Customize with Values

```bash
# See all configurable values:
helm show values bitnami/nginx | head -50

# Install with custom values:
helm install my-nginx bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=ClusterIP

# Or use a values file (better for many values):
helm install my-nginx bitnami/nginx -f my-values.yaml
```

```yaml
# my-values.yaml
replicaCount: 3

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

ingress:
  enabled: true
  hostname: app.example.com
```

### Upgrade and Rollback

```bash
# Upgrade (change values or chart version):
helm upgrade my-nginx bitnami/nginx \
  --set replicaCount=5

# Check release history:
helm history my-nginx
# REVISION   STATUS       CHART          DESCRIPTION
# 1          superseded   nginx-18.2.1   Install complete
# 2          deployed     nginx-18.2.1   Upgrade complete

# Rollback to previous version:
helm rollback my-nginx 1

# Uninstall:
helm uninstall my-nginx
```

---

## Creating Your Own Chart

```bash
# Create a chart skeleton:
helm create my-app

# This creates:
my-app/
  Chart.yaml          # chart metadata (name, version, description)
  values.yaml         # default configuration values
  templates/          # Kubernetes YAML templates
    deployment.yaml
    service.yaml
    ingress.yaml
    hpa.yaml
    serviceaccount.yaml
    _helpers.tpl      # template helper functions
    NOTES.txt         # post-install message
  charts/             # sub-charts (dependencies)
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My web application
type: application
version: 1.0.0                    # chart version
appVersion: "2.1.0"               # your app version
```

### values.yaml

```yaml
# Default values for my-app
replicaCount: 2

image:
  repository: my-app
  tag: "2.1.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  hostname: ""

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

env:
  LOG_LEVEL: info
  DATABASE_HOST: postgres
```

### Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
```

```
Template syntax:
  {{ .Values.replicaCount }}     → reads from values.yaml
  {{ .Release.Name }}            → the release name (helm install NAME)
  {{- toYaml .Values.resources | nindent 12 }}  → inject YAML block
  {{- range }}...{{- end }}      → loop
  {{ if }}...{{ end }}           → conditional
```

### Test Your Chart

```bash
# Render templates without installing (see the generated YAML):
helm template my-release ./my-app

# Render with custom values:
helm template my-release ./my-app -f production-values.yaml

# Dry run (checks against cluster):
helm install my-release ./my-app --dry-run

# Lint (check for errors):
helm lint ./my-app
```

### Install Your Chart

```bash
# Install from local directory:
helm install my-release ./my-app

# Install with environment-specific values:
helm install my-release ./my-app -f environments/production.yaml

# Upgrade:
helm upgrade my-release ./my-app -f environments/production.yaml
```

---

## Environment-Specific Values

```
my-app/
  values.yaml                    # defaults
  environments/
    dev.yaml                     # dev overrides
    staging.yaml                 # staging overrides
    production.yaml              # production overrides
```

```yaml
# environments/dev.yaml
replicaCount: 1
image:
  tag: "latest"
resources:
  requests:
    cpu: 50m
    memory: 64Mi
env:
  LOG_LEVEL: debug
  DATABASE_HOST: dev-db
```

```yaml
# environments/production.yaml
replicaCount: 5
image:
  tag: "2.1.0"
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
env:
  LOG_LEVEL: warn
  DATABASE_HOST: prod-db-cluster
ingress:
  enabled: true
  hostname: app.example.com
```

```bash
# Deploy to dev:
helm install my-app-dev ./my-app -f environments/dev.yaml -n dev

# Deploy to production:
helm install my-app-prod ./my-app -f environments/production.yaml -n production
```

---

## Helm in CI/CD

```yaml
# GitLab CI example:
deploy:
  stage: deploy
  script:
    - helm upgrade --install my-app ./helm/my-app
      -f environments/${ENVIRONMENT}.yaml
      --namespace ${NAMESPACE}
      --set image.tag=${CI_COMMIT_SHORT_SHA}
      --wait
      --timeout 5m
```

```
--install:   install if it doesn't exist, upgrade if it does
--wait:      wait until all pods are ready
--timeout:   fail if not ready within 5 minutes
--set:       override a single value (image tag from CI)
```

---

## Useful Helm Commands

```bash
# List installed releases:
helm list
helm list --all-namespaces

# Get values of a release:
helm get values my-release

# Get all info about a release:
helm get all my-release

# See release history:
helm history my-release

# Package a chart (for sharing):
helm package ./my-app
# Creates: my-app-1.0.0.tgz
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
Charts                     → Packaging applications for Kubernetes
                           → Reusable across teams and environments
Values                     → Environment-specific configuration
                           → dev/staging/prod from one chart
Repositories               → Sharing charts across teams
                           → Internal chart museum / OCI registries
Upgrade/Rollback           → Safe deployments
                           → Quick rollback if something breaks
CI/CD integration          → Automated deployments
                           → helm upgrade in every pipeline
                           → Image tag from Git commit SHA
```

---

> **Next:** 20 — Monitoring & Logging → Prometheus, Grafana, and log aggregation.
