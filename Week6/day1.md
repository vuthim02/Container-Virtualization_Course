# Week 6 • Day 1 • Advanced
# CI/CD Integration

## Progress Checklist
- [x] Day 1 — CI/CD Integration
- [ ] Day 2 — Monitoring
- [ ] Day 3 — Scaling
- [ ] Day 4 — Production Best Practices

---

← [Roadmap](../roadmap.md) | [Week 5 Day 4](../../Week5/day4.md) | [Day 2 →](day2.md)

---

## 1. What is CI/CD?

**CI/CD** stands for **Continuous Integration / Continuous Delivery (or Deployment)**.

```
Code → Build → Test → Package → Deploy → Monitor
 │                                              │
 └──────────────────────────────────────────────┘
              Continuous Feedback
```

| Stage | Description |
|-------|-------------|
| **CI (Continuous Integration)** | Developers merge code frequently, automated builds and tests run |
| **CD (Continuous Delivery)** | Every change is automatically prepared for release |
| **CD (Continuous Deployment)** | Every change is automatically deployed to production |

### CI/CD for Containers

```
Traditional CI/CD:
Code → Build (JAR/WAR) → Test → Deploy to Server

Container CI/CD:
Code → Build Docker Image → Test Container → Push to Registry → Deploy to K8s
```

---

## 2. CI/CD Pipeline for Kubernetes

```
┌─────────────────────────────────────────────────────────────────┐
│                  CI/CD Pipeline                                 │
│                                                                 │
│  1. Developer pushes code to Git                               │
│         │                                                       │
│         ▼                                                       │
│  2. CI Server (GitHub Actions, GitLab CI, Jenkins)              │
│         │                                                       │
│         ├──▶ Build Docker Image                                 │
│         ├──▶ Run Unit Tests                                     │
│         ├──▶ Run Integration Tests                              │
│         ├──▶ Scan Image for Vulnerabilities (Trivy)             │
│         └──▶ Push Image to Registry (Docker Hub, ECR, GCR)      │
│         │                                                       │
│         ▼                                                       │
│  3. CD Tool (ArgoCD, Flux, Helm)                                │
│         │                                                       │
│         ├──▶ Update Kubernetes Manifests                        │
│         ├──▶ Apply to Cluster (kubectl, Helm, Kustomize)        │
│         └──▶ Verify Deployment                                  │
│         │                                                       │
│         ▼                                                       │
│  4. Kubernetes Cluster                                          │
│         │                                                       │
│         ├──▶ Rolling Update                                     │
│         ├──▶ Health Checks                                      │
│         └──▶ Rollback on Failure                                │
│                                                                 │
│  5. Monitoring (Prometheus, Grafana)                            │
│         │                                                       │
│         └──▶ Alert on Issues → Back to Step 1                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Building Docker Images in CI

### GitHub Actions Example

```yaml
# .github/workflows/ci.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Build and test
        run: |
          docker build -t myapp:test .
          docker run myapp:test npm test
      
      - name: Push to Docker Hub
        if: github.ref == 'refs/heads/main'
        run: |
          docker tag myapp:test ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}
          docker push ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}
```

### Multi-stage Dockerfile for CI/CD

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Test stage
FROM node:20-alpine AS tester
WORKDIR /app
COPY --from=builder /app .
RUN npm test

# Production stage
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production
USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Docker Build Best Practices for CI

```bash
# Use BuildKit for faster builds
DOCKER_BUILDKIT=1 docker build -t myapp:latest .

# Use cache from previous builds
docker build --cache-from myapp:latest -t myapp:latest .

# Build for multiple platforms
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myapp:latest \
  --push .

# Scan for vulnerabilities
trivy image myapp:latest

# Check image size
docker images myapp:latest
```

---

## 4. Deployment Strategies

### Blue-Green Deployment

```
┌─────────────────────────────────────────────────┐
│  Blue (v1.0)           Green (v2.0)            │
│  ┌─────┐ ┌─────┐       ┌─────┐ ┌─────┐        │
│  │Pod 1│ │Pod 2│       │Pod 1│ │Pod 2│        │
│  └──┬──┘ └──┬──┘       └──┬──┘ └──┬──┘        │
│     └───────┘              └───────┘            │
│         │                        │              │
│         └────────┬───────────────┘              │
│                  │                              │
│            ┌─────▼─────┐                        │
│            │  Service  │                        │
│            │  (Switch) │                        │
│            └───────────┘                        │
│                                                 │
│  1. Deploy Green alongside Blue                │
│  2. Test Green thoroughly                      │
│  3. Switch service to point to Green           │
│  4. Blue is now idle (can be kept as backup)   │
│  5. Rollback: Switch back to Blue              │
└─────────────────────────────────────────────────┘
```

```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: app
          image: myapp:1.0
---
# Service initially points to blue
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
    version: blue  # <-- Switch this to 'green' to flip
  ports:
    - port: 80
```

```bash
# Deploy green version
kubectl apply -f deployment-green.yaml

# Test green version
kubectl port-forward svc/app-service 8080:80
# ... manual testing ...

# Switch to green (instant)
kubectl patch svc app-service -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback to blue (instant)
kubectl patch svc app-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

### Canary Deployment

```
┌─────────────────────────────────────────────────┐
│  Stable (v1.0)         Canary (v2.0)           │
│  ┌─────┐ ┌─────┐ ┌─────┐    ┌─────┐           │
│  │ 90% │ │ 90% │ │ 90% │    │ 10% │           │
│  └──┬──┘ └──┬──┘ └──┬──┘    └──┬──┘           │
│     └───────┴───────┴─────────┘                │
│                  │                              │
│            ┌─────▼─────┐                        │
│            │ Ingress   │                        │
│            │ (Weighted)│                        │
│            └───────────┘                        │
│                                                 │
│  1. Deploy canary with small % of traffic      │
│  2. Monitor canary metrics                     │
│  3. Gradually increase traffic to canary       │
│  4. If all good, replace stable with canary    │
│  5. If issues, remove canary immediately       │
└─────────────────────────────────────────────────┘
```

```yaml
# Canary deployment with nginx ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% traffic to canary
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-canary
                port:
                  number: 80
```

### Rolling Update (Default in Kubernetes)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%   # At most 25% pods unavailable
      maxSurge: 25%         # At most 25% extra pods
```

---

## 5. GitOps with ArgoCD

**GitOps** is a way of managing Kubernetes clusters using Git as the single source of truth.

```
┌─────────────────────────────────────────────────┐
│  Git Repository (Source of Truth)               │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │  k8s-manifests/                           │ │
│  │  ├── base/                                │ │
│  │  │   ├── deployment.yaml                  │ │
│  │  │   ├── service.yaml                     │ │
│  │  │   └── ingress.yaml                     │ │
│  │  └── overlays/                            │ │
│  │      ├── dev/                             │ │
│  │      ├── staging/                         │ │
│  │      └── production/                      │ │
│  └───────────────────────────────────────────┘ │
│         │                                      │
│         ▼ (ArgoCD watches Git)                 │
│  ┌──────────────────────────────────────────┐ │
│  │  ArgoCD                                  │ │
│  │  - Detects changes in Git               │ │
│  │  - Compares Git state vs cluster state  │ │
│  │  - Syncs cluster to match Git           │ │
│  └───────────────┬─────────────────────────┘ │
│                  │                            │
│                  ▼                            │
│  ┌──────────────────────────────────────────┐ │
│  │  Kubernetes Cluster                      │ │
│  │  State always matches Git                │ │
│  └───────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### Installing ArgoCD

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access the ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Install ArgoCD CLI
brew install argocd

# Login
argocd login localhost:8080
```

### Creating an Application in ArgoCD

```yaml
# app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myuser/k8s-manifests.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Delete resources not in Git
      selfHeal: true     # Re-sync if cluster drifts
    syncOptions:
      - CreateNamespace=true
```

```bash
# Apply the application
kubectl apply -f app.yaml

# Or use the ArgoCD CLI
argocd app create myapp \
  --repo https://github.com/myuser/k8s-manifests.git \
  --path overlays/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated

# Sync manually
argocd app sync myapp

# Check status
argocd app get myapp
```

---

## 6. Helm — The Kubernetes Package Manager

**Helm** is a package manager for Kubernetes. It allows you to define, install, and upgrade complex Kubernetes applications.

### Helm Concepts

```
Chart = A Helm package (template + values)
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── charts/             # Subcharts (dependencies)

Release = A running instance of a chart
```

### Installing Helm

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version

# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo nginx
```

### Creating a Helm Chart

```bash
# Create a new chart
helm create myapp

# Chart structure
myapp/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── _helpers.tpl
    ├── tests/
    │   └── test-connection.yaml
    └── NOTES.txt
```

### Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### Values Example

```yaml
# values.yaml
replicaCount: 3

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25"

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

### Helm Commands

```bash
# Install a chart
helm install my-release ./myapp

# Install with custom values
helm install my-release ./myapp -f custom-values.yaml

# Install from a repository
helm install my-release bitnami/nginx

# Upgrade a release
helm upgrade my-release ./myapp \
  --set image.tag=1.26 \
  --set replicaCount=5

# Rollback
helm rollback my-release 1  # Rollback to revision 1

# View release history
helm history my-release

# List releases
helm list

# Uninstall
helm uninstall my-release

# Dry-run (preview)
helm install my-release ./myapp --dry-run --debug

# Template rendering
helm template my-release ./myapp
```

---

## 7. Kustomize — Configuration Management

**Kustomize** is a configuration management tool built into `kubectl`. It allows you to customize raw YAML files without templates.

### Kustomize Structure

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-count.yaml
    └── production/
        ├── kustomization.yaml
        └── replica-count.yaml
```

### Base Configuration

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  app: myapp
```

```yaml
# base/deployment.yaml
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
          image: myapp:1.0
          ports:
            - containerPort: 8080
```

### Overlay Configuration

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: myapp
    count: 5

images:
  - name: myapp
    newTag: 1.26

patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |
      - op: replace
        path: /spec/template/spec/containers/0/resources
        value:
          limits:
            cpu: "500m"
            memory: "256Mi"
          requests:
            cpu: "250m"
            memory: "128Mi"
```

### Kustomize Commands

```bash
# Build (render) the configuration
kubectl kustomize overlays/production

# Apply directly
kubectl apply -k overlays/production

# Build and save
kubectl kustomize overlays/production > production.yaml
```

### Helm vs Kustomize

| Feature | Helm | Kustomize |
|---------|------|-----------|
| **Templating** | Go templates | No templating |
| **Complexity** | Higher | Lower |
| **Reusability** | Charts (packages) | Base + Overlays |
| **Learning Curve** | Steeper | Flatter |
| **Best For** | Community charts, complex apps | Simple custom apps |

---

## 8. Complete CI/CD Pipeline Example

### GitHub Actions + ArgoCD + Kubernetes

```yaml
# .github/workflows/deploy.yml
name: Build, Push, Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Build and push
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }} .
          docker push ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}
      
      - name: Update image tag in Git
        run: |
          git clone https://x-access-token:${{ secrets.GH_PAT }}@github.com/myuser/k8s-manifests.git
          cd k8s-manifests
          sed -i "s|image:.*|image: ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}|" overlays/production/deployment.yaml
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git commit -am "Update image to ${{ github.sha }}"
          git push

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Wait for ArgoCD sync
        run: |
          echo "ArgoCD will automatically sync the new image tag"
          echo "Check status at: https://argocd.example.com"
```

---

## 📝 Hands-On Exercises

### Exercise 1: Build and Test Docker Image in CI

```bash
# Create a simple Node.js app
mkdir ~/ci-cd-demo && cd ~/ci-cd-demo

cat > package.json << 'EOF'
{
  "name": "ci-cd-demo",
  "version": "1.0.0",
  "scripts": {
    "test": "echo \"All tests passed\" && exit 0",
    "start": "node server.js"
  }
}
EOF

cat > server.js << 'EOF'
const http = require('http');
const server = http.createServer((req, res) => {
  res.writeHead(200);
  res.end('Hello from CI/CD demo!');
});
server.listen(3000, () => console.log('Server running on port 3000'));
EOF

cat > Dockerfile << 'EOF'
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
EOF

# Build and test locally
docker build -t myapp:test .
docker run --rm myapp:test npm test
docker run --rm -d -p 3000:3000 --name test-container myapp:test
curl http://localhost:3000
docker stop test-container
```

### Exercise 2: Helm Deployment

```bash
# Create a Helm chart
helm create myapp

# Customize values
cat > myapp/values.yaml << 'EOF'
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
EOF

# Install the chart
helm install my-release ./myapp

# Check the deployment
kubectl get pods -l app.kubernetes.io/instance=my-release
helm list

# Upgrade the release
helm upgrade my-release ./myapp --set image.tag=1.26

# Rollback
helm rollback my-release 1

# Clean up
helm uninstall my-release
```

### Exercise 3: Kustomize Multi-Environment

```bash
# Create base configuration
mkdir -p ~/kustomize-demo/{base,overlays/{dev,prod}}
cd ~/kustomize-demo

cat > base/deployment.yaml << 'EOF'
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
          image: myapp:1.0
          ports:
            - containerPort: 8080
EOF

cat > base/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
EOF

cat > base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app: myapp
EOF

# Create dev overlay
cat > overlays/dev/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
replicas:
  - name: myapp
    count: 1
images:
  - name: myapp
    newTag: dev
EOF

# Create prod overlay
cat > overlays/prod/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
replicas:
  - name: myapp
    count: 3
images:
  - name: myapp
    newTag: 1.26
EOF

# Build and apply dev
kubectl kustomize overlays/dev
kubectl apply -k overlays/dev

# Build and apply prod
kubectl kustomize overlays/prod
kubectl apply -k overlays/prod

# Clean up
kubectl delete -k overlays/dev
kubectl delete -k overlays/prod
```

---

## 🧠 Common Questions

**Q: What's the difference between Helm and Kustomize?**
A: Helm uses templates (Go templates) to generate Kubernetes manifests. Kustomize modifies existing YAML files without templates. Helm is better for packaging and sharing, Kustomize is simpler for custom configurations.

**Q: How do I rollback a failed deployment?**
A: With Deployments: `kubectl rollout undo deployment/myapp`. With Helm: `helm rollback my-release <revision>`. With ArgoCD: `argocd app rollback myapp <revision>`.

**Q: Should I use GitOps (ArgoCD/Flux) or push-based CD (GitHub Actions)?**
A: GitOps is better for Kubernetes because it provides a single source of truth, automatic drift detection, and self-healing. Push-based CD is simpler for small teams. Use GitOps for production environments.

**Q: How do I manage secrets in CI/CD pipelines?**
A: Never store secrets in Git. Use GitHub Secrets, GitLab Variables, or a secrets manager (Vault, AWS Secrets Manager). For Kubernetes secrets in GitOps, use Sealed Secrets or External Secrets Operator.

**Q: Can I test Helm charts locally?**
A: Yes! `helm template ./myapp` renders the templates locally. `helm install --dry-run --debug ./myapp` validates the chart without deploying. `helm unittest` (plugin) allows writing unit tests for charts.

**Q: What's the best CI/CD tool for Kubernetes?**
A: There's no single best. GitHub Actions is great for simplicity. GitLab CI is excellent for integrated pipelines. Jenkins is highly customizable. ArgoCD/Flux are the best GitOps tools. Choose based on your team's needs.

---

## ✅ Day 1 Summary

**What you learned today:**

- ✅ CI/CD concepts: Continuous Integration, Delivery, Deployment
- ✅ Container CI/CD pipeline: Build → Test → Push → Deploy
- ✅ Building Docker images in CI (GitHub Actions)
- ✅ Deployment strategies: Blue-Green, Canary, Rolling Update
- ✅ GitOps with ArgoCD (Git as source of truth)
- ✅ Helm package manager (charts, releases, upgrades)
- ✅ Kustomize for configuration management (base + overlays)
- ✅ Complete CI/CD pipeline example
- ✅ Rollback strategies for failed deployments

**Coming up tomorrow:** Monitoring — setting up Prometheus and Grafana to monitor your Kubernetes cluster and applications.

---

*💡 Tip: Always automate your deployments. Manual deployments are not reproducible, not auditable, and prone to errors. Use GitOps (ArgoCD/Flux) to keep your cluster state in sync with your Git repository. Your Git history becomes your deployment history.*