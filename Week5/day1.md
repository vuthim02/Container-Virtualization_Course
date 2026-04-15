# Week 5 • Day 1 • Advanced
# Kubernetes Architecture

## Progress Checklist
- [x] Day 1 — K8s Architecture
- [ ] Day 2 — Pods & Deployments
- [ ] Day 3 — Services & Ingress
- [ ] Day 4 — Config & Secrets

---

← [Roadmap](../roadmap.md) | [Day 2 →](day2.md)

---

## 1. What is Kubernetes?

Kubernetes (often abbreviated as K8s) is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.

### The Problem Kubernetes Solves

```
Without Kubernetes:
┌─────────────────────────────────────────┐
│  Docker Host                           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │ App: 1  │ │ App: 2  │ │ App: 3  │  │
│  └─────────┘ └─────────┘ └─────────┘  │
│                                        │
│  ❌ Manual scaling                     │
│  ❌ No automatic restart on failure    │
│  ❌ No rolling updates                 │
│  ❌ No self-healing                    │
│  ❌ No centralized management          │
└─────────────────────────────────────────┘

With Kubernetes:
┌─────────────────────────────────────────┐
│  Kubernetes Cluster                    │
│  ┌───────────────────────────────────┐ │
│  │  Control Plane (Brain)           │ │
│  │  - API Server                    │ │
│  │  - Scheduler                     │ │
│  │  - Controller Manager            │ │
│  │  - etcd                          │ │
│  └───────────────────────────────────┘ │
│                                        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │ Node 1  │ │ Node 2  │ │ Node 3  │ │
│  │ ┌─────┐ │ │ ┌─────┐ │ │ ┌─────┐ │ │
│  │ │Pod  │ │ │ │Pod  │ │ │ │Pod  │ │ │
│  │ └─────┘ │ │ └─────┘ │ │ └─────┘ │ │
│  └─────────┘ └─────────┘ └─────────┘ │
│                                        │
│  ✅ Auto-scaling                       │
│  ✅ Self-healing                       │
│  ✅ Rolling updates                    │
│  ✅ Service discovery                  │
│  ✅ Load balancing                     │
│  ✅ Declarative configuration          │
└─────────────────────────────────────────┘
```

### Kubernetes vs Docker Compose

| Feature | Docker Compose | Kubernetes |
|---------|---------------|------------|
| **Scope** | Single host | Multi-host cluster |
| **Scaling** | Manual (`--scale`) | Automatic (HPA) |
| **Self-healing** | Limited | Full (restart, reschedule) |
| **Rolling updates** | No | Yes |
| **Service discovery** | DNS-based | DNS + Environment variables |
| **Load balancing** | External (nginx, Traefik) | Built-in |
| **Configuration** | YAML (docker-compose.yml) | YAML (multiple resource files) |
| **Complexity** | Low | High |
| **Use case** | Development, small apps | Production, large-scale |

---

## 2. Kubernetes Cluster Architecture

A Kubernetes cluster consists of two main parts:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                          │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    Control Plane (Master)                 │ │
│  │                                                           │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐  │ │
│  │  │  API Server │  │  Scheduler  │  │  Controller Mgr  │  │ │
│  │  │  (kube-     │  │             │  │                  │  │ │
│  │  │   apiserver)│  │             │  │                  │  │ │
│  │  └──────┬──────┘  └──────┬──────┘  └────────┬─────────┘  │ │
│  │         │                │                   │            │ │
│  │         └────────────────┼───────────────────┘            │ │
│  │                          │                                │ │
│  │                  ┌───────▼───────┐                        │ │
│  │                  │     etcd      │                        │ │
│  │                  │ (Key-Value    │                        │ │
│  │                  │  Store)       │                        │ │
│  │                  └───────────────┘                        │ │
│  └───────────────────────────────────────────────────────────┘ │
│                           │                                    │
│              ┌────────────┼────────────┐                      │
│              │            │            │                      │
│         ┌────▼───┐  ┌────▼───┐  ┌────▼───┐                   │
│         │ Node 1 │  │ Node 2 │  │ Node 3 │                   │
│         │        │  │        │  │        │                   │
│         │┌──────┐│  │┌──────┐│  │┌──────┐│                   │
│         ││Pod   ││  ││Pod   ││  ││Pod   ││                   │
│         │└──────┘│  │└──────┘│  │└──────┘│                   │
│         │┌──────┐│  │┌──────┐│  │┌──────┐│                   │
│         ││Pod   ││  ││Pod   ││  ││Pod   ││                   │
│         │└──────┘│  │└──────┘│  │└──────┘│                   │
│         └────────┘  └────────┘  └────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Control Plane Components

The Control Plane is the brain of the cluster. It makes global decisions, detects and responds to cluster events, and maintains the desired state.

### 3.1 API Server (kube-apiserver)

The **single entry point** to the cluster. All communication between components goes through the API server.

```
┌─────────────────────────────────────────┐
│           API Server                    │
│                                         │
│  ┌───────────────────────────────────┐ │
│  │  REST API (HTTP/HTTPS)           │ │
│  │  - Authentication                │ │
│  │  - Authorization (RBAC)          │ │
│  │  - Admission Control             │ │
│  │  - Validation                    │ │
│  └───────────────────────────────────┘ │
│                                         │
│  Port: 6443 (default)                  │
│  Protocol: HTTPS                       │
└─────────────────────────────────────────┘
         ▲           ▲           ▲
         │           │           │
    ┌────┴───┐  ┌────┴───┐  ┌────┴───┐
    │ kubectl│  │  UI   │  │  API   │
    │ (CLI)  │  │(Dash   │  │Clients │
    │        │  │ board) │  │        │
    └────────┘  └────────┘  └────────┘
```

**Key responsibilities:**
- **Authentication**: Who are you? (certificates, tokens, OIDC)
- **Authorization**: What can you do? (RBAC, ABAC, Node, Webhook)
- **Admission Control**: Enforce policies before persisting objects
- **Validation**: Ensure the request is well-formed
- **API Versioning**: Support multiple API versions (v1, apps/v1, etc.)

```bash
# The API server is the only component that talks to etcd directly
# All kubectl commands go through the API server:

kubectl get pods
# kubectl → API Server → etcd → return pods

kubectl apply -f deployment.yml
# kubectl → API Server → validate → persist in etcd

# Check the API server
kubectl cluster-info
# Kubernetes control plane is running at https://<ip>:6443

# View API server logs (on the control plane node)
kubectl logs -n kube-system -l component=kube-apiserver
```

### 3.2 etcd

A **consistent and highly-available key-value store** used as Kubernetes' backing store for all cluster data.

```
┌─────────────────────────────────────────┐
│              etcd                       │
│                                         │
│  Key-Value Store:                       │
│  ┌───────────────────────────────────┐ │
│  │  /registry/pods/default/nginx-abc │ │
│  │  → {status: Running, IP: 10.0.0.5}│ │
│  │                                   │ │
│  │  /registry/services/default/web   │ │
│  │  → {type: ClusterIP, port: 80}    │ │
│  │                                   │ │
│  │  /registry/configmaps/default/app │ │
│  │  → {data: {key1: value1}}         │ │
│  └───────────────────────────────────┘ │
│                                         │
│  Raft Consensus Algorithm               │
│  - Requires odd number of nodes (1, 3, 5)│
│  - Leader election                      │
│  - Fault tolerant (can lose N/2 nodes)  │
│                                         │
│  Port: 2379 (client), 2380 (peer)      │
└─────────────────────────────────────────┘
```

**Key characteristics:**
- **Consistent**: Every read returns the latest data
- **Highly available**: Tolerates node failures (needs odd number of nodes: 1, 3, 5)
- **Watch support**: Clients can watch for changes in real-time
- **Compact**: Old revisions are automatically cleaned up

```bash
# Access etcd directly (on control plane node)
ETCDCTL_API=3 etcdctl --endpoints=127.0.0.1:2379 get /registry/pods --prefix --keys-only

# Backup etcd
ETCDCTL_API=3 etcdctl --endpoints=127.0.0.1:2379 snapshot save /backup/snapshot.db

# Restore from backup
ETCDCTL_API=3 etcdctl --endpoints=127.0.0.1:2379 snapshot restore /backup/snapshot.db
```

### 3.3 Scheduler (kube-scheduler)

Watches for newly created **Pods with no assigned node** and selects the best node for them.

```
┌─────────────────────────────────────────┐
│            Scheduler                    │
│                                         │
│  Scheduling Process:                    │
│  ┌───────────────────────────────────┐ │
│  │  1. Filter (Feasibility)         │ │
│  │     - Does the node have enough   │ │
│  │       CPU/memory?                 │ │
│  │     - Does the node match the     │ │
│  │       nodeSelector/affinity?      │ │
│  │     - Are there port conflicts?   │ │
│  │                                   │ │
│  │  2. Score (Priority)             │ │
│  │     - Balance resource usage      │ │
│  │     - Prefer nodes with more      │ │
│  │       available resources         │ │
│  │     - Spread pods across zones    │ │
│  │                                   │ │
│  │  3. Bind                         │ │
│  │     - Assign pod to the highest-  │ │
│  │       scoring node                │ │
│  └───────────────────────────────────┘ │
│                                         │
└─────────────────────────────────────────┘
```

**Scheduling factors:**
- Resource requirements (CPU, memory requests)
- Node selectors and affinity
- Taints and tolerations
- Pod affinity/anti-affinity
- Data locality
- Hardware constraints (GPU, SSD)

```bash
# View scheduler logs
kubectl logs -n kube-system -l component=kube-scheduler

# Example: Pod waiting to be scheduled
kubectl get pods
# NAME    READY   STATUS    REASON
# nginx   0/1     Pending   Unschedulable

# Why is it pending?
kubectl describe pod nginx
# Events:
#   Type     Reason            Age   From               Message
#   ----     ------            ----  ----               -------
#   Warning  FailedScheduling  30s   default-scheduler  0/3 nodes are available:
#     3 Insufficient memory.
```

### 3.4 Controller Manager (kube-controller-manager)

Runs multiple **controller processes** that regulate the state of the cluster. Each controller compares the **current state** with the **desired state** and takes action to reconcile differences.

```
┌──────────────────────────────────────────────────────┐
│              Controller Manager                      │
│                                                      │
│  ┌────────────┐  ┌────────────┐  ┌───────────────┐  │
│  │Node        │  │Replication │  │Deployment     │  │
│  │Controller  │  │Controller  │  │Controller     │  │
│  └────────────┘  └────────────┘  └───────────────┘  │
│  ┌────────────┐  ┌────────────┐  ┌───────────────┐  │
│  │Endpoint    │  │Service     │  │Job Controller │  │
│  │Controller  │  │Account     │  │               │  │
│  └────────────┘  └────────────┘  └───────────────┘  │
│  ┌────────────┐  ┌────────────┐                     │
│  │Namespace   │  │Persistent  │                     │
│  │Controller  │  │Volume      │                     │
│  │            │  │Controller  │                     │
│  └────────────┘  └────────────┘                     │
└──────────────────────────────────────────────────────┘
```

**How controllers work (Reconciliation Loop):**

```
┌─────────────────────────────────────────────────────┐
│              Reconciliation Loop                    │
│                                                     │
│        Desired State        Current State           │
│        (from etcd)          (from nodes)            │
│             │                    │                  │
│             ▼                    ▼                  │
│        ┌─────────────────────────────┐              │
│        │     Compare States          │              │
│        └──────────────┬──────────────┘              │
│                       │                             │
│              ┌────────▼────────┐                    │
│              │    Different?   │                    │
│              └───┬─────────┬───┘                    │
│                  │         │                        │
│                 Yes        No                       │
│                  │         │                        │
│                  ▼         │                        │
│        ┌───────────────────┘                        │
│        │                                            │
│        ▼                                            │
│  Take Action to Reconcile                          │
│  (create pods, delete pods, etc.)                  │
│        │                                            │
│        ▼                                            │
│  Update Current State                              │
│  (loop continues forever)                          │
└─────────────────────────────────────────────────────┘
```

**Example: ReplicaSet Controller**

```
Desired: 3 replicas
Current: 1 replica

ReplicaSet Controller:
  "I need 3 pods, but only see 1. I'll create 2 more."

  Creates pod-2, Creates pod-3

Desired: 3 replicas
Current: 3 replicas

ReplicaSet Controller:
  "All good. Nothing to do."

---

Desired: 3 replicas
Current: 5 replicas (user manually created extra pods)

ReplicaSet Controller:
  "I need 3 pods, but see 5. I'll delete 2."

  Deletes extra pods
```

---

## 4. Worker Node Components

Worker nodes (also called Minions) are the machines that run the actual containerized applications.

### 4.1 Kubelet

The **primary Kubernetes agent** running on each worker node. It communicates with the API server and manages containers on its node.

```
┌───────────────────────────────────────────────────────┐
│                    Worker Node                        │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │              Kubelet                            │ │
│  │                                                 │ │
│  │  Responsibilities:                             │ │
│  │  - Register node with the cluster              │ │
│  │  - Watch for pod assignments from API server   │ │
│  │  - Ensure containers are running in desired    │ │
│  │    state (via container runtime)               │ │
│  │  - Run health checks (liveness, readiness)     │ │
│  │  - Report node and pod status to API server    │ │
│  │  - Execute pod lifecycle hooks                 │ │
│  └─────────────────────────────────────────────────┘ │
│                       │                               │
│              ┌────────▼────────┐                      │
│              │Container Runtime│                      │
│              │(containerd/     │                      │
│              │ CRI-O/Docker)   │                      │
│              └────────┬────────┘                      │
│                       │                               │
│         ┌─────────────┼─────────────┐                │
│         ▼             ▼             ▼                │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐            │
│    │  Pod 1  │  │  Pod 2  │  │  Pod 3  │            │
│    │(container│  │(container│  │(container│           │
│    │ s)      │  │ s)      │  │ s)      │            │
│    └─────────┘  └─────────┘  └─────────┘            │
└───────────────────────────────────────────────────────┘
```

```bash
# Kubelet communicates with the container runtime via CRI
# (Container Runtime Interface)

# View kubelet logs (on the worker node)
journalctl -u kubelet -f

# Check node status from the control plane
kubectl get nodes
# NAME     STATUS   ROLES    AGE   VERSION
# node-1   Ready    worker   10d   v1.29.0
# node-2   Ready    worker   10d   v1.29.0
# node-3   Ready    worker   10d   v1.29.0

# Detailed node information
kubectl describe node node-1
# Conditions:
#   Type                 Status
#   MemoryPressure       False
#   DiskPressure         False
#   PIDPressure          False
#   Ready                True
#
# Allocated resources:
#   Resource     Requests    Limits
#   cpu          500m (25%)  1000m (50%)
#   memory       512Mi (13%) 1Gi (26%)
```

### 4.2 Kube Proxy

A **network proxy** running on each node. It maintains network rules that allow communication between pods inside and outside the cluster.

```
┌───────────────────────────────────────────────────────┐
│                    Worker Node                        │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │              Kube-Proxy                         │ │
│  │                                                 │ │
│  │  Responsibilities:                             │ │
│  │  - Maintain network rules on each node         │ │
│  │  - Implement Service abstraction (iptables,    │ │
│  │    IPVS, or nftables)                          │ │
│  │  - Load balance traffic across pod endpoints   │ │
│  │  - Handle ClusterIP, NodePort, LoadBalancer    │ │
│  └─────────────────────────────────────────────────┘ │
│                       │                               │
│         ┌─────────────▼─────────────┐                │
│         │    iptables / IPVS rules  │                │
│         │    (network routing)      │                │
│         └─────────────┬─────────────┘                │
│                       │                               │
│              ┌────────▼────────┐                      │
│              │   Network Stack │                      │
│              └────────┬────────┘                      │
│                       │                               │
│         ┌─────────────┼─────────────┐                │
│         ▼             ▼             ▼                │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐            │
│    │  Pod 1  │  │  Pod 2  │  │  Pod 3  │            │
│    │:80      │  │:80      │  │:80      │            │
│    └─────────┘  └─────────┘  └─────────┘            │
│                                                       │
│  Service: ClusterIP 10.96.0.1:80                     │
│  Kube-proxy creates rules to distribute traffic      │
│  across all 3 pods                                   │
└───────────────────────────────────────────────────────┘
```

**How kube-proxy works (iptables mode):**

```bash
# Kube-proxy creates iptables rules like:

# Traffic to Service IP 10.96.0.1:80
iptables -t nat -A KUBE-SERVICES -d 10.96.0.1/32 -p tcp --dport 80 \
  -j KUBE-SVC-ABC123

# Distribute across 3 pods (statistical mode)
iptables -t nat -A KUBE-SVC-ABC123 \
  -m statistic --mode random --probability 0.33333 \
  -j KUBE-SEP-POD1

iptables -t nat -A KUBE-SVC-ABC123 \
  -m statistic --mode random --probability 0.50000 \
  -j KUBE-SEP-POD2

iptables -t nat -A KUBE-SVC-ABC123 \
  -j KUBE-SEP-POD3

# View kube-proxy rules
iptables -t nat -L KUBE-SERVICES -n
```

### 4.3 Container Runtime

The software responsible for **actually running containers**. Kubernetes supports multiple runtimes through the **Container Runtime Interface (CRI)**.

```
┌──────────────────────────────────────────────────────┐
│                  Container Runtime                   │
│                                                      │
│  Supported Runtimes:                                │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐ │
│  │ containerd │  │   CRI-O    │  │     Docker     │ │
│  │ (Recommended)│  │(Lightweight)│  │(via dockershim │ │
│  │            │  │            │  │ - DEPRECATED)  │ │
│  └────────────┘  └────────────┘  └────────────────┘ │
│                                                      │
│  Docker support was removed in Kubernetes 1.24!     │
│  Use containerd or CRI-O instead.                   │
│                                                      │
│  Container Runtime Interface (CRI):                 │
│  - Standard API for container management            │
│  - Allows Kubernetes to use different runtimes      │
│  - Operations: Create, Start, Stop, Delete, Exec   │
└──────────────────────────────────────────────────────┘
```

```bash
# Check the container runtime on a node
kubectl get nodes -o wide
# NAME     STATUS   CONTAINER-RUNTIME
# node-1   Ready    containerd://1.7.0

# Container runtime versions
kubectl describe node node-1 | grep "Container Runtime"
#   Container Runtime Version:  containerd://1.7.0
```

---

## 5. Kubernetes Objects and Resources

Everything in Kubernetes is represented as **API objects** — persistent entities that represent the state of your cluster.

### Core Objects

```
┌──────────────────────────────────────────────────────────┐
│                Kubernetes Objects                        │
│                                                          │
│  Workloads:                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐│
│  │  Pod     │  │ReplicaSet│  │Deployment│  │Stateful  ││
│  │          │  │          │  │          │  │Set       ││
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘│
│  ┌──────────┐  ┌──────────┐                             │
│  │DaemonSet │  │   Job    │                             │
│  │          │  │          │                             │
│  └──────────┘  └──────────┘                             │
│                                                          │
│  Networking:                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Service  │  │  Ingress │  │Network   │               │
│  │          │  │          │  │Policy    │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                          │
│  Storage:                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │Volume    │  │Persistent│  │Storage   │               │
│  │          │  │Volume    │  │Class     │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                          │
│  Configuration:                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ConfigMap │  │  Secret  │  │Service   │               │
│  │          │  │          │  │Account   │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                          │
│  Metadata:                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │Namespace │  │  Label   │  │Annotation│               │
│  │          │  │          │  │          │               │
│  └──────────┘  └──────────┘  └──────────┘               │
└──────────────────────────────────────────────────────────┘
```

### Object Structure

Every Kubernetes object follows the same YAML structure:

```yaml
apiVersion: apps/v1          # Which API version to use
kind: Deployment             # What type of object
metadata:                    # Identifying information
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
spec:                        # Desired state
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:                  # Pod template
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
status:                      # Current state (managed by Kubernetes)
  replicas: 3
  readyReplicas: 3
  availableReplicas: 3
```

---

## 6. Namespaces

Namespaces provide a **logical separation** of cluster resources, like virtual clusters within a physical cluster.

```
┌─────────────────────────────────────────────────────┐
│              Kubernetes Cluster                     │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Namespace: default                         │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │ nginx   │  │ redis   │  │ my-app  │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘    │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Namespace: staging                         │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │ nginx   │  │ redis   │  │ my-app  │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘    │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Namespace: production                      │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │ nginx   │  │ redis   │  │ my-app  │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘    │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  Resources (CPU, memory, etc.) can be limited     │
│  per namespace using ResourceQuotas                │
└─────────────────────────────────────────────────────┘
```

```bash
# List all namespaces
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   10d
# kube-system       Active   10d
# kube-public       Active   10d
# kube-node-lease   Active   10d

# Create a namespace
kubectl create namespace staging
kubectl create namespace production

# Create resources in a namespace
kubectl run nginx --image=nginx -n staging
kubectl get pods -n staging

# Set default namespace for current context
kubectl config set-context --current --namespace=staging

# View all resources across namespaces
kubectl get all --all-namespaces
```

**Built-in namespaces:**
- `default`: Default namespace for objects with no other namespace
- `kube-system`: Kubernetes system components (API server, scheduler, etc.)
- `kube-public`: Resources accessible by all users (cluster info)
- `kube-node-lease`: Node lease objects (heartbeat for nodes)

---

## 7. Setting Up a Local Kubernetes Cluster

### Option 1: minikube

minikube runs a single-node Kubernetes cluster in a VM or container.

```bash
# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start a cluster
minikube start
# 😄  minikube v1.32.0 on Ubuntu 22.04
# ✨  Automatically selected docker driver
# 📌  Using Docker driver with root privileges
# 📌  Starting control plane node minikube in cluster minikube
# 🚜  Pulling base image...
# 🎉  minikube cluster started!

# Check the cluster
minikube status
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: configured

# Use kubectl with minikube
kubectl cluster-info
# Kubernetes control plane is running at https://192.168.49.2:8443

kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   60s   v1.28.3

# Open the Kubernetes dashboard
minikube dashboard

# Stop the cluster
minikube stop

# Delete the cluster
minikube delete
```

### Option 2: kind (Kubernetes IN Docker)

kind runs Kubernetes clusters inside Docker containers.

```bash
# Install kind
go install sigs.k8s.io/kind@v0.20.0

# Create a cluster
kind create cluster --name my-cluster
# Creating cluster "my-cluster" ...
# ✓ Ensuring node image (kindest/node:v1.28.0)
# ✓ Preparing nodes
# ✓ Writing configuration
# ✓ Starting control-plane
# ✓ Installing CNI
# ✓ Installing StorageClass
# Set kubectl context to "kind-my-cluster"

# Check the cluster
kubectl get nodes
# NAME                     STATUS   ROLES           AGE   VERSION
# my-cluster-control-plane Ready    control-plane   90s   v1.28.0

# Create a multi-node cluster
kind create cluster --name multi-node \
  --config=- <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# List clusters
kind get clusters
# my-cluster
# multi-node

# Delete a cluster
kind delete cluster --name my-cluster
```

### Option 3: k3s (Lightweight Kubernetes)

k3s is a certified Kubernetes distribution designed for resource-constrained environments.

```bash
# Install k3s
curl -sfL https://get.k3s.io | sh -

# Check the cluster
sudo k3s kubectl get nodes
# NAME     STATUS   ROLES                  AGE   VERSION
# myhost   Ready    control-plane,master   60s   v1.28.2+k3s1

# Copy the kubeconfig file
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
kubectl get nodes

# Stop k3s
sudo systemctl stop k3s
sudo systemctl start k3s
```

---

## 8. kubectl — The Kubernetes CLI

`kubectl` is the primary tool for interacting with a Kubernetes cluster.

```bash
# Cluster information
kubectl cluster-info
kubectl version
kubectl api-versions
kubectl api-resources

# Get resources
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get nodes
kubectl get all            # All resources in current namespace
kubectl get pods -n kube-system  # Specific namespace
kubectl get pods -o wide   # Extended information
kubectl get pods -o yaml   # YAML output
kubectl get pods -o json   # JSON output

# Describe resources (detailed info)
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe svc <service-name>

# Create resources
kubectl apply -f file.yaml          # Apply from file
kubectl apply -f directory/         # Apply all YAML in directory
kubectl create deployment nginx --image=nginx  # Imperative create

# Delete resources
kubectl delete -f file.yaml
kubectl delete pod <pod-name>
kubectl delete deployment <deployment-name>

# Execute commands in pods
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- sh
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -f          # Follow (tail -f)
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> --tail=100  # Last 100 lines

# Port forwarding
kubectl port-forward <pod-name> 8080:80
# Access pod's port 80 via localhost:8080

# Debugging
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
```

---

## 📝 Hands-On Exercises

### Exercise 1: Set Up a Local Cluster with minikube

```bash
# Start minikube
minikube start

# Verify the cluster
kubectl cluster-info
kubectl get nodes
kubectl get namespaces

# Explore the kube-system namespace
kubectl get pods -n kube-system

# View detailed information about a system pod
kubectl describe pod -n kube-system -l component=kube-apiserver

# Stop the cluster
minikube stop
```

### Exercise 2: Deploy Your First Application

```bash
# Ensure you have a running cluster
kubectl get nodes

# Create a deployment
kubectl create deployment nginx --image=nginx:alpine

# Check the deployment
kubectl get deployments
kubectl get pods
kubectl get replicaset

# Describe the pod
kubectl describe pod -l app=nginx

# View logs
kubectl logs -l app=nginx

# Expose the deployment as a service
kubectl expose deployment nginx --port=80 --type=NodePort

# Get the service
kubectl get services

# Access the application (minikube)
minikube service nginx

# Or use port forwarding
kubectl port-forward svc/nginx 8080:80
# Then open: http://localhost:8080

# Clean up
kubectl delete service nginx
kubectl delete deployment nginx
```

### Exercise 3: Multi-Node Cluster with kind

```bash
# Create a multi-node cluster
kind create cluster --name exercise-cluster --config=- <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# Check nodes
kubectl get nodes -o wide

# Deploy a web application
kubectl create deployment web --image=nginx:alpine --replicas=3

# Check that pods are scheduled across worker nodes
kubectl get pods -o wide
# NAME                  READY   STATUS    NODE
# web-abc123-def1       1/1     Running   exercise-cluster-worker
# web-abc123-def2       1/1     Running   exercise-cluster-worker2
# web-abc123-def3       1/1     Running   exercise-cluster-control-plane

# Clean up
kind delete cluster --name exercise-cluster
```

### Exercise 4: Explore Kubernetes Objects

```bash
# Start a fresh cluster
minikube start

# Create a namespace
kubectl create namespace myapp

# Create a deployment in the namespace
kubectl create deployment web --image=nginx:alpine -n myapp

# List all objects in the namespace
kubectl get all -n myapp

# View the YAML of the deployment
kubectl get deployment web -n myapp -o yaml

# Describe the deployment
kubectl describe deployment web -n myapp

# Scale the deployment
kubectl scale deployment web --replicas=3 -n myapp

# Watch the pods being created
kubectl get pods -n myapp -w

# Clean up
kubectl delete namespace myapp
```

---

## 🧠 Common Questions

**Q: What's the difference between a Pod and a Deployment?**
A: A Pod is the smallest unit — it runs one or more containers. A Deployment manages ReplicaSets, which manage Pods. You almost never create Pods directly; you create Deployments.

```
Deployment → manages → ReplicaSet → manages → Pods → run → Containers
```

**Q: Why does Kubernetes use etcd instead of a traditional database?**
A: etcd provides strong consistency, distributed consensus (via Raft), and watch capabilities — all essential for cluster state management. Traditional databases don't offer the same consistency guarantees or the ability to watch for changes in real-time.

**Q: Can I run Kubernetes on a single machine?**
A: Yes! minikube, kind, and k3s all allow you to run K8s on a single machine. Production clusters use multiple nodes for high availability, but single-node clusters are perfect for development and testing.

**Q: How is Kubernetes different from Docker Swarm?**
A: Kubernetes is more feature-rich (auto-scaling, self-healing, rolling updates, extensive ecosystem) but also more complex. Swarm is simpler to set up but has fewer features and a smaller community. Kubernetes is the industry standard for production orchestration.

**Q: What happens when the API server goes down?**
A: Existing pods continue running, but you can't create/update/delete resources, and controllers can't reconcile state. The cluster becomes read-only until the API server recovers. etcd and the controllers will retry once the API server is back.

**Q: How does Kubernetes know which node to schedule a pod on?**
A: The scheduler evaluates multiple factors: resource requests (CPU/memory), node selectors, affinity/anti-affinity rules, taints/tolerations, and available capacity. It filters out unsuitable nodes, scores the remaining ones, and binds the pod to the highest-scoring node.

---

## ✅ Day 1 Summary

**What you learned today:**

- ✅ What Kubernetes is and the problems it solves
- ✅ Kubernetes cluster architecture (Control Plane + Worker Nodes)
- ✅ Control Plane components: API Server, etcd, Scheduler, Controller Manager
- ✅ Worker Node components: Kubelet, Kube-Proxy, Container Runtime
- ✅ Kubernetes objects and their structure (apiVersion, kind, metadata, spec, status)
- ✅ Namespaces for logical isolation
- ✅ Setting up local clusters (minikube, kind, k3s)
- ✅ kubectl basics (get, describe, apply, delete, exec, logs)

**Coming up tomorrow:** Pods & Deployments — the core workload resources in Kubernetes, including lifecycle, health checks, and update strategies.

---

*💡 Tip: Always use a local cluster (minikube or kind) for learning and development. Never practice on a production cluster. Get comfortable with kubectl and the core concepts before moving to more advanced topics.*