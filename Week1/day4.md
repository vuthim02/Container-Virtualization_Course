# Week 1 • Day 4 • Beginner
# Basic Commands

## Progress Checklist
- [x] Day 1 — What is Containerization?
- [x] Day 2 — Docker Architecture
- [x] Day 3 — Images & Containers
- [ ] Day 4 — Basic Commands

---

← [Day 3](day3.md) | [Roadmap](../roadmap.md) | [Week 2 Day 1 →](../Week2/day1.md)

---

## 1. Why Commands Matter

You've learned the theory. Now it's time to build muscle memory. These are the commands you'll use **every single day** as someone working with containers.

By the end of this lesson, you should be able to:
- Launch any container with the right flags
- Debug containers that won't start
- Inspect what's happening inside containers
- Clean up resources properly

---

## 2. Container Lifecycle Commands

### `docker run` — The Most Important Command

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

This single command does three things:
1. **Creates** a container from the image
2. **Configures** it with the options you provide
3. **Starts** it running

#### Basic Patterns

```bash
# Run in the background (detached mode) — most common
docker run -d nginx

# Run in the foreground (attached to your terminal)
docker run nginx

# Run and remove automatically when it exits
docker run --rm nginx

# Run with a human-friendly name
docker run -d --name my-web-server nginx

# Run a specific version
docker run -d nginx:1.25.3-alpine
```

#### Port Mapping (`-p`)

Containers have their own network namespace. To access a service inside a container from your host machine, you need to map ports:

```bash
# Map host port 8080 to container port 80
docker run -d -p 8080:80 nginx
# Visit: http://localhost:8080 → reaches nginx inside the container

# Map to a specific host IP
docker run -d -p 127.0.0.1:8080:80 nginx
# Only accessible from localhost

# Map a range of ports
docker run -d -p 8080-8090:8080-8090 myapp

# Map UDP ports
docker run -d -p 53:53/udp dns-server

# Multiple port mappings
docker run -d -p 80:80 -p 443:443 nginx
```

**Visual:**
```
Host Machine                    Container
┌──────────────────┐           ┌──────────────────┐
│                  │           │                  │
│  Port 8080       │ ◀───────▶ │  Port 80         │
│  (host)          │  -p 8080:80│  (container)     │
│                  │           │  nginx listening │
└──────────────────┘           └──────────────────┘

Your browser: http://localhost:8080
                     │
                     ▼
              Reaches nginx on container port 80
```

#### Environment Variables (`-e`)

```bash
# Single variable
docker run -d -e NODE_ENV=production myapp

# Multiple variables
docker run -d \
  -e DB_HOST=postgres \
  -e DB_PORT=5432 \
  -e DB_USER=admin \
  -e DB_PASSWORD=secret \
  myapp

# Variables from a file
docker run -d --env-file .env myapp

# Example with official MySQL image
docker run -d \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  -e MYSQL_DATABASE=myapp \
  -p 3306:3306 \
  mysql:8.0
```

#### Resource Limits

```bash
# Limit memory
docker run -d --memory=256m nginx

# Limit CPU (0.5 = half a core, 2.0 = two cores)
docker run -d --cpus=0.5 nginx

# Limit both
docker run -d --memory=512m --cpus=1.0 myapp

# Set restart policy
docker run -d --restart unless-stopped nginx
docker run -d --restart always nginx
docker run -d --restart on-failure:5 nginx  # Retry 5 times on failure
docker run -d --restart no nginx             # Never restart (default)
```

**Restart Policy Comparison:**

| Policy | Behavior | When to Use |
|--------|----------|-------------|
| `no` | Never restart | Testing, one-off jobs |
| `on-failure[:max]` | Restart only on non-zero exit | Background workers |
| `always` | Always restart (even on `docker stop`) | Critical services |
| `unless-stopped` | Always restart (except when manually stopped) | ✅ Most common for services |

---

### `docker start` / `docker stop` / `docker restart`

```bash
# Start a stopped container
docker start my-web-server

# Stop gracefully (SIGTERM → wait 10s → SIGKILL)
docker stop my-web-server

# Stop with custom timeout
docker stop -t 30 my-web-server

# Restart (stop + start)
docker restart my-web-server

# You can operate multiple containers at once:
docker stop web1 web2 web3
docker start web1 web2
```

### `docker rm` — Remove Containers

```bash
# Remove a stopped container
docker rm my-web-server

# Force remove a running container
docker rm -f my-web-server

# Remove all stopped containers
docker container prune

# Remove all containers (⚠️ dangerous!)
docker rm -f $(docker ps -aq)
```

---

## 3. Interactive Container Access

### `docker run -it` — Interactive Shell

The `-it` flags combine to give you a terminal inside the container:

- `-i` (--interactive): Keep STDIN open
- `-t` (--tty): Allocate a pseudo-terminal

```bash
# Get a bash shell inside Ubuntu
docker run -it ubuntu bash

# You're now INSIDE the container!
root@a1b2c3d4e5f6:/# ls
root@a1b2c3d4e5f6:/# apt-get update
root@a1b2c3d4e5f6:/# exit   # Type 'exit' or press Ctrl+D to leave
```

**What happens:**
```
┌─────────────────────────────────────────┐
│             Your Terminal               │
│                                         │
│  $ docker run -it ubuntu bash           │
│                                         │
│  root@a1b2c3d4e5f6:/# ← You are HERE   │
│  (inside the container's namespace)     │
│                                         │
│  The container's filesystem, processes, │
│  and network are all isolated from host │
└─────────────────────────────────────────┘
```

### `docker exec` — Enter a Running Container

Use `exec` to run a command inside a container that's **already running**:

```bash
# Get a shell inside a running container
docker exec -it my-web-server bash
# or for Alpine-based containers:
docker exec -it my-web-server sh

# Run a single command (doesn't open interactive shell)
docker exec my-web-server cat /etc/nginx/nginx.conf

# Run as a different user
docker exec -u root -it my-web-server bash

# Set environment variables for the exec session
docker exec -e MY_VAR=hello my-web-server env

# View running processes inside the container
docker exec my-web-server ps aux

# Test network connectivity from inside the container
docker exec my-web-server curl -s http://localhost
```

**`exec` vs `run -it`:**

| Feature | `docker run -it` | `docker exec -it` |
|---------|-----------------|-------------------|
| Container state | Creates a NEW container | Enters EXISTING container |
| Use case | Debugging, exploration | Inspecting running services |
| Main process | New process (PID 1) | Additional process alongside PID 1 |
| After `exit` | Container stops (if no background process) | Container keeps running |

---

## 4. Diagnostic Commands

### `docker logs` — View Container Output

Containers write to STDOUT and STDERR. Docker captures both.

```bash
# View all logs
docker logs my-web-server

# Follow logs in real-time (like tail -f)
docker logs -f my-web-server

# Show only last N lines
docker logs --tail 50 my-web-server

# Show logs with timestamps
docker logs -t my-web-server

# Show logs since a specific time
docker logs --since 2024-01-15T10:00:00 my-web-server
docker logs --since 30m my-web-server   # Last 30 minutes

# Combine flags: follow + tail + timestamps
docker logs -f --tail 20 -t my-web-server
```

**Understanding logs output:**
```bash
# Start a container that produces logs
docker run -d --name log-demo nginx

# View logs
docker logs log-demo
# Output:
# /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty...
# /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
# ...
# 2024/01/15 10:00:00 [notice] 1#1: start worker processes
```

**Pro tip:** If a container crashes immediately, logs are your first debugging tool:
```bash
docker run -d --name broken mybrokenimage
docker ps -a          # Check if it exited
docker logs broken    # See why it failed
docker rm broken      # Clean up
```

### `docker inspect` — Deep Dive into Container Details

Returns low-level information about containers, images, networks, and volumes.

```bash
# Full JSON output (massive)
docker inspect my-web-server

# Get specific fields with Go templates
docker inspect my-web-server --format '{{.State.Status}}'
# Output: running

docker inspect my-web-server --format '{{.State.Pid}}'
# Output: 12345 (host PID of the container's main process)

docker inspect my-web-server --format '{{.NetworkSettings.IPAddress}}'
# Output: 172.17.0.2

docker inspect my-web-server --format '{{.HostConfig.PortBindings}}'
# Output: map[80/tcp:[{HostIp: HostPort:8080}]]

# Get multiple fields at once
docker inspect my-web-server --format '{{.Name}} | {{.State.Status}} | {{.NetworkSettings.IPAddress}}'
# Output: /my-web-server | running | 172.17.0.2

# Output as raw JSON (for scripting)
docker inspect my-web-server --format '{{json .}}' | python3 -m json.tool
```

**Common inspect patterns:**

```bash
# What is the container's status?
docker inspect --format '{{.State.Status}}' <container>

# What is the exit code? (for stopped containers)
docker inspect --format '{{.State.ExitCode}}' <container>

# When did it start?
docker inspect --format '{{.State.StartedAt}}' <container>

# Why did it die? (error message)
docker inspect --format '{{.State.Error}}' <container>

# What volumes are mounted?
docker inspect --format '{{json .Mounts}}' <container> | python3 -m json.tool

# What environment variables are set?
docker inspect --format '{{json .Config.Env}}' <container> | python3 -m json.tool

# What command was used to start it?
docker inspect --format '{{.Path}} {{.Args}}' <container>
```

### `docker stats` — Monitor Resource Usage

Real-time monitoring of container resource consumption.

```bash
# Live stream of all running containers
docker stats

# One-time snapshot
docker stats --no-stream

# Stats for specific containers
docker stats my-web-server my-database

# Show all containers (including stopped)
docker stats --no-stream -a
```

**Example output:**
```
CONTAINER ID   NAME            CPU %     MEM USAGE / LIMIT   MEM %     NET I/O         BLOCK I/O    PIDS
a1b2c3d4e5f6   my-web-server   0.02%     5.47MiB / 256MiB    2.14%     1.2kB / 890B    0B / 0B      4
b2c3d4e5f6g7   my-database     1.50%     142.3MiB / 512MiB   27.79%    45.2kB / 12kB   1.2MB / 0B   18
```

| Column | Meaning |
|--------|---------|
| **CPU %** | Percentage of host CPU the container is using |
| **MEM USAGE / LIMIT** | Current memory usage / maximum allowed |
| **MEM %** | Memory usage as percentage of limit |
| **NET I/O** | Network data received / sent |
| **BLOCK I/O** | Disk data read / written |
| **PIDS** | Number of processes running inside the container |

---

## 5. Image Management Commands

```bash
# List all images
docker images
docker images -a              # Include intermediate images
docker images --digests       # Show image digests
docker images --no-trunc      # Show full image IDs

# Search Docker Hub
docker search nginx
docker search --filter stars=100 nginx   # Only popular images

# Pull an image
docker pull nginx
docker pull nginx:1.25.3-alpine

# Remove an image
docker rmi nginx:latest
docker rmi 605c77e624dd

# Remove all unused images
docker image prune -a

# Build an image from Dockerfile
docker build -t myapp:v1 .

# Tag an existing image
docker tag nginx:latest myregistry.com/nginx:latest

# Save/load images
docker save nginx > nginx.tar
docker load < nginx.tar
```

---

## 6. Volume Commands

```bash
# List volumes
docker volume ls

# Create a volume
docker volume create mydata

# Inspect a volume
docker volume inspect mydata

# Remove a volume
docker volume rm mydata

# Remove all unused volumes
docker volume prune

# Use a volume in a container
docker run -d -v mydata:/app/data nginx

# Use a bind mount (host directory → container)
docker run -d -v /home/user/data:/app/data nginx
```

---

## 7. Network Commands

```bash
# List networks
docker network ls

# Create a custom bridge network
docker network create mynet

# Inspect a network
docker network inspect mynet

# Run a container on a custom network
docker run -d --network mynet --name web nginx

# Connect a running container to a network
docker network connect mynet mycontainer

# Disconnect a container from a network
docker network disconnect mynet mycontainer

# Remove a network
docker network rm mynet
docker network prune   # Remove all unused networks
```

---

## 8. System-Wide Commands

```bash
# Full system information
docker info

# Disk usage summary
docker system df

# Disk usage (verbose — shows individual items)
docker system df -v

# Remove ALL unused data (images, containers, volumes, networks)
docker system prune

# Remove ALL unused data including stopped containers and unused volumes
docker system prune -a --volumes

# ⚠️ This is the nuclear option — removes everything
docker system prune -a --volumes --all
```

**`docker system df` output:**
```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          7         2         1.234GB   890MB (72%)
Containers      3         1         12.5MB    8.2MB (65%)
Local Volumes   4         1         256MB     128MB (50%)
Build Cache     0         0         0B        0B
```

---

## 9. Useful Command Shortcuts & Aliases

Add these to your `~/.bashrc` or `~/.zshrc`:

```bash
# Quick container list
alias dps='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
alias dpsa='docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'

# Quick image list
alias dim='docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"'

# Quick stats
alias dstats='docker stats --no-stream'

# Cleanup functions
alias dclean='docker system prune -f'
alias dcleanall='docker system prune -a --volumes -f'

# Enter running container
alias dexec='docker exec -it'

# View logs
alias dlog='docker logs -f --tail 50'
```

---

## 10. Real-World Workflow Example

Here's a realistic scenario putting all the commands together:

```bash
# === SCENARIO: Deploy a web application ===

# 1. Pull the latest image
docker pull nginx:1.25.3-alpine

# 2. Run it with proper configuration
docker run -d \
  --name production-web \
  --restart unless-stopped \
  -p 80:80 \
  -v /var/www/html:/usr/share/nginx/html:ro \
  -e NGINX_HOST=example.com \
  nginx:1.25.3-alpine

# 3. Verify it's running
docker ps
docker stats --no-stream

# 4. Check logs
docker logs production-web

# 5. Test connectivity from inside the container
docker exec production-web curl -s http://localhost

# 6. Inspect details
docker inspect production-web --format '{{.NetworkSettings.IPAddress}}'

# 7. Something's wrong? Debug:
docker exec -it production-web sh
# Inside the container, investigate...
cat /etc/nginx/nginx.conf
ls -la /usr/share/nginx/html
exit

# 8. Need to update? Stop, remove, recreate
docker stop production-web
docker rm production-web
# (Now run step 2 again with updated config)

# 9. End of day cleanup (if this was just testing)
docker rm -f production-web
docker system prune -f
```

---

## 📝 Hands-On Exercises — Full Day Task

### Task: Master the nginx Container

Complete ALL of the following steps. This is your capstone exercise for Week 1.

#### Part 1: Launch and Verify
```bash
# 1. Pull the nginx image
docker pull nginx:1.25.3-alpine

# 2. Run nginx in detached mode with port mapping
docker run -d --name week1-nginx -p 8080:80 nginx:1.25.3-alpine

# 3. Verify it's running
docker ps
docker stats --no-stream

# 4. Open your browser to http://localhost:8080
#    You should see the "Welcome to nginx!" page

# 5. Alternatively, test with curl:
curl http://localhost:8080
```

#### Part 2: Inspect and Explore
```bash
# 6. View the container's logs
docker logs week1-nginx

# 7. Inspect the container's configuration
docker inspect week1-nginx --format '{{.Name}}'
docker inspect week1-nginx --format '{{.State.Status}}'
docker inspect week1-nginx --format '{{.NetworkSettings.IPAddress}}'
docker inspect week1-nginx --format '{{.HostConfig.PortBindings}}'

# 8. Enter the running container
docker exec -it week1-nginx sh

# Inside the container, run these commands:
#   nginx -v
#   cat /etc/nginx/nginx.conf
#   ls -la /usr/share/nginx/html/
#   ps aux
#   exit

# 9. Run a single command without entering interactively
docker exec week1-nginx nginx -v
docker exec week1-nginx cat /etc/nginx/conf.d/default.conf
```

#### Part 3: Lifecycle Management
```bash
# 10. Stop the container
docker stop week1-nginx

# 11. Verify it stopped
docker ps -a  # Should show "Exited"

# 12. Start it again
docker start week1-nginx

# 13. Verify it's running
docker ps

# 14. Restart it
docker restart week1-nginx
```

#### Part 4: Resource Monitoring
```bash
# 15. Watch resource usage
docker stats week1-nginx

# 16. View one-time snapshot
docker stats --no-stream week1-nginx
```

#### Part 5: Cleanup
```bash
# 17. Stop and remove the container
docker rm -f week1-nginx

# 18. Verify it's gone
docker ps -a

# 19. Check disk usage
docker system df

# 20. (Optional) Remove the image too
docker rmi nginx:1.25.3-alpine
```

---

## 🧠 Troubleshooting Guide

### Container Exits Immediately

```bash
# Common cause: the main process finished and exited
docker run -d ubuntu  # Exits immediately — 'ubuntu' has no long-running process

# Fix: give it something to do
docker run -d ubuntu sleep 3600

# Or use an interactive shell
docker run -it ubuntu bash
```

### Port Already in Use

```bash
# Error: "bind: address already in use"
# Something is already listening on that port

# Find what's using the port:
sudo lsof -i :8080
sudo ss -tlnp | grep 8080

# Fix: use a different host port
docker run -d -p 9090:80 nginx   # Map to port 9090 instead
```

### Container Won't Start

```bash
# Check logs first
docker logs <container_name>

# Check if it's in a restart loop
docker ps -a  # Look for "Restarting" status

# Inspect the state
docker inspect <container_name> --format '{{.State.Error}}'
```

### Can't Remove a Container

```bash
# Error: "cannot remove running container"
docker rm -f <container_name>   # Force remove

# Or stop first, then remove
docker stop <container_name>
docker rm <container_name>
```

---

## ✅ Day 4 Summary

**What you learned today:**

- ✅ `docker run` — all the essential flags (`-d`, `-p`, `-e`, `-v`, `--name`, `--rm`, `--restart`)
- ✅ Container lifecycle: `start`, `stop`, `restart`, `rm`
- ✅ Interactive access: `docker run -it` vs `docker exec -it`
- ✅ Diagnostics: `logs`, `inspect`, `stats`
- ✅ Image management: `images`, `pull`, `rmi`, `build`, `tag`
- ✅ Volume and network management basics
- ✅ System-wide commands: `info`, `df`, `prune`
- ✅ Real-world deployment workflow
- ✅ Troubleshooting common issues

**🎉 Week 1 Complete!**

You now have a solid foundation in container fundamentals. You understand:
- What containers are and how they differ from VMs
- Docker's architecture and all its components
- The critical distinction between images and containers
- All the essential commands for daily container management

**Coming up in Week 2:** We'll dive into Dockerfiles — how to build your own custom images from scratch.

---

*💡 Tip: Practice these commands until they feel natural. They're the foundation for everything that follows. Try running containers, inspecting them, breaking them, and fixing them. The more you experiment, the more comfortable you'll become.*