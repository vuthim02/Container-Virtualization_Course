# Week 6 • Day 4 • Advanced
# Production Best Practices

## Progress Checklist
- [x] Day 1 — CI/CD Integration
- [x] Day 2 — Monitoring
- [x] Day 3 — Scaling
- [x] Day 4 — Production Best Practices

---

← [Day 3](day3.md) | [Roadmap](../roadmap.md)

---

## 1. Security Best Practices

### Run Containers as Non-root

```dockerfile
# Bad: Running as root (default)
FROM node:20-alpine
WORKDIR /app
COPY . .
CMD ["node", "server.js"]
# Container runs as root → Security risk!

# Good: Running as non-root
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["node", "server.js"]
```

```yaml
# Enforce in Kubernetes
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true       # Prevent running as root
    runAsUser: 1000          # Specific user ID
    fsGroup: 2000            # Group ID for volumes
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        allowPrivilegeEscalation: false  # No sudo
        readOnlyRootFilesystem: true     # Immutable filesystem
        capabilities:
          drop:
            - ALL           # Drop all Linux capabilities
```

### Network Policies

Network policies control traffic flow between pods:

```yaml
# Default deny all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# Allow traffic from specific pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web
      ports:
        - protocol: TCP
          port: 3000
---
# Allow traffic from specific namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9090
```

```
Network Policy:
┌─────────────────────────────────────────────────┐
│  Namespace: default                             │
│                                                 │
│  ┌─────┐         ┌─────┐         ┌─────┐      │
│  │Web  │ ──────▶ │ API │         │ DB  │      │
│  │     │  ✅     │     │         │     │      │
│  └─────┘         └──┬──┘         └─────┘      │
│                     │                          │
│                     │ ❌ Denied                │
│                     ▼                          │
│                   ┌─────┐                     │
│                   │Test │ (cannot reach DB)   │
│                   └─────┘                     │
│                                                 │
│  Network Policy:                                │
│  - Web → API: ✅ Allowed                       │
│  - API → DB: ✅ Allowed                        │
│  - Test → DB: ❌ Denied                        │
└─────────────────────────────────────────────────┘
```

### Pod Security Standards

Kubernetes provides three Pod Security Standards:

| Standard | Description | Use Case |
|----------|-------------|----------|
| **Privileged** | No restrictions | System pods, monitoring |
| **Baseline** | Minimal restrictions | Most workloads |
| **Restricted** | Heavily restricted (best practice) | Security-sensitive apps |

```yaml
# Enforce restricted policy on namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

```bash
# Pods that violate the restricted policy will be rejected:
kubectl run insecure --image=nginx --namespace=production
# Error: pods "insecure" is forbidden: violates PodSecurity "restricted"
```

### Image Security

```yaml
# Always use specific image tags (never 'latest')
containers:
  - name: app
    image: myapp:1.2.3  # ✅ Specific version
    # image: myapp:latest  # ❌ Don't use latest!

# Use image pull policies
imagePullPolicy: IfNotPresent  # Pull only if not cached
imagePullPolicy: Always        # Always pull (ensure latest)
imagePullPolicy: Never         # Never pull (use local image)
```

```bash
# Scan images for vulnerabilities
trivy image myapp:1.2.3
# ┌─────────────────────────────────────┐
# │ myapp:1.2.3 (debian 11.7)          │
# ├─────────────────────────────────────┤
# │ Total: 15 (0 critical, 5 high)     │
# └─────────────────────────────────────┘

# Sign images with Cosign
cosign sign --key env:// myapp:1.2.3

# Verify images
cosign verify --key env:// myapp:1.2.3
```

---

## 2. Resource Management

### Set Resource Requests and Limits

```yaml
containers:
  - name: web
    image: myapp:1.0
    resources:
      requests:
        cpu: "100m"       # Guaranteed CPU
        memory: "128Mi"   # Guaranteed memory
      limits:
        cpu: "500m"       # Max CPU
        memory: "256Mi"   # Max memory (OOM killed if exceeded)
```

**What happens when limits are exceeded:**
- CPU limit exceeded: Container is throttled (slowed down)
- Memory limit exceeded: Container is OOM killed (restarted)

### Resource Quotas

Limit resource consumption per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"        # Total CPU requests: 4 cores
    requests.memory: 8Gi     # Total memory requests: 8 GiB
    limits.cpu: "8"          # Total CPU limits: 8 cores
    limits.memory: 16Gi      # Total memory limits: 16 GiB
    pods: "50"               # Max 50 pods
    services: "20"           # Max 20 services
    persistentvolumeclaims: "10"  # Max 10 PVCs
```

```bash
# Check quota usage
kubectl describe resourcequota production-quota -n production
# Name:       production-quota
# Namespace:  production
# Resource    Used  Hard
# --------    ----  ----
# cpu         2     8
# memory      4Gi   16Gi
# pods        15    50
```

### Limit Ranges

Set default resource requests/limits for all containers in a namespace:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "1"
        memory: "1Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
```

---

## 3. High Availability

### Multi-Replica Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3  # At least 3 for HA
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # At most 1 pod down during update
      maxSurge: 1        # At most 1 extra pod during update
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
          image: myapp:1.0
```

### Pod Anti-Affinity

Spread pods across nodes to avoid single points of failure:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - web-app
                topologyKey: kubernetes.io/hostname
```

```
Pod Anti-Affinity:
┌─────────────────────────────────────────────────┐
│  Node 1          Node 2          Node 3         │
│  ┌─────┐         ┌─────┐         ┌─────┐       │
│  │Pod 1│         │Pod 2│         │Pod 3│       │
│  └─────┘         └─────┘         └─────┘       │
│                                                 │
│  ✅ Pods spread across nodes                   │
│  If Node 1 fails, Pods 2 and 3 still serve     │
│                                                 │
│  Without anti-affinity:                        │
│  Node 1          Node 2          Node 3         │
│  ┌─────┐ ┌─────┐ ┌─────┐                      │
│  │Pod 1│ │Pod 2│ │Pod 3│                      │
│  └─────┘ └─────┘ └─────┘                      │
│  ❌ All pods on one node → Single point of failure│
└─────────────────────────────────────────────────┘
```

### Pod Disruption Budgets (PDB)

Ensure a minimum number of pods are available during voluntary disruptions:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2  # At least 2 pods must be available
  selector:
    matchLabels:
      app: web-app
```

```yaml
# Or use maxUnavailable
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  maxUnavailable: 1  # At most 1 pod can be unavailable
  selector:
    matchLabels:
      app: web-app
```

```bash
# Check PDB status
kubectl get pdb
# NAME     MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# web-pdb  2               N/A               1                     5m

# Test disruptions
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
# PDB ensures at least 2 pods remain available during drain
```

---

## 4. Backup and Disaster Recovery

### etcd Backup

```bash
# Backup etcd (on control plane node)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db

# Restore from backup
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd-restore
```

### Velero for Cluster Backup

[Velero](https://velero.io/) is a popular tool for backing up Kubernetes clusters.

```bash
# Install Velero CLI
brew install velero

# Install Velero server in cluster
velero install \
  --provider aws \
  --bucket my-backup-bucket \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=us-east-1,s3ForcePathStyle=true,s3Url=https://s3.amazonaws.com

# Create a backup
velero backup create my-backup --include-namespaces production

# Check backup status
velero backup describe my-backup
velero backup logs my-backup

# Restore from backup
velero restore create --from-backup my-backup

# Schedule daily backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production
```

### Backup YAML Manifests

Always store your Kubernetes manifests in Git (GitOps):

```
k8s-manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── secret.yaml
├── overlays/
│   ├── dev/
│   ├── staging/
│   └── production/
└── README.md

# If cluster is destroyed, recreate from Git:
kubectl apply -k overlays/production
```

---

## 5. Operational Best Practices

### Use Namespaces for Isolation

```bash
# Separate environments
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Separate teams/projects
kubectl create namespace team-a
kubectl create namespace team-b

# Separate applications
kubectl create namespace monitoring
kubectl create namespace logging
kubectl create namespace ingress
```

### Use Labels Consistently

```yaml
metadata:
  labels:
    app: myapp
    version: "1.2.3"
    environment: production
    team: backend
    managed-by: helm
```

**Recommended labels:**
- `app`: Application name
- `version`: Application version
- `environment`: dev/staging/production
- `team`: Owning team
- `managed-by`: helm/kustomize/argocd

### Rolling Updates Best Practices

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%   # Allow 25% to be down
      maxSurge: 25%         # Allow 25% extra
  minReadySeconds: 10       # Wait 10s before considering pod ready
  progressDeadlineSeconds: 600  # Fail if not ready in 10 min
  revisionHistoryLimit: 5   # Keep 5 old revisions for rollback
```

### Health Checks

```yaml
containers:
  - name: web-app
    image: myapp:1.0
    
    # Liveness: Restart if unhealthy
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 2
      failureThreshold: 3
    
    # Readiness: Remove from service if not ready
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
    
    # Startup: Give app time to start
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

### Graceful Shutdown

```yaml
containers:
  - name: web-app
    image: myapp:1.0
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]  # Wait for LB to update
    terminationGracePeriodSeconds: 30  # Wait 30s before killing
```

```javascript
// Application should handle SIGTERM gracefully
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
  // Force close after 10s
  setTimeout(() => process.exit(1), 10000);
});
```

### Logging Best Practices

```yaml
containers:
  - name: web-app
    image: myapp:1.0
    env:
      - name: LOG_LEVEL
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: LOG_LEVEL
```

```javascript
// Structured logging (JSON)
const winston = require('winston');
const logger = winston.createLogger({
  format: winston.format.json(),
  transports: [new winston.transports.Console()]
});

logger.info('Request processed', {
  method: req.method,
  path: req.path,
  duration: durationMs,
  statusCode: res.statusCode
});
```

```bash
# View logs
kubectl logs pod-name
kubectl logs -l app=myapp --tail=100  # Last 100 lines
kubectl logs -l app=myapp -f           # Follow
```

---

## 6. Production Deployment Checklist

### Before Deployment

- [ ] Images are scanned for vulnerabilities
- [ ] Resource requests and limits are set
- [ ] Liveness, readiness, and startup probes configured
- [ ] Network policies defined
- [ ] ConfigMaps and Secrets created
- [ ] PDB configured for critical services
- [ ] Pod anti-affinity configured for HA
- [ ] Rollback plan documented

### After Deployment

- [ ] Pods are running and ready
- [ ] No crash loops in logs
- [ ] Health checks passing
- [ ] Metrics flowing to Prometheus
- [ ] Logs visible in Grafana/Loki
- [ ] Alert rules tested
- [ ] Traffic reaching new pods

### Ongoing

- [ ] Monitor error rates
- [ ] Monitor resource usage
- [ ] Review and rotate secrets
- [ ] Update images regularly
- [ ] Test disaster recovery
- [ ] Run chaos engineering experiments

---

## 7. Complete Production-Ready Deployment

```yaml
# production-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
    version: "1.2.3"
    environment: production
    team: backend
spec:
  replicas: 3
  revisionHistoryLimit: 5
  minReadySeconds: 10
  progressDeadlineSeconds: 600
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        version: "1.2.3"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - web-app
                topologyKey: kubernetes.io/hostname
      terminationGracePeriodSeconds: 30
      containers:
        - name: web-app
          image: myregistry/web-app:1.2.3
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: metrics
              containerPort: 8080
              protocol: TCP
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /healthz
              port: http
            failureThreshold: 30
            periodSeconds: 10
          env:
            - name: NODE_ENV
              value: production
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: web-app-config
                  key: LOG_LEVEL
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: web-app-secret
                  key: DB_PASSWORD
          volumeMounts:
            - name: tmp
              mountPath: /tmp
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]
      volumes:
        - name: tmp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## 📝 Hands-On Exercises

### Exercise 1: Create a Secure Deployment

```bash
# Create namespace
kubectl create namespace secure-demo

# Apply pod security standard
kubectl label namespace secure-demo \
  pod-security.kubernetes.io/enforce=restricted

# Try to run a pod that violates the policy
kubectl run insecure --image=nginx -n secure-demo
# Error: violates PodSecurity "restricted"

# Run a compliant pod
cat > secure-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: secure-demo
  labels:
    app: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:alpine
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        readOnlyRootFilesystem: true
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 100m
          memory: 128Mi
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
    - name: run
      emptyDir: {}
EOF

kubectl apply -f secure-pod.yaml
kubectl get pods -n secure-demo
```

### Exercise 2: Network Policies

```bash
# Create namespaces
kubectl create namespace frontend
kubectl create namespace backend

# Deploy services
kubectl run web --image=nginx -n frontend
kubectl run api --image=nginx -n backend

# Default deny all traffic
cat > default-deny.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF

kubectl apply -f default-deny.yaml

# Allow only frontend → backend
cat > allow-frontend.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      run: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
      ports:
        - protocol: TCP
          port: 80
EOF

kubectl apply -f allow-frontend.yaml

# Clean up
kubectl delete namespace frontend backend
```

### Exercise 3: PDB and Anti-Affinity

```bash
# Create deployment with anti-affinity
cat > ha-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ha-app
  template:
    metadata:
      labels:
        app: ha-app
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - ha-app
                topologyKey: kubernetes.io/hostname
      containers:
        - name: app
          image: nginx:alpine
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF

kubectl apply -f ha-deployment.yaml

# Create PDB
cat > pdb.yaml << 'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ha-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: ha-app
EOF

kubectl apply -f pdb.yaml

# Check PDB
kubectl get pdb
kubectl get pods -o wide  # Verify pods spread across nodes

# Clean up
kubectl delete -f ha-deployment.yaml
kubectl delete -f pdb.yaml
```

### Exercise 4: Simple Chaos Engineering (Manual)

Test your cluster's resilience by simulating a failure:

```bash
# 1. Start a high-availability deployment (from Exercise 3)
kubectl apply -f ha-deployment.yaml
kubectl apply -f pdb.yaml

# 2. Monitor the pods in one terminal
watch kubectl get pods -o wide

# 3. In another terminal, simulate a node failure by draining it
# Replace 'node-name' with one of your node names from 'kubectl get nodes'
NODE_TO_FAIL=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl drain $NODE_TO_FAIL --ignore-daemonsets --delete-emptydir-data

# 4. Observe the results:
# - Did the PDB prevent the drain if it would violate availability?
# - How quickly did Kubernetes reschedule the pods to other nodes?
# - Was there any downtime? (Try running a 'curl' loop in the background)

# 5. Restore the node
kubectl uncordon $NODE_TO_FAIL
```

---

## 🧠 Common Questions

**Q: Should I run containers as non-root?**
A: Yes, always. Running as root means a compromised container has root access to the host. Use `runAsNonRoot: true` and `runAsUser: 1000` in the security context.

**Q: How do I choose resource requests?**
A: Monitor actual usage with `kubectl top pods` for at least a week. Set requests to your average usage, limits to 2-5x requests. Don't over-provision (wastes resources) or under-provision (causes throttling/OOM).

**Q: What's the minimum number of replicas for HA?**
A: At least 3 replicas spread across different nodes. With 3 replicas and a PDB of `minAvailable: 2`, you can lose 1 node without downtime.

**Q: How often should I backup etcd?**
A: Daily at minimum. Before any major operation (upgrades, migrations), take a manual backup. Store backups off-cluster (S3, GCS). Test restoration periodically.

**Q: Should I use PodDisruptionBudgets for all deployments?**
A: Use PDBs for critical services that need high availability. For non-critical services (batch jobs, dev tools), PDBs are optional.

**Q: How do I test my disaster recovery plan?**
A: Regularly practice restoring from backups in a staging environment. Use chaos engineering tools (Litmus, Chaos Mesh) to simulate node failures and verify PDBs work.

---

## ✅ Day 4 Summary

**What you learned today:**

- ✅ Security: Non-root containers, network policies, pod security standards
- ✅ Image security: Specific tags, scanning, signing
- ✅ Resource management: Requests/limits, quotas, limit ranges
- ✅ High availability: Multi-replica deployments, pod anti-affinity, PDBs
- ✅ Backup and disaster recovery: etcd backup, Velero, GitOps
- ✅ Operational best practices: Namespaces, labels, rolling updates
- ✅ Health checks: Liveness, readiness, startup probes
- ✅ Graceful shutdown and structured logging
- ✅ Production deployment checklist
- ✅ Complete production-ready deployment example

---

## 🎉 Course Complete!

**Congratulations!** You've completed the Container Completely Course!

### What You've Learned

**Week 1: Container Fundamentals**
- ✅ What is containerization
- ✅ Docker architecture
- ✅ Images and containers
- ✅ Basic commands

**Week 2: Docker Deep Dive**
- ✅ Dockerfile basics
- ✅ Building images
- ✅ Volume management
- ✅ Optimization

**Week 3: Docker Compose**
- ✅ Introduction to Compose
- ✅ Multi-container apps
- ✅ Networking in Compose
- ✅ Advanced patterns

**Week 4: Container Networking**
- ✅ Network basics
- ✅ Service discovery
- ✅ Load balancing
- ✅ Security

**Week 5: Kubernetes Essentials**
- ✅ K8s architecture
- ✅ Pods and deployments
- ✅ Services and ingress
- ✅ Config and secrets

**Week 6: Production Deployment**
- ✅ CI/CD integration
- ✅ Monitoring
- ✅ Scaling
- ✅ Production best practices

### Recommended Next Steps

1. **Practice:** Build a real project using everything you've learned
2. **Certifications:** Consider CKA (Certified Kubernetes Administrator) or CKAD
3. **Advanced Topics:** Service mesh (Istio), GitOps (ArgoCD), serverless (Knative)
4. **Community:** Join Kubernetes Slack, attend local meetups, contribute to open source

---

*💡 Tip: The best way to master Kubernetes is to use it daily. Start small, iterate often, and always automate your deployments. Your future self will thank you when things break at 3 AM.*