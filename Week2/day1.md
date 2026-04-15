# Week 2 • Day 1 • Beginner
# Dockerfile Basics

## Progress Checklist
- [x] Day 1 — Dockerfile Basics
- [ ] Day 2 — Building Images
- [ ] Day 3 — Volume Management
- [ ] Day 4 — Image Optimization

---

← [Roadmap](../roadmap.md) | [Day 2 →](day2.md)

---

## 1. What is a Dockerfile?

A **Dockerfile** is a text file containing step-by-step instructions to build a Docker image. Think of it as a **recipe** that Docker follows to create a reproducible, portable environment for your application.

### Why Dockerfiles Matter

Before Dockerfiles, you might have:
- Manually installed dependencies on a server
- Written a 50-line shell script to set up an environment
- Hoped the next person followed the same steps

With a Dockerfile:
```bash
# One command, identical environment every time
docker build -t myapp:v1 .
```

### The Golden Rule

> **If it's not in the Dockerfile, it doesn't exist in the image.**

Every dependency, every file, every configuration must be explicitly declared. This makes your builds **reproducible** and **auditable**.

---

## 2. Dockerfile Instructions — Complete Guide

### 2.1 `FROM` — The Foundation

Every Dockerfile starts with `FROM`. It specifies the **base image** your image is built on.

```dockerfile
# Official Python image
FROM python:3.12-slim

# Official Node.js image
FROM node:20-alpine

# Official Ubuntu image
FROM ubuntu:24.04

# Use an image from a specific registry
FROM ghcr.io/org/base-image:v1.0

# Use a digest (maximum reproducibility)
FROM python@sha256:abc123def456...
```

**Choosing a base image:**

| Base Image | Size | When to Use |
|------------|------|-------------|
| `ubuntu:24.04` | ~78 MB | Need full Ubuntu toolchain |
| `debian:bookworm-slim` | ~75 MB | Debian ecosystem, smaller footprint |
| `alpine:3.19` | ~7 MB | Minimal size, musl libc (beware of compatibility) |
| `scratch` | 0 bytes | Statically compiled binaries (Go, Rust) |
| `distroless` | ~2-20 MB | Google's minimal runtime-only images |

**⚠️ Important:** Always use specific tags, never `latest`:

```dockerfile
# ❌ Bad — unpredictable builds
FROM python:latest

# ✅ Good — pinned version
FROM python:3.12-slim

# ✅ Best — pinned to exact version
FROM python:3.12.1-slim-bookworm
```

---

### 2.2 `LABEL` — Metadata

Add metadata to your image. This is visible with `docker inspect`.

```dockerfile
LABEL maintainer="devops@mycompany.com"
LABEL version="1.0"
LABEL description="My web application"

# Or all at once:
LABEL maintainer="devops@mycompany.com" \
      version="1.0" \
      description="My web application"
```

---

### 2.3 `WORKDIR` — Set Working Directory

Sets the working directory for all subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, and `ADD` instructions.

```dockerfile
FROM python:3.12-slim

# Creates /app if it doesn't exist, then cd's into it
WORKDIR /app

# Now all relative paths are relative to /app
COPY requirements.txt .      # Copies to /app/requirements.txt
COPY src/ ./src/             # Copies to /app/src/
RUN pip install -r requirements.txt   # Runs in /app
CMD ["python", "app.py"]     # Runs in /app
```

**⚠️ Never do this:**
```dockerfile
# ❌ Don't use RUN cd — it only applies to that one RUN instruction
RUN cd /app && pip install -r requirements.txt

# ✅ Use WORKDIR instead
WORKDIR /app
RUN pip install -r requirements.txt
```

**You can use `WORKDIR` multiple times:**
```dockerfile
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

WORKDIR /app/src
COPY src/ .
CMD ["python", "app.py"]
```

---

### 2.4 `COPY` — Copy Files

Copies files/directories from your **build context** (host) into the image.

```dockerfile
# Copy a single file
COPY requirements.txt .

# Copy a directory
COPY src/ ./src/

# Copy multiple files
COPY package.json package-lock.json ./

# Copy with wildcard
COPY *.conf /etc/nginx/conf.d/

# Copy and rename
COPY config/production.env ./.env
```

**Build Context:** When you run `docker build .`, the `.` is the build context. Docker sends **all files** in that directory (recursively) to the Docker daemon before building. This is why `.dockerignore` matters (covered on Day 2).

**`COPY` vs `ADD`:**

| Feature | `COPY` | `ADD` |
|---------|--------|-------|
| Copy local files | ✅ | ✅ |
| Auto-extract tarballs | ❌ | ✅ |
| Download from URLs | ❌ | ✅ |
| Recommended | ✅ **Yes** | ❌ Avoid |

**Best Practice:** Always use `COPY` unless you specifically need `ADD`'s auto-extraction feature.

---

### 2.5 `RUN` — Execute Commands

Runs commands **during image build**. Each `RUN` creates a new layer.

```dockerfile
# Install system packages
RUN apt-get update && apt-get install -y \
    curl \
    git \
    vim \
    && rm -rf /var/lib/apt/lists/*

# Install Python packages
RUN pip install --no-cache-dir -r requirements.txt

# Install Node packages
RUN npm install --production

# Multiple RUN instructions (each creates a layer)
RUN apt-get update && apt-get install -y curl
RUN pip install flask
```

**Critical patterns:**

```bash
# ✅ ALWAYS chain apt-get update with install in one RUN
RUN apt-get update && apt-get install -y curl

# ❌ NEVER separate them — caching causes stale packages
RUN apt-get update
RUN apt-get install -y curl

# ✅ ALWAYS clean up package manager cache
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

# ✅ Use --no-cache-dir for pip
RUN pip install --no-cache-dir flask

# ✅ Use --production for npm (skip dev dependencies)
RUN npm install --production
```

**Why cache cleanup matters:**
```
Without cleanup:    Layer size = 150 MB (curl + cached package files)
With cleanup:       Layer size = 15 MB (curl only)
```

---

### 2.6 `ENV` — Environment Variables

Sets environment variables that persist at **both build time and runtime**.

```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables
ENV APP_HOME=/app \
    APP_PORT=8080 \
    LOG_LEVEL=info

# Use variables in subsequent instructions
ENV APP_HOME=/app
WORKDIR $APP_HOME
COPY . $APP_HOME/

# Variables available at runtime
ENV DATABASE_URL=postgres://localhost:5432/mydb
```

**Accessing at runtime:**
```bash
# Inside the container:
echo $APP_HOME          # /app
echo $DATABASE_URL      # postgres://localhost:5432/mydb
```

**Override at runtime:**
```bash
# Dockerfile sets DATABASE_URL, but you can override:
docker run -e DATABASE_URL=postgres://prod-db:5432/prod myapp
```

---

### 2.7 `EXPOSE` — Document Ports

Documents which ports the container listens on. **Does NOT publish ports** — you still need `-p` at runtime.

```dockerfile
# Single port
EXPOSE 80

# Multiple ports
EXPOSE 80 443

# UDP port
EXPOSE 53/udp

# TCP port (explicit)
EXPOSE 8080/tcp
```

**What `EXPOSE` actually does:**
- Serves as **documentation** for image users
- Used by `docker run -P` (capital P) to publish all exposed ports
- Used by orchestration tools (Kubernetes, Docker Compose)

```bash
# -P publishes all EXPOSE ports on random high ports
docker run -d -P nginx
docker ps
# Output: 0.0.0.0:32768->80/tcp  (random host port → container port 80)
```

---

### 2.8 `CMD` — Default Runtime Command

Specifies the **default command** to run when the container starts. Only the **last** `CMD` instruction takes effect.

```dockerfile
# Exec form (RECOMMENDED)
CMD ["python", "app.py"]

# Shell form
CMD python app.py

# With arguments
CMD ["nginx", "-g", "daemon off;"]
```

**Exec form vs Shell form:**

```dockerfile
# ✅ Exec form — runs directly as PID 1
CMD ["python", "app.py"]
# Process tree: python (PID 1)

# ❌ Shell form — runs via /bin/sh -c, PID 1 is sh
CMD python app.py
# Process tree: /bin/sh -c (PID 1) → python (PID 7)
```

**Why exec form matters:**
- PID 1 receives signals properly (SIGTERM, SIGINT)
- Shell form can cause containers to not stop gracefully
- Shell form is needed only when you need shell features (variables, pipes, etc.)

```dockerfile
# When you NEED shell form:
CMD python app.py --port $PORT --workers $WORKERS
CMD npm start | tee /var/log/app.log
```

**CMD can be overridden at runtime:**
```bash
# Dockerfile: CMD ["python", "app.py"]
docker run myapp                  # Runs: python app.py
docker run myapp python shell.py  # Runs: python shell.py (overrides CMD)
```

---

### 2.9 `ENTRYPOINT` — Fixed Command

Like `CMD`, but **cannot be overridden** by default runtime arguments. Use it for the "main command" of your container.

```dockerfile
# ENTRYPOINT + CMD combination
ENTRYPOINT ["python"]
CMD ["app.py"]

# docker run myapp               → python app.py
# docker run myapp script.py     → python script.py (CMD replaced, ENTRYPOINT kept)
```

**Common pattern: ENTRYPOINT for executable, CMD for defaults:**

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y curl

ENTRYPOINT ["curl"]
CMD ["--help"]

# docker run myimage              → curl --help
# docker run myimage google.com   → curl google.com
```

**Override ENTRYPOINT at runtime:**
```bash
docker run --entrypoint /bin/bash myapp -it
```

---

### 2.10 `USER` — Run as Non-Root

Sets the user for subsequent `RUN`, `CMD`, and `ENTRYPOINT` instructions.

```dockerfile
FROM python:3.12-slim

# Create a non-root user
RUN useradd -m appuser

# Switch to non-root user
USER appuser

WORKDIR /home/appuser
COPY --chown=appuser:appuser . .

CMD ["python", "app.py"]
```

**Why this matters:** Running as root inside a container is a security risk. If an attacker escapes the container, they may have root access to the host. (Covered in depth in Week 4, Day 4.)

---

### 2.11 `HEALTHCHECK` — Container Health

Tells Docker how to check if the container is still working properly.

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

| Flag | Default | Meaning |
|------|---------|---------|
| `--interval` | 30s | Time between checks |
| `--timeout` | 30s | Max time for a single check |
| `--start-period` | 0s | Grace period before checking starts |
| `--retries` | 3 | Failures before marking unhealthy |

```bash
# Check health status
docker ps
# Output: Up 5 minutes (health: starting)
# Output: Up 10 minutes (healthy)
# Output: Up 15 minutes (unhealthy)
```

---

### 2.12 `ARG` — Build-Time Variables

Variables that exist **only during build**, not at runtime.

```dockerfile
ARG APP_VERSION=1.0
ARG BUILD_DATE

FROM python:3.12-slim
LABEL version=${APP_VERSION}
LABEL build-date=${BUILD_DATE}

# Build with custom values:
docker build --build-arg APP_VERSION=2.0 --build-arg BUILD_DATE=2024-01-15 -t myapp .
```

**ARG vs ENV:**

| Feature | `ARG` | `ENV` |
|---------|-------|-------|
| Available during build | ✅ | ✅ |
| Available in running container | ❌ | ✅ |
| Set with `--build-arg` | ✅ | ❌ |
| Set with `-e` at runtime | ❌ | ✅ |

---

## 3. Complete Dockerfile Examples

### Example 1: Python Web App

```dockerfile
# Base image
FROM python:3.12-slim

# Metadata
LABEL maintainer="devops@mycompany.com"
LABEL description="Flask web application"

# Set working directory
WORKDIR /app

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    FLASK_APP=app.py \
    FLASK_ENV=production

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first (leverage Docker cache)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# Document port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

# Default command
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

### Example 2: Static Site with nginx

```dockerfile
FROM nginx:1.25-alpine

# Remove default nginx config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom config
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy static files
COPY dist/ /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Example 3: Go Application (Minimal Image)

```dockerfile
# Build stage
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .

# Runtime stage
FROM scratch
COPY --from=builder /app/myapp /myapp
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

This produces an image that's **only the binary** — no OS, no shell, nothing else. Total size: ~5-10 MB.

---

## 📝 Hands-On Exercises

### Exercise 1: Create a Simple Python Dockerfile

Create a directory and these files:

```bash
mkdir ~/docker-exercise && cd ~/docker-exercise
```

**`app.py`:**
```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class MyHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/plain')
        self.end_headers()
        self.wfile.write(b"Hello from inside a Docker container!\n")

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 8080), MyHandler)
    print("Server running on port 8080...")
    server.serve_forever()
```

**`Dockerfile`:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python", "app.py"]
```

Don't build yet — just make sure the files are correct.

### Exercise 2: Understand the Layers

For the Dockerfile above, answer these questions:
1. How many layers will the image have?
2. Which instruction creates the largest layer?
3. If you change `app.py` and rebuild, which layers are reused?

Write your answers down. We'll verify on Day 2.

### Exercise 3: Write a Dockerfile from Scratch

Create a Dockerfile for a simple Node.js application that:
- Uses Node.js 20 on Alpine
- Sets `/app` as working directory
- Copies `package.json` and runs `npm install`
- Copies the rest of the code
- Exposes port 3000
- Runs `node server.js`

---

## 🧠 Common Questions

**Q: Can I have multiple `FROM` instructions?**
A: Yes — that's called a multi-stage build (covered on Day 4). Only the last `FROM` determines the final image.

**Q: Do I need `RUN apt-get update` every time?**
A: Yes, if you're installing packages. But chain it with install and cleanup in a single `RUN` to avoid caching issues.

**Q: What's the difference between `CMD` and `ENTRYPOINT`?**
A: `CMD` provides default arguments that can be overridden. `ENTRYPOINT` is the fixed executable. Together, `ENTRYPOINT` + `CMD` gives you a command with defaults.

**Q: Why does Dockerfile order matter?**
A: Docker caches layers. Instructions that change frequently (like copying source code) should come last. Instructions that rarely change (like installing dependencies) should come first.

---

## ✅ Day 1 Summary

**What you learned today:**

- ✅ All major Dockerfile instructions (`FROM`, `WORKDIR`, `COPY`, `RUN`, `ENV`, `EXPOSE`, `CMD`, `ENTRYPOINT`, `USER`, `HEALTHCHECK`, `ARG`)
- ✅ Exec form vs shell form for `CMD` and `ENTRYPOINT`
- ✅ The difference between `ARG` and `ENV`
- ✅ Why `COPY` is preferred over `ADD`
- ✅ Best practices for writing efficient Dockerfiles
- ✅ Complete real-world Dockerfile examples

**Coming up tomorrow:** We'll learn how to build images from Dockerfiles, understand the build cache, and explore best practices for fast, efficient builds.

---

*💡 Tip: Dockerfiles are declarative — you describe the desired state, not the steps to get there. Think about what the final image should look like, not how to transform one image into another.*