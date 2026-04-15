# Week 4 • Day 1 • Intermediate
# Network Basics

## Progress Checklist
- [x] Day 1 — Network Basics
- [ ] Day 2 — Service Discovery
- [ ] Day 3 — Load Balancing
- [ ] Day 4 — Container Security

---

← [Roadmap](../roadmap.md) | [Day 2 →](day2.md)

---

## 1. How Docker Networking Works

Every container has its own **network namespace** — an isolated network stack with its own:

- Network interfaces (eth0, lo)
- IP addresses
- Routing tables
- Firewall rules
- Port space

```
Container A                  Container B
┌───────────────────┐       ┌───────────────────┐
│  eth0: 172.17.0.2 │       │  eth0: 172.17.0.3 │
│  lo: 127.0.0.1    │       │  lo: 127.0.0.1    │
│  Routes...        │       │  Routes...        │
│                   │       │                   │
│  Process: nginx   │       │  Process: python  │
│  Listening: :80   │       │  Listening: :8080 │
└────────┬──────────┘       └────────┬──────────┘
         │                           │
         └───────────┬───────────────┘
                     │
              ┌──────▼──────┐
              │  Docker     │
              │  Bridge     │
              │  (docker0)  │
              │ 172.17.0.1  │
              └──────┬──────┘
                     │
              ┌──────▼──────┐
              │   Host      │
              │  Network    │
              │             │
              │  eth0       │
              │  (real NIC) │
              └─────────────┘
```

When Container A talks to Container B:
```
Container A (172.17.0.2) → Docker Bridge → Container B (172.17.0.3)
```

---

## 2. Docker Network Drivers

Docker provides several network drivers for different use cases.

### 2.1 Bridge Network (Default)

The bridge driver creates an internal private network on the host. Containers attached to the same bridge network can communicate with each other.

```bash
# Docker creates a default bridge network (docker0) automatically
docker network ls
# NAME      DRIVER
# bridge    bridge     ← The default network
# host      host
# none      null

# Inspect the default bridge
docker network inspect bridge
# {
#   "Name": "bridge",
#   "Driver": "bridge",
#   "IPAM": {
#     "Config": [{"Subnet": "172.17.0.0/16", "Gateway": "172.17.0.1"}]
#   }
# }
```

**Default bridge behavior:**
- Containers on the default bridge can communicate by **IP address** only
- Service name resolution does NOT work on the default bridge
- Containers must be linked manually or use `--link` (legacy)

```bash
# Run two containers on the default bridge
docker run -d --name container-a nginx
docker run -d --name container-b nginx

# They CAN reach each other by IP:
docker exec container-a ping 172.17.0.3   # Works

# They CANNOT reach each other by name:
docker exec container-a ping container-b  # Fails! (on default bridge)
```

### Custom Bridge Networks

Create your own bridge network:

```bash
# Create a custom bridge network
docker network create mynet

# Verify
docker network ls
# NAME      DRIVER
# bridge    bridge
# mynet     bridge      ← Our new network
# host      host
# none      null

# Inspect it
docker network inspect mynet
# {
#   "Name": "mynet",
#   "Driver": "bridge",
#   "IPAM": {
#     "Config": [{"Subnet": "172.18.0.0/16", "Gateway": "172.18.0.1"}]
#   }
# }
```

**Custom bridge networks enable DNS-based service discovery:**

```bash
# Run containers on the custom network
docker run -d --network mynet --name web nginx
docker run -d --network mynet --name api alpine sleep infinity

# Now service name resolution WORKS:
docker exec web ping api       # Works! Resolves to 172.18.0.3
docker exec api ping web       # Works! Resolves to 172.18.0.2
```

**Always use custom bridge networks instead of the default bridge.**

### Custom Network Options

```bash
# Specify a subnet
docker network create \
  --subnet 10.0.1.0/24 \
  --gateway 10.0.1.1 \
  mynet

# Specify IP range (for DHCP-style allocation)
docker network create \
  --subnet 10.0.1.0/24 \
  --ip-range 10.0.1.0/28 \
  mynet

# Create an internal network (no internet access)
docker network create \
  --internal \
  internal-net

# Set the MTU (Maximum Transmission Unit)
docker network create \
  --opt com.docker.network.driver.mtu=1450 \
  mynet
```

---

### 2.2 Host Network

The host driver removes network isolation entirely. The container **shares the host's network namespace**.

```bash
docker run -d --network host nginx

# The container uses the host's IP, interfaces, and ports directly
# No port mapping needed — port 80 in the container IS port 80 on the host
```

```
Container with host network:
┌──────────────────────────────────────────┐
│           Host Network Stack             │
│  ┌────────────┐  ┌────────────┐          │
│  │  Container │  │  Host      │          │
│  │  nginx:80  │  │  ssh:22    │          │
│  │  app:8080  │  │  ...       │          │
│  └────────────┘  └────────────┘          │
│  ← ALL share the SAME network namespace  │
└──────────────────────────────────────────┘
```

**Pros:**
- Maximum network performance (no NAT overhead)
- Simple — no port mapping needed

**Cons:**
- No network isolation (container sees all host traffic)
- Port conflicts (container and host can't use the same port)
- Security risk (container can access host services)
- Only works on Linux (not Mac/Windows)

**When to use:**
- Network-intensive applications where performance is critical
- Monitoring tools that need to see all network traffic
- Containers that must bind to specific host interfaces

---

### 2.3 None Network

The none driver gives a container **no networking at all**:

```bash
docker run -d --network none alpine sleep infinity

# Container has only a loopback interface (lo: 127.0.0.1)
# No eth0, no external connectivity
docker exec <container> ip addr
# 1: lo: <LOOPBACK>
#    inet 127.0.0.1/8 scope host lo
```

**When to use:**
- Batch jobs that don't need network access
- Security-sensitive containers (complete network isolation)
- Testing network failure scenarios

---

### 2.4 Overlay Network (Swarm Mode)

Overlay networks span **multiple Docker hosts**. They're used in Docker Swarm and require Swarm mode to be enabled.

```bash
# Initialize Swarm mode
docker swarm init

# Create an overlay network
docker network create \
  --driver overlay \
  --subnet 10.0.0.0/24 \
  my-overlay

# Containers on different hosts can communicate through this network
# Traffic is encrypted (optional) and routed through the Swarm mesh
```

```
Host A                          Host B
┌──────────────────────┐       ┌──────────────────────┐
│ Container A1         │       │ Container B1         │
│ 10.0.0.2             │◀─────▶│ 10.0.0.3             │
│ (overlay network)    │  VXLAN│ (overlay network)    │
│                      │ tunnel│                      │
└──────────────────────┘       └──────────────────────┘
         │                             │
         └─────────────┬───────────────┘
                       │
              Overlay Network
              (encapsulated traffic)
```

**Covered in depth if you study Docker Swarm. For now, know that it exists.**

---

### 2.5 Macvlan Network

Macvlan assigns a **unique MAC address** to each container, making it appear as a physical device on your network.

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-macvlan

docker run -d --network my-macvlan --name mycontainer nginx
```

```
Physical Network Switch
├── Host (192.168.1.10) — MAC: AA:BB:CC:DD:EE:01
├── Container A (192.168.1.100) — MAC: AA:BB:CC:DD:EE:02
├── Container B (192.168.1.101) — MAC: AA:BB:CC:DD:EE:03
└── Other physical devices...
```

**Pros:**
- Containers get IPs from your physical network's DHCP
- Containers appear as real devices on the network
- No NAT, no port mapping

**Cons:**
- Requires network infrastructure support (promiscuous mode on switch)
- Each container consumes an IP from your physical network
- Not supported on all network hardware

**When to use:**
- Legacy applications that expect to be on the physical network
- Integration with existing network monitoring/management tools
- When you can't use NAT or port mapping

---

## 3. Comparing Network Drivers

| Driver | Isolation | IP Assignment | Multi-Host | Use Case |
|--------|-----------|---------------|------------|----------|
| **bridge** | Container-level | Docker-assigned | ❌ | Default, single-host apps |
| **host** | None | Host's IP | ❌ | Maximum performance |
| **none** | Complete | Loopback only | ❌ | No networking needed |
| **overlay** | Container-level | Docker-assigned | ✅ | Swarm, multi-host |
| **macvlan** | Physical network | DHCP/static | ❌ | Physical network integration |

---

## 4. Connecting and Disconnecting Containers

```bash
# Create networks
docker network create frontend
docker network create backend

# Run a container on one network
docker run -d --network frontend --name web nginx

# Connect the container to another network
docker network connect backend web

# Now 'web' is on BOTH networks
docker network inspect frontend  # Shows web
docker network inspect backend   # Also shows web

# Disconnect from a network
docker network disconnect backend web

# Now 'web' is only on frontend
docker network inspect backend   # No longer shows web
```

**Practical example:**
```bash
# Create networks
docker network create frontend
docker network create backend

# Run services
docker run -d --network frontend --name nginx nginx
docker run -d --network backend --name db postgres:16-alpine -e POSTGRES_PASSWORD=secret

# API is the bridge between frontend and backend
docker run -d \
  --network frontend \
  --name api \
  myapp-api

docker network connect backend api

# Traffic flow:
# nginx (frontend) → api (frontend + backend) → db (backend)
# nginx CANNOT reach db directly (different networks)
```

---

## 5. Network Management Commands

```bash
# List all networks
docker network ls

# List with detailed info
docker network ls --no-trunc

# Inspect a network (full JSON)
docker network inspect bridge

# Get specific info
docker network inspect bridge --format '{{.IPAM.Config}}'
docker network inspect mynet --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{end}}'

# Show which containers are on a network
docker network inspect mynet --format '{{range .Containers}}{{.Name}} → {{.IPv4Address}}
{{end}}'

# Remove a network
docker network rm mynet

# Remove ALL unused networks
docker network prune

# Connect a running container to a network
docker network connect mynet container-name

# Disconnect a container from a network
docker network disconnect mynet container-name
```

---

## 6. Port Mapping Deep Dive

When you publish a port with `-p`, Docker creates a NAT rule on the host.

```bash
docker run -d -p 8080:80 nginx
```

```
Host Machine                    Container
┌───────────────────────┐      ┌───────────────────────┐
│  Host IP: 0.0.0.0     │      │  Container IP:        │
│  Port: 8080           │      │  172.17.0.2           │
│                       │      │  Port: 80             │
│  Incoming request     │      │                       │
│  on port 8080 ────────┼─────▶│  nginx listening on   │
│                       │ NAT  │  port 80              │
└───────────────────────┘      └───────────────────────┘
```

### Port Mapping Variations

```bash
# Map to all interfaces
docker run -d -p 8080:80 nginx
# Accessible at: http://<host-ip>:8080

# Map to localhost only
docker run -d -p 127.0.0.1:8080:80 nginx
# Only accessible from the host machine

# Map to a specific host IP
docker run -d -p 192.168.1.100:8080:80 nginx
# Only accessible through that specific IP

# Map UDP ports
docker run -d -p 53:53/udp dns-server

# Map both TCP and UDP
docker run -d -p 53:53/tcp -p 53:53/udp dns-server

# Random host port (Docker assigns)
docker run -d -p 80 nginx
# docker port <container> → shows the assigned port

# Multiple ports
docker run -d -p 80:80 -p 443:443 nginx
```

### Viewing Port Mappings

```bash
# See port mappings for a container
docker port <container>
# Output: 80/tcp -> 0.0.0.0:8080

# See in docker ps
docker ps
# PORTS: 0.0.0.0:8080->80/tcp
```

---

## 7. DNS and Hostname Resolution

Docker's embedded DNS server runs at `127.0.0.11` inside every container on a user-defined network.

```bash
# Inside a container, check DNS config
cat /etc/resolv.conf
# nameserver 127.0.0.11    ← Docker's DNS server
# options ndots:0          ← Don't add search domains for single-label names

# Test DNS resolution
nslookup service-name
# Server:    127.0.0.11
# Address:   127.0.0.11:53
# Name: service-name
# Address: 172.18.0.3

# Check all names on the network
getent hosts service-name
```

**How Docker DNS works:**

```
Container A wants to reach "database":
1. Container checks /etc/resolv.conf → nameserver 127.0.0.11
2. Query sent to Docker's embedded DNS server
3. Docker DNS looks up "database" in the network's service registry
4. Returns: 172.18.0.3 (the IP of the "database" container)
5. Container A connects to 172.18.0.3
```

**What Docker DNS resolves:**
- Service names (`database`, `web`, `api`)
- Container names (`my-container-name`)
- Network aliases (custom names defined in the network config)
- Container hostnames (short container ID, e.g., `a1b2c3d4e5f6`)

---

## 📝 Hands-On Exercises

### Exercise 1: Create a Custom Bridge Network

```bash
# Create a custom bridge network with a specific subnet
docker network create \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1 \
  --driver bridge \
  exercise-net

# Verify it was created
docker network ls
docker network inspect exercise-net

# Run two containers on the network
docker run -d --network exercise-net --name web1 nginx
docker run -d --network exercise-net --name web2 nginx

# Test communication by name
docker exec web1 ping -c 2 web2
# PING web2 (10.10.0.3): 56 data bytes
# 64 bytes from 10.10.0.3: seq=0 ttl=64 time=0.089 ms

# Test communication by IP
docker exec web1 ping -c 2 10.10.0.3

# Clean up
docker rm -f web1 web2
docker network rm exercise-net
```

### Exercise 2: Compare Default vs Custom Bridge

```bash
# Run two containers on the DEFAULT bridge
docker run -d --name default-a nginx
docker run -d --name default-b nginx

# Try to reach by IP (works)
docker exec default-a ping -c 2 $(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' default-b)

# Try to reach by name (fails on default bridge)
docker exec default-a ping -c 2 default-b
# ping: bad address 'default-b'

# Clean up
docker rm -f default-a default-b

# Run two containers on a CUSTOM bridge
docker network create custom-net
docker run -d --network custom-net --name custom-a nginx
docker run -d --network custom-net --name custom-b nginx

# Try to reach by name (works on custom bridge!)
docker exec custom-a ping -c 2 custom-b
# PING custom-b (172.19.0.3): 56 data bytes
# 64 bytes from 172.19.0.3: seq=0 ttl=64 time=0.067 ms

# Clean up
docker rm -f custom-a custom-b
docker network rm custom-net
```

### Exercise 3: Host Network

```bash
# Run nginx with host network
docker run -d --network host --name host-nginx nginx

# No port mapping needed — port 80 is directly on the host
curl http://localhost:80
# <html>...Welcome to nginx!...

# Check that the container shares the host's network
docker exec host-nginx ip addr
# Compare with host's: ip addr
# They should be IDENTICAL

# Clean up
docker rm -f host-nginx
```

### Exercise 4: Connect/Disconnect

```bash
# Create two networks
docker network create net-a
docker network create net-b

# Run containers on different networks
docker run -d --network net-a --name container-a alpine sleep infinity
docker run -d --network net-b --name container-b alpine sleep infinity

# They CANNOT reach each other
docker exec container-a ping -c 1 container-b
# ping: bad address 'container-b'

# Connect container-a to net-b
docker network connect net-b container-a

# NOW they CAN reach each other
docker exec container-a ping -c 2 container-b
# PING container-b (172.21.0.2): 56 data bytes

# Disconnect
docker network disconnect net-b container-a

# They CANNOT reach each other again
docker exec container-a ping -c 1 container-b
# ping: bad address 'container-b'

# Clean up
docker rm -f container-a container-b
docker network rm net-a net-b
```

---

## 🧠 Common Questions

**Q: Why doesn't name resolution work on the default bridge?**
A: The default bridge is a legacy behavior. Custom user-defined networks have built-in DNS resolution. Always create your own networks instead of using the default bridge.

**Q: Can I change a container's network after it's running?**
A: You can ADD networks with `docker network connect`, but you can't REMOVE the primary network (the one set with `--network` at `docker run` time).

**Q: What IP range does Docker use?**
A: Default bridge: 172.17.0.0/16. Custom bridges: auto-assigned from the 172.18.0.0/16 range onward. You can specify custom subnets.

**Q: Can two networks communicate?**
A: Only if a container is connected to both networks, acting as a bridge between them.

**Q: How do I give a container internet access on an internal network?**
A: An internal network (`--internal`) has no gateway, so no internet. Remove the `--internal` flag or add a proxy/NAT container that's on both the internal network and a network with internet access.

---

## ✅ Day 1 Summary

**What you learned today:**

- ✅ How Docker networking works (network namespaces, bridge)
- ✅ All network drivers: bridge, host, none, overlay, macvlan
- ✅ Why custom bridge networks are better than the default bridge
- ✅ Creating and managing custom networks with subnets and options
- ✅ Connecting and disconnecting containers from networks
- ✅ Port mapping (NAT) in detail
- ✅ Docker's embedded DNS server for service resolution

**Coming up tomorrow:** Service discovery in depth — how containers find and communicate with each other by name.

---

*💡 Tip: Always create custom bridge networks for your containers. Never use the default bridge for production. Custom networks give you DNS-based service discovery, better isolation, and more control.*