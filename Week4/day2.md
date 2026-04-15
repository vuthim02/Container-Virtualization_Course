# Week 4 • Day 2 • Intermediate
# Service Discovery

## Progress Checklist
- [x] Day 1 — Network Basics
- [ ] Day 2 — Service Discovery
- [ ] Day 3 — Load Balancing
- [ ] Day 4 — Container Security

---

← [Day 1](day1.md) | [Roadmap](../roadmap.md) | [Day 3 →](day3.md)

---

## 1. What is Service Discovery?

Service discovery is the mechanism by which containers find and communicate with each other **by name** rather than by IP address.

### The Problem Without Service Discovery

```
Traditional infrastructure:
┌──────────┐    ┌──────────┐
│  Web App │───▶│ Database │
│          │    │ 10.0.1.5 │  ← Hardcoded IP
└──────────┘    └──────────┘

Problem: If the database IP changes (restart, move, scale),
         the web app's configuration must be updated.
```

### The Docker Solution: DNS-Based Service Discovery

```
Docker with service discovery:
┌──────────┐    ┌──────────┐
│  Web App │───▶│ Database │
│          │    │  "db"    │  ← Resolved by DNS
│          │    │ → 172.18.0.3 (dynamic IP)
└──────────┘    └──────────┘

Benefit: The web app always reaches "db",
         regardless of what IP it currently has.
```

---

## 2. Docker's Embedded DNS Server

Every container on a user-defined network uses Docker's embedded DNS server at `127.0.0.11`.

### How DNS Resolution Works

```
┌──────────────────────────────────────────────────────┐
│                    Container A                       │
│                                                      │
│  Application: "I need to reach 'database'"           │
│         │                                            │
│         ▼                                            │
│  /etc/resolv.conf:                                   │
│    nameserver 127.0.0.11                             │
│    options ndots:0                                   │
│         │                                            │
│         ▼                                            │
│  ┌─────────────────┐                                 │
│  │ DNS Client      │                                 │
│  │ (glibc/musl)    │                                 │
│  └────────┬────────┘                                 │
└───────────┼──────────────────────────────────────────┘
            │ DNS Query: "Where is 'database'?"
            ▼
┌───────────────────────────────────────────────────────┐
│              Docker DNS Server (127.0.0.11)           │
│                                                       │
│  Looks up 'database' in the network's registry...    │
│  Found: 172.18.0.3                                    │
│                                                       │
│  Returns: 172.18.0.3                                 │
└───────────────┬───────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────┐
│                    Container A                        │
│                                                       │
│  "database is at 172.18.0.3, connecting..."          │
│  TCP connection → 172.18.0.3:5432                    │
└───────────────────────────────────────────────────────┘
```

### What Docker DNS Resolves

| Name Type | Example | Resolves To |
|-----------|---------|-------------|
| **Service name** | `database`, `api`, `web` | Container(s) with that service name |
| **Container name** | `my-container-name` | Specific container |
| **Network alias** | `db.internal` | Container(s) with that alias |
| **Container hostname** | `a1b2c3d4e5f6` (short ID) | Specific container |
| **Container ID** | `a1b2c3d4e5f6...` (full ID) | Specific container |

### DNS vs /etc/hosts

```bash
# Docker does NOT use /etc/hosts for service discovery
docker exec mycontainer cat /etc/hosts
# 127.0.0.1    localhost
# 172.18.0.2   a1b2c3d4e5f6    ← Only the container's own hostname
# 172.18.0.3   database         ← Other containers are NOT here

# Service discovery happens through the DNS server (127.0.0.11)
# NOT through /etc/hosts entries
```

**Why DNS instead of /etc/hosts?**
- Containers can start/stop/dynamically change IPs
- /etc/hosts would need constant updating
- DNS can resolve names on-the-fly
- DNS supports round-robin for multiple containers with the same name

---

## 3. DNS Resolution in Practice

### Testing DNS Between Containers

```bash
# Create a network
docker network create service-discovery-net

# Run a database container
docker run -d --network service-discovery-net \
  --name database \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine

# Run a web container
docker run -d --network service-discovery-net \
  --name web \
  nginx

# Test DNS from the web container
docker exec web nslookup database
# Server:    127.0.0.11
# Address:   127.0.0.11:53
# Name: database
# Address: 172.20.0.2    ← Resolved!

# Test with ping
docker exec web ping -c 2 database
# PING database (172.20.0.2): 56 data bytes
# 64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.089 ms
# 64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.067 ms

# Test with curl (HTTP request)
docker exec web curl -s http://database:5432
# (PostgreSQL will reject HTTP, but the connection attempt shows DNS works)

# Test reverse lookup (IP → name)
docker exec web nslookup 172.20.0.2
# 2.0.20.172.in-addr.arpa    name = database.service-discovery-net
```

### DNS Caching

Docker DNS has a small TTL (Time To Live). This means:

```bash
# Container starts at IP X
docker exec web nslookup database  → 172.20.0.2

# Container restarts, gets a new IP
docker rm -f database
docker run -d --network service-discovery-net --name database postgres:16-alpine
docker exec web nslookup database  → 172.20.0.3  ← New IP!

# DNS updates automatically — no configuration change needed
```

### DNS Search Domains

```bash
# Inside a container on network 'mynet':
nslookup web          → Resolves (short name)
nslookup web.mynet    → Also resolves (fully qualified)
nslookup web.othernet → Fails (wrong network)
```

Docker automatically adds the network name as a search domain.

---

## 4. Load Balancing via DNS (Round-Robin)

When multiple containers share the same service name, Docker DNS returns all their IPs in round-robin fashion.

```bash
# Create a network
docker network create lb-net

# Run three containers with the same service name
docker run -d --network lb-net --name api-1 alpine sleep infinity
docker run -d --network lb-net --name api-2 alpine sleep infinity
docker run -d --network lb-net --name api-3 alpine sleep infinity

# DNS lookup returns ALL IPs
docker exec api-1 nslookup api-1
# Address: 172.21.0.2

# But Docker DNS also resolves shared service names:
# If we used a compose service with replicas:
docker compose up -d --scale api=3

# DNS for 'api' returns all 3 IPs:
docker exec web nslookup api
# Name: api
# Address: 172.21.0.2   ← api-1
# Name: api
# Address: 172.21.0.3   ← api-2
# Name: api
# Address: 172.21.0.4   ← api-3
```

**Round-robin behavior:**
- First query: returns [IP1, IP2, IP3]
- Second query: returns [IP2, IP3, IP1]
- Third query: returns [IP3, IP1, IP2]

**Important:** This is DNS-level round-robin. It does NOT provide health checking or connection-based load balancing. For production load balancing, use a proper load balancer (covered Day 3).

---

## 5. Network Aliases

Give containers additional names for DNS resolution:

### Using `docker network connect`

```bash
docker network create mynet
docker run -d --network mynet --name database postgres:16-alpine

# Add aliases to the container
docker network connect --alias db-primary --alias db-master mynet database

# All these resolve to the same container:
docker exec <another-container> nslookup database    # Works
docker exec <another-container> nslookup db-primary   # Works
docker exec <another-container> nslookup db-master    # Works
```

### Using Compose

```yaml
services:
  database:
    image: postgres:16-alpine
    networks:
      app-network:
        aliases:
          - db-primary
          - db-master
          - postgres-server

  api:
    build: .
    networks:
      - app-network

networks:
  app-network:
```

```bash
# From the api container, all these work:
curl http://database:5432
curl http://db-primary:5432
curl http://db-master:5432
curl http://postgres-server:5432
```

**Use case:** Legacy applications that expect specific hostnames, or providing migration paths when renaming services.

---

## 6. External DNS Servers

Containers can also resolve external DNS (like `google.com`) through Docker's DNS forwarding.

```bash
# Inside a container:
nslookup google.com
# Server:    127.0.0.11
# Address:   127.0.0.11:53
#
# Non-authoritative answer:
# Name: google.com
# Address: 142.250.80.46

# Docker's DNS server:
# 1. Checks if the name is a Docker service (local lookup)
# 2. If not found, forwards to the host's DNS server
# 3. Returns the external DNS result
```

### Configuring DNS Servers

```bash
# Use specific DNS servers for all containers
docker daemon --dns 8.8.8.8 --dns 8.8.4.4

# Or in /etc/docker/daemon.json:
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}

# Override DNS for a specific container
docker run --dns 1.1.1.1 alpine nslookup google.com
```

---

## 7. Service Discovery with Docker Compose

Docker Compose makes service discovery even easier because it automatically:

1. Creates a network for the project
2. Registers all service names in DNS
3. Handles container naming

```yaml
services:
  web:
    image: nginx
  api:
    image: node:20-alpine
  db:
    image: postgres:16-alpine
  redis:
    image: redis:7-alpine
```

```bash
docker compose up -d

# From any container, all service names resolve:
docker compose exec web ping api     → Works
docker compose exec web ping db      → Works
docker compose exec web ping redis   → Works
docker compose exec api ping db      → Works
docker compose exec db ping redis    → Works

# Every service can reach every other service by name
# No manual network creation or DNS configuration needed
```

### DNS with Scaled Services

```yaml
services:
  api:
    build: .
    ports:
      - "3000"    # No host port, just internal
```

```bash
# Start 3 replicas
docker compose up -d --scale api=3

# DNS resolution:
docker compose exec web nslookup api
# Name: api
# Address: 172.20.0.3   ← api-1
# Name: api
# Address: 172.20.0.4   ← api-2
# Name: api
# Address: 172.20.0.5   ← api-3

# Connecting to 'api' will reach one of the replicas
# (round-robin at the DNS level)
```

---

## 8. Debugging Service Discovery Issues

### Common Problems

**Problem: "Could not resolve host"**
```bash
# Check if both containers are on the same network
docker network inspect <network-name>

# Verify the container name is correct
docker ps --format '{{.Names}}'

# Test DNS from inside the container
docker exec <container> nslookup <target-service>

# Check the DNS server configuration
docker exec <container> cat /etc/resolv.conf
# Should show: nameserver 127.0.0.11
```

**Problem: "Connection refused" (DNS works, connection doesn't)**
```bash
# DNS resolves correctly, but service isn't listening
docker exec <container> nslookup <target>  # Returns IP
docker exec <container> ping <target>       # Works
docker exec <container> curl <target>:port  # Connection refused

# Check if the target service is actually running:
docker exec <target> ps aux
docker logs <target>
```

**Problem: DNS returning wrong IP**
```bash
# Stale DNS cache — force a fresh lookup
docker exec <container> nslookup <target>

# If still wrong, check if an old container with the same name exists:
docker ps -a --filter name=<target>

# Remove stale containers:
docker rm <old-container>
```

### Useful Debugging Commands

```bash
# See all containers on a network
docker network inspect <network> --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'

# Check DNS resolution from inside a container
docker exec <container> getent hosts <service-name>

# Use the +short flag for clean output
docker exec <container> nslookup +short <service-name>

# Trace the full DNS resolution path
docker exec <container> nslookup -debug <service-name>
```

---

## 📝 Hands-On Exercises

### Exercise 1: Basic Service Discovery

```bash
# Create a network
docker network create sd-test

# Run two containers
docker run -d --network sd-test --name web nginx
docker run -d --network sd-test --name api alpine sleep infinity

# Test DNS resolution from api to web
docker exec api nslookup web
# Should return the IP of the web container

# Test DNS resolution from web to api
docker exec web nslookup api
# Should return the IP of the api container

# Test connectivity
docker exec web ping -c 2 api
docker exec api ping -c 2 web

# Clean up
docker rm -f web api
docker network rm sd-test
```

### Exercise 2: Multiple Aliases

```bash
docker network create alias-test

# Run a container with aliases
docker run -d --network alias-test --name primary alpine sleep infinity
docker network connect --alias secondary --alias tertiary alias-test primary

# Run another container
docker run -d --network alias-test --name client alpine sleep infinity

# From the client, all names should resolve to the same container:
docker exec client nslookup primary
docker exec client nslookup secondary
docker exec client nslookup tertiary

# All should return the same IP

# Verify with ping:
docker exec client ping -c 1 primary
docker exec client ping -c 1 secondary
docker exec client ping -c 1 tertiary

# Clean up
docker rm -f primary client
docker network rm alias-test
```

### Exercise 3: Dynamic IP Changes

```bash
docker network create dynamic-test

# Run a container
docker run -d --network dynamic-test --name database \
  -e POSTGRES_PASSWORD=secret postgres:16-alpine

# Note its IP
docker exec database ip addr show eth0
# Or: docker inspect database --format '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

# From another container, resolve it
docker run --rm --network dynamic-test alpine nslookup database

# Remove and recreate the container
docker rm -f database
docker run -d --network dynamic-test --name database \
  -e POSTGRES_PASSWORD=secret postgres:16-alpine

# The IP has likely changed, but DNS still resolves correctly:
docker run --rm --network dynamic-test alpine nslookup database

# The name 'database' always reaches the current container,
# regardless of its IP address

# Clean up
docker rm -f database
docker network rm dynamic-test
```

### Exercise 4: Compose Service Discovery

```bash
mkdir ~/compose-sd && cd ~/compose-sd

cat > docker-compose.yml << 'EOF'
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"

  api:
    image: alpine
    command: sleep infinity

  db:
    image: alpine
    command: sleep infinity

  redis:
    image: alpine
    command: sleep infinity
EOF

docker compose up -d

# From web, resolve all other services:
docker compose exec web nslookup api
docker compose exec web nslookup db
docker compose exec web nslookup redis

# From db, reach web and api:
docker compose exec db nslookup web
docker compose exec db nslookup api

# All services can reach all other services by name
docker compose exec api ping web
docker compose exec redis ping db

# Clean up
docker compose down
```

---

## 🧠 Common Questions

**Q: Does DNS work across different Docker networks?**
A: No. DNS resolution is scoped to the network. A container on `network-a` cannot resolve a service name from `network-b` unless it's connected to both networks.

**Q: What's the difference between container name and hostname?**
A: The container name is what you set with `--name` and what Docker DNS resolves. The hostname (inside the container) defaults to the container's short ID. They can be different.

```bash
docker run -d --name myapp --hostname custom-host nginx
# DNS resolves: myapp → IP
# Inside the container: hostname = custom-host
```

**Q: Can I use custom DNS servers inside containers?**
A: Yes, set them with `--dns` on `docker run` or in `daemon.json`. Docker's embedded DNS (127.0.0.11) will still handle service name resolution and forward external queries to your custom DNS servers.

**Q: Does Docker DNS support SRV records?**
A: No. Docker DNS only supports A/AAAA records (IP address resolution). For SRV records or more advanced service discovery, use Consul, etcd, or a service mesh.

**Q: Why does `nslookup` sometimes return multiple IPs?**
A: When multiple containers share the same service name (e.g., scaled Compose services), Docker DNS returns all IPs in round-robin order for load distribution.

---

## ✅ Day 2 Summary

**What you learned today:**

- ✅ What service discovery is and why it matters
- ✅ Docker's embedded DNS server (127.0.0.11)
- ✅ How DNS resolution works inside containers
- ✅ What names Docker DNS resolves (services, containers, aliases, IDs)
- ✅ Round-robin DNS for basic load distribution
- ✅ Network aliases for alternative service names
- ✅ External DNS forwarding and configuration
- ✅ Service discovery with Docker Compose
- ✅ Debugging DNS resolution issues

**Coming up tomorrow:** Load balancing — distributing traffic across multiple container instances using nginx and Docker's built-in mechanisms.

---

*💡 Tip: Service discovery is the foundation of dynamic infrastructure. Embrace it — never hardcode IP addresses in containerized applications. Always use service names.*