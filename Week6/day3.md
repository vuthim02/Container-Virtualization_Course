# Week 6 • Day 3 • Advanced
# Scaling

## Progress Checklist
- [x] Day 1 — CI/CD Integration
- [x] Day 2 — Monitoring
- [x] Day 3 — Scaling
- [ ] Day 4 — Production Best Practices

---

← [Day 2](day2.md) | [Roadmap](../roadmap.md) | [Day 4 →](day4.md)

---

## 1. Why Scaling Matters

```
Without Auto-scaling:
┌─────────────────────────────────────────────────┐
│  Normal traffic: 3 pods sufficient              │
│  ┌─────┐  ┌─────┐  ┌─────┐                    │
│  │Pod 1│  │Pod 2│  │Pod 3│                    │
│  └─────┘  └─────┘  └─────┘                    │
│                                                 │
│  Traffic spike: 3 pods overwhelmed              │
│  ┌─────┐  ┌─────┐  ┌─────┐                    │
│  │ 95% │  │ 98% │  │ 92% │ ← CPU at 95%+     │
│  └─────┘  └─────┘  └─────┘                    │
│  Users experience slow responses/errors        │
│                                                 │
│  ❌ Manual scaling takes time                  │
│  ❌ Over-provisioning wastes resources         │
│  ❌ Under-provisioning hurts users             │
└─────────────────────────────────────────────────┘

With Auto-scaling:
┌─────────────────────────────────────────────────┐
│  Normal traffic: 3 pods                         │
│  ┌─────┐  ┌─────┐  ┌─────┐                    │
│  │ 30% │  │ 25% │  │ 35% │                     │
│  └─────┘  └─────┘  └─────┘                    │
│                                                 │
│  Traffic spike detected                         │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐ │
│  │ 70% │  │ 65% │  │ 75% │  │NEW  │  │NEW  │ │
│  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘ │
│  HPA creates new pods automatically            │
│                                                 │
│  Traffic subsides                               │
│  ┌─────┐  ┌─────┐  ┌─────┐                    │
│  │ 30% │  │ 25% │  │ 35% │ ← Scaled back down │
│  └─────┘  └─────┘  └─────┘                    │
│                                                 │
│  ✅ Automatic scaling                          │
│  ✅ Right-sized resources                      │
│  ✅ Cost efficient                             │
└─────────────────────────────────────────────────┘
```

---

## 2. Types of Scaling in Kubernetes

| Type | What it Scales | Tool |
|------|---------------|------|
| **Horizontal Pod Autoscaling (HPA)** | Number of pod replicas | metrics-server |
| **Vertical Pod Autoscaling (VPA)** | CPU/memory requests per pod | VPA controller |
| **Cluster Autoscaling** | Number of nodes in the cluster | Cluster Autoscaler |
| **KEDA (Event-Driven)** | Pods based on external events | KEDA |

---

## 3. Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of pod replicas based on metrics.

### How HPA Works

```
┌─────────────────────────────────────────────────┐
│              HPA Controller                       │
│                                                 │
│  1. Check current metrics (every 15s)           │
│     - CPU usage: 80%                            │
│     - Memory usage: 70%                         │
│                                                 │
│  2. Compare with target                         │
│     - Target CPU: 50%                           │
│     - Current: 80% > Target: 50%               │
│                                                 │
│  3. Calculate desired replicas                  │
│     desired = current * (current / target)      │
│     desired = 3 * (80 / 50) = 4.8 → 5          │
│                                                 │
│  4. Update Deployment replicas                  │
│     3 → 5 replicas                              │
│                                                 │
│  5. Wait for cooldown (5 min) before next       │
│     scale-down                                  │
└─────────────────────────────────────────────────┘
```

### Prerequisites: Metrics Server

HPA requires the metrics-server to be installed:

```bash
# Check if metrics-server is already running
kubectl get deployment metrics-server -n kube-system

# If not, install it
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For minikube, enable the addon
minikube addons enable metrics-server

# Verify metrics are working
kubectl top nodes
kubectl top pods
```

### Basic HPA (CPU-based)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50  # Scale if CPU > 50%
```

```bash
# Create HPA imperatively
kubectl autoscale deployment web-deployment \
  --cpu-percent=50 \
  --min=2 \
  --max=10

# Check HPA status
kubectl get hpa
# NAME        REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# web-hpa     Deployment/web-deployment     80%/50%   2         10        5          2m

# Detailed HPA status
kubectl describe hpa web-hpa
# Events:
#   Type    Reason             Age   From                       Message
#   ----    ------             ----  ----                       -------
#   Normal  SuccessfulRescale  2m    horizontal-pod-autoscaler  New size: 5; reason: cpu resource utilization (percentage of request) above target
```

### HPA with Multiple Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    # CPU metric
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    
    # Memory metric
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
```

**HPA picks the highest calculated replica count from all metrics.**

### HPA with Custom Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    # Custom metric: HTTP requests per second
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100  # Scale if > 100 req/s per pod
```

**Custom metrics require Prometheus Adapter or a similar solution:**

```bash
# Install Prometheus Adapter
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --set prometheus.url=http://prometheus-server.monitoring.svc.cluster.local \
  --set prometheus.port=80
```

### HPA Calculation Formula

```
desiredReplicas = currentReplicas × (currentMetricValue / targetMetricValue)

Example:
- Current replicas: 3
- Current CPU: 80%
- Target CPU: 50%

desiredReplicas = 3 × (80 / 50) = 3 × 1.6 = 4.8 → ceil to 5

Scale up: 3 → 5 pods
```

### HPA Behavior

```
Scaling Up:
- Triggered when metrics exceed target
- Happens quickly (within 15-30 seconds)
- Limited by maxReplicas

Scaling Down:
- Triggered when metrics fall below target
- Has a cooldown period (5 minutes default)
- Prevents "flapping" (rapid scale up/down)
- Limited by minReplicas
```

---

## 4. Vertical Pod Autoscaler (VPA)

VPA automatically adjusts CPU and memory **requests and limits** for pods.

### How VPA Works

```
┌─────────────────────────────────────────────────┐
│              VPA Controller                       │
│                                                 │
│  1. Observe actual resource usage              │
│     Pod A: CPU request=100m, actual=20m        │
│     Pod B: CPU request=100m, actual=150m       │
│                                                 │
│  2. Recommend optimal values                   │
│     Pod A: CPU request=30m (reduce waste)      │
│     Pod B: CPU request=200m (avoid throttling) │
│                                                 │
│  3. Apply recommendations                      │
│     - Mode: Auto (automatic update)            │
│     - Mode: Initial (only on new pods)         │
│     - Mode: Off (recommendations only)         │
└─────────────────────────────────────────────────┘
```

### Installing VPA

```bash
# Clone the autoscaler repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/

# Install VPA
./hack/vpa-up.sh

# Check VPA components
kubectl get pods -n kube-system | grep vpa
# vpa-admission-controller-abc123
# vpa-recommender-def456
# vpa-updater-ghi789
```

### VPA Configuration

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  updatePolicy:
    updateMode: "Auto"  # Auto, Initial, or Off
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "1"
          memory: 1Gi
```

```bash
# Check VPA recommendations
kubectl get vpa web-vpa -o yaml
# spec:
#   recommendation:
#     containerRecommendations:
#       - containerName: web
#         lowerBound:
#           cpu: 50m
#           memory: 128Mi
#         target:
#           cpu: 100m
#           memory: 256Mi
#         upperBound:
#           cpu: 200m
#           memory: 512Mi
```

**Important:** VPA and HPA can conflict if used on the same resource. Use VPA for resource requests/limits, and HPA for replica count. Don't use VPA's Auto mode with HPA on CPU/memory metrics.

---

## 5. Cluster Autoscaler

Cluster Autoscaler automatically adds or removes nodes based on pod scheduling failures.

### How Cluster Autoscaler Works

```
┌─────────────────────────────────────────────────┐
│           Cluster Autoscaler                      │
│                                                 │
│  1. Detect unschedulable pods                   │
│     - Pod cannot fit on any existing node       │
│     - Insufficient CPU/memory                   │
│                                                 │
│  2. Check if new node can be added              │
│     - Cloud provider API (AWS, GCP, Azure)      │
│     - Max node limit not reached                │
│                                                 │
│  3. Provision new node                          │
│     - Node joins cluster                        │
│     - Pod is scheduled on new node              │
│                                                 │
│  4. Scale down when nodes are underutilized     │
│     - All pods on node can move elsewhere       │
│     - Node removed from cluster                 │
└─────────────────────────────────────────────────┘
```

### Cloud Provider Setup

**AWS EKS:**
```bash
# Annotate the autoscaler service account
kubectl annotate serviceaccount cluster-autoscaler \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT:role/eksctl-cluster-autoscaler

# Deploy Cluster Autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-one-asg.yaml
```

**GCP GKE:**
```bash
# Enable Cluster Autoscaler on node pool
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --zone=us-central1-a
```

**Azure AKS:**
```bash
# Enable Cluster Autoscaler
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5
```

### Cluster Autoscaler Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.28.0
          command:
            - ./cluster-autoscaler
            - --cloud-provider=aws
            - --nodes=1:10:my-node-group  # min:max:node-group
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
            - --skip-nodes-with-local-storage=false
            - --skip-nodes-with-system-pods=false
```

---

## 6. KEDA — Event-Driven Autoscaling

KEDA (Kubernetes Event-driven Autoscaling) scales pods based on external events, not just CPU/memory.

### Supported Event Sources

| Event Source | Use Case |
|-------------|----------|
| **AWS SQS** | Queue depth |
| **Azure Service Bus** | Message count |
| **RabbitMQ** | Queue length |
| **Kafka** | Consumer group lag |
| **Redis** | List length |
| **Prometheus** | Custom metrics |
| **HTTP** | Request rate |
| **Cron** | Time-based scaling |

### Installing KEDA

```bash
# Install KEDA with Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

### KEDA ScaledObject Example

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-worker-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: queue-worker
  minReplicaCount: 1
  maxReplicaCount: 10
  pollingInterval: 10  # Check every 10s
  cooldownPeriod: 60   # Wait 60s before scaling down
  triggers:
    - type: aws-sqs
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
        queueLength: "5"  # 1 pod per 5 messages
        awsRegion: us-east-1
```

```
KEDA Scaling:
┌─────────────────────────────────────────────────┐
│  SQS Queue: 50 messages                        │
│                                                 │
│  KEDA detects queue depth                      │
│  Queue length target: 5 messages per pod       │
│  Desired pods: 50 / 5 = 10 pods                │
│                                                 │
│  ┌─────┐  ┌─────┐  ┌─────┐  ...  ┌─────┐     │
│  │Pod 1│  │Pod 2│  │Pod 3│       │Pod10│     │
│  └─────┘  └─────┘  └─────┘       └─────┘     │
│                                                 │
│  Messages processed, queue empties             │
│  KEDA scales back to minReplicaCount: 1        │
│  ┌─────┐                                       │
│  │Pod 1│  (idle, waiting for messages)        │
│  └─────┘                                       │
└─────────────────────────────────────────────────┘
```

### KEDA with Prometheus Trigger

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: http-scaler
spec:
  scaleTargetRef:
    name: web-deployment
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-server.monitoring.svc.cluster.local:80
        metricName: http_requests_per_second
        threshold: "100"
        query: sum(rate(http_requests_total[2m]))
```

---

## 7. Scaling Best Practices

### Resource Requests and Limits

HPA and VPA depend on accurate resource requests:

```yaml
containers:
  - name: web
    image: myapp:1.0
    resources:
      requests:
        cpu: "100m"    # Be realistic
        memory: "128Mi"
      limits:
        cpu: "500m"    # Higher than requests
        memory: "256Mi"
```

**Don't over-provision requests:**
```yaml
# Bad: Request much higher than actual usage
resources:
  requests:
    cpu: "1000m"   # Pod uses only 100m
    memory: "1Gi"  # Pod uses only 128Mi

# HPA will never trigger because CPU% is low (100/1000 = 10%)
# But the pod is wasting 90% of allocated CPU
```

**Don't under-provision requests:**
```yaml
# Bad: Request much lower than actual usage
resources:
  requests:
    cpu: "10m"     # Pod uses 200m
    memory: "32Mi" # Pod uses 256Mi

# HPA triggers too late (200/10 = 2000% → immediate scale up)
# Pod may be throttled before scaling kicks in
```

### HPA Configuration Best Practices

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 2      # At least 2 for HA
  maxReplicas: 20     # Reasonable upper bound
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60  # 60-70% is a good target
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 60s before next scale up
      policies:
        - type: Pods
          value: 4                      # Max 4 pods at a time
          periodSeconds: 60
        - type: Percent
          value: 100                    # Or 100% increase
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
      policies:
        - type: Percent
          value: 10                    # Max 10% decrease
          periodSeconds: 60
```

### Scaling Strategy Summary

| Scenario | Tool | Configuration |
|----------|------|--------------|
| Variable traffic | HPA | CPU/memory targets |
| Queue processing | KEDA | Queue depth |
| Scheduled traffic | KEDA Cron | Time-based |
| Resource optimization | VPA | Auto mode |
| Infrastructure scaling | Cluster Autoscaler | Cloud provider |

---

## 📝 Hands-On Exercises

### Exercise 1: HPA with CPU

```bash
# Create a deployment with resource requests
cat > hpa-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          ports:
            - containerPort: 80
EOF

kubectl apply -f hpa-deployment.yaml

# Create HPA
kubectl autoscale deployment hpa-demo \
  --cpu-percent=50 \
  --min=1 \
  --max=5

# Check HPA
kubectl get hpa
# NAME       REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# hpa-demo   Deployment/hpa-demo  0%/50%    1         5         1          30s

# Generate load (in another terminal)
kubectl run -it load-generator --image=busybox --restart=Never -- /bin/sh
# while true; do wget -q -O- http://hpa-demo; done

# Watch HPA scale up
kubectl get hpa -w
# NAME       REFERENCE            TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
# hpa-demo   Deployment/hpa-demo  150%/50%   1         5         3          2m
# hpa-demo   Deployment/hpa-demo  200%/50%   1         5         5          3m

# Stop load generator and wait for scale down
# (takes 5 minutes due to cooldown)

# Clean up
kubectl delete deployment hpa-demo
kubectl delete hpa hpa-demo
kubectl delete pod load-generator
```

### Exercise 2: HPA with Custom YAML

```bash
# Create deployment
kubectl create deployment web-app --image=nginx --replicas=2

# Set resource requests
kubectl set resources deployment/web-app \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi

# Create HPA from YAML
cat > web-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
EOF

kubectl apply -f web-hpa.yaml

# Check HPA
kubectl get hpa web-app-hpa
kubectl describe hpa web-app-hpa

# Clean up
kubectl delete hpa web-app-hpa
kubectl delete deployment web-app
```

### Exercise 3: VPA Recommendations

```bash
# Create a deployment
kubectl create deployment vpa-demo --image=nginx --replicas=2
kubectl set resources deployment/vpa-demo \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi

# Generate some traffic
for i in $(seq 1 100); do
  kubectl run test-$i --image=busybox --restart=Never --rm -it -- \
    wget -q -O- http://vpa-demo
done

# Create VPA in "Off" mode (recommendations only)
cat > vpa.yaml << 'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-demo
  updatePolicy:
    updateMode: "Off"
EOF

kubectl apply -f vpa.yaml

# Wait a few minutes, then check recommendations
kubectl get vpa vpa-demo -o jsonpath='{.status.recommendation.containerRecommendations}'

# Clean up
kubectl delete vpa vpa-demo
kubectl delete deployment vpa-demo
```

### Exercise 4: Testing Scale Down Behavior

```bash
# Create deployment and HPA
kubectl create deployment scale-test --image=nginx --replicas=3
kubectl set resources deployment/scale-test \
  --requests=cpu=100m,memory=128Mi
kubectl autoscale deployment scale-test \
  --cpu-percent=50 \
  --min=2 \
  --max=10

# Scale up manually to test scale down
kubectl scale deployment scale-test --replicas=8

# Watch HPA maintain the manual scale (no metric pressure)
kubectl get hpa -w

# After 5 minutes, HPA will scale back to minReplicas
# This demonstrates the cooldown behavior

# Clean up
kubectl delete hpa scale-test
kubectl delete deployment scale-test
```

---

## 🧠 Common Questions

**Q: HPA is not scaling up. What's wrong?**
A: Check these:
1. Are resource requests set? (HPA needs requests to calculate %)
2. Is metrics-server running? (`kubectl top pods`)
3. Are the metrics within target? (check `kubectl describe hpa`)
4. Is the deployment ready? (HPA only works on ready deployments)

```bash
kubectl describe hpa my-hpa
# Look for:
# - Warning  FailedComputeReplicas  ...  missing request for cpu
# - Warning  FailedGetResourceMetric  ...  unable to fetch metrics
```

**Q: Can I use HPA and VPA together?**
A: Yes, but with caution. Use VPA in "Initial" or "Off" mode to set resource requests/limits, and HPA to scale replicas. Don't use VPA Auto mode with HPA on CPU/memory metrics — they'll fight each other.

**Q: How fast does HPA scale?**
A: HPA checks metrics every 15 seconds. Scale-up is fast (within 30-60 seconds). Scale-down has a 5-minute cooldown to prevent flapping. You can tune this with `behavior.stabilizationWindowSeconds`.

**Q: What happens if HPA reaches maxReplicas?**
A: HPA stops scaling. All traffic goes to maxReplicas pods. If they're overwhelmed, you'll see increased latency and errors. Set maxReplicas high enough for your peak traffic.

**Q: Does Cluster Autoscaler work with HPA?**
A: Yes! HPA creates new pods, and if there's no room on existing nodes, Cluster Autoscaler adds new nodes. The chain is: HPA → pending pods → Cluster Autoscaler → new nodes → pods scheduled.

**Q: When should I use KEDA instead of HPA?**
A: Use HPA for CPU/memory-based scaling (web servers, APIs). Use KEDA for event-driven scaling (queue processors, batch jobs, scheduled tasks). KEDA supports many more event sources.

---

## ✅ Day 3 Summary

**What you learned today:**

- ✅ Why auto-scaling is essential for production
- ✅ Horizontal Pod Autoscaler (HPA) — scaling pod replicas
- ✅ HPA configuration with CPU, memory, and custom metrics
- ✅ Vertical Pod Autoscaler (VPA) — scaling resource requests
- ✅ Cluster Autoscaler — scaling nodes
- ✅ KEDA — event-driven autoscaling (SQS, Kafka, Prometheus, Cron)
- ✅ Scaling best practices (resource requests, HPA behavior, cooldown)
- ✅ HPA calculation formula and behavior

**Coming up tomorrow:** Production Best Practices — security, resource management, high availability, and operational excellence.

---

*💡 Tip: Set resource requests accurately! HPA, VPA, and the scheduler all depend on them. Base requests on actual usage (monitor with `kubectl top pods`), not guesswork. A good rule: set requests to your average usage, and limits to 2-5x requests.*