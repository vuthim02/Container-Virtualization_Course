# Week 5 • Day 2 • Advanced
# Pods & Deployments

## Progress Checklist
- [x] Day 1 — K8s Architecture
- [x] Day 2 — Pods & Deployments
- [ ] Day 3 — Services & Ingress
- [ ] Day 4 — Config & Secrets

---

← [Day 1](day1.md) | [Roadmap](../roadmap.md) | [Day 3 →](day3.md)

---

## 1. Pods — The Smallest Deployable Unit

A **Pod** is the smallest and simplest Kubernetes object. It represents a single instance of a running process in your cluster.

### What is a Pod?

```
┌─────────────────────────────────────────┐
│              Pod                        │
│                                         │
│  ┌─────────────┐  ┌─────────────┐      │
│  │ Container 1 │  │ Container 2 │      │
│  │  (App)      │  │  (Sidecar)  │      │
│  │  nginx:80   │  │  log-agent  │      │
│  └──────┬──────┘  └──────┬──────┘      │
│         │                │             │
│         └────────┬───────┘             │
│                  │                     │
│  ┌───────────────▼───────────────┐    │
│  │     Shared Volumes            │    │
│  │     (emptyDir, persistent)    │    │
│  └───────────────────────────────┘    │
│                                         │
│  Shared network namespace:              │
│  - Same IP address                      │
│  - Same port space (localhost)          │
│  - Can communicate via localhost        │
└─────────────────────────────────────────┘
```

**Key characteristics:**
- **One or more containers** that run together
- **Shared network**: Containers in a pod share the same IP address and port space
- **Shared storage**: Containers can share volumes
- **Co-scheduled**: All containers in a pod run on the same node
- **Co-located**: They share the same lifecycle (created, scheduled, terminated together)

### Single-Container Pod (Most Common)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

### Multi-Container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
    - name: web-app
      image: myapp:1.0
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app

    - name: log-agent
      image: fluentd:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app

  volumes:
    - name: shared-logs
      emptyDir: {}
```

### Pod Lifecycle

```
┌──────────────────────────────────────────────────────┐
│                  Pod Lifecycle                       │
│                                                      │
│  Pending → ContainerCreating → Running → Terminated │
│                                                      │
│  • Pending: Pod accepted, but containers not created │
│  • ContainerCreating: Pulling images, setting up     │
│  • Running: At least one container is running        │
│  • Succeeded: All containers terminated successfully │
│  • Failed: At least one container failed             │
│  • Unknown: Pod state cannot be determined           │
└──────────────────────────────────────────────────────┘
```

```bash
# Create a pod
kubectl apply -f pod.yaml

# Check pod status
kubectl get pods
# NAME        READY   STATUS    RESTARTS   AGE
# nginx-pod   1/1     Running   0          30s

# Detailed pod status
kubectl get pod nginx-pod -o wide
# NAME        READY   STATUS    IP           NODE
# nginx-pod   1/1     Running   10.244.0.5   minikube

# Describe the pod (shows events and detailed status)
kubectl describe pod nginx-pod
# Events:
#   Type    Reason     Age   From               Message
#   ----    ------     ----  ----               -------
#   Normal  Scheduled  40s   default-scheduler  Successfully assigned default/nginx-pod to minikube
#   Normal  Pulling    39s   kubelet            Pulling image "nginx:1.25"
#   Normal  Pulled     35s   kubelet            Successfully pulled image "nginx:1.25"
#   Normal  Created    35s   kubelet            Created container nginx
#   Normal  Started    35s   kubelet            Started container nginx

# View pod logs
kubectl logs nginx-pod
# /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty...
```

### Pod Restart Policy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-example
spec:
  restartPolicy: Always  # Options: Always, OnFailure, Never
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo Hello; sleep 30; exit 0"]
```

**Restart policies:**
- `Always` (default for Pods): Restart regardless of exit code
- `OnFailure`: Restart only if exit code is non-zero
- `Never`: Never restart the pod

```bash
# Watch pod lifecycle
kubectl get pods -w
# NAME              READY   STATUS              RESTARTS   AGE
# restart-example   0/1     ContainerCreating   0          0s
# restart-example   1/1     Running             0          5s
# restart-example   0/1     Completed           0          35s
# restart-example   1/1     Running             1          36s    ← Restarted
# restart-example   0/1     Completed           1          71s
# restart-example   1/1     Running             2          72s    ← Restarted again
```

---

## 2. Working with Pods

### Creating Pods Imperatively

```bash
# Run a single pod imperatively
kubectl run nginx --image=nginx:alpine

# Run a pod with a command
kubectl run mypod --image=busybox --command -- sh -c "while true; do echo hello; sleep 10; done"

# Run a pod with labels
kubectl run nginx --image=nginx --labels="app=web,tier=frontend"

# Run a pod and expose it
kubectl run nginx --image=nginx --port=80
kubectl expose pod nginx --port=80 --type=NodePort
```

### Executing Commands in Pods

```bash
# Execute a command in a running pod
kubectl exec nginx-pod -- ls /etc/nginx

# Open an interactive shell
kubectl exec -it nginx-pod -- /bin/bash
# root@nginx-pod:/# 
# root@nginx-pod:/# cat /etc/nginx/nginx.conf
# root@nginx-pod:/# exit

# Multi-container pod - specify container
kubectl exec -it app-with-sidecar -c web-app -- /bin/bash

# View logs
kubectl logs nginx-pod
kubectl logs nginx-pod -f          # Follow logs
kubectl logs nginx-pod --tail=50   # Last 50 lines
kubectl logs nginx-pod -c web-app  # Specific container
```

### Pod Resource Management

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "100m"      # 0.1 CPU core
          memory: "128Mi"  # 128 MiB
        limits:
          cpu: "500m"      # 0.5 CPU core
          memory: "256Mi"  # 256 MiB
```

**Resource units:**
- CPU: `1` = 1 core, `100m` = 0.1 cores, `1000m` = 1 core
- Memory: `128Mi` = 128 MiB, `1Gi` = 1 GiB

```bash
# View resource usage (requires metrics-server)
kubectl top pods
# NAME           CPU(cores)   MEMORY(bytes)
# resource-pod   5m           8Mi

# View node resources
kubectl top nodes
# NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# minikube 150m         7%     1200Mi          31%
```

### Pod Probes (Health Checks)

Kubernetes uses probes to check the health of containers:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: health-pod
spec:
  containers:
    - name: app
      image: nginx
      ports:
        - containerPort: 80
      
      # Liveness probe: Is the container alive?
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10   # Wait before first check
        periodSeconds: 5          # Check every 5s
        timeoutSeconds: 2         # Timeout after 2s
        failureThreshold: 3       # 3 failures = restart
        
      # Readiness probe: Is the container ready to accept traffic?
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 3
        
      # Startup probe: Has the application started?
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 10
```

**Probe types:**

| Probe | Purpose | Action on Failure |
|-------|---------|-------------------|
| **livenessProbe** | Is the container alive? | Restart the container |
| **readinessProbe** | Is the container ready? | Remove from service endpoints |
| **startupProbe** | Has the app started? | No action (waits for success) |

**Probe methods:**
```yaml
# HTTP GET
livenessProbe:
  httpGet:
    path: /health
    port: 8080

# TCP Socket
livenessProbe:
  tcpSocket:
    port: 3306

# Command execution
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
```

---

## 3. ReplicaSets — Ensuring Desired Replicas

A **ReplicaSet** ensures that a specified number of pod replicas are running at any given time.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```
┌─────────────────────────────────────────┐
│              ReplicaSet                 │
│                                         │
│  Desired: 3 replicas                    │
│  Current: 1 replica                     │
│                                         │
│  Action: Create 2 more pods             │
│  ┌─────┐  ┌─────┐  ┌─────┐            │
│  │Pod 1│  │Pod 2│  │Pod 3│            │
│  └─────┘  └─────┘  └─────┘            │
│                                         │
│  Controller loop:                       │
│  while true:                            │
│    current = get_pods()                 │
│    if current < desired:                │
│      create_pods(desired - current)     │
│    if current > desired:                │
│      delete_pods(current - desired)     │
│    sleep(5)                             │
└─────────────────────────────────────────┘
```

```bash
# Create a ReplicaSet
kubectl apply -f replicaset.yaml

# Check ReplicaSet status
kubectl get replicasets
# NAME         DESIRED   CURRENT   READY   AGE
# nginx-rs     3         3         3       2m

# Scale a ReplicaSet
kubectl scale replicaset nginx-rs --replicas=5

# View pods managed by the ReplicaSet
kubectl get pods -l app=nginx
# NAME              READY   STATUS    RESTARTS   AGE
# nginx-rs-abc123   1/1     Running   0          2m
# nginx-rs-def456   1/1     Running   0          2m
# nginx-rs-ghi789   1/1     Running   0          2m
```

**Note:** You rarely create ReplicaSets directly. Deployments manage ReplicaSets automatically.

---

## 4. Deployments — The Recommended Way to Run Apps

A **Deployment** provides declarative updates for Pods and ReplicaSets. It's the recommended way to run stateless applications.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # Max pods unavailable during update
      maxSurge: 1           # Max extra pods during update
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /
              port: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
```

### Deployment Strategies

```yaml
# Rolling Update (default) - Zero downtime
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%     # 25% of pods can be down
    maxSurge: 25%           # 25% extra pods can be created

# Recreate - All old pods terminated first
strategy:
  type: Recreate
```

```
Rolling Update Process:
┌─────────────────────────────────────────────────────┐
│  Version 1.25 (3 pods)                              │
│  ┌─────┐  ┌─────┐  ┌─────┐                        │
│  │v1.25│  │v1.25│  │v1.25│                        │
│  └─────┘  └─────┘  └─────┘                        │
│                                                     │
│  Update to v1.26:                                   │
│  1. Create 1 pod v1.26                              │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐              │
│  │v1.25│  │v1.25│  │v1.25│  │v1.26│              │
│  └─────┘  └─────┘  └─────┘  └─────┘              │
│                                                     │
│  2. Wait for readiness, then remove 1 v1.25         │
│  ┌─────┐  ┌─────┐  ┌─────┐                        │
│  │v1.25│  │v1.25│  │v1.26│                        │
│  └─────┘  └─────┘  └─────┘                        │
│                                                     │
│  3. Repeat until all pods are v1.26                 │
│  ┌─────┐  ┌─────┐  ┌─────┐                        │
│  │v1.26│  │v1.26│  │v1.26│                        │
│  └─────┘  └─────┘  └─────┘                        │
└─────────────────────────────────────────────────────┘
```

### Deployment Operations

```bash
# Create a deployment
kubectl create deployment nginx --image=nginx:1.25 --replicas=3

# Or apply from file
kubectl apply -f deployment.yaml

# Check deployment status
kubectl get deployments
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           5m

# View rollout history
kubectl rollout history deployment/nginx-deployment
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# Update the image (triggers rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26
# OR
kubectl edit deployment/nginx-deployment  # Edit image field

# Watch the rollout
kubectl rollout status deployment/nginx-deployment
# Waiting for deployment "nginx-deployment" rollout to finish:
#   2 out of 3 new replicas have been updated...
# deployment "nginx-deployment" successfully rolled out

# Pause the rollout (to debug)
kubectl rollout pause deployment/nginx-deployment

# Resume the rollout
kubectl rollout resume deployment/nginx-deployment

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Scale a deployment
kubectl scale deployment/nginx-deployment --replicas=5

# Check pods
kubectl get pods -l app=nginx
# NAME                               READY   STATUS    RESTARTS   AGE
# nginx-deployment-5d8f7c4b7-abc12   1/1     Running   0          5m
# nginx-deployment-5d8f7c4b7-def34   1/1     Running   0          5m
# nginx-deployment-5d8f7c4b7-ghi56   1/1     Running   0          5m
```

---

## 5. Labels and Selectors

Labels are key-value pairs attached to Kubernetes objects. Selectors filter objects based on labels.

### Labels

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    app: nginx
    version: "1.25"
    environment: production
    tier: frontend
spec:
  containers:
    - name: nginx
      image: nginx:1.25
```

### Selectors

```bash
# Equality-based selectors
kubectl get pods -l app=nginx
kubectl get pods -l environment=production
kubectl get pods -l app=nginx,version=1.25

# Set-based selectors
kubectl get pods -l 'app in (nginx, apache)'
kubectl get pods -l 'environment notin (development)'
kubectl get pods -l 'app'  # Pods with label 'app' (any value)

# Combined selectors
kubectl get pods -l 'app=nginx,environment=production'
```

**Selector types:**

| Type | Example | Meaning |
|------|---------|---------|
| **Equality** | `app=nginx` | app equals nginx |
| **Inequality** | `app!=nginx` | app not equal to nginx |
| **Set-based** | `app in (nginx, apache)` | app is nginx or apache |
| **Set-based** | `environment notin (dev, test)` | environment is not dev or test |
| **Existence** | `app` | Pod has label 'app' (any value) |

---

## 6. Annotations

Annotations are similar to labels, but they store **non-identifying metadata**. They're used by tools and libraries, not by Kubernetes itself.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotated-pod
  annotations:
    description: "This is a sample pod"
    contact: "admin@example.com"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    kubernetes.io/change-cause: "Updated to nginx 1.26"
spec:
  containers:
    - name: nginx
      image: nginx:1.25
```

```bash
# View annotations
kubectl get pod annotated-pod -o yaml | grep annotations

# Add annotation
kubectl annotate pod annotated-pod description="Updated description"

# Remove annotation
kubectl annotate pod annotated-pod description-
```

**Labels vs Annotations:**

| Feature | Labels | Annotations |
|---------|--------|-------------|
| **Purpose** | Identification and selection | Non-identifying metadata |
| **Used by** | Kubernetes (selectors, services) | External tools (Prometheus, CI/CD) |
| **Query efficiency** | Indexed and efficient | Not indexed |
| **Size limit** | 63 characters per value | No strict limit |

---

## 7. StatefulSets — Stateful Applications

**StatefulSets** manage stateful applications that require:
- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

```
StatefulSet Pod Names (stable, predictable):
┌─────────────────────────────────────────┐
│  StatefulSet: web (3 replicas)          │
│                                         │
│  web-0  ← Always named web-0           │
│  web-1  ← Always named web-1           │
│  web-2  ← Always named web-2           │
│                                         │
│  If web-1 dies, new pod is web-1        │
│  If scaled to 5, new pods: web-3, web-4 │
│  Scale down: web-4, web-3 removed first │
└─────────────────────────────────────────┘
```

**When to use StatefulSets:**
- Databases (MySQL, PostgreSQL, MongoDB)
- Key-value stores (etcd, Consul)
- Message queues (Kafka, RabbitMQ)
- Any application requiring stable identity

---

## 8. DaemonSets — Run on Every Node

**DaemonSets** ensure that a copy of a pod runs on **all** (or selected) nodes.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
spec:
  selector:
    matchLabels:
      app: monitoring
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      containers:
        - name: agent
          image: datadog/agent:latest
          env:
            - name: DD_API_KEY
              valueFrom:
                secretKeyRef:
                  name: datadog-secret
                  key: api-key
```

```
DaemonSet: One pod per node
┌─────────────────────────────────────────┐
│  Node 1     Node 2     Node 3           │
│  ┌─────┐    ┌─────┐    ┌─────┐         │
│  │Agent│    │Agent│    │Agent│         │
│  └─────┘    └─────┘    └─────┘         │
│                                         │
│  If Node 4 joins: Agent auto-deploys    │
│  If Node 2 leaves: Agent removed        │
└─────────────────────────────────────────┘
```

**Common DaemonSet use cases:**
- Log collection (Fluentd, Logstash)
- Monitoring agents (Datadog, Prometheus Node Exporter)
- Network plugins (Calico, Weave)
- Storage plugins (Ceph, GlusterFS)

---

## 📝 Hands-On Exercises

### Exercise 1: Create and Manage Pods

```bash
# Create a simple pod
kubectl run nginx --image=nginx:alpine --port=80

# Check pod status
kubectl get pods
kubectl describe pod nginx

# Execute commands inside the pod
kubectl exec nginx -- ls /etc/nginx
kubectl exec -it nginx -- /bin/sh

# View logs
kubectl logs nginx
kubectl logs nginx -f

# Port forward to access the pod
kubectl port-forward nginx 8080:80
# Open: http://localhost:8080

# Clean up
kubectl delete pod nginx
```

### Exercise 2: Create a Deployment with Rolling Updates

```bash
# Create a deployment
kubectl create deployment nginx --image=nginx:1.25 --replicas=3

# Verify the deployment
kubectl get deployments
kubectl get pods -o wide

# Update the image (triggers rolling update)
kubectl set image deployment/nginx nginx=nginx:1.26

# Watch the rollout
kubectl rollout status deployment/nginx

# Check rollout history
kubectl rollout history deployment/nginx

# Rollback to previous version
kubectl rollout undo deployment/nginx

# Scale up
kubectl scale deployment nginx --replicas=5

# Scale down
kubectl scale deployment nginx --replicas=2

# Clean up
kubectl delete deployment nginx
```

### Exercise 3: Health Checks

```bash
# Create a deployment with health checks
cat > health-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-check-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: health-demo
  template:
    metadata:
      labels:
        app: health-demo
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 3
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
EOF

kubectl apply -f health-deployment.yaml

# Check pods
kubectl get pods
kubectl describe pod -l app=health-demo

# Clean up
kubectl delete -f health-deployment.yaml
```

### Exercise 4: Labels and Selectors

```bash
# Create pods with different labels
kubectl run web --image=nginx --labels="app=web,tier=frontend"
kubectl run api --image=nginx --labels="app=api,tier=backend"
kubectl run db --image=nginx --labels="app=db,tier=backend"

# Filter by labels
kubectl get pods -l app=web
kubectl get pods -l tier=backend
kubectl get pods -l 'app in (web,api)'
kubectl get pods -l 'tier=backend,app!=db'

# Add labels to existing pods
kubectl label pod web environment=production

# Remove labels
kubectl label pod web environment-

# Clean up
kubectl delete pod web api db
```

---

## 🧠 Common Questions

**Q: What's the difference between a Pod and a Container?**
A: A Pod is a Kubernetes concept that wraps one or more containers. Containers are the actual runtime processes (Docker containers). A Pod provides shared networking and storage for its containers.

**Q: Should I create Pods directly or use Deployments?**
A: Always use Deployments for stateless applications. Deployments manage ReplicaSets, which manage Pods, providing self-healing, scaling, and rolling updates. Create Pods directly only for very specific use cases (testing, debugging, or when you need exact control).

**Q: What happens when a Pod dies?**
A: If managed by a ReplicaSet/Deployment, a new Pod is automatically created to replace it. If it's a standalone Pod, it's gone forever (unless restartPolicy is set). The new pod gets a new IP address.

**Q: Why use multi-container pods?**
A: Multi-container pods are for tightly coupled processes that need to share resources (network, storage, lifecycle). Examples: app + log collector, app + proxy, app + monitoring agent. They always run together on the same node.

**Q: How do rolling updates work?**
A: Kubernetes creates new pods with the updated version one at a time, waits for them to be ready, then terminates old pods. This ensures zero-downtime deployments. You can control the pace with `maxUnavailable` and `maxSurge`.

**Q: What's the difference between liveness and readiness probes?**
A: Liveness checks if the container is alive. If it fails, Kubernetes restarts the container. Readiness checks if the container is ready to serve traffic. If it fails, the pod is removed from service endpoints (but not restarted). Use both for production applications.

---

## ✅ Day 2 Summary

**What you learned today:**

- ✅ Pods: The smallest deployable unit in Kubernetes
- ✅ Single-container vs multi-container pods (sidecar pattern)
- ✅ Pod lifecycle and restart policies
- ✅ Pod resource requests and limits
- ✅ Health probes: liveness, readiness, and startup
- ✅ ReplicaSets: Ensuring desired number of replicas
- ✅ Deployments: Declarative updates and rolling deployments
- ✅ Deployment strategies (RollingUpdate, Recreate)
- ✅ Rollout management: history, pause, resume, undo
- ✅ Labels and selectors for object identification
- ✅ Annotations for non-identifying metadata
- ✅ StatefulSets for stateful applications
- ✅ DaemonSets for running pods on every node

**Coming up tomorrow:** Services & Ingress — how to expose your applications to internal and external traffic.

---

*💡 Tip: Always use Deployments instead of creating Pods directly. Deployments give you self-healing, scaling, and rolling updates out of the box. Pods should only be created directly for very specific use cases like testing or debugging.*