# 09 — ConfigMaps & Secrets

## The Problem: Hardcoded Configuration

```
Hardcoding config in your container image is bad:

  FROM nginx:1.25
  ENV DATABASE_HOST=10.0.0.50      ← hardcoded!
  ENV DATABASE_PORT=5432           ← hardcoded!

  To change the database host:
    1. Edit Dockerfile
    2. Rebuild image
    3. Push to registry
    4. Redeploy

  That's a lot of work just to change a config value.
  And now you have different images for dev/staging/prod.
```

```
The Kubernetes way:
  Same image for all environments.
  Configuration is injected at runtime via ConfigMaps and Secrets.

  dev:  image=my-app:1.0 + ConfigMap(DATABASE_HOST=dev-db)
  prod: image=my-app:1.0 + ConfigMap(DATABASE_HOST=prod-db)
  
  Same image. Different config. No rebuild needed.
```

---

## ConfigMaps — Non-Sensitive Configuration

A ConfigMap stores key-value pairs of configuration data.

### Creating ConfigMaps

```bash
# From literal values:
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=postgres \
  --from-literal=DATABASE_PORT=5432 \
  --from-literal=LOG_LEVEL=info

# From a file:
kubectl create configmap nginx-config --from-file=nginx.conf

# From a directory (each file becomes a key):
kubectl create configmap configs --from-file=./config-dir/

# From an env file:
kubectl create configmap app-env --from-env-file=app.env
```

### ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: postgres
  DATABASE_PORT: "5432"          # values are always strings in ConfigMaps
  LOG_LEVEL: info
  APP_MODE: production
```

```yaml
# ConfigMap with a full config file:
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
```

---

## Using ConfigMaps in Pods

### Method 1: Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: my-app:1.0
      env:
        - name: DATABASE_HOST           # env var name in the container
          valueFrom:
            configMapKeyRef:
              name: app-config           # ConfigMap name
              key: DATABASE_HOST         # key in the ConfigMap
        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_PORT
```

### Method 2: All Keys as Environment Variables

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      envFrom:                           # inject ALL keys as env vars
        - configMapRef:
            name: app-config
```

```
This injects:
  DATABASE_HOST=postgres
  DATABASE_PORT=5432
  LOG_LEVEL=info
  APP_MODE=production

All as environment variables inside the container.
```

### Method 3: Mount as Files (Volume)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d    # mount point in container
  volumes:
    - name: config-volume
      configMap:
        name: nginx-config                # ConfigMap name
```

```
This creates a file inside the container:
  /etc/nginx/conf.d/nginx.conf  ← content from the ConfigMap

Each key in the ConfigMap becomes a file.
The key name = file name.
The key value = file content.
```

---

## Secrets — Sensitive Configuration

Secrets are like ConfigMaps but for **sensitive data**: passwords, API keys, TLS certificates, tokens.

```
ConfigMap:  DATABASE_HOST=postgres          (not sensitive)
Secret:     DATABASE_PASSWORD=MyS3cretP@ss  (sensitive!)
```

### Creating Secrets

```bash
# From literal values:
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=MyS3cretP@ss

# From files:
kubectl create secret generic tls-cert \
  --from-file=tls.crt=./cert.pem \
  --from-file=tls.key=./key.pem

# TLS secret (special type):
kubectl create secret tls my-tls-secret \
  --cert=./cert.pem \
  --key=./key.pem

# Docker registry credentials:
kubectl create secret docker-registry my-registry \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```

### Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque                         # generic secret
data:
  username: YWRtaW4=                 # base64 encoded "admin"
  password: TXlTM2NyZXRQQHNz         # base64 encoded "MyS3cretP@ss"
```

```bash
# Values in YAML must be base64 encoded:
echo -n "admin" | base64
# YWRtaW4=

echo -n "MyS3cretP@ss" | base64
# TXlTM2NyZXRQQHNz

# Decode:
echo "YWRtaW4=" | base64 -d
# admin
```

```yaml
# Or use stringData (plain text — K8s encodes for you):
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:                          # plain text (easier)
  username: admin
  password: MyS3cretP@ss
```

---

## Using Secrets in Pods

### As Environment Variables

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

### All Keys as Environment Variables

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      envFrom:
        - secretRef:
            name: db-credentials
```

### Mount as Files

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
```

```
This creates files in the container:
  /etc/secrets/username    ← contains "admin"
  /etc/secrets/password    ← contains "MyS3cretP@ss"
```

---

## Secret Types

```
Type                           Use
──────                         ──────
Opaque                         Generic key-value (default)
kubernetes.io/tls              TLS certificates (tls.crt, tls.key)
kubernetes.io/dockerconfigjson Docker registry credentials
kubernetes.io/basic-auth       Username and password
kubernetes.io/ssh-auth         SSH private key
kubernetes.io/service-account-token  Auto-generated for service accounts
```

---

## ConfigMap vs Secret — When to Use Which

```
ConfigMap:
  ✓ Database hostnames
  ✓ Port numbers
  ✓ Log levels
  ✓ Feature flags
  ✓ Config files (nginx.conf, app.yaml)
  ✓ Non-sensitive data

Secret:
  ✓ Passwords
  ✓ API keys
  ✓ TLS certificates
  ✓ OAuth tokens
  ✓ SSH keys
  ✓ Anything sensitive

Rule: If someone seeing the value would be a security problem → Secret
      Everything else → ConfigMap
```

---

## Important Security Notes About Secrets

```
⚠️ Kubernetes Secrets are NOT strongly encrypted by default!

  1. Secrets are stored in etcd (base64 encoded, not encrypted)
  2. Anyone with access to etcd can read all secrets
  3. Anyone with API access to the namespace can read secrets
  4. Secrets appear in pod specs (kubectl get pod -o yaml)
  5. base64 is NOT encryption (it's just encoding)

To make Secrets actually secure:
  ✓ Enable etcd encryption at rest (EncryptionConfiguration)
  ✓ Use RBAC to restrict who can read Secrets
  ✓ Use external secret managers:
    - HashiCorp Vault
    - AWS Secrets Manager
    - Azure Key Vault
    - External Secrets Operator (syncs from external to K8s)
  ✓ Don't commit Secrets to Git (ever!)
```

---

## Updating ConfigMaps and Secrets

```bash
# Edit a ConfigMap:
kubectl edit configmap app-config

# Replace:
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=new-host \
  --dry-run=client -o yaml | kubectl apply -f -

# What happens to running Pods?

# Environment variables: NOT updated until Pod restarts
#   The values were injected when the Pod started.
#   To get new values, delete the Pod (Deployment recreates it).

# Volume mounts: Updated automatically (within ~1 minute)
#   kubelet periodically checks for updates.
#   Your app needs to watch for file changes to pick them up.
```

---

## Complete Example

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  DATABASE_HOST: postgres-service
  DATABASE_PORT: "5432"
  LOG_LEVEL: info
---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: SuperS3cret!
  API_KEY: abc123def456
---
# deployment.yaml
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
        - name: web
          image: my-app:1.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: web-config           # all ConfigMap keys as env vars
            - secretRef:
                name: web-secrets          # all Secret keys as env vars
```

```
Inside the container, all of these are available as env vars:
  DATABASE_HOST=postgres-service
  DATABASE_PORT=5432
  LOG_LEVEL=info
  DATABASE_PASSWORD=SuperS3cret!
  API_KEY=abc123def456
```

---

## How This Connects to DevOps

```
This File                  Where It Matters
─────────                  ─────────────────
ConfigMaps                 → 12-factor app configuration
                           → Same image, different config per environment
                           → Config files for nginx, haproxy, etc.
Secrets                    → Database credentials
                           → API keys for external services
                           → TLS certificates
Environment injection      → Standard container configuration pattern
                           → Supported by every framework
Volume mounts              → Config files that apps watch for changes
                           → TLS certificate mounts
Secret management          → HashiCorp Vault integration
                           → External Secrets Operator
                           → NEVER put secrets in Git
```

---

> **Next:** 10 — Storage → Persistent data that survives pod restarts.
