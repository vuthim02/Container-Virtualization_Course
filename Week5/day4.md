# Week 5 • Day 4 • Advanced
# Config & Secrets

## Progress Checklist
- [x] Day 1 — K8s Architecture
- [x] Day 2 — Pods & Deployments
- [x] Day 3 — Services & Ingress
- [x] Day 4 — Config & Secrets

---

← [Day 3](day3.md) | [Roadmap](../roadmap.md) | [Week 6 Day 1 →](../../Week6/day1.md)

---

## 1. The Problem: Configuration in Containers

Applications need configuration data:
- Database connection strings
- API endpoints
- Feature flags
- Environment-specific settings
- Passwords and certificates

**Without ConfigMaps/Secrets:**
```yaml
# Bad: Configuration baked into the image
FROM node:20
ENV DATABASE_URL=postgres://user:pass@db:5432/mydb
ENV API_KEY=abc123secret
COPY . .
```
```
❌ Need to rebuild image for every config change
❌ Secrets stored in image layers (visible in docker history)
❌ Same image can't be used across environments
❌ No way to update config without restarting pods
```

**With ConfigMaps and Secrets:**
```yaml
# Good: Configuration injected at runtime
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: postgres://user:pass@db:5432/mydb
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  API_KEY: YWJjMTIzc2VjcmV0  # base64 encoded
```
```
✅ Same image across all environments
✅ Config changed without rebuilding
✅ Secrets not in image layers
✅ Environment-specific configurations
```

---

## 2. ConfigMaps — Non-sensitive Configuration

A **ConfigMap** is an API object used to store non-sensitive configuration data in key-value pairs.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Simple key-value pairs
  APP_ENV: production
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"
  
  # Database configuration
  DATABASE_HOST: postgres.default.svc.cluster.local
  DATABASE_PORT: "5432"
  DATABASE_NAME: myapp
  
  # Feature flags
  ENABLE_CACHE: "true"
  CACHE_TTL: "3600"
```

### Creating ConfigMaps

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# From a file
kubectl create configmap app-config \
  --from-file=config.properties

# From a .env file
kubectl create configmap app-config \
  --from-file=.env

# From a directory
kubectl create configmap app-config \
  --from-file=configs/

# From a YAML file
kubectl create configmap app-config --from-file=app-config.yaml
kubectl apply -f app-config.yaml
```

### Using ConfigMaps in Pods

#### Method 1: Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        # Single key
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

```bash
# Inside the container:
echo $APP_ENV      # production
echo $LOG_LEVEL    # info
```

#### Method 2: All Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      envFrom:
        - configMapRef:
            name: app-config
```

```bash
# All keys become environment variables:
echo $APP_ENV           # production
echo $LOG_LEVEL         # info
echo $MAX_CONNECTIONS   # 100
echo $DATABASE_HOST     # postgres.default.svc.cluster.local
```

#### Method 3: Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

```bash
# Inside the container:
ls /etc/config/
# APP_ENV  LOG_LEVEL  MAX_CONNECTIONS  DATABASE_HOST  DATABASE_PORT

cat /etc/config/APP_ENV
# production

cat /etc/config/DATABASE_HOST
# postgres.default.svc.cluster.local
```

#### Method 4: Mount Specific Keys as Files

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;
      
      location / {
        proxy_pass http://backend:3000;
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
  volumes:
    - name: nginx-config-volume
      configMap:
        name: nginx-config
```

### ConfigMap Configuration File Example

```yaml
# application.properties
database.host=postgres.default.svc.cluster.local
database.port=5432
database.name=myapp
cache.enabled=true
cache.ttl=3600

# Create ConfigMap
kubectl create configmap app-properties \
  --from-file=application.properties=application.properties

# Mount as a single file
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: properties-volume
          mountPath: /etc/config/application.properties
          subPath: application.properties
  volumes:
    - name: properties-volume
      configMap:
        name: app-properties
```

---

## 3. Secrets — Sensitive Data

A **Secret** is an object that contains sensitive data such as passwords, tokens, or keys.

**Key differences from ConfigMaps:**
- Secrets are stored in base64 encoding
- Secrets are not written to disk by default (tmpfs)
- Secrets are not shown in `kubectl describe` output
- Secrets can be encrypted at rest (cluster configuration)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # Values MUST be base64 encoded
  DB_PASSWORD: cGFzc3dvcmQxMjM=    # password123
  API_KEY: c2VjcmV0LWFwaS1rZXk=  # secret-api-key
  TLS_CERT: LS0tLS1CRUdJTi...     # certificate data
```

### Secret Types

| Type | Use Case |
|------|----------|
| **Opaque** | Generic secrets (default) |
| **kubernetes.io/tls** | TLS certificates |
| **kubernetes.io/dockerconfigjson** | Docker registry credentials |
| **kubernetes.io/basic-auth** | Basic authentication |
| **kubernetes.io/ssh-auth** | SSH credentials |
| **kubernetes.io/service-account-token** | Service account tokens |

### Creating Secrets

```bash
# From literal values (automatically base64 encoded)
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=password123 \
  --from-literal=API_KEY=secret-api-key

# From files
kubectl create secret generic app-secret \
  --from-file=DB_PASSWORD=./db-password.txt \
  --from-file=API_KEY=./api-key.txt

# From a .env file
kubectl create secret generic app-secret \
  --from-file=.env

# From a YAML file
kubectl apply -f secret.yaml
```

### Base64 Encoding/Decoding

```bash
# Encode to base64
echo -n "password123" | base64
# cGFzc3dvcmQxMjM=

# Decode from base64
echo "cGFzc3dvcmQxMjM=" | base64 -d
# password123
```

### Using Secrets in Pods

#### Method 1: Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: API_KEY
```

#### Method 2: All Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      envFrom:
        - secretRef:
            name: app-secret
```

#### Method 3: Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: app-secret
        defaultMode: 0400  # Only owner can read
```

```bash
# Inside the container:
ls -la /etc/secrets/
# -r-------- 1 root root 12 Apr 15 10:00 DB_PASSWORD
# -r-------- 1 root root 15 Apr 15 10:00 API_KEY

cat /etc/secrets/DB_PASSWORD
# password123
```

### TLS Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...  # base64 encoded certificate
  tls.key: LS0tLS1CRUdJTi...  # base64 encoded private key
```

```bash
# Create from files
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Use in Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
    - hosts:
        - example.com
      secretName: tls-secret
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

### Image Pull Secrets

```bash
# Create docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com

# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  containers:
    - name: app
      image: myregistry/myimage:1.0
  imagePullSecrets:
    - name: regcred
```

---

## 4. Environment-Specific Configuration

A common pattern is to use **ConfigMaps for non-sensitive config** and **Secrets for sensitive data**, with different values per environment.

```
Development                    Production
┌─────────────────────┐       ┌─────────────────────┐
│ ConfigMap:          │       │ ConfigMap:          │
│   APP_ENV=dev       │       │   APP_ENV=production│
│   LOG_LEVEL=debug   │       │   LOG_LEVEL=info    │
│   DB_HOST=localhost │       │   DB_HOST=db.prod   │
│                     │       │                     │
│ Secret:             │       │ Secret:             │
│   DB_PASSWORD=dev   │       │   DB_PASSWORD=****  │
│   API_KEY=test      │       │   API_KEY=****      │
└─────────────────────┘       └─────────────────────┘
         │                              │
         └──────────┬───────────────────┘
                    │
              Same Deployment
              ┌─────────────────┐
              │ replicas: 3     │
              │ image: myapp:1.0│
              └─────────────────┘
```

### Using Namespaces for Environment Separation

```bash
# Create namespaces
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Create environment-specific ConfigMaps
kubectl create configmap app-config \
  --from-literal=APP_ENV=development \
  --from-literal=DB_HOST=db.dev \
  -n development

kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=DB_HOST=db.prod \
  -n production

# Create environment-specific Secrets
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=devpass123 \
  -n development

kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=supersecure456 \
  -n production

# Deploy the same application to all environments
kubectl apply -f deployment.yaml -n development
kubectl apply -f deployment.yaml -n staging
kubectl apply -f deployment.yaml -n production
```

### Deployment Using ConfigMaps and Secrets

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          ports:
            - containerPort: 8080
          
          # Environment variables from ConfigMap
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_HOST
          
          # Environment variables from Secret
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
          
          # All config as env variables
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: db-secret
          
          # Config files via volume mount
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
            - name: secret-volume
              mountPath: /etc/secrets
              readOnly: true
      
      # Volumes
      volumes:
        - name: config-volume
          configMap:
            name: app-config
        - name: secret-volume
          secret:
            secretName: db-secret
            defaultMode: 0400
```

---

## 5. Updating ConfigMaps and Secrets

### ConfigMap Updates

When a ConfigMap is updated, pods using it as environment variables **will NOT see the changes** until they are restarted. Pods using it as volume mounts **will see the changes automatically** (after a short delay).

```bash
# Update a ConfigMap
kubectl edit configmap app-config
# OR
kubectl create configmap app-config \
  --from-literal=APP_ENV=staging \
  --from-literal=LOG_LEVEL=warn \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart pods to pick up env var changes
kubectl rollout restart deployment/myapp

# Or delete and recreate pods
kubectl delete pod -l app=myapp
```

### Volume Mount Auto-Update

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

```bash
# Update the ConfigMap
kubectl edit configmap app-config
# Change LOG_LEVEL from info to debug

# The file on the pod will update automatically (within ~1 minute)
kubectl exec config-pod -- cat /etc/config/LOG_LEVEL
# debug  ← Updated automatically!
```

### Secret Updates

Secrets behave the same way as ConfigMaps when used as volume mounts:

```bash
# Update a Secret
kubectl edit secret app-secret
# OR
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=newpassword123 \
  --dry-run=client -o yaml | kubectl apply -f -

# Volume-mounted secrets update automatically
kubectl exec secret-pod -- cat /etc/secrets/DB_PASSWORD
# newpassword123  ← Updated automatically!

# Environment variable references need pod restart
kubectl rollout restart deployment/myapp
```

---

## 6. ConfigMap and Secret Quotas

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Max Size** | 1 MB | 1 MB |
| **Storage** | etcd | etcd (tmpfs in pods) |
| **Visibility** | Visible in kubectl describe | Values hidden in describe |
| **Encryption** | At rest (if enabled) | At rest (if enabled) |
| **Encoding** | Plain text | Base64 encoded |
| **Use Case** | Non-sensitive config | Sensitive data |

---

## 7. External Configuration Management

For production environments, consider external configuration management tools:

### HashiCorp Vault

```
┌─────────────────────────────────────────────────┐
│              Vault Server                       │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │  Secrets:                                │  │
│  │  - Database credentials                  │  │
│  │  - API keys                              │  │
│  │  - TLS certificates                      │  │
│  │  - Encryption keys                       │  │
│  └──────────────────────────────────────────┘  │
│                                                 │
│  Features:                                     │
│  - Dynamic secrets (auto-generate credentials) │
│  - Automatic rotation                          │
│  - Audit logging                               │
│  - Access control policies                     │
│  - Encryption as a Service                     │
└─────────────────────────────────────────────────┘
         ▲
         │
    ┌────┴────┐
    │ K8s Pod │ (uses Vault Agent or CSI Driver)
    └─────────┘
```

```yaml
# Using Vault Agent Injector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-app
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/mydb"
    vault.hashicorp.com/agent-inject-template-db-creds: |
      {{- with secret "database/creds/mydb" -}}
      DB_USERNAME={{ .Data.username }}
      DB_PASSWORD={{ .Data.password }}
      {{- end -}}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault-app
  template:
    metadata:
      labels:
        app: vault-app
    spec:
      containers:
        - name: app
          image: myapp:1.0
          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: vault-app-db-creds
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: vault-app-db-creds
                  key: password
```

### Sealed Secrets (for GitOps)

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Install kubeseal CLI
brew install kubeseal

# Create a regular secret
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=password123 \
  --dry-run=client -o yaml > secret.yaml

# Encrypt it to a SealedSecret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# The SealedSecret can be safely stored in Git
cat sealed-secret.yaml
# apiVersion: bitnami.com/v1alpha1
# kind: SealedSecret
# metadata:
#   name: db-secret
# spec:
#   encryptedData:
#     DB_PASSWORD: AgBx...  ← Encrypted!

# Deploy to cluster (controller decrypts it)
kubectl apply -f sealed-secret.yaml

# The regular secret is created automatically
kubectl get secret db-secret
```

---

## 📝 Hands-On Exercises

### Exercise 1: Working with ConfigMaps

```bash
# Create a ConfigMap from literal values
kubectl create configmap my-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# View ConfigMap
kubectl get configmaps
kubectl describe configmap my-config
kubectl get configmap my-config -o yaml

# Create a pod using ConfigMap as env vars
cat > configmap-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "while true; do echo APP_ENV=$APP_ENV LOG_LEVEL=$LOG_LEVEL; sleep 10; done"]
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: APP_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: LOG_LEVEL
EOF

kubectl apply -f configmap-pod.yaml

# Check the pod's environment
kubectl logs configmap-pod
# APP_ENV=production LOG_LEVEL=info

# Create a pod using ConfigMap as volume
cat > configmap-volume-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "while true; do cat /etc/config/APP_ENV; sleep 10; done"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: my-config
EOF

kubectl apply -f configmap-volume-pod.yaml

# Clean up
kubectl delete pod configmap-pod configmap-volume-pod
kubectl delete configmap my-config
```

### Exercise 2: Working with Secrets

```bash
# Create a Secret from literal values
kubectl create secret generic my-secret \
  --from-literal=DB_PASSWORD=supersecretpassword \
  --from-literal=API_KEY=my-api-key-123

# View Secret (values are hidden)
kubectl get secrets
kubectl describe secret my-secret
kubectl get secret my-secret -o yaml

# Decode a secret value
kubectl get secret my-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# supersecretpassword

# Create a pod using Secret as env vars
cat > secret-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "while true; do echo Password is set; sleep 10; done"]
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: API_KEY
EOF

kubectl apply -f secret-pod.yaml

# Verify env vars
kubectl exec secret-pod -- printenv | grep DB_PASSWORD
# DB_PASSWORD=supersecretpassword

# Clean up
kubectl delete pod secret-pod
kubectl delete secret my-secret
```

### Exercise 3: Complete Application with ConfigMap and Secret

```bash
# Create ConfigMaps and Secrets for a web application
kubectl create configmap web-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=REDIS_HOST=redis.default.svc.cluster.local

kubectl create secret generic web-secret \
  --from-literal=SESSION_SECRET=my-session-secret-123

# Deploy the application
cat > web-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web-app
          image: nginx:alpine
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: web-config
          env:
            - name: SESSION_SECRET
              valueFrom:
                secretKeyRef:
                  name: web-secret
                  key: SESSION_SECRET
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
            - name: secret-volume
              mountPath: /etc/secrets
              readOnly: true
      volumes:
        - name: config-volume
          configMap:
            name: web-config
        - name: secret-volume
          secret:
            secretName: web-secret
            defaultMode: 0400
EOF

kubectl apply -f web-deployment.yaml

# Expose the application
kubectl expose deployment web-app --port=80 --type=NodePort

# Check the deployment
kubectl get pods -l app=web-app
kubectl logs -l app=web-app

# Verify config inside the pod
POD_NAME=$(kubectl get pods -l app=web-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- cat /etc/config/APP_ENV
kubectl exec $POD_NAME -- cat /etc/secrets/SESSION_SECRET

# Clean up
kubectl delete service web-app
kubectl delete deployment web-app
kubectl delete configmap web-config
kubectl delete secret web-secret
```

### Exercise 4: Environment Separation with Namespaces

```bash
# Create namespaces
kubectl create namespace dev
kubectl create namespace prod

# Create dev config
kubectl create configmap app-config \
  --from-literal=APP_ENV=development \
  --from-literal=DB_HOST=db.dev \
  -n dev

kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=devpass \
  -n dev

# Create prod config
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=DB_HOST=db.prod \
  -n prod

kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=supersecure456 \
  -n prod

# Deploy the same app to both environments
cat > app-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: busybox
          command: ["sh", "-c", "while true; do echo APP=$APP_ENV DB=$DB_HOST; sleep 10; done"]
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_HOST
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
EOF

kubectl apply -f app-deployment.yaml -n dev
kubectl apply -f app-deployment.yaml -n prod

# Compare environments
kubectl logs -l app=myapp -n dev
# APP=development DB=db.dev

kubectl logs -l app=myapp -n prod
# APP=production DB=db.prod

# Clean up
kubectl delete namespace dev prod
```

---

## 🧠 Common Questions

**Q: Are Secrets really secure in Kubernetes?**
A: Secrets are base64 encoded (not encrypted!) by default. Anyone with access to etcd or the API server can read them. For real security: enable encryption at rest, use RBAC to limit access, or use an external secrets manager (Vault, AWS Secrets Manager, etc.).

**Q: What happens when a ConfigMap is deleted?**
A: Pods using the ConfigMap as env vars will continue running (they got the values at startup). Pods using it as volumes will see the volume become empty. New pods will fail to start.

**Q: Can I use a Secret and ConfigMap together?**
A: Yes! Use `envFrom` with both:

```yaml
envFrom:
  - configMapRef:
      name: my-config
  - secretRef:
      name: my-secret
```

**Q: What's the maximum size of a ConfigMap or Secret?**
A: 1 MB. For larger configurations, consider splitting into multiple ConfigMaps or using a different approach (e.g., init containers to pull config from external storage).

**Q: How often do volume-mounted ConfigMaps/Secrets update?**
A: kubelet syncs the changes periodically (default: every minute). There's a small delay between updating the ConfigMap/Secret and seeing the changes in the pod.

**Q: Should I store configurations in ConfigMaps or environment variables?**
A: Use ConfigMaps! They provide a centralized way to manage configuration, can be updated without rebuilding images, and are easily auditable. Environment variables in the Dockerfile are only for defaults.

**Q: How do I rotate secrets without downtime?**
A: Create a new secret with updated credentials, then update the pod spec to reference the new secret name. The pods will restart automatically (if using Deployments) and pick up the new credentials.

---

## ✅ Day 4 Summary

**What you learned today:**

- ✅ Why ConfigMaps and Secrets are needed for container configuration
- ✅ Creating ConfigMaps from literals, files, and directories
- ✅ Using ConfigMaps as environment variables and volume mounts
- ✅ Creating Secrets (base64 encoding, secret types)
- ✅ Using Secrets as env vars, volume mounts, and image pull secrets
- ✅ TLS secrets for HTTPS certificates
- ✅ Environment-specific configuration with namespaces
- ✅ ConfigMap and Secret update behavior (env vars vs volumes)
- ✅ External configuration management (Vault, Sealed Secrets)
- ✅ Security best practices for sensitive data

**Week 5 Complete!** You now understand:
- ✅ Kubernetes architecture and components
- ✅ Pods, ReplicaSets, Deployments, StatefulSets, and DaemonSets
- ✅ Services, Ingress, and service discovery
- ✅ ConfigMaps and Secrets for configuration management

**Coming up next week:** Production deployment — CI/CD integration, monitoring, scaling, and production best practices.

---

*💡 Tip: Never store secrets in Git repositories, even in private repos. Use Sealed Secrets, Vault, or your cloud provider's secret manager. For ConfigMaps, keep them small (< 1MB) and use them for environment-specific configuration that changes between deployments.*