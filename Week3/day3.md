# Week 3 • Day 3 • Intermediate
# Networking in Docker Compose

## Progress Checklist
- [x] Day 1 — Introduction to Compose
- [x] Day 2 — Multi-container Apps
- [ ] Day 3 — Networking in Compose
- [ ] Day 4 — Advanced Compose Patterns

---

← [Day 2](day2.md) | [Roadmap](../roadmap.md) | [Day 4 →](day4.md)

---

## 1. Why Networking Matters

In a multi-container application, services need to talk to each other. Understanding how Docker Compose handles networking is crucial for:

- Making services discover each other
- Isolating sensitive services (databases) from the outside world
- Debugging connection issues
- Designing secure architectures

---

## 2. The Default Network

When you run `docker compose up`, Docker Compose **automatically creates a network** for your project.

```yaml
# No networks section needed
services:
  web:
    image: nginx
    ports:
      - "80:80"

  api:
    image: node:20-alpine

  db:
    image: postgres:16-alpine
```

```bash
docker compose up -d

# Compose creates a network named: <project>_default
docker network ls
# NAME                      DRIVER
# flask-postgres_default    bridge    ← Created automatically
```

### Service Discovery

All services on the default network can reach each other by **service name**:

```yaml
services:
  web:
    image: nginx
  api:
    image: node:20-alpine
  db:
    image: postgres:16-alpine
```

```
From the web container:
  curl http://api:3000     → Reaches the api service
  curl http://db:5432      → Reaches the db service (TCP connection)

From the api container:
  fetch("http://web:80")   → Reaches the web service
  connect("db", 5432)      → Connects to the db service
```

### How DNS Resolution Works

Docker Compose runs an **embedded DNS server** (127.0.0.11) inside each container:

```
Container A wants to reach "db":
1. Container A asks DNS server at 127.0.0.11: "Where is 'db'?"
2. DNS server looks up the service name
3. DNS server returns: "db is at 172.20.0.3"
4. Container A connects to 172.20.0.3
```

```bash
# Inside any container, verify the DNS server:
cat /etc/resolv.conf
# nameserver 127.0.0.11   ← Docker's embedded DNS

# Test DNS resolution manually:
nslookup api
nslookup db
dig web
```

### Ports: Internal vs Exposed

**Critical distinction:**

| Port Type | What It Does | Example |
|-----------|-------------|---------|
| **Internal (no `ports:`)** | Service reachable by OTHER containers, NOT from host | `db` at port 5432 |
| **Exposed (`ports:`)** | Service reachable from BOTH host AND other containers | `web` at port 80 |

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"     # ← Host port 8080 → Container port 80

  api:
    image: node:20-alpine
    # No ports mapping!

  db:
    image: postgres:16-alpine
    # No ports mapping!
```

```
Access from HOST (your machine):
  http://localhost:8080    → web ✅
  http://localhost:3000    → api  ❌ (not exposed)
  localhost:5432           → db   ❌ (not exposed)

Access FROM web container:
  http://api:3000          → api  ✅
  db:5432                  → db   ✅

Access FROM api container:
  http://web:80            → web  ✅
  db:5432                  → db   ✅
```

**Best Practice:** Only expose ports for services that need to be accessed from the host. Databases and internal services should NOT have port mappings in production.

```yaml
# ✅ Good — database only reachable internally
  db:
    image: postgres:16-alpine
    # No ports: line

# ❌ Bad — database exposed to the whole machine
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"    # Only do this for local development/debugging!
```

---

## 3. Custom Networks

Sometimes the default network isn't enough. You might want to segment your services for security or organizational reasons.

### Example: Frontend/Backend Segmentation

```
┌──────────────────────────────────────────────────────────┐
│                    frontend network                      │
│  ┌─────────┐         ┌─────────┐                         │
│  │  nginx  │────────▶│  api    │                         │
│  │ (web)   │         │         │                         │
│  └─────────┘         └────┬────┘                         │
└───────────────────────────┼──────────────────────────────┘
                            │
┌───────────────────────────┼──────────────────────────────┐
│                    backend network                       │
│                            ▼                              │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────┐  │
│  │   api   │───▶│  PostgreSQL  │    │     Redis       │  │
│  │         │    │     (db)     │    │    (cache)      │  │
│  └─────────┘    └──────────────┘    └─────────────────┘  │
│                                                          │
│  nginx CANNOT reach db or redis directly                │
│  db and redis CANNOT reach nginx directly               │
└──────────────────────────────────────────────────────────┘
```

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - frontend

  api:
    build: .
    networks:
      - frontend
      - backend

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend

  redis:
    image: redis:7-alpine
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  pgdata:
```

**What this achieves:**

| From → To | web | api | db | redis |
|-----------|-----|-----|----| ----|
| **web** | — | ✅ (frontend) | ❌ | ❌ |
| **api** | ✅ (frontend) | — | ✅ (backend) | ✅ (backend) |
| **db** | ❌ | ✅ (backend) | — | ✅ (backend) |
| **redis** | ❌ | ✅ (backend) | ✅ (backend) | — |

The `api` service is the **only** service on both networks, acting as a gateway. The database and Redis are completely isolated from the nginx frontend.

### Testing Network Segmentation

```bash
docker compose up -d

# From web, reach api (should work — same frontend network)
docker compose exec web curl http://api:3000

# From web, reach db (should FAIL — different networks)
docker compose exec web ping db
# ping: bad address 'db'   ← DNS resolution fails!

# From api, reach db (should work — same backend network)
docker compose exec api ping db
# PING db (172.20.0.3): 56 data bytes   ← Works!

# From db, try to reach web (should FAIL)
docker compose exec db ping web
# ping: bad address 'web'   ← DNS resolution fails!
```

---

## 4. Network Configuration Options

### Network Drivers

```yaml
networks:
  # Default bridge network (most common)
  mybridge:
    driver: bridge

  # Overlay network (Docker Swarm, multi-host)
  myoverlay:
    driver: overlay

  # Host network (no isolation, shares host network stack)
  myhost:
    driver: host

  # None (no networking)
  mynone:
    driver: none
```

### Custom Subnets and Gateway

```yaml
networks:
  mynetwork:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1
```

**Why specify subnets?**
- Avoid IP conflicts with other Docker networks
- Set up specific firewall rules
- Consistent IP addressing across environments

### External Networks

Connect to a network created outside of Compose:

```yaml
networks:
  # This network is created/managed outside of this compose file
  monitoring:
    external: true
    name: infra_monitoring   # Actual network name on the host

services:
  web:
    image: nginx
    networks:
      - default
      - monitoring
```

```bash
# Create the external network manually first
docker network create infra_monitoring

# Then run compose
docker compose up -d
```

### Default Network Override

Change the default network settings:

```yaml
networks:
  default:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
```

---

## 5. Aliases and Network Names

Give a service an alternative name on a specific network:

```yaml
services:
  db:
    image: postgres:16-alpine
    networks:
      backend:
        aliases:
          - database
          - postgres
          - mydb.local

networks:
  backend:
```

```bash
# From another service on the backend network:
curl http://db:5432        # Works (service name)
curl http://database:5432  # Works (alias)
curl http://postgres:5432  # Works (alias)
curl http://mydb.local:5432 # Works (alias)
```

**Use case:** Legacy applications that expect a specific hostname.

---

## 6. Linking (Legacy — Avoid)

```yaml
services:
  web:
    image: nginx
    links:
      - api    # ← Legacy feature, don't use

  api:
    image: node:20-alpine
```

**Why avoid `links`?**
- Networks provide the same functionality natively
- Links are a legacy Docker feature
- They create implicit dependencies (replaced by `depends_on`)
- All services on the same network can already reach each other by name

**Only use `links` if:** You're maintaining very old Docker Compose files.

---

## 7. Port Publishing Options

```yaml
services:
  web:
    image: nginx
    ports:
      # Short syntax: "host:container"
      - "80:80"

      # With specific host IP
      - "127.0.0.1:8080:80"

      # Port range
      - "8080-8090:8080-8090"

      # UDP port
      - "53:53/udp"

      # Long syntax (more explicit)
      - target: 80
        published: 8080
        protocol: tcp
        mode: host
```

### Host Mode vs Ingress Mode

| Mode | Behavior | When to Use |
|------|----------|-------------|
| `host` | Port bound on the host machine directly | Single-host setups |
| `ingress` | Port managed by Docker's routing mesh | Swarm mode, multi-host |

For standalone Compose, always use `host` mode (the default).

---

## 8. Debugging Network Issues

### Common Problems and Solutions

**Problem: Service not reachable**
```bash
# Check if services are on the same network
docker compose ps
docker network inspect <project>_default

# Check DNS resolution inside a container
docker compose exec web nslookup api
docker compose exec web ping api
```

**Problem: Port already in use**
```bash
# Error: "bind: address already in use"
# Something is already listening on that port

# Find the conflicting process:
sudo lsof -i :80
sudo ss -tlnp | grep 80

# Fix: Use a different port
docker compose down
# Change ports in docker-compose.yml
docker compose up -d
```

**Problem: Connection refused**
```bash
# From one container to another:
docker compose exec web curl -v http://api:3000
# Look for:
#   "Could not resolve host" → DNS/network issue (not on same network)
#   "Connection refused"     → Service not running/listening on that port
#   "Connection timed out"   → Firewall or service crashed

# Check if the target service is actually running:
docker compose ps api
docker compose logs api
```

### Network Inspection

```bash
# List all Docker networks
docker network ls

# Inspect a specific network
docker network inspect <project>_default

# See which containers are on a network
docker network inspect <project>_default --format '{{range .Containers}}{{.Name}} {{end}}'
```

---

## 📝 Hands-On Exercises

### Exercise 1: Default Network Communication

```bash
mkdir ~/compose-networking && cd ~/compose-networking

cat > docker-compose.yml << 'EOF'
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"

  api:
    image: nginx:alpine
    ports:
      - "8081:80"

  db:
    image: alpine
    command: sleep infinity
    # No ports exposed — internal only
EOF

docker compose up -d

# Test default network communication
docker compose exec web ping api     # Should work
docker compose exec web ping db      # Should work
docker compose exec api ping web     # Should work
docker compose exec db ping web      # Should work

# From your host machine:
curl http://localhost:8080           # web — works
curl http://localhost:8081           # api — works
# db — no port exposed, unreachable from host

docker compose down
```

### Exercise 2: Custom Networks with Segmentation

```bash
cat > docker-compose.yml << 'EOF'
services:
  frontend:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - frontend-net

  backend:
    image: nginx:alpine
    networks:
      - frontend-net
      - backend-net

  database:
    image: alpine
    command: sleep infinity
    networks:
      - backend-net

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge
EOF

docker compose up -d

# Test connectivity:
# Frontend → Backend (should work)
docker compose exec frontend ping backend

# Frontend → Database (should FAIL)
docker compose exec frontend ping database

# Backend → Database (should work)
docker compose exec backend ping database

# Database → Frontend (should FAIL)
docker compose exec database ping frontend

docker compose down
```

### Exercise 3: Network Aliases

```bash
cat > docker-compose.yml << 'EOF'
services:
  db:
    image: alpine
    command: sleep infinity
    networks:
      appnet:
        aliases:
          - mydb.internal
          - postgres-primary

  app:
    image: alpine
    command: sleep infinity
    networks:
      - appnet

networks:
  appnet:
    driver: bridge
EOF

docker compose up -d

# All these should resolve to the same container:
docker compose exec app ping db
docker compose exec app ping mydb.internal
docker compose exec app ping postgres-primary

docker compose down
```

---

## 🧠 Common Questions

**Q: Can two separate compose projects communicate?**
A: Not by default — they get separate networks. But you can connect them by putting services on a shared external network.

```yaml
# Project A
networks:
  shared:
    external: true
    name: global-shared

# Project B
networks:
  shared:
    external: true
    name: global-shared
```

**Q: Does Docker Compose support host networking?**
A: Yes, but only on Linux. Set `network_mode: host` on a service. The service shares the host's network namespace (no isolation).

```yaml
services:
  monitoring:
    image: prom/prometheus
    network_mode: host   # Uses host ports directly
```

**Q: What's the difference between `networks:` and `ports:`?**
A: `networks:` controls which internal services can reach each other. `ports:` controls which internal services are accessible from the host machine. They're independent concepts.

**Q: Can I assign static IPs to containers?**
A: Yes, but you need to define a subnet in the network config:

```yaml
networks:
  mynet:
    ipam:
      config:
        - subnet: 172.20.0.0/16

services:
  db:
    image: postgres
    networks:
      mynet:
        ipv4_address: 172.20.0.10
```

**Best practice:** Avoid static IPs. Use service names and DNS resolution instead.

---

## ✅ Day 3 Summary

**What you learned today:**

- ✅ How Compose's default network works (automatic creation, DNS-based service discovery)
- ✅ Service name resolution (127.0.0.11 DNS server)
- ✅ Internal vs exposed ports
- ✅ Custom networks for service segmentation
- ✅ Frontend/backend network isolation patterns
- ✅ Network configuration (drivers, subnets, gateways, external networks)
- ✅ Network aliases for alternative hostnames
- ✅ Debugging network issues

**Coming up tomorrow:** Advanced Compose patterns — environment variables, volumes, healthchecks, profiles, and production-ready configurations.

---

*💡 Tip: Think about your network topology before writing your compose file. Which services need to talk to which? Which should be isolated? Design your networks with security in mind.*