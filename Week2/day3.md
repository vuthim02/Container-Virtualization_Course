# Week 2 • Day 3 • Beginner
# Volume Management

## Progress Checklist
- [x] Day 1 — Dockerfile Basics
- [x] Day 2 — Building Images
- [ ] Day 3 — Volume Management
- [ ] Day 4 — Image Optimization

---

← [Day 2](day2.md) | [Roadmap](../roadmap.md) | [Day 4 →](day4.md)

---

## 1. The Persistence Problem

Remember what we learned about containers on Day 3 of Week 1?

> **Containers have a thin writable layer. When the container is deleted, this layer is deleted too. All data inside is lost.**

```bash
# Demonstrate the problem:
docker run -d --name temp-db postgres:16-alpine \
  -e POSTGRES_PASSWORD=secret

# Create a database inside the container
docker exec temp-db psql -U postgres -c "CREATE TABLE users (id int);"

# Verify it exists
docker exec temp-db psql -U postgres -c "\dt"
# Output: users table exists

# Stop and remove the container
docker rm -f temp-db

# Run a fresh container
docker run -d --name new-db postgres:16-alpine \
  -e POSTGRES_PASSWORD=secret

# The table is gone!
docker exec new-db psql -U postgres -c "\dt"
# Output: Did not find any relations.
```

This is the **persistence problem**. Containers are ephemeral (temporary). Data inside them is not permanent.

### What Needs Persistence?

| Data Type | Examples |
|-----------|----------|
| **Databases** | PostgreSQL, MySQL, MongoDB data files |
| **Configuration** | Nginx configs, application settings |
| **Logs** | Application logs, access logs |
| **Uploads** | User-uploaded files, images, documents |
| **Cache** | Redis data, application cache |
| **Code (dev)** | Source code mounted for live editing |

---

## 2. Docker's Solution: Volumes and Mounts

Docker provides three ways to persist data:

| Type | Managed By | Stored Where | Best For |
|------|-----------|--------------|----------|
| **Volumes** | Docker | `/var/lib/docker/volumes/` | Database data, app data |
| **Bind Mounts** | You (the user) | Anywhere on host | Development, config files |
| **tmpfs** | Docker | In memory (RAM) | Temporary, sensitive data |

```
Container Filesystem Without Mounts:
┌─────────────────────────────────┐
│ Writable Layer (deleted on rm)  │
├─────────────────────────────────┤
│ Image Layers (read-only)        │
└─────────────────────────────────┘

Container Filesystem With a Mount:
┌─────────────────────────────────┐
│ Writable Layer                  │
│  /var/log (unmounted)           │
├─────────────────────────────────┤ ← Mount point
│ /data ← Mounted to host/volume  │ ← This data PERSISTS
├─────────────────────────────────┤
│ Image Layers (read-only)        │
└─────────────────────────────────┘
```

---

## 3. Volumes — Docker-Managed Storage

Volumes are the **preferred** method for persisting data in Docker.

### Creating and Using Volumes

```bash
# Create a volume
docker volume create mydata

# List all volumes
docker volume ls

# Inspect a volume
docker volume inspect mydata
# Output shows:
# {
#   "Driver": "local",
#   "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
#   "Name": "mydata",
#   ...
# }

# Use a volume in a container
docker run -d \
  --name my-app \
  -v mydata:/app/data \
  myimage

# The `-v volume_name:container_path` syntax:
#   mydata        → name of the Docker volume
#   /app/data     → path inside the container where it's mounted
```

### Volume Demo: Data Persistence

```bash
# Step 1: Create a volume and write data
docker volume create test-data
docker run -d --name writer \
  -v test-data:/data \
  alpine sh -c "echo 'Hello from volume!' > /data/message.txt"
docker rm -f writer

# Step 2: The volume still exists!
docker volume ls | grep test-data

# Step 3: Read the data with a new container
docker run --rm \
  -v test-data:/data \
  alpine cat /data/message.txt
# Output: Hello from volume!

# Step 4: Clean up
docker volume rm test-data
```

### Named vs Anonymous Volumes

```bash
# Named volume (you choose the name)
docker run -d -v mydata:/app/data nginx

# Anonymous volume (Docker generates a random name)
docker run -d -v /app/data nginx
# Docker creates a volume like: a1b2c3d4e5f6...

# Inspect anonymous volume
docker volume ls
# local     a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0

# Best practice: Always use named volumes for clarity
```

### Volume Drivers

```bash
# Default driver: local (stores on host filesystem)
docker volume create --driver local mydata

# Other drivers (require plugins):
docker volume create --driver vieux/sshfs \
  -o sshcmd=user@host:/path \
  -o password=secret \
  sshvolume

# Common third-party drivers:
# - vieux/sshfs: Mount remote directories via SSH
# - netshare: NFS, CIFS, EFS
# - rex-ray: AWS EBS, GCE PD, Azure Disk
```

---

## 4. Bind Mounts — Host Directory Mapping

Bind mounts map a **specific directory on your host** to a directory inside the container.

### Syntax

```bash
# Long syntax (recommended — clearer)
docker run -d \
  --type bind \
  --source /home/user/myapp \
  --target /app \
  myimage

# Short syntax (common)
docker run -d \
  -v /home/user/myapp:/app \
  myimage

# Or using --mount (most explicit)
docker run -d \
  --mount type=bind,source=/home/user/myapp,target=/app \
  myimage
```

### Bind Mount Demo: Development Workflow

```bash
# Create a project directory
mkdir -p ~/myproject
echo '<h1>Hello from bind mount!</h1>' > ~/myproject/index.html

# Run nginx with bind mount (live code editing)
docker run -d --name dev-nginx \
  -p 8080:80 \
  -v ~/myproject:/usr/share/nginx/html:ro \
  nginx

# Visit http://localhost:8080
# You'll see "Hello from bind mount!"

# Now edit the file ON YOUR HOST
echo '<h1>I changed this on my host!</h1>' > ~/myproject/index.html

# Refresh the browser — the change is immediate!
# No need to rebuild or restart the container
```

**`:ro` flag** — Mount as read-only:
```bash
# Container can READ and WRITE
-v /host/path:/container/path

# Container can only READ (safer for configs, static files)
-v /host/path:/container/path:ro
```

### Absolute Paths Required

```bash
# ✅ Absolute path
docker run -v /home/user/data:/data myimage

# ❌ Relative path (Docker will create a named volume called "data" instead!)
docker run -v data:/data myimage

# Exception: Docker Compose supports relative paths
# (covered in Week 3)
```

---

## 5. tmpfs — In-Memory Storage

tmpfs mounts store data in the host's **memory (RAM)**. Data is never written to disk and is lost when the container stops.

```bash
# Mount a tmpfs volume
docker run -d \
  --tmpfs /app/temp \
  --tmpfs-size 100m \
  myimage

# Or with --mount:
docker run -d \
  --mount type=tmpfs,destination=/app/temp,tmpfs-size=100m \
  myimage
```

**When to use tmpfs:**

| Use Case | Why tmpfs |
|----------|-----------|
| Sensitive data (tokens, keys) | Never written to disk |
| Temporary processing files | Fast I/O, auto-cleanup |
| Session storage | Fast access, ephemeral |
| Cache data | Speed, no need for persistence |

**When NOT to use tmpfs:**

| Don't Use For | Why |
|---------------|-----|
| Database files | Lost on container stop |
| User uploads | Lost on container stop |
| Anything important | It's in RAM, it's volatile |

---

## 6. Comparing All Three Methods

```
┌──────────────────┬──────────────┬──────────────┬──────────────┐
│   Feature        │   Volumes    │ Bind Mounts  │   tmpfs      │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│ Persist data?    │ ✅ Yes       │ ✅ Yes       │ ❌ No        │
│ Managed by?      │ Docker       │ You          │ Docker       │
│ Stored where?    │ Docker dir   │ Any path     │ RAM          │
│ Share between    │ ✅ Easy      │ ✅ Easy      │ ❌ No        │
│ containers?      │              │              │              │
│ Works on any OS? │ ✅ Yes       │ ⚠️ Path dep. │ ⚠️ Linux only│
│ Performance?     │ Good         │ Good         │ Best (RAM)   │
│ Backup easily?   │ ✅ Yes       │ ✅ Yes       │ ❌ N/A       │
│ Best for:        │ DB data,     │ Dev, config  │ Temp,        │
│                  │ production   │ files        │ sensitive    │
└──────────────────┴──────────────┴──────────────┴──────────────┘
```

### `-v` vs `--mount` Syntax

```bash
# -v syntax (older, shorter)
docker run -v myvolume:/app/data myimage

# --mount syntax (newer, more explicit)
docker run \
  --mount type=volume,source=myvolume,target=/app/data \
  myimage

# --mount supports more options:
docker run \
  --mount type=volume,source=myvolume,target=/app/data,volume-driver=local,readonly \
  myimage
```

**Best Practice:** Use `--mount` in scripts and production. Use `-v` for quick commands and development.

---

## 7. Volume Management Commands

```bash
# List all volumes
docker volume ls

# List with sizes (requires Docker 20.10+)
docker volume ls -f dangling=false

# Create a volume
docker volume create mydata

# Create with specific driver
docker volume create --driver local mydata

# Inspect a volume (shows mount point, driver, labels)
docker volume inspect mydata

# Remove a volume
docker volume rm mydata

# Remove ALL unused volumes
docker volume prune

# Remove volumes filtered by label
docker volume prune --filter "label=mylabel"

# Backup a volume
docker run --rm \
  -v mydata:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/mydata-backup.tar.gz -C /source .

# Restore a volume
docker run --rm \
  -v mydata:/target \
  -v $(pwd):/backup \
  alpine tar xzf /backup/mydata-backup.tar.gz -C /target
```

---

## 8. Multiple Volumes and Complex Mounts

```bash
# Multiple volumes on one container
docker run -d \
  -v app-data:/app/data \
  -v app-logs:/app/logs \
  -v app-config:/app/config:ro \
  myimage

# Mix volume types
docker run -d \
  -v myvolume:/app/data \
  -v /home/user/config:/app/config:ro \
  --tmpfs /app/temp \
  myimage

# Mount a single file (not directory)
docker run -d \
  -v /home/user/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx

# Mount multiple config files
docker run -d \
  -v /home/user/conf.d:/etc/nginx/conf.d:ro \
  -v /home/user/certs:/etc/nginx/certs:ro \
  nginx
```

---

## 9. Volumes in Dockerfiles

You can declare volumes in a Dockerfile, but you **cannot** specify the host path. The volume is created at runtime.

```dockerfile
FROM postgres:16-alpine

# Declare a volume (documentation + automatic creation)
VOLUME /var/lib/postgresql/data

# You can also declare multiple volumes
VOLUME ["/data", "/logs", "/config"]
```

**Important:** `VOLUME` in a Dockerfile is **documentation**. It tells Docker "this directory should be a volume." The actual volume is created when you run the container.

```bash
# With VOLUME in Dockerfile, this creates an anonymous volume automatically:
docker run -d mypostgres

# To use a named volume instead:
docker run -d -v mypgdata:/var/lib/postgresql/data mypostgres

# To override with a bind mount:
docker run -d -v /mnt/data:/var/lib/postgresql/data mypostgres
```

---

## 10. Real-World Examples

### PostgreSQL with Persistent Data

```bash
# Create a volume for PostgreSQL data
docker volume create pgdata

# Run PostgreSQL with persistent data
docker run -d \
  --name production-db \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=super-secret \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  postgres:16-alpine

# Create some data
docker exec production-db psql -U postgres -d myapp -c \
  "CREATE TABLE users (id serial PRIMARY KEY, name VARCHAR(50));"
docker exec production-db psql -U postgres -d myapp -c \
  "INSERT INTO users (name) VALUES ('Alice'), ('Bob');"
docker exec production-db psql -U postgres -d myapp -c \
  "SELECT * FROM users;"

# Stop and remove the container
docker rm -f production-db

# Start a NEW container with the SAME volume
docker run -d \
  --name new-db \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=super-secret \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  postgres:16-alpine

# Data is still there!
docker exec new-db psql -U postgres -d myapp -c "SELECT * FROM users;"
# Output: Alice and Bob are still in the table
```

### Nginx with Bind-Mounted Config

```bash
# Create custom nginx config
mkdir -p ~/nginx-config
cat > ~/nginx-config/default.conf << 'EOF'
server {
    listen 80;
    server_name localhost;
    
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
    
    location /api {
        proxy_pass http://backend:3000;
    }
}
EOF

# Run nginx with bind-mounted config
docker run -d \
  --name my-nginx \
  -p 80:80 \
  -v ~/nginx-config/default.conf:/etc/nginx/conf.d/default.conf:ro \
  nginx
```

### Redis with tmpfs for Speed

```bash
# Run Redis with data in RAM (fastest possible)
docker run -d \
  --name fast-redis \
  --tmpfs /data:rw,size=256m \
  -p 6379:6379 \
  redis:7-alpine

# Data is ultra-fast but lost on container restart
```

---

## 📝 Hands-On Exercises

### Exercise 1: Named Volume Persistence

```bash
# 1. Create a named volume
docker volume create persist-test

# 2. Run a container that writes to the volume
docker run --rm \
  -v persist-test:/data \
  alpine sh -c "echo 'Persistent data!' > /data/file.txt && cat /data/file.txt"

# 3. Run another container to read the data
docker run --rm \
  -v persist-test:/data \
  alpine cat /data/file.txt
# Output: Persistent data!

# 4. Inspect the volume
docker volume inspect persist-test

# 5. Check where Docker stores the data
sudo ls -la /var/lib/docker/volumes/persist-test/_data/

# 6. Clean up
docker volume rm persist-test
```

### Exercise 2: PostgreSQL with Persistent Data

```bash
# 1. Create a volume
docker volume create pg-data

# 2. Run PostgreSQL
docker run -d \
  --name week2-postgres \
  -v pg-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=test123 \
  -p 5432:5432 \
  postgres:16-alpine

# 3. Wait a few seconds for it to start, then create a table
docker exec week2-postgres psql -U postgres -c \
  "CREATE TABLE products (id int, name varchar(50));"
docker exec week2-postgres psql -U postgres -c \
  "INSERT INTO products VALUES (1, 'Widget');"

# 4. Verify
docker exec week2-postgres psql -U postgres -c "SELECT * FROM products;"

# 5. Stop and remove
docker rm -f week2-postgres

# 6. Start fresh container with same volume
docker run -d \
  --name week2-postgres-2 \
  -v pg-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=test123 \
  -p 5432:5432 \
  postgres:16-alpine

# 7. Data persists!
docker exec week2-postgres-2 psql -U postgres -c "SELECT * FROM products;"
# Output: Widget is still there!

# 8. Clean up
docker rm -f week2-postgres-2
docker volume rm pg-data
```

### Exercise 3: Bind Mount for Development

```bash
# 1. Create a directory with an HTML file
mkdir -p ~/docker-site
echo '<h1>My Docker Site</h1><p>Edit me!</p>' > ~/docker-site/index.html

# 2. Run nginx with bind mount
docker run -d \
  --name dev-site \
  -p 8080:80 \
  -v ~/docker-site:/usr/share/nginx/html \
  nginx

# 3. Visit http://localhost:8080

# 4. Edit the file ON YOUR HOST (use any editor)
echo '<h1>I updated this without rebuilding!</h1>' > ~/docker-site/index.html

# 5. Refresh the browser — the change is instant!

# 6. Clean up
docker rm -f dev-site
rm -rf ~/docker-site
```

### Exercise 4: tmpfs for Temporary Data

```bash
# 1. Run a container with tmpfs
docker run --rm \
  --tmpfs /tmp:rw,size=50m \
  alpine sh -c "echo 'Secret temp data!' > /tmp/secret.txt && cat /tmp/secret.txt"

# 2. Run another container — the data is gone (new container, new tmpfs)
docker run --rm \
  --tmpfs /tmp:rw,size=50m \
  alpine ls /tmp/ 2>/dev/null || echo "tmpfs is empty or doesn't have the file"

# 3. Verify no volume was created
docker volume ls | grep tmpfs   # Should show nothing
```

---

## 🧠 Common Questions

**Q: Can I use a volume with `docker build`?**
A: No. Volumes are a runtime feature. Builds only work with the build context and COPY instructions.

**Q: What happens to anonymous volumes when I remove a container?**
A: They persist! Docker doesn't automatically delete them. Use `docker volume prune` to clean them up.

**Q: Can two containers share the same volume?**
A: Yes! This is a common pattern for sharing data between containers.

```bash
docker volume create shared-data
docker run -d -v shared-data:/data --name writer alpine sh -c "while true; do date >> /data/log.txt; sleep 1; done"
docker run -d -v shared-data:/data --name reader alpine sh -c "tail -f /data/log.txt"
```

**Q: Are volumes faster than bind mounts?**
A: On Linux, performance is similar. On macOS/Windows (using VMs), volumes are significantly faster because they avoid filesystem translation overhead.

**Q: Can I mount a volume as read-only?**
A: Yes! Add `:ro` to the mount: `-v mydata:/data:ro` or `--mount type=volume,source=mydata,target=/data,readonly`.

---

## ✅ Day 3 Summary

**What you learned today:**

- ✅ Why containers need persistent storage (the writable layer is ephemeral)
- ✅ Three types of mounts: volumes, bind mounts, and tmpfs
- ✅ How to create, use, and manage Docker volumes
- ✅ Bind mounts for development workflows
- ✅ tmpfs for in-memory, temporary storage
- ✅ Volume management commands (ls, inspect, rm, prune)
- ✅ Backup and restore volumes
- ✅ Real-world examples (PostgreSQL, nginx, Redis)

**Coming up tomorrow:** We'll learn how to optimize Docker images for size, speed, and security.

---

*💡 Tip: Volumes are the backbone of stateful containers. Whenever your application needs to remember something between restarts, reach for a volume.*