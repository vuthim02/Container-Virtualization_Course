# Week 6 • Day 2 • Advanced
# Monitoring

## Progress Checklist
- [x] Day 1 — CI/CD Integration
- [x] Day 2 — Monitoring
- [ ] Day 3 — Scaling
- [ ] Day 4 — Production Best Practices

---

← [Day 1](day1.md) | [Roadmap](../roadmap.md) | [Day 3 →](day3.md)

---

## 1. Why Monitoring Matters

Without monitoring, you're flying blind:

```
Without Monitoring:
┌─────────────────────────────────────────────────┐
│  User: "The app is slow!"                      │
│  You: *checking manually*                      │
│         - Is the pod running?                  │
│         - How much CPU/memory is it using?     │
│         - Are there any errors in the logs?    │
│         - Is the database overloaded?          │
│                                                 │
│  ❌ Reactive (wait for users to complain)      │
│  ❌ Slow diagnosis                              │
│  ❌ No historical data                         │
│  ❌ No alerts for critical issues              │
└─────────────────────────────────────────────────┘

With Monitoring:
┌─────────────────────────────────────────────────┐
│  Prometheus: "CPU at 95% for 5 minutes!"       │
│  Alert: 🔔 High CPU usage on web-deployment    │
│  Grafana Dashboard:                            │
│    - CPU: 95% (spike started 5 min ago)        │
│    - Memory: 70% (stable)                      │
│    - Requests: 500/min → 200/min               │
│    - Errors: 0.5% → 15%                        │
│                                                 │
│  You: Scale up the deployment before users     │
│       notice                                   │
│                                                 │
│  ✅ Proactive (detect before users notice)     │
│  ✅ Fast diagnosis (visualize trends)          │
│  ✅ Historical data (compare with past)        │
│  ✅ Automated alerts                          │
└─────────────────────────────────────────────────┘
```

---

## 2. The Kubernetes Monitoring Stack

```
┌─────────────────────────────────────────────────┐
│              Monitoring Stack                   │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │  Grafana (Visualization)                  │ │
│  │  - Dashboards                             │ │
│  │  - Alerts                                 │ │
│  │  - Data source: Prometheus                │ │
│  └───────────────┬───────────────────────────┘ │
│                  │                              │
│  ┌───────────────▼───────────────────────────┐ │
│  │  Prometheus (Metrics Collection)          │ │
│  │  - Scrapes metrics from targets           │ │
│  │  - Stores time-series data                 │ │
│  │  - PromQL query language                  │ │
│  │  - Alerting rules                         │ │
│  └───────────────┬───────────────────────────┘ │
│                  │                              │
│  ┌───────────────▼───────────────────────────┐ │
│  │  Metrics Sources                          │ │
│  │  - kubelet (node/pod metrics)             │ │
│  │  - cAdvisor (container metrics)           │ │
│  │  - kube-state-metrics (K8s objects)       │ │
│  │  - Application metrics (custom)           │ │
│  └───────────────────────────────────────────┘ │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │  Logging (EFK/Loki)                       │ │
│  │  - Elasticsearch/ Loki (storage)          │ │
│  │  - Fluentd/ Fluent Bit (collection)       │ │
│  │  - Kibana/ Grafana (visualization)        │ │
│  └───────────────────────────────────────────┘ │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │  Tracing (Jaeger)                         │ │
│  │  - Distributed tracing                    │ │
│  │  - Request flow visualization             │ │
│  │  - Latency analysis                       │ │
│  └───────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

---

## 3. Prometheus — Metrics Collection

Prometheus is an open-source monitoring system that:
- Scrapes metrics from configured targets
- Stores them as time-series data
- Provides a powerful query language (PromQL)
- Triggers alerts based on rules

### Architecture

```
┌─────────────────────────────────────────────────┐
│              Prometheus Server                  │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │  Service Discovery                        │ │
│  │  - Kubernetes API (pods, services, nodes) │ │
│  │  - File-based (static targets)            │ │
│  └───────────────┬───────────────────────────┘ │
│                  │                              │
│  ┌───────────────▼───────────────────────────┐ │
│  │  Scraping                                 │ │
│  │  - HTTP /metrics endpoint                 │ │
│  │  - Every 15s (configurable)               │ │
│  └───────────────┬───────────────────────────┘ │
│                  │                              │
│  ┌───────────────▼───────────────────────────┐ │
│  │  TSDB (Time-Series Database)              │ │
│  │  - Local storage                          │ │
│  │  - Retention: 15 days (configurable)      │ │
│  └───────────────┬───────────────────────────┘ │
│                  │                              │
│  ┌───────────────▼───────────────────────────┐ │
│  │  PromQL Engine                            │ │
│  │  - Query metrics                          │ │
│  │  - Generate alerts                        │ │
│  └───────────────────────────────────────────┘ │
│                                                 │
│  Alertmanager (separate component)             │
│  - Receives alerts from Prometheus             │
│  - Routes to Slack, Email, PagerDuty, etc.     │
└─────────────────────────────────────────────────┘
```

### Installing Prometheus with Helm

```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace

# Check installation
kubectl get pods -n monitoring
# NAME                                       READY   STATUS
# prometheus-alertmanager-abc123             2/2     Running
# prometheus-kube-state-metrics-def456       1/1     Running
# prometheus-node-exporter-ghi789            1/1     Running
# prometheus-pushgateway-jkl012              1/1     Running
# prometheus-server-mno345                   2/2     Running

# Access Prometheus server
kubectl port-forward svc/prometheus-server -n monitoring 9090:80
# Open: http://localhost:9090
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s     # How often to scrape
  evaluation_interval: 15s # How often to evaluate rules

scrape_configs:
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
```

### Metrics Endpoints

Prometheus scrapes metrics from HTTP endpoints that expose `/metrics`:

```
# Node metrics (from kubelet)
http://10.0.1.10:10250/metrics

# Pod metrics (from cAdvisor)
http://10.0.1.10:10250/metrics/cadvisor

# Kubernetes object metrics (from kube-state-metrics)
http://kube-state-metrics:8080/metrics

# Application metrics (custom)
http://myapp:8080/metrics
```

Example metrics output:
```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 10234
http_requests_total{method="POST",status="200"} 567
http_requests_total{method="GET",status="500"} 12

# HELP http_request_duration_seconds HTTP request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 8000
http_request_duration_seconds_bucket{le="0.5"} 9500
http_request_duration_seconds_bucket{le="1.0"} 10000
http_request_duration_seconds_sum 2500.5
http_request_duration_seconds_count 10234
```

---

## 4. PromQL — Prometheus Query Language

PromQL is a functional query language for retrieving and manipulating time-series data.

### Basic Queries

```promql
# Instant query: Current value
node_cpu_seconds_total

# With label filter
node_cpu_seconds_total{mode="idle"}

# Multiple label filters
node_cpu_seconds_total{mode="idle", instance="10.0.1.10:9100"}

# Rate of change (over 5 minutes)
rate(node_cpu_seconds_total{mode="idle"}[5m])

# Average across all instances
avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

### Query Types

| Type | Example | Description |
|------|---------|-------------|
| **Instant Vector** | `node_cpu_seconds_total` | Single value per time series |
| **Range Vector** | `node_cpu_seconds_total[5m]` | Values over time range |
| **Scalar** | `avg(node_cpu_seconds_total)` | Single numeric value |

### Common Functions

| Function | Example | Description |
|----------|---------|-------------|
| `rate()` | `rate(http_requests_total[5m])` | Per-second rate of increase |
| `irate()` | `irate(http_requests_total[5m])` | Instant rate (last 2 points) |
| `increase()` | `increase(http_requests_total[1h])` | Total increase over time |
| `sum()` | `sum(node_cpu_seconds_total)` | Sum across all series |
| `avg()` | `avg(node_cpu_usage_percent)` | Average across all series |
| `max()` | `max(node_memory_usage)` | Maximum across all series |
| `min()` | `min(node_disk_free)` | Minimum across all series |
| `count()` | `count(up{job="kubernetes-pods"})` | Count of series |
| `histogram_quantile()` | `histogram_quantile(0.95, ...)` | Percentile from histogram |

### Practical Queries

```promql
# CPU usage percentage per pod
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)

# Memory usage per pod
sum(container_memory_working_set_bytes{namespace="default"}) by (pod)

# Pod restart count in last hour
increase(kube_pod_container_status_restarts_total{namespace="default"}[1h])

# HTTP request rate (5 min average)
sum(rate(http_requests_total{status=~"2.."}[5m]))

# HTTP error rate (5xx errors)
sum(rate(http_requests_total{status=~"5.."}[5m]))

# Request latency (95th percentile)
histogram_quantile(0.95, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Node disk usage
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

# Available memory percentage
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

---

## 5. Grafana — Visualization

Grafana is an open-source visualization tool that connects to Prometheus (and other data sources) to create dashboards.

### Installing Grafana

```bash
# Install Grafana with Helm
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set adminPassword=admin123

# Get the admin password
kubectl get secret --namespace monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d

# Access Grafana
kubectl port-forward svc/grafana -n monitoring 3000:80
# Open: http://localhost:3000
# Login: admin / admin123
```

### Adding Prometheus as Data Source

```
1. Go to Configuration → Data Sources
2. Click "Add data source"
3. Select "Prometheus"
4. URL: http://prometheus-server.monitoring.svc.cluster.local:80
5. Click "Save & Test"
```

### Importing Pre-built Dashboards

Grafana has a library of community dashboards:

```
1. Go to Create → Import
2. Enter dashboard ID:
   - 315  (Node Exporter)
   - 6417 (Kubernetes Cluster)
   - 13105 (Kubernetes Pods)
   - 1860 (Node Exporter Full)
   - 8588 (Kubernetes API Server)

3. Select Prometheus data source
4. Click Import
```

### Creating a Custom Dashboard

```
1. Create → New Dashboard
2. Add a new panel
3. Query:
   - Data source: Prometheus
   - Query: sum(rate(http_requests_total[5m])) by (service)
4. Visualization: Time series
5. Title: "Request Rate by Service"
6. Save dashboard
```

---

## 6. Application Metrics

To monitor your application, expose metrics in the Prometheus format:

### Node.js Example

```javascript
const prometheus = require('prom-client');

// Create a Registry
const register = new prometheus.Registry();

// Enable default metrics (CPU, memory, GC, etc.)
prometheus.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5]  // seconds
});

const httpRequestTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'status_code']
});

register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestTotal);

// Middleware to collect metrics
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .observe(duration);
    httpRequestTotal.labels(req.method, res.statusCode).inc();
  });
  next();
});

// Expose /metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### Python Example

```python
from prometheus_client import start_http_server, Counter, Histogram, generate_latest
from flask import Flask, Response

app = Flask(__name__)

# Custom metrics
REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'status'])
REQUEST_DURATION = Histogram('http_request_duration_seconds', 'Request duration')

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype='text/plain')

@app.route('/')
def index():
    REQUEST_COUNT.labels(method='GET', status=200).inc()
    REQUEST_DURATION.observe(0.1)
    return 'Hello!'

if __name__ == '__main__':
    start_http_server(8000)  # Expose metrics on port 8000
    app.run(port=5000)
```

### Prometheus Service Discovery for Apps

```yaml
# Add to prometheus.yml
scrape_configs:
  - job_name: 'myapp'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

```yaml
# Add annotation to your pod
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8000"
    prometheus.io/path: "/metrics"
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      ports:
        - containerPort: 5000  # App port
        - containerPort: 8000  # Metrics port
```

---

## 7. Logging with EFK Stack

The EFK stack (Elasticsearch, Fluentd, Kibana) collects and stores logs from all pods.

### Installing EFK Stack

```bash
# Add Helm repository
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=1 \
  --set volumeClaimTemplate.storageClassName=standard

# Install Kibana
helm install kibana elastic/kibana \
  --namespace logging \
  --set elasticsearch.hosts=http://elasticsearch-master:9200

# Install Fluentd
kubectl apply -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch-rbac.yaml

# Access Kibana
kubectl port-forward svc/kibana-kibana -n logging 5601:5601
# Open: http://localhost:5601
```

### Alternative: Loki + Fluent Bit

Loki is a lighter-weight alternative to Elasticsearch, designed for Kubernetes:

```bash
# Install Loki stack with Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --set promtail.enabled=true \
  --set grafana.enabled=false

# Add Loki as Grafana data source
# URL: http://loki.logging.svc.cluster.local:3100
```

---

## 8. Alerting with Alertmanager

Prometheus generates alerts, and Alertmanager routes them to the right channels.

### Alert Rules

```yaml
# alert-rules.yml
groups:
  - name: kubernetes
    rules:
      # Pod restart alert
      - alert: PodHighRestartRate
        expr: increase(kube_pod_container_status_restarts_total[1h]) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} restarting frequently"
          description: "Pod {{ $labels.pod }} has restarted {{ $value }} times in the last hour"

      # Pod not ready alert
      - alert: PodNotReady
        expr: kube_pod_status_ready{condition="true"} == 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} not ready"
          description: "Pod {{ $labels.pod }} has been not ready for 10 minutes"

      # Node high CPU alert
      - alert: NodeHighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} high CPU usage"
          description: "CPU usage is above 90% for 10 minutes"

      # Node disk space alert
      - alert: NodeDiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} disk space low"
          description: "Disk usage is above 90%"

      # High error rate alert
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High HTTP 5xx error rate"
          description: "Error rate is {{ $value | humanizePercentage }}"
```

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'namespace']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'slack-notifications'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      repeat_interval: 5m
    - match:
        severity: warning
      receiver: 'slack-notifications'
      repeat_interval: 1h

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
```

---

## 📝 Hands-On Exercises

### Exercise 1: Install Prometheus and Grafana

```bash
# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace \
  --set server.persistentVolume.enabled=true \
  --set server.persistentVolume.size=10Gi

# Install Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set adminPassword=admin123 \
  --set persistence.enabled=true \
  --set persistence.size=5Gi

# Check pods
kubectl get pods -n monitoring

# Access Prometheus
kubectl port-forward svc/prometheus-server -n monitoring 9090:80

# Access Grafana
kubectl port-forward svc/grafana -n monitoring 3000:80
```

### Exercise 2: Query Metrics with PromQL

```bash
# Access Prometheus UI
kubectl port-forward svc/prometheus-server -n monitoring 9090:80

# Go to http://localhost:9090 and try these queries:

# 1. CPU usage per namespace
sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)

# 2. Memory usage per pod
sum(container_memory_working_set_bytes{container!=""}) by (pod)

# 3. Node memory usage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# 4. Pod restarts
increase(kube_pod_container_status_restarts_total[1h])

# 5. Container uptime
time() - container_start_time_seconds
```

### Exercise 3: Create a Grafana Dashboard

```bash
# Access Grafana
kubectl port-forward svc/grafana -n monitoring 3000:80

# Login: admin / admin123

# Import pre-built dashboards:
# 1. Go to Dashboards → Import
# 2. Enter ID: 13105 (Kubernetes Pods)
# 3. Select Prometheus data source
# 4. Click Import

# Create a custom dashboard:
# 1. Create → Dashboard
# 2. Add panel → Query:
#    - sum(rate(http_requests_total[5m])) by (service)
# 3. Visualization: Time series
# 4. Title: "Request Rate"
# 5. Save

# Import more dashboards:
# - Node Exporter Full: 1860
# - Kubernetes Cluster: 6417
# - Nginx Ingress Controller: 9614
```

### Exercise 4: Application Metrics

```bash
# Create a simple app with metrics
cat > metrics-app.py << 'EOF'
from prometheus_client import start_http_server, Counter, Histogram
import time
import random

REQUEST_COUNT = Counter('http_requests_total', 'Total requests', ['method', 'status'])
REQUEST_DURATION = Histogram('http_request_duration_seconds', 'Request duration')

# Simulate HTTP requests
while True:
    method = random.choice(['GET', 'POST'])
    status = random.choice([200, 200, 200, 404, 500])
    duration = random.uniform(0.01, 0.5)
    
    REQUEST_COUNT.labels(method=method, status=status).inc()
    REQUEST_DURATION.observe(duration)
    
    time.sleep(1)
EOF

# Run the app and expose metrics
pip install prometheus_client
python metrics-app.py &
start_http_server(8000)

# Check metrics endpoint
curl http://localhost:8000/metrics

# Configure Prometheus to scrape the app
# (Add the pod annotation: prometheus.io/scrape: "true")
```

---

## 🧠 Common Questions

**Q: What's the difference between Prometheus and Grafana?**
A: Prometheus collects and stores metrics. Grafana visualizes them. Prometheus has basic alerting, Grafana has advanced dashboards. You need both for a complete monitoring solution.

**Q: How long does Prometheus store data?**
A: By default, 15 days. You can increase this with `--storage.tsdb.retention.time=30d`, but for long-term storage, use Thanos or Cortex.

**Q: How do I monitor Kubernetes itself?**
A: Use kube-state-metrics for K8s objects (deployments, pods, services) and kubelet/cAdvisor for node/pod resource usage. The Kubernetes cluster dashboard (Grafana ID 6417) provides a good overview.

**Q: Should I use EFK or Loki for logging?**
A: EFK (Elasticsearch, Fluentd, Kibana) is more feature-rich but heavier. Loki is lighter, integrates well with Grafana, and is designed specifically for Kubernetes. For small/medium clusters, use Loki. For large-scale, use EFK.

**Q: How do I set up alerts for production?**
A: Start with critical alerts (pod not ready, high error rate, node down). Use Alertmanager to route alerts to Slack/PagerDuty. Set appropriate thresholds and `for` durations to avoid alert fatigue.

**Q: Can I monitor containers without Prometheus?**
A: Yes, Docker provides `docker stats` and Kubernetes provides `kubectl top pods/nodes`. But for historical data, alerting, and dashboards, you need a proper monitoring stack.

---

## ✅ Day 2 Summary

**What you learned today:**

- ✅ Why monitoring is essential for production systems
- ✅ Kubernetes monitoring stack (Prometheus, Grafana, EFK/Loki)
- ✅ Prometheus architecture and installation
- ✅ PromQL query language (rate, sum, avg, histogram_quantile)
- ✅ Grafana dashboards (pre-built and custom)
- ✅ Exposing application metrics in Prometheus format
- ✅ Logging with EFK stack and Loki
- ✅ Alertmanager for routing alerts (Slack, PagerDuty)
- ✅ Alert rules for common issues (restarts, CPU, disk, errors)

**Coming up tomorrow:** Scaling — horizontal and vertical pod autoscaling, cluster autoscaling, and best practices.

---

*💡 Tip: Set up monitoring from day one, not after things break. Start with basic metrics (CPU, memory, requests, errors), add application-specific metrics later. Use pre-built Grafana dashboards before creating custom ones. Alert on symptoms (high error rate, slow responses), not causes (high CPU, low memory).*