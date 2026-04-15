# Week 5 • Day 3 • Advanced
# Services & Ingress

## Progress Checklist
- [x] Day 1 — K8s Architecture
- [x] Day 2 — Pods & Deployments
- [x] Day 3 — Services & Ingress
- [ ] Day 4 — Config & Secrets

---

← [Day 2](day2.md) | [Roadmap](../roadmap.md) | [Day 4 →](day4.md)

---

## 1. The Problem: Pod IP Addresses Are Ephemeral

Pods are **ephemeral** — they can die and be recreated at any time. When this happens, they get **new IP addresses**.

```
Without Services:
┌─────────────────────────────────────────────────┐
│  Client App wants to talk to Database           │
│                                                 │
│  Database Pod 1: 10.244.0.5 (dies)             │
│  New Database Pod 2: 10.244.1.8 (new IP!)      │
│                                                 │
│  ❌ Client's hardcoded IP is now invalid        │
│  ❌ Client needs to discover the new IP         │
│  ❌ No load balancing across multiple pods      │
└─────────────────────────────────────────────────┘

With Services:
┌─────────────────────────────────────────────────┐
│  Client App ──▶ Service: 10.96.0.1:5432        │
│                         │                       │
│                    ┌────┼────┐                 │
│                    ▼    ▼    ▼                 │
│                 [Pod1] [Pod2] [Pod3]           │
│                 10.244.0.5  10.244.1.8 ...     │
│                                                 │
│  ✅ Service IP is stable                       │
│  ✅ Automatic load balancing                   │
│  ✅ Automatic endpoint updates                 │
│  ✅ DNS resolution (service-name.namespace)     │
└─────────────────────────────────────────────────┘
```

A **Service** is an abstraction that defines:
- A stable IP address (ClusterIP)
- A stable DNS name
- Load balancing across a set of pods (selected by labels)
- Network policies for accessing those pods

---

## 2. Service Types

Kubernetes provides four service types:

| Type | Purpose | Use Case |
|------|---------|----------|
| **ClusterIP** (default) | Internal-only IP | Internal microservices |
| **NodePort** | Exposes on each node's IP | Development, testing |
| **LoadBalancer** | External load balancer | Production (cloud providers) |
| **ExternalName** | Maps to external DNS | External services |

---

## 3. ClusterIP — Internal Services

The default service type. Creates an internal IP accessible only within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP  # Default
  selector:
    app: web
  ports:
    - protocol: TCP
      port: 80        # Service port
      targetPort: 8080  # Container port
```

```
ClusterIP Service:
┌─────────────────────────────────────────────────┐
│  Cluster: 10.96.0.0/16                         │
│                                                 │
│  Service: web-service                           │
│  ClusterIP: 10.96.1.100                         │
│  Port: 80 → targetPort: 8080                   │
│                                                 │
│  Inside cluster:                                │
│  curl http://10.96.1.100:80                     │
│  curl http://web-service:80                     │
│  curl http://web-service.default.svc.cluster.local │
│                                                 │
│  Outside cluster:                               │
│  ❌ Cannot reach 10.96.1.100                   │
└─────────────────────────────────────────────────┘
```

```bash
# Create a deployment
kubectl create deployment web --image=nginx --replicas=3

# Expose it as a ClusterIP service
kubectl expose deployment web --port=80 --target-port=80

# Check the service
kubectl get services
# NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP   10d
# web           ClusterIP   10.96.145.231   <none>        80/TCP    5s

# Describe the service
kubectl describe service web
# Name:              web
# Namespace:         default
# Type:              ClusterIP
# IP:                10.96.145.231
# Port:              80/TCP
# TargetPort:        80/TCP
# Endpoints:         10.244.0.5:80,10.244.0.6:80,10.244.0.7:80
# Session Affinity:  None

# Test the service from inside a pod
kubectl run test --image=busybox --command -- sleep infinity
kubectl exec test -- wget -qO- http://web:80
# <html>...Welcome to nginx!...</html>

# View endpoints
kubectl get endpoints web
# NAME   ENDPOINTS                                    AGE
# web    10.244.0.5:80,10.244.0.6:80,10.244.0.7:80   1m

# Clean up
kubectl delete service web
kubectl delete deployment web
kubectl delete pod test
```

### Service Discovery via DNS

Kubernetes automatically creates DNS records for services:

```
DNS Format: <service-name>.<namespace>.svc.cluster.local

Examples:
- web.default.svc.cluster.local
- web.default
- web  (from within the same namespace)

# From another pod in the default namespace:
curl http://web
curl http://web:80
curl http://web.default.svc.cluster.local:80

# From another namespace:
curl http://web.other-namespace.svc.cluster.local:80
```

---

## 4. NodePort — Expose on Node IP

Exposes the service on a static port on each node's IP address.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - protocol: TCP
      port: 80          # Service port (ClusterIP)
      targetPort: 80    # Container port
      nodePort: 30080   # Node port (30000-32767)
```

```
NodePort Service:
┌─────────────────────────────────────────────────┐
│  Node 1: 192.168.1.10                          │
│  Node 2: 192.168.1.11                          │
│  Node 3: 192.168.1.12                          │
│                                                 │
│  Access from outside:                           │
│  http://192.168.1.10:30080                      │
│  http://192.168.1.11:30080                      │
│  http://192.168.1.12:30080                      │
│                                                 │
│  kube-proxy distributes traffic to:             │
│  ┌─────┐  ┌─────┐  ┌─────┐                    │
│  │Pod 1│  │Pod 2│  │Pod 3│                    │
│  └─────┘  └─────┘  └─────┘                    │
└─────────────────────────────────────────────────┘
```

```bash
# Create a NodePort service
kubectl expose deployment web --port=80 --type=NodePort

# Check the service
kubectl get services
# NAME   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# web    NodePort   10.96.145.231   <none>        80:30080/TCP   5s

# Access the service
# On minikube:
minikube service web

# On a real cluster:
curl http://<node-ip>:30080

# Get the node port
kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}'

# Clean up
kubectl delete service web
```

---

## 5. LoadBalancer — Cloud Provider Integration

Creates an external load balancer (provided by your cloud provider).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```
LoadBalancer Service:
┌─────────────────────────────────────────────────┐
│              External Load Balancer             │
│              (Cloud Provider)                   │
│              External IP: 203.0.113.1           │
│                      │                          │
│         ┌────────────┼────────────┐            │
│         ▼            ▼            ▼            │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│    │ Node 1  │  │ Node 2  │  │ Node 3  │     │
│    │  Pod 1  │  │  Pod 2  │  │  Pod 3  │     │
│    └─────────┘  └─────────┘  └─────────┘     │
│                                                 │
│  User: http://203.0.113.1:80                   │
│  → Load balancer distributes traffic           │
└─────────────────────────────────────────────────┘
```

```bash
# Create a LoadBalancer service
kubectl expose deployment web --port=80 --type=LoadBalancer

# Check the service
kubectl get services
# NAME   TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
# web    LoadBalancer   10.96.145.231   203.0.113.1    80:30080/TCP   1m

# The EXTERNAL-IP may show <pending> initially
# It takes time for the cloud provider to provision the LB

# Wait until EXTERNAL-IP is assigned
kubectl get svc web -w
# NAME   TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
# web    LoadBalancer   10.96.145.231   <pending>       80:30080/TCP
# web    LoadBalancer   10.96.145.231   203.0.113.1    80:30080/TCP  ← Assigned!

# Access the service
curl http://203.0.113.1

# Clean up
kubectl delete service web
```

**Cloud provider support:**
- AWS: Classic Load Balancer, Network Load Balancer, Application Load Balancer
- GCP: Google Cloud Load Balancing
- Azure: Azure Load Balancer
- DigitalOcean: DigitalOcean Load Balancer

---

## 6. Multiple Ports in a Service

A service can expose multiple ports:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: myapp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
    - name: grpc
      protocol: TCP
      port: 9090
      targetPort: 9090
```

---

## 7. Headless Services

A headless service has no ClusterIP. It's used for direct pod access (typically with StatefulSets).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None  # Headless!
  selector:
    app: database
  ports:
    - port: 5432
      targetPort: 5432
```

```
Headless Service:
┌─────────────────────────────────────────────────┐
│  Service: headless-service                      │
│  clusterIP: None                                │
│                                                 │
│  DNS resolves to ALL pod IPs directly:          │
│  headless-service.default.svc.cluster.local     │
│  → 10.244.0.5                                   │
│  → 10.244.0.6                                   │
│  → 10.244.0.7                                   │
│                                                 │
│  Client picks one pod directly (no load balancing)│
│  Useful for:                                    │
│  - Database replication (direct pod-to-pod)     │
│  - StatefulSets (predictable DNS names)         │
│  - Direct pod access                            │
└─────────────────────────────────────────────────┘
```

```bash
# Headless service DNS returns all pod IPs
kubectl run test --image=busybox --command -- sleep infinity
kubectl exec test -- nslookup headless-service
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# Name: headless-service.default.svc.cluster.local
# Address: 10.244.0.5
# Name: headless-service.default.svc.cluster.local
# Address: 10.244.0.6
# Name: headless-service.default.svc.cluster.local
# Address: 10.244.0.7
```

---

## 8. Ingress — HTTP/HTTPS Routing

**Ingress** is an API object that manages external access to services, typically HTTP/HTTPS.

```
Without Ingress:
┌─────────────────────────────────────────────────┐
│  User → LoadBalancer → Service → Pod            │
│                                                 │
│  Need a separate LB for each service            │
│  Expensive and complex                          │
└─────────────────────────────────────────────────┘

With Ingress:
┌─────────────────────────────────────────────────┐
│  User → Ingress Controller → Ingress Rules      │
│                              │                  │
│               ┌──────────────┼──────────────┐  │
│               ▼              ▼              ▼  │
│         /api → api-svc   /web → web-svc   / → default│
│                                                 │
│  Single entry point for multiple services       │
│  Path-based and host-based routing              │
│  SSL/TLS termination                            │
└─────────────────────────────────────────────────┘
```

### Ingress Controller

Ingress rules need an **Ingress Controller** to work. Popular options:

| Controller | Description | Best For |
|------------|-------------|----------|
| **nginx** | Most popular, feature-rich | Production |
| **Traefik** | Cloud-native, auto-discovery | Dynamic environments |
| **HAProxy** | High performance | Enterprise |
| **Contour** | Envoy-based, Kubernetes-native | Modern stacks |
| **Istio Gateway** | Service mesh integration | Microservices |

```bash
# Install nginx ingress controller (minikube)
minikube addons enable ingress

# Or install manually
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Check the ingress controller
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Basic Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: myapp.example.com
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

### Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: default-service
                port:
                  number: 80
```

```
Path-Based Routing:
┌─────────────────────────────────────────────────┐
│  myapp.example.com                             │
│                                                 │
│  /api/*    → api-service:8080                  │
│  /web/*    → web-service:80                    │
│  /*        → default-service:80                │
│                                                 │
│  User requests:                                 │
│  myapp.example.com/api/users → api-service     │
│  myapp.example.com/web/home → web-service      │
│  myapp.example.com/ → default-service          │
└─────────────────────────────────────────────────┘
```

### TLS/SSL Termination

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
    - hosts:
        - myapp.example.com
      secretName: tls-secret
  rules:
    - host: myapp.example.com
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

```
TLS Termination:
┌─────────────────────────────────────────────────┐
│  User ──HTTPS──▶ Ingress Controller             │
│                │                                │
│                │ (TLS terminated here)          │
│                │                                │
│                ▼                                │
│          ┌─────┴─────┐                         │
│          │  Service  │                         │
│          │  (HTTP)   │                         │
│          └───────────┘                         │
│                                                 │
│  Benefits:                                     │
│  - Single TLS termination point                │
│  - Backend communication is HTTP (internal)    │
│  - Easy certificate management                 │
└─────────────────────────────────────────────────┘
```

```bash
# Create a TLS secret
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Apply the ingress
kubectl apply -f ingress.yaml

# Check ingress rules
kubectl get ingress
# NAME            CLASS   HOSTS                ADDRESS        PORTS
# tls-ingress     nginx   myapp.example.com    203.0.113.1   80, 443

# Describe the ingress
kubectl describe ingress tls-ingress

# Test (with host header)
curl -k https://203.0.113.1 -H "Host: myapp.example.com"
```

### Ingress Annotations

Annotations control Ingress Controller behavior:

```yaml
metadata:
  annotations:
    # Rewrite the URL path
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    
    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
    
    # Basic authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    
    # Proxy body size
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    
    # Session affinity (sticky sessions)
    nginx.ingress.kubernetes.io/affinity: cookie
    nginx.ingress.kubernetes.io/session-cookie-name: route
```

---

## 9. Service Selectors and Endpoint Slices

Services use **selectors** to find pods. The **Endpoint Controller** watches pods and updates the service's endpoints automatically.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
    version: v2  # Only pods with BOTH labels
  ports:
    - port: 80
      targetPort: 8080
```

```
Selector Matching:
┌─────────────────────────────────────────────────┐
│  Service selector: app=myapp, version=v2        │
│                                                 │
│  Pod 1: app=myapp, version=v1    ❌ No match   │
│  Pod 2: app=myapp, version=v2    ✅ Match      │
│  Pod 3: app=myapp, version=v2    ✅ Match      │
│  Pod 4: app=other, version=v2    ❌ No match   │
│                                                 │
│  Endpoints: Pod 2, Pod 3                        │
└─────────────────────────────────────────────────┘
```

### Manual Endpoints (Headless use case)

You can also manually specify endpoints (rarely used):

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
  - addresses:
      - ip: 192.168.1.100
      - ip: 192.168.1.101
    ports:
      - port: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
    - port: 3306
```

---

## 📝 Hands-On Exercises

### Exercise 1: Create Services

```bash
# Create a deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Create a ClusterIP service
kubectl expose deployment nginx --port=80 --name=nginx-clusterip

# Check the service
kubectl get svc nginx-clusterip
kubectl describe svc nginx-clusterip

# Test from a test pod
kubectl run test --image=busybox --command -- sleep infinity
kubectl exec test -- wget -qO- http://nginx-clusterip

# Create a NodePort service
kubectl expose deployment nginx --port=80 --type=NodePort --name=nginx-nodeport
kubectl get svc nginx-nodeport

# Access via minikube
minikube service nginx-nodeport --url

# Clean up
kubectl delete svc nginx-clusterip nginx-nodeport
kubectl delete deployment nginx
kubectl delete pod test
```

### Exercise 2: Path-Based Ingress

```bash
# Ensure ingress addon is enabled
minikube addons enable ingress

# Create deployments
kubectl create deployment web --image=nginx --replicas=2
kubectl create deployment api --image=nginx --replicas=2

# Create services
kubectl expose deployment web --port=80 --name=web-svc
kubectl expose deployment api --port=80 --name=api-svc

# Create an ingress resource
cat > ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: demo.local
      http:
        paths:
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
EOF

kubectl apply -f ingress.yaml

# Check ingress
kubectl get ingress
kubectl describe ingress demo-ingress

# Test (using minikube IP)
MINIKUBE_IP=$(minikube ip)
curl -H "Host: demo.local" http://$MINIKUBE_IP/web
curl -H "Host: demo.local" http://$MINIKUBE_IP/api

# Clean up
kubectl delete ingress demo-ingress
kubectl delete svc web-svc api-svc
kubectl delete deployment web api
```

### Exercise 3: Service Discovery via DNS

```bash
# Create a deployment and service
kubectl create deployment redis --image=redis:alpine
kubectl expose deployment redis --port=6379 --name=redis-svc

# Create a test pod
kubectl run test --image=busybox --command -- sleep infinity

# Test DNS resolution
kubectl exec test -- nslookup redis-svc
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# Name: redis-svc.default.svc.cluster.local
# Address: 10.96.x.x

# Test connectivity
kubectl exec test -- wget -qO- http://redis-svc:6379 || echo "Connection attempted"

# Test fully qualified name
kubectl exec test -- nslookup redis-svc.default.svc.cluster.local

# Clean up
kubectl delete svc redis-svc
kubectl delete deployment redis
kubectl delete pod test
```

---

## 🧠 Common Questions

**Q: What happens when a pod is deleted and recreated? Does the service update automatically?**
A: Yes! The Endpoint Controller watches for pod changes. When a pod dies, it's removed from the service's endpoints. When a new pod starts, it's added. This happens in seconds.

**Q: Can a service route to pods in different namespaces?**
A: No. Services only select pods in their own namespace. To reach a service in another namespace, use the fully qualified DNS name: `service-name.namespace.svc.cluster.local`.

**Q: What's the difference between port, targetPort, and nodePort?**
A: 
- `port`: The port the service listens on (ClusterIP port)
- `targetPort`: The port on the container
- `nodePort`: The port on the node (for NodePort type)

```yaml
ports:
  - port: 80        # Service port (what clients use)
    targetPort: 8080  # Container port (what the app listens to)
    nodePort: 30080   # Node port (external access)
```

**Q: How does Ingress differ from a LoadBalancer service?**
A: A LoadBalancer creates a cloud provider LB for each service (expensive). Ingress uses a single LB (the Ingress Controller) and routes to multiple services based on rules (cheap and flexible).

**Q: Why use a headless service?**
A: Headless services (clusterIP: None) return all pod IPs directly via DNS. This is useful for stateful applications (databases, message queues) where clients need to connect to specific pods directly.

**Q: Can I use multiple ingress controllers?**
A: Yes. Use the `ingressClassName` field to specify which controller should handle the ingress:

```yaml
spec:
  ingressClassName: nginx  # or traefik, contour, etc.
```

---

## ✅ Day 3 Summary

**What you learned today:**

- ✅ Why services are needed (pod IPs are ephemeral)
- ✅ Service types: ClusterIP, NodePort, LoadBalancer, ExternalName
- ✅ ClusterIP for internal communication
- ✅ NodePort for external access (development/testing)
- ✅ LoadBalancer for cloud provider integration
- ✅ DNS-based service discovery
- ✅ Headless services for direct pod access
- ✅ Ingress for HTTP/HTTPS routing
- ✅ Path-based and host-based routing
- ✅ TLS/SSL termination with Ingress
- ✅ Ingress annotations for advanced configuration
- ✅ Ingress controllers (nginx, Traefik, HAProxy, etc.)
- ✅ Service selectors and endpoint management

**Coming up tomorrow:** Config & Secrets — managing configuration and sensitive data in Kubernetes.

---

*💡 Tip: Always use Services to expose your pods, never access pod IPs directly. Services give you stable IPs, DNS names, and automatic load balancing. For HTTP traffic, prefer Ingress over multiple LoadBalancer services — it's more cost-effective and easier to manage.*