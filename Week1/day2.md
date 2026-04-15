# Week 1 • Day 2 • Beginner
# Docker Architecture

## Progress Checklist
- [x] Day 1 — What is Containerization?
- [ ] Day 2 — Docker Architecture
- [ ] Day 3 — Images & Containers
- [ ] Day 4 — Basic Commands

---

← [Day 1](day1.md) | [Roadmap](../roadmap.md) | [Day 3 →](day3.md)

---

## 1. Docker's Big Picture

Docker didn't invent containers — but it made them **accessible**. Before Docker (2013), working with containers meant writing complex LXC configurations. Docker wrapped all that complexity into simple commands.

### Why Architecture Matters

Understanding Docker's architecture helps you:
- **Debug problems** (is it the CLI? the daemon? the network?)
- **Design secure setups** (who can talk to the daemon?)
- **Troubleshoot performance** (where's the bottleneck?)
- **Understand alternatives** (why Podman is daemonless, what containerd does)

---

## 2. Docker's Client-Server Architecture

Docker uses a classic **client-server** model:

```
┌─────────────────────────────────────────────────────────────┐
│                        Your Machine                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐         ┌──────────────────────────────┐  │
│  │              │         │                              │  │
│  │  Docker CLI  │────────▶│      Docker Daemon           │  │
│  │  (docker)    │  REST   │      (dockerd)               │  │
│  │              │  API    │                              │  │
│  │  $ docker    │  (Unix  │  ┌────────────────────────┐  │  │
│  │    run nginx │   socket│  │   containerd           │  │  │
│  │  $ docker    │  or     │  │   (container runtime)  │  │  │
│  │    build .   │   TCP)  │  └──────────┬─────────────┘  │  │
│  │              │         │             │                │  │
│  └──────────────┘         │  ┌──────────▼─────────────┐  │  │
│                           │  │   runc                 │  │  │
│                           │  │   (OCI runtime)        │  │  │
│                           │  └────────────────────────┘  │  │
│                           │                              │  │
│                           └──────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Let's break down every piece.

---

## 3. Core Components — Deep Dive

### 3.1 Docker CLI (Client)

The `docker` command you type in your terminal. It's a **thin client** — it doesn't do any heavy lifting itself. It just translates your commands into API calls.

```bash
# What happens when you type this:
docker run -d -p 80:80 nginx

# Behind the scenes:
# 1. CLI connects to Docker daemon via Unix socket (/var/run/docker.sock)
# 2. CLI sends a REST API call: POST /containers/create
# 3. CLI sends another call: POST /containers/{id}/start
# 4. CLI returns the container ID to your terminal
```

**Key point:** The CLI is stateless. It doesn't store container info — it asks the daemon every time.

```bash
# This is why both commands work even if you didn't "create" anything locally:
docker ps          # Asks daemon for running containers
docker images      # Asks daemon for stored images
```

### 3.2 Docker Daemon (dockerd)

The **brain** of Docker. It's a persistent background process that:

- Manages the full container lifecycle
- Stores images, containers, volumes, and networks
- Enforces security policies
- Communicates with registries
- Handles networking between containers

```bash
# The daemon runs as root (usually) and listens on:
# - Unix socket: /var/run/docker.sock (default)
# - TCP socket: 2375 (unencrypted, rarely used)
# - TCP socket: 2376 (TLS encrypted, production)

# Check if dockerd is running:
systemctl status docker

# View daemon logs:
journalctl -u docker --no-pager | tail -30
```

**⚠️ Security Warning:** The Docker daemon runs as root. Anyone with access to `/var/run/docker.sock` effectively has root access to the host. This is a critical security consideration.

```bash
# Check who can access the Docker socket:
ls -la /var/run/docker.sock
# Output: srw-rw---- 1 root docker ... /var/run/docker.sock
# Only root and docker group members can use it

# Add a user to the docker group (be careful!):
sudo usermod -aG docker $USER
```

### 3.3 containerd

Between the Docker daemon and the actual container runtime sits **containerd** — an industry-standard container runtime.

**What containerd does:**
- Pulls images from registries
- Stores images locally
- Starts/stops containers via runc
- Manages container execution state
- Handles port forwarding and I/O

```
Docker Daemon (dockerd)
        │
        │ gRPC API
        ▼
    containerd          ← Industry standard, used by Docker, Kubernetes, etc.
        │
        │
        ▼
     containerd-shim
        │
        ▼
       runc            ← Low-level OCI runtime
```

**Why this matters:** Docker didn't keep containerd as an internal module — it became its own open-source project and is now used by Kubernetes directly. This means Kubernetes can use containerd **without Docker** (which is what happened in Kubernetes 1.24+ when they removed Docker support).

### 3.4 runc

The lowest-level piece in the stack. **runc** is the reference implementation of the OCI runtime specification.

**What runc actually does:**
1. Creates Linux namespaces (PID, NET, MNT, etc.)
2. Sets up cgroups (resource limits)
3. Prepares the root filesystem (chroot/pivot_root)
4. Executes the container's ENTRYPOINT/CMD process

```bash
# You can actually use runc directly (though nobody does in practice):
# 1. Export a container config
runc spec

# 2. Run it
runc run mycontainer

# This proves that Docker is mostly orchestration around runc.
# runc does the actual "container magic" using Linux kernel features.
```

### 3.5 The Full Request Flow

Here's exactly what happens when you run `docker run nginx`:

```
Step 1: User types "docker run nginx"
              │
              ▼
Step 2: Docker CLI sends POST /containers/create to dockerd
              │
              ▼
Step 3: Docker daemon checks if nginx image exists locally
              │
         ┌────┴────┐
         │  No?    │
         ▼         │
Step 4: dockerd calls containerd to pull image from Docker Hub
              │
              ▼
Step 5: containerd downloads layers, unpacks image
              │
              ▼
Step 6: dockerd creates container metadata
              │
              ▼
Step 7: dockerd tells containerd to start container
              │
              ▼
Step 8: containerd starts containerd-shim
              │
              ▼
Step 9: containerd-shim calls runc
              │
              ▼
Step 10: runc sets up namespaces, cgroups, filesystem
              │
              ▼
Step 11: runc exec's the nginx process inside the namespace
              │
              ▼
Step 12: nginx is now running as a containerized process
```

---

## 4. Docker Registries

A registry is where images are stored and retrieved from.

### Docker Hub (Default Registry)

```bash
# When you run this:
docker pull nginx

# Docker actually pulls from:
docker pull docker.io/library/nginx:latest
```

**Registry URL breakdown:**
```
docker.io/library/nginx:latest
├─────┬────┤├──┬──┤├─┬──┤├──┬───┤
│  Registry  │Namespace│Image│ Tag │
│            │(or org) │     │     │
```

| Part | Meaning |
|------|---------|
| `docker.io` | Default registry (Docker Hub) |
| `library` | Official images (no organization prefix needed) |
| `nginx` | Image name |
| `latest` | Tag (defaults to latest if not specified) |

### Other Registries

```bash
# GitHub Container Registry
docker pull ghcr.io/myorg/myapp:v1.0

# Google Container Registry / Artifact Registry
docker pull gcr.io/my-project/myapp:v1.0

# AWS Elastic Container Registry
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0

# Azure Container Registry
docker pull myregistry.azurecr.io/myapp:v1.0

# Self-hosted (Harbor, Nexus, etc.)
docker pull registry.mycompany.com/myapp:v1.0
```

### Official vs Community Images

| Type | Example | Maintained By | Trust Level |
|------|---------|---------------|-------------|
| **Official** | `nginx`, `python`, `node` | Docker Inc + vendor | ✅ High |
| **Verified** | `bitnami/nginx` | Verified publisher | ✅ Good |
| **Community** | `someuser/myapp` | Anyone | ⚠️ Use caution |

**Best Practice:** Always prefer official images. When using community images, inspect the Dockerfile and tags before trusting them.

---

## 5. The Linux Kernel Features Behind Docker

Docker relies on several Linux kernel features. Understanding these helps you understand what containers *actually* are.

### 5.1 Namespaces (Isolation)

We covered these briefly on Day 1. Let's go deeper.

```bash
# See what namespaces a running container has:
docker run -d --name test-ns nginx

# Find the container's PID on the host:
docker inspect --format '{{.State.Pid}}' test-ns
# Output: 12345 (example)

# See its namespaces:
ls -la /proc/12345/ns/
# Output shows: cgroup ipc mnt net pid user uts
```

### 5.2 Cgroups (Resource Limits)

```bash
# Run a container with CPU and memory limits:
docker run -d --name limited \
  --cpus="1.0" \
  --memory="256m" \
  --memory-swap="256m" \
  nginx

# Verify the limits:
docker inspect limited --format '{{.HostConfig.Memory}}'
# Output: 268435456 (256 MB in bytes)

docker inspect limited --format '{{.HostConfig.NanoCpus}}'
# Output: 1000000000 (1 CPU in nanocpus)
```

### 5.3 Union File Systems (Layers)

Docker images are built in **layers**. Each Dockerfile instruction creates a new layer.

```
nginx:latest image structure:
┌────────────────────────────────────┐
│ Layer 5: CMD ["nginx", "-g", ...]  │ ← Tiny (metadata)
├────────────────────────────────────┤
│ Layer 4: EXPOSE 80                 │ ← Tiny (metadata)
├────────────────────────────────────┤
│ Layer 3: nginx config files        │ ← ~5 KB
├────────────────────────────────────┤
│ Layer 2: nginx binaries            │ ← ~5 MB
├────────────────────────────────────┤
│ Layer 1: debian:bookworm-slim      │ ← ~75 MB
└────────────────────────────────────┘

Total: ~80 MB
```

**Why layers matter:**
- Layers are **shared** between images
- If you have 10 containers using `nginx:latest`, the base layers are loaded once
- Rebuilding an image reuses unchanged layers (fast rebuilds)

```bash
# See layers of an image:
docker image inspect nginx --format '{{json .RootFS.Layers}}' | python3 -m json.tool

# Or use:
docker history nginx
```

### 5.4 Copy-on-Write (CoW)

When a container writes a file:

```
Image Layers (Read-Only)
├── Layer 1: base OS
├── Layer 2: app binary
└── Layer 3: config file
        │
        │ Container writes to /config/settings.json
        ▼
Container Layer (Read-Write, Thin)
└── Only the changed file is stored here
```

The original image layer is never modified. Only the changed files exist in the container's thin writable layer.

---

## 6. Docker Storage Drivers

The storage driver manages how layers and the writable layer work.

| Driver | Best For | Notes |
|--------|----------|-------|
| **overlay2** | Most modern Linux setups | Default and recommended |
| **btrfs** | Btrfs filesystem setups | Supports snapshots |
| **zfs** | ZFS filesystem setups | Enterprise storage features |
| **vfs** | Testing, no special kernel reqs | Slow, don't use in production |

```bash
# Check your storage driver:
docker info | grep "Storage Driver"
# Output: Storage Driver: overlay2 (most likely)
```

---

## 7. Docker's Configuration Files

```bash
# Main daemon config:
/etc/docker/daemon.json

# Example configuration:
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "data-root": "/var/lib/docker",
  "dns": ["8.8.8.8", "8.8.4.4"]
}

# After editing, restart the daemon:
sudo systemctl restart docker
```

---

## 8. Docker vs Podman — Architectural Difference

```
Docker Architecture:              Podman Architecture:
┌──────────────┐                  ┌──────────────┐
│  Docker CLI  │                  │  Podman CLI  │
└──────┬───────┘                  └──────┬───────┘
       │                                 │
       │ REST API                        │ Direct fork/exec
       ▼                                 ▼
┌──────────────┐                  ┌──────────────┐
│   Docker     │                  │     runc     │
│   Daemon     │                  │  (via conmon)│
│  (root)      │                  └──────────────┘
└──────────────┘
   │
   │ Manages everything
   ▼
┌──────────────┐
│   containerd │
│     + runc   │
└──────────────┘

Key Difference:
- Docker: daemon required, runs as root
- Podman: daemonless, can run rootless
```

---

## 📝 Hands-On Exercises

### Exercise 1: Verify Docker Installation
```bash
# Check both client and server versions
docker version

# Expected output should show:
# - Client: Docker Engine - Community
# - Server: Docker Engine - Community
# Both should have version numbers
```

### Exercise 2: Inspect the Docker Daemon
```bash
# View full Docker system information
docker info

# Key things to look for:
# - Number of containers (running/paused/stopped)
# - Number of images
# - Storage driver
# - Cgroup version
# - Operating system
# - Kernel version
```

### Exercise 3: Explore the Docker Socket
```bash
# Verify the socket exists
ls -la /var/run/docker.sock

# Check the Docker daemon process
ps aux | grep dockerd

# View recent daemon logs
sudo journalctl -u docker -n 20 --no-pager
```

### Exercise 4: Trace a Container Run
```bash
# In one terminal, watch Docker events:
docker events

# In another terminal, run a quick container:
docker run --rm hello-world

# Go back to the first terminal and observe the events:
# You'll see: image pull, container create, container start, container die
```

### Exercise 5: Diagram Exercise
On paper or digitally, draw Docker's architecture from memory:
- Docker CLI
- Docker Daemon
- containerd
- runc
- Registry

Label the communication paths between each component.

---

## 🧠 Common Questions

**Q: Can Docker run without the daemon?**
A: No. Docker requires dockerd to be running. If you want daemonless containers, look at Podman.

**Q: What happens if the Docker daemon crashes?**
A: Running containers **continue running** (they're separate processes). But you can't start/stop/manage containers until dockerd restarts.

**Q: Is Docker just a wrapper around runc?**
A: Essentially, yes. Docker adds image management, networking, volumes, API, and UX on top of runc's raw container execution.

**Q: Why did Kubernetes drop Docker support?**
A: Kubernetes now uses containerd directly because Docker added an unnecessary abstraction layer. The kubelet talks to containerd natively via CRI (Container Runtime Interface).

---

## ✅ Day 2 Summary

**What you learned today:**

- ✅ Docker's client-server architecture (CLI → Daemon → Runtime)
- ✅ The role of each component (dockerd, containerd, runc)
- ✅ How registries work and image naming conventions
- ✅ Linux kernel features behind containers (namespaces, cgroups, layers, CoW)
- ✅ Storage drivers and Docker configuration
- ✅ How Docker differs architecturally from Podman

**Coming up tomorrow:** We'll dive deep into images and containers — what they are, how they differ, and how to work with them.

---

*💡 Tip: The architecture can feel overwhelming. Focus on the flow: CLI → Daemon → containerd → runc → Container. Everything else builds on this.*