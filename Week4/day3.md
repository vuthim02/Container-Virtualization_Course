# Week 4 • Day 3 • Intermediate
# Load Balancing

## Progress Checklist
- [x] Day 1 — Network Basics
- [x] Day 2 — Service Discovery
- [ ] Day 3 — Load Balancing
- [ ] Day 4 — Container Security

---

← [Day 2](day2.md) | [Roadmap](../roadmap.md) | [Day 4 →](day4.md)

---

## 1. Why Load Balancing?

When your application grows, one container isn't enough:

```
Single container:
User ─────▶ [ Container ]
            ↓
            Single point of failure
            Limited throughput
            No horizontal scaling

Multiple containers with load balancer:
            ┌─────────────┐
User ─────▶│     LB      │
            │             │
            ├─────┬───────┤
            ▼     ▼       ▼
         [ C1 ] [ C2 ]  [ C3 ]
            │     │       │
            └─────┴───────┘
                  ↓
         High availability
         Horizontal scaling
         Better throughput
```

Load balancing distributes incoming traffic across multiple backend instances.

---

## 2. DNS-Based Load Balancing (Round-Robin)

The simplest form of load balancing is DNS round-robin, which we covered on Day 2.

```yaml
services:
  api:
    build: .
    deploy:
      replicas: 3

  web:
    image: nginx
```

```
DNS query for 'api':
  Query 1 → [172.20.0.2, 172.20.0.3, 172.20.0.4]
  Query 2 → [172.20.0.3, 172.20.0.4, 172.20.0.2]
  Query 3 → [172.20.0.4, 172.20.0.2, 172.20.0.3]

Client picks the first IP → traffic distributed
```

**Limitations of DNS-based LB:**
- No health checking (dead containers still get traffic)
- Client-side DNS caching breaks round-robin
- No connection-aware balancing
- No weighted distribution
- No session persistence (sticky sessions)

For production, you need a **real load balancer**.

---

## 3. nginx as a Load Balancer

nginx is the most popular software load balancer for container environments.

### Basic Setup

```
User → Port 80 → [ nginx LB ] → Port 3000 → api-1
                            → Port 3001 → api-2
                            → Port 3002 → api-3
```

### nginx Configuration

`nginx.conf`:
```nginx
upstream backend {
    server api-1:3000;
    server api-2:3000;
    server api-3:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**How it works:**
- nginx listens on port 80
- All requests to `/` are forwarded to the `backend` upstream group
- nginx round-robins between the three backend servers

### Docker Compose Setup

```yaml
services:
  lb:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api

  api:
    build: .
    deploy:
      replicas: 3
```

**Problem:** In Docker, scaled containers get names like `app-api-1`, `app-api-2`. Hardcoding them in nginx.conf doesn't scale well.

### Solution: DNS-based upstream

```nginx
upstream backend {
    # Use Docker DNS to resolve the service name
    # This returns all container IPs for the 'api' service
    server api:3000;
    
    # Enable DNS resolution via Docker's embedded DNS
    resolver 127.0.0.11 valid=10s;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
    }
}
```

With the `resolver` directive, nginx re-resolves DNS names periodically, adapting to container restarts and IP changes.

---

## 4. nginx Load Balancing Methods

### Round-Robin (Default)

```nginx
upstream backend {
    server api-1:3000;
    server api-2:3000;
    server api-3:3000;
}
# Requests distributed evenly in order: 1, 2, 3, 1, 2, 3...
```

### Least Connections

```nginx
upstream backend {
    least_conn;
    server api-1:3000;
    server api-2:3000;
    server api-3:3000;
}
# New requests go to the server with fewest active connections
# Best for: long-running requests (WebSockets, streaming)
```

### IP Hash (Sticky Sessions)

```nginx
upstream backend {
    ip_hash;
    server api-1:3000;
    server api-2:3000;
    server api-3:3000;
}
# Same client IP always reaches the same backend
# Best for: session-based applications
```

### Weighted Distribution

```nginx
upstream backend {
    server api-1:3000 weight=3;  # 60% of traffic
    server api-2:3000 weight=2;  # 40% of traffic
}
# Useful when servers have different capacities
```

### Health Checking (nginx Plus only — open-source alternative below)

```nginx
upstream backend {
    server api-1:3000 max_fails=3 fail_timeout=30s;
    server api-2:3000 max_fails=3 fail_timeout=30s;
    server api-3:3000 max_fails=3 fail_timeout=30s backup;
}
# After 3 failed attempts, server is marked unavailable for 30s
# api-3 is a backup server — only used when primary servers are down
```

---

## 5. Complete nginx LB Example

### Project Structure

```
nginx-lb/
├── docker-compose.yml
├── nginx.conf
└── api/
    ├── Dockerfile
    └── server.js
```

### `api/server.js`

```javascript
const http = require('http');
const os = require('os');

const PORT = 3000;

const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end(`Response from ${os.hostname()}\n`);
});

server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}, hostname: ${os.hostname()}`);
});
```

### `api/Dockerfile`

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
```

### `nginx.conf`

```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend {
        # nginx will resolve 'api' via Docker DNS
        server api:3000;
        resolver 127.0.0.11 valid=10s ipv6=off;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # Health check endpoint for the LB itself
        location /lb-health {
            return 200 "LB OK\n";
            add_header Content-Type text/plain;
        }
    }
}
```

### `docker-compose.yml`

```yaml
services:
  lb:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api

  api:
    build: ./api
    deploy:
      replicas: 3
```

### Running and Testing

```bash
cd nginx-lb

docker compose up -d

# Check all containers
docker compose ps
# lb, api-1, api-2, api-3 all running

# Make several requests — observe different backends
curl http://localhost
# Response from nginx-lb-api-1

curl http://localhost
# Response from nginx-lb-api-2

curl http://localhost
# Response from nginx-lb-api-3

curl http://localhost
# Response from nginx-lb-api-1  (round-robin cycles back)

# Check nginx logs to see the distribution
docker compose logs lb | grep upstream

# Scale up to 5 replicas
docker compose up -d --scale api=5

# Now traffic is distributed across 5 backends
for i in $(seq 1 10); do curl -s http://localhost; done

# Clean up
docker compose down
```

---

## 6. Docker's Built-in Routing Mesh (Swarm Mode)

Docker Swarm provides built-in load balancing when you use the `ingress` network.

```bash
# Initialize Swarm
docker swarm init

# Create a service with 3 replicas
docker service create \
  --name myapp \
  --replicas 3 \
  --publish 8080:3000 \
  myapp-image

# Docker automatically load balances across all 3 replicas
# Any request to port 8080 on ANY node reaches a healthy replica
```

This is beyond the scope of standalone Docker. Covered in Swarm/Kubernetes courses.

---

## 7. HAProxy as Load Balancer

HAProxy is another popular load balancer, known for high performance.

### HAProxy Configuration

`haproxy.cfg`:
```
global
    log stdout format raw local0
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    log global
    option httplog

frontend http_front
    bind *:80
    default_backend api_backends

backend api_backends
    balance roundrobin
    option httpchk GET /health
    server api-1 api-1:3000 check
    server api-2 api-2:3000 check
    server api-3 api-3:3000 check
```

### HAProxy Compose

```yaml
services:
  lb:
    image: haproxy:2.9-alpine
    ports:
      - "80:80"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - api

  api:
    build: ./api
    deploy:
      replicas: 3
```

---

## 8. Traefik — The Cloud-Native Load Balancer

Traefik is designed specifically for container environments. It **automatically discovers services** and configures routes.

```yaml
services:
  traefik:
    image: traefik:v2.10
    ports:
      - "80:80"
      - "8080:8080"   # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--api.dashboard=true"

  api:
    build: ./api
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=PathPrefix(`/`)"
      - "traefik.http.services.api.loadbalancer.server.port=3000"
    deploy:
      replicas: 3
```

**Key advantage:** No separate load balancer config file needed. Traefik reads Docker labels and configures itself automatically. When containers start/stop, Traefik updates its routes instantly.

---

## 9. Comparing Load Balancing Options

| Solution | Auto-Discovery | Health Checks | Config Complexity | Best For |
|----------|---------------|---------------|-------------------|----------|
| **DNS Round-Robin** | ✅ Yes | ❌ No | None | Simple setups |
| **nginx** | ⚠️ Partial | ⚠️ Basic | Medium | Most use cases |
| **HAProxy** | ⚠️ Partial | ✅ Yes | Medium-High | High-performance |
| **Traefik** | ✅ Yes | ✅ Yes | Low (auto-config) | Dynamic environments |
| **Docker Swarm** | ✅ Yes | ✅ Yes | Low | Swarm clusters |
| **Kubernetes** | ✅ Yes | ✅ Yes | Medium-High | K8s clusters |

---

## 10. Health Checks for Load Balancers

A good load balancer only sends traffic to **healthy** backends.

### nginx Passive Health Checks (Open Source)

```nginx
upstream backend {
    server api-1:3000 max_fails=3 fail_timeout=30s;
    server api-2:3000 max_fails=3 fail_timeout=30s;
    server api-3:3000 max_fails=3 fail_timeout=30s;
}
# If a server fails 3 times, it's removed from the pool for 30 seconds
```

### HAProxy Active Health Checks

```
backend api_backends
    option httpchk GET /health
    server api-1 api-1:3000 check inter 5s fall 3 rise 2
    server api-2 api-2:3000 check inter 5s fall 3 rise 2
# Checks /health every 5s. After 3 failures, marks down. After 2 successes, marks up.
```

### Application Health Endpoint

Your application should expose a health endpoint:

```javascript
// Node.js
app.get('/health', (req, res) => {
    const health = {
        status: 'ok',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
        memory: process.memoryUsage(),
    };
    
    // Check database connection
    try {
        db.query('SELECT 1');
        health.database = 'connected';
    } catch (err) {
        health.status = 'degraded';
        health.database = 'disconnected';
    }
    
    const statusCode = health.status === 'ok' ? 200 : 503;
    res.status(statusCode).json(health);
});
```

```python
# Flask
@app.route('/health')
def health():
    health_data = {
        'status': 'ok',
        'timestamp': datetime.utcnow().isoformat(),
        'uptime': time.time() - start_time,
    }
    
    try:
        db.execute('SELECT 1')
        health_data['database'] = 'connected'
    except:
        health_data['status'] = 'degraded'
        health_data['database'] = 'disconnected'
    
    status_code = 200 if health_data['status'] == 'ok' else 503
    return jsonify(health_data), status_code
```

---

## 📝 Hands-On Exercises

### Exercise: Set Up nginx Load Balancer

```bash
mkdir ~/nginx-lb-exercise && cd ~/nginx-lb-exercise
```

Create `api/app.py`:
```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import socket

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-Type', 'text/plain')
        self.end_headers()
        hostname = socket.gethostname()
        self.wfile.write(f"Response from {hostname}\n".encode())

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 3000), Handler)
    print(f"Running on port 3000, hostname: {socket.gethostname()}")
    server.serve_forever()
```

Create `api/Dockerfile`:
```dockerfile
FROM python:3.12-alpine
WORKDIR /app
COPY app.py .
EXPOSE 3000
CMD ["python", "app.py"]
```

Create `nginx.conf`:
```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server api:3000;
        resolver 127.0.0.11 valid=10s ipv6=off;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

Create `docker-compose.yml`:
```yaml
services:
  lb:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro

  api:
    build: ./api
    deploy:
      replicas: 3
```

```bash
# Start everything
docker compose up -d

# Test load balancing — make 10 requests
for i in $(seq 1 10); do
  curl -s http://localhost:8080
done

# You should see responses from different container hostnames

# Scale up to 5 replicas
docker compose up -d --scale api=5

# Make 10 more requests — now distributed across 5 backends
for i in $(seq 1 10); do
  curl -s http://localhost:8080
done

# Check nginx access logs to see the distribution
docker compose logs lb

# Clean up
docker compose down
```

---

## 🧠 Common Questions

**Q: Can nginx reload its config without downtime?**
A: Yes! `nginx -s reload` reloads the configuration gracefully. In Docker, you can also send a SIGHUP signal: `docker kill -s SIGHUP <nginx-container>`.

**Q: Does DNS round-robin work with HTTP keep-alive connections?**
A: No. With keep-alive, the client resolves DNS once and reuses the same connection. All subsequent requests go to the same backend until the connection closes. This is why proper load balancers (nginx, HAProxy) are preferred.

**Q: Can I do SSL/TLS termination at the load balancer?**
A: Yes! This is one of the main benefits. The LB handles HTTPS, and traffic to backends is HTTP (internal network).

```nginx
server {
    listen 443 ssl;
    ssl_certificate /certs/cert.pem;
    ssl_certificate_key /certs/key.pem;
    
    location / {
        proxy_pass http://backend;  # HTTP to backends
    }
}
```

**Q: What's the difference between Layer 4 and Layer 7 load balancing?**
A: Layer 4 (TCP/UDP) routes based on IP/port. Layer 7 (HTTP) routes based on URL paths, headers, cookies. nginx and HAProxy support both.

```nginx
# Layer 4 (TCP) — route by port
stream {
    upstream db_backend {
        server db-1:5432;
        server db-2:5432;
    }
    server {
        listen 5432;
        proxy_pass db_backend;
    }
}

# Layer 7 (HTTP) — route by URL path
http {
    server {
        location /api { proxy_pass http://api_backend; }
        location /web { proxy_pass http://web_backend; }
    }
}
```

---

## ✅ Day 3 Summary

**What you learned today:**

- ✅ Why load balancing is essential for scaling
- ✅ DNS round-robin and its limitations
- ✅ nginx as a load balancer (configuration, methods: round-robin, least_conn, ip_hash, weighted)
- ✅ Complete nginx LB example with Docker Compose
- ✅ HAProxy and Traefik as alternatives
- ✅ Health checks (passive and active)
- ✅ Application health endpoints
- ✅ Layer 4 vs Layer 7 load balancing

**Coming up tomorrow:** Container security — running containers safely, limiting privileges, scanning images, and managing secrets.

---

*💡 Tip: A load balancer is only as good as your health checks. Always implement a meaningful health endpoint in your application. The LB needs to know when a backend is struggling.*