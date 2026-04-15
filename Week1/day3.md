# Week 1 • Day 3 • Beginner
# Images & Containers

## Progress Checklist
- [x] Day 1 — What is Containerization?
- [x] Day 2 — Docker Architecture
- [ ] Day 3 — Images & Containers
- [ ] Day 4 — Basic Commands

---

← [Day 2](day2.md) | [Roadmap](../roadmap.md) | [Day 4 →](day4.md)

---

## 1. Images vs Containers — The Core Concept

This is the single most important concept in Docker. If you understand this, everything else falls into place.

### The Analogy

```
Docker Image    =  A recipe (or a class in programming)
Docker Container =  The baked cake (or an instance of that class)
```

| Analogy | Image | Container |
|---------|-------|-----------|
| **Cooking** | Recipe card | The actual dish you eat |
| **Programming** | Class definition | An object created from that class |
| **Photography** | Negative/template | A printed photograph |
| **Virtualization** | Installation ISO | A running VM |

### Technical Definitions

**Docker Image:**
- A **read-only template** for creating containers
- Built from a Dockerfile
- Made of stacked layers (filesystem changes)
- Identified by a name and tag (`nginx:1.25`, `python:3.12-slim`)
- Can be stored in registries and shared
- **Does not change** once built

**Docker Container:**
- A **running instance** of an image
- Has a writable layer on top of the image layers
- Has its own process space, network interfaces, and filesystem
- Has a lifecycle (create → start → stop → remove)
- **Can be modified** during its lifetime (changes go in the writable layer)

```
Image (read-only, on disk)          Container (running, in memory)
┌───────────────────────────┐       ┌───────────────────────────┐
│ CMD ["nginx"]             │       │ Image layers (read-only)  │
│ EXPOSE 80                 │  ───▶ │ + Writable layer          │
│ nginx binary              │       │ + Process (PID)           │
│ base OS files             │       │ + Network namespace       │
└───────────────────────────┘       │ + Cgroup resource limits  │
                                    └───────────────────────────┘
   One image can create →    MANY containers (all identical at start)
```

---

## 2. Understanding Image Layers

Every Docker image is a stack of **read-only layers**. Each layer represents a filesystem change.

### How Layers Work

Consider this Dockerfile:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y python3
COPY app.py /app/app.py
CMD ["python3", "/app/app.py"]
```

When built, it produces these layers:

```
Layer 4: CMD ["python3", "/app/app.py"]   ← 0 bytes (metadata only)
Layer 3: COPY app.py /app/app.py           ← ~2 KB (the file)
Layer 2: RUN apt-get update && install...  ← ~150 MB (python3 + deps)
Layer 1: FROM ubuntu:24.04                 ← ~75 MB (base OS)
────────────────────────────────────────────────
Total image size:                         ~225 MB
```

### Layer Sharing (The Magic of Layers)

```
Image A: python:3.12 + Flask app     Image B: python:3.12 + Django app
┌───────────────────────────┐        ┌───────────────────────────┐
│ Layer 3: COPY flask_app/  │        │ Layer 3: COPY django_app/ │
├───────────────────────────┤        ├───────────────────────────┤
│ Layer 2: pip install flask│        │ Layer 2: pip install django│
├───────────┬───────────────┤        ├───────────┬───────────────┤
│           │  SHARED LAYERS          │           │               │
│ Layer 1:  │  python:3.12-slim       │ Layer 1:  │               │
│           │  (75 MB, loaded ONCE)   │           │               │
└───────────┴───────────────┘        └───────────┴───────────────┘
```

**Key insight:** Docker only stores each unique layer once on disk. If you have 10 images based on `python:3.12-slim`, the base Python layer is stored only once. This saves enormous disk space.

### Exploring Layers

```bash
# See the layers of a local image:
docker history nginx:latest

# Example output:
# IMAGE          CREATED       CREATED BY                                      SIZE
# a836e6f0e742   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]               0B
# <missing>      2 weeks ago   EXPOSE map[80/tcp:{}]                           0B
# <missing>      2 weeks ago   COPY ...                                        5.5kB
# <missing>      2 weeks ago   RUN /bin/sh -c apt-get update ...               60.3MB
# <missing>      2 weeks ago   debian:bookworm-slim                            74.8MB
```

Notice:
- Each row is a layer
- `CMD` and `EXPOSE` are 0B (they're metadata, not files)
- `COPY` and `RUN` add actual bytes
- The base image is the largest layer

---

## 3. Image Tags — More Important Than You Think

### What is a Tag?

A tag is a **label** attached to a specific image version:

```
nginx:1.25.3-alpine
├──┬──┤├─┬───┤
│    │  │
│    │  └── Specific tag
│    └── Image name
└── Image prefix (could include registry and organization)
```

### Common Tag Conventions

| Tag | Meaning | When to Use |
|-----|---------|-------------|
| `nginx:latest` | Most recent version | ❌ Avoid in production |
| `nginx:1.25` | Latest 1.25.x patch | ✅ Good balance |
| `nginx:1.25.3` | Exact version | ✅ Best for production |
| `nginx:1.25.3-alpine` | Exact version, Alpine base | ✅ Best for size |

### The `latest` Trap

```bash
# These are the SAME command — latest is the default:
docker pull nginx
docker pull nginx:latest

# PROBLEM: "latest" changes over time!
# Today:  nginx:latest → nginx:1.25.3
# Next month: nginx:latest → nginx:1.26.0

# This breaks reproducibility:
# Your Dockerfile works today, but next month pulls a different image!
```

**Best Practice:** Always use specific tags in Dockerfiles and production:

```dockerfile
# ❌ Bad — unpredictable
FROM node:latest

# ✅ Good — pinned to major version
FROM node:20

# ✅ Best — pinned to exact version
FROM node:20.11.0-alpine
```

---

## 4. Working with Images

### Pulling Images

```bash
# Pull from Docker Hub (default registry)
docker pull nginx

# Pull a specific tag
docker pull nginx:1.25.3-alpine

# Pull from another registry
docker pull ghcr.io/nginx/nginx:1.25.3

# Pull all tags for an image (rarely needed)
docker pull --all-tags nginx

# Pull only the metadata (no download)
docker pull --quiet nginx
```

### Listing Images

```bash
# List all local images
docker images

# List with digest (cryptographic hash)
docker images --digests

# Show only image IDs (useful for scripts)
docker images -q

# Show full image IDs (not truncated)
docker images --no-trunc

# Human-readable format
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

**Example output:**
```
REPOSITORY    TAG              IMAGE ID       CREATED        SIZE
nginx         1.25.3-alpine    a836e6f0e742   2 weeks ago    42.6MB
nginx         latest           605c77e624dd   2 weeks ago    187MB
python        3.12-slim        4b99c4cc0f27   3 days ago     123MB
ubuntu        24.04            59a6c97b2b2c   2 weeks ago    78.1MB
```

### Inspecting Images

```bash
# Full JSON metadata
docker image inspect nginx

# Get specific fields
docker inspect nginx --format '{{.Config.ExposedPorts}}'
# Output: map[80/tcp:{}]

docker inspect nginx --format '{{.Config.Cmd}}'
# Output: [nginx -g daemon off;]

docker inspect nginx --format '{{.Size}}'
# Output: 187700000 (approximately)
```

### Removing Images

```bash
# Remove a specific image
docker rmi nginx:latest

# Remove by image ID
docker rmi 605c77e624dd

# Remove all unused images
docker image prune

# Remove ALL unused images (not just dangling)
docker image prune -a

# ⚠️ Remove ALL images (nuclear option)
docker rmi $(docker images -q)

# Force remove (even if containers use it)
docker rmi -f nginx
```

### Saving and Loading Images

```bash
# Save an image to a tar file (offline transfer)
docker save nginx:latest > nginx.tar
docker save nginx:latest | gzip > nginx.tar.gz  # Compressed

# Load an image from a tar file
docker load < nginx.tar
docker load < nginx.tar.gz

# Useful for:
# - Air-gapped environments (no internet)
# - Sharing images without a registry
# - Backup purposes
```

---

## 5. Containers — The Running Instances

### Creating vs Running

There's an important distinction:

```bash
# Creates the container but does NOT start it:
docker create nginx

# Creates AND starts the container:
docker run nginx

# These are equivalent:
docker create nginx && docker start <container_id>
docker run nginx
```

### Container Lifecycle

```
         docker create
              │
              ▼
         ┌────────┐     docker start     ┌──────────┐
         │Created │ ────────────────────▶│ Running  │
         └────────┘                      └────┬─────┘
                                              │
                                    ┌─────────┼─────────┐
                                    │         │         │
                               docker stop  docker   docker
                                    │      restart   pause
                                    ▼         │         │
                               ┌────────┐     │    ┌─────────┐
                               │Stopped │ ◀───┘    │ Paused  │
                               └───┬────┘          └────┬────┘
                                   │                    │
                          docker start            docker unpause
                                   │                    │
                                   ▼                    ▼
                             ┌──────────┐         ┌──────────┐
                             │ Running  │         │ Running  │
                             └──────────┘         └──────────┘
                                   │
                              docker rm
                                   │
                                   ▼
                             ┌──────────┐
                             │ Removed  │
                             └──────────┘
```

### Running Containers

```bash
# Basic run (detached mode — runs in background)
docker run -d nginx

# Run with a custom name
docker run -d --name my-web-server nginx

# Run with port mapping (host:container)
docker run -d -p 8080:80 nginx

# Run with environment variables
docker run -d -e MY_VAR=hello nginx

# Run with a volume mount
docker run -d -v mydata:/data nginx

# Run with resource limits
docker run -d --memory=256m --cpus=0.5 nginx

# Run and automatically remove on exit
docker run --rm nginx

# Run interactively (get a shell)
docker run -it ubuntu bash
```

### Common `docker run` Flags

| Flag | Short | Purpose | Example |
|------|-------|---------|---------|
| `--detach` | `-d` | Run in background | `docker run -d nginx` |
| `--name` | — | Give container a name | `--name myapp` |
| `--publish` | `-p` | Map host port to container port | `-p 8080:80` |
| `--env` | `-e` | Set environment variable | `-e DB_HOST=localhost` |
| `--volume` | `-v` | Mount a volume | `-v data:/app/data` |
| `--interactive` | `-i` | Keep STDIN open | `-it` (with `-t`) |
| `--tty` | `-t` | Allocate pseudo-TTY | `-it` (with `-i`) |
| `--restart` | — | Restart policy | `--restart unless-stopped` |
| `--rm` | — | Auto-remove on exit | `--rm` |

### Listing Containers

```bash
# List running containers
docker ps

# List ALL containers (including stopped)
docker ps -a

# Show only container IDs
docker ps -q

# Show only container IDs (including stopped)
docker ps -aq

# Filter by status
docker ps --filter "status=running"
docker ps --filter "status=exited"

# Filter by name
docker ps --filter "name=nginx"

# Format output
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Example output of `docker ps`:**
```
CONTAINER ID   IMAGE          COMMAND                  CREATED        STATUS        PORTS                  NAMES
a1b2c3d4e5f6   nginx:latest   "nginx -g 'daemon of…"   2 minutes ago  Up 2 mins     0.0.0.0:8080->80/tcp   my-web-server
```

### Inspecting Containers

```bash
# Get full JSON metadata
docker inspect my-web-server

# Get container's IP address
docker inspect my-web-server --format '{{.NetworkSettings.IPAddress}}'

# Get container's state
docker inspect my-web-server --format '{{.State.Status}}'

# Get the main process PID on the host
docker inspect my-web-server --format '{{.State.Pid}}'

# Get mount information
docker inspect my-web-server --format '{{json .Mounts}}' | python3 -m json.tool
```

### Stopping and Starting

```bash
# Graceful stop (sends SIGTERM, waits 10s, then SIGKILL)
docker stop my-web-server

# Stop with custom timeout (waits 30 seconds)
docker stop -t 30 my-web-server

# Start a stopped container
docker start my-web-server

# Restart (stop + start)
docker restart my-web-server

# Pause (freezes all processes — uses cgroups freezer)
docker pause my-web-server

# Unpause
docker unpause my-web-server

# Kill (immediate SIGKILL — no graceful shutdown)
docker kill my-web-server
```

### Removing Containers

```bash
# Remove a stopped container
docker rm my-web-server

# Remove a running container (force stop first)
docker rm -f my-web-server

# Remove ALL stopped containers
docker container prune

# Remove ALL containers (including running)
docker rm -f $(docker ps -aq)
```

---

## 6. The Container Writable Layer

When a container runs, Docker adds a thin writable layer on top:

```
Running Container's Filesystem:
┌────────────────────────────────────┐
│ Writable Layer (container-specific)│ ← Changes made here
│  /var/log/nginx/access.log         │
│  /tmp/some-temp-file               │
├────────────────────────────────────┤
│ Image Layer 3: CMD config          │ ← Read-only
├────────────────────────────────────┤
│ Image Layer 2: nginx binary        │ ← Read-only
├────────────────────────────────────┤
│ Image Layer 1: base OS              │ ← Read-only
└────────────────────────────────────┘
```

**Important:** When the container is deleted, the writable layer is deleted too. Any data stored there is **lost**.

This is why **volumes** exist — they provide persistent storage that survives container deletion (covered in Week 2, Day 3).

```bash
# Demonstrate the writable layer:
docker run -d --name demo nginx

# Create a file inside the container
docker exec demo touch /tmp/hello.txt

# Verify it exists
docker exec demo ls /tmp/hello.txt
# Output: /tmp/hello.txt

# Stop and remove the container
docker rm -f demo

# Run a new container from the same image
docker run -d --name demo2 nginx

# The file is gone — writable layer was deleted
docker exec demo2 ls /tmp/hello.txt
# Output: ls: cannot access '/tmp/hello.txt': No such file or directory
```

---

## 7. Container States Explained

| State | Meaning | How to Fix |
|-------|---------|-----------|
| **Created** | Container exists but hasn't started | `docker start <name>` |
| **Running** | Container is actively running | Nothing needed |
| **Paused** | Processes frozen (cgroups freezer) | `docker unpause <name>` |
| **Restarting** | Currently restarting (restart policy) | Wait, or `docker stop` to cancel |
| **Exited** | Container stopped (exit code shown) | `docker start <name>` |
| **Dead** | Failed to delete (storage driver error) | `docker rm -f <name>` |

```bash
# See exit codes of stopped containers
docker ps -a
# Look for: "Exited (0)" = clean exit, "Exited (1)" = error
```

---

## 📝 Hands-On Exercises

### Exercise 1: Pull and Inspect an Image
```bash
# Pull the nginx image
docker pull nginx:1.25.3-alpine

# List local images and note the size difference
docker images

# Inspect the image layers
docker history nginx:1.25.3-alpine

# Inspect the image metadata
docker inspect nginx:1.25.3-alpine --format '{{.Config.ExposedPorts}}'
docker inspect nginx:1.25.3-alpine --format '{{.Config.Cmd}}'
```

### Exercise 2: Run Your First Container
```bash
# Run nginx with a custom name and port mapping
docker run -d --name my-nginx -p 8080:80 nginx

# Verify it's running
docker ps

# Open a browser and visit: http://localhost:8080
# You should see the "Welcome to nginx!" page

# Check the container logs
docker logs my-nginx
```

### Exercise 3: Multiple Containers from One Image
```bash
# Run two nginx containers on different ports
docker run -d --name web1 -p 8081:80 nginx
docker run -d --name web2 -p 8082:80 nginx

# Verify both are running
docker ps

# Visit http://localhost:8081 and http://localhost:8082
# Both serve the same nginx page (same image, different containers)

# Clean up
docker rm -f web1 web2
```

### Exercise 4: Container Lifecycle Practice
```bash
# Create a container without starting it
docker create --name paused-container nginx

# Check its state
docker ps -a  # Should show "Created"

# Start it
docker start paused-container

# Stop it gracefully
docker stop paused-container

# Start it again
docker start paused-container

# Remove it (must stop first)
docker stop paused-container
docker rm paused-container

# Or force remove in one step
docker run -d --name temp nginx
docker rm -f temp
```

### Exercise 5: Explore the Writable Layer
```bash
# Run a container
docker run -d --name layer-demo ubuntu sleep 3600

# Create files in the writable layer
docker exec layer-demo touch /tmp/file1.txt
docker exec layer-demo sh -c "echo 'Hello from container!' > /tmp/file2.txt"

# Verify
docker exec layer-demo cat /tmp/file2.txt

# Remove the container
docker rm -f layer-demo

# Run a new container — files are gone
docker run -d --name layer-demo-2 ubuntu sleep 3600
docker exec layer-demo-2 cat /tmp/file2.txt  # Will fail

# Clean up
docker rm -f layer-demo-2
```

---

## 🧠 Common Questions

**Q: Can I modify an image after building it?**
A: No. Images are immutable. To "modify" an image, you rebuild it with a new Dockerfile.

**Q: How many containers can I create from one image?**
A: Unlimited (limited only by system resources). Each container is an independent instance.

**Q: What happens to the writable layer when a container exits?**
A: It persists on disk until you remove the container with `docker rm`. You can commit it to a new image with `docker commit` (but this is not best practice — use Dockerfiles instead).

**Q: Is `docker pull` the same as `docker run`?**
A: No. `docker pull` only downloads the image. `docker run` creates AND starts a container from the image (pulling it first if needed).

---

## ✅ Day 3 Summary

**What you learned today:**

- ✅ The fundamental difference between images (templates) and containers (instances)
- ✅ How image layers work and why layer sharing saves disk space
- ✅ Image tags, why `latest` is dangerous, and how to pin versions
- ✅ All essential image commands (pull, list, inspect, save, load, remove)
- ✅ Container lifecycle (create → start → stop → remove)
- ✅ The writable layer and why data is lost when containers are deleted
- ✅ Container states and how to transition between them

**Coming up tomorrow:** We'll master all the essential Docker commands you'll use every day.

---

*💡 Tip: The image vs container distinction is fundamental. If you're ever confused, go back to the analogy: image = recipe, container = cake.*