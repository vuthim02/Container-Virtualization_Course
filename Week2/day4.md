# Week 2 • Day 4 • Beginner
# Image Optimization

## Progress Checklist
- [x] Day 1 — Dockerfile Basics
- [x] Day 2 — Building Images
- [x] Day 3 — Volume Management
- [ ] Day 4 — Image Optimization

---

← [Day 3](day3.md) | [Roadmap](../roadmap.md) | [Week 3 Day 1 →](../Week3/day1.md)

---

## 1. Why Image Size Matters

You might be thinking: *"Disk space is cheap. Why should I care about image size?"*

Here's why:

| Factor | Small Image (50 MB) | Bloated Image (800 MB) |
|--------|---------------------|------------------------|
| **Pull time** | 5 seconds | 80 seconds |
| **Push time** | 3 seconds | 50 seconds |
| **Deployment speed** | Fast rollouts | Slow deployments |
| **CI/CD cost** | Less bandwidth, faster builds | More bandwidth, slower builds |
| **Attack surface** | Fewer packages, fewer vulnerabilities | More packages = more CVEs |
| **Storage costs** | Cheap | Expensive at scale |

At scale, the difference is enormous:

```
100 containers × 800 MB = 80 GB transferred per deployment
100 containers × 50 MB  =  5 GB transferred per deployment

Difference: 75 GB per deployment × multiple deployments per day = massive cost
```

### Security Perspective

```
Ubuntu-based image:          Alpine-based image:
┌───────────────────────┐    ┌───────────────────────┐
│  Your app             │    │  Your app             │
│  + 400 system packages│    │  + 14 system packages │
│  + curl, vim, ssh     │    │  + nothing extra      │
│  + compilers          │    │                       │
│  + documentation      │    │                       │
│  + man pages          │    │                       │
│                       │    │                       │
│  Attack surface:      │    │  Attack surface:      │
│  LARGE ❌             │    │  MINIMAL ✅           │
└───────────────────────┘    └───────────────────────┘
```

Every extra package in your image is a potential vulnerability.

---

## 2. Strategy 1: Choose the Right Base Image

The base image is the biggest factor in your final image size.

### Common Base Images Compared

| Base Image | Size | Package Manager | libc | When to Use |
|------------|------|----------------|------|-------------|
| `ubuntu:24.04` | ~78 MB | apt | glibc | Full Ubuntu ecosystem needed |
| `debian:bookworm-slim` | ~75 MB | apt | glibc | Debian, smaller than Ubuntu |
| `alpine:3.19` | ~7 MB | apk | musl | Minimal size (beware compatibility) |
| `scratch` | 0 bytes | N/A | None | Statically compiled binaries |
| `distroless` | ~2-20 MB | N/A | glibc | Runtime-only, Google-maintained |
| `wolfi` | ~8 MB | apk | glibc | Minimal + glibc compatibility |

### Size Comparison for the Same App

```dockerfile
# Ubuntu: ~170 MB total
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y python3
# Total: ~170 MB

# Debian slim: ~150 MB total
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y python3
# Total: ~150 MB

# Alpine: ~50 MB total
FROM alpine:3.19
RUN apk add --no-cache python3
# Total: ~50 MB
```

### Alpine: The musl libc Gotcha

Alpine uses **musl libc** instead of the more common **glibc**. Most packages work fine, but some have issues:

```dockerfile
# ❌ This might not work on Alpine (glibc-specific features)
FROM alpine:3.19
RUN apk add some-glibc-only-package

# ✅ Use debian-slim if you need glibc compatibility
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y some-package
```

**Common Alpine issues:**
- Some Python packages with C extensions fail to compile
- Some Node.js native modules don't work
- `glibc`-specific tools (like `gdb`) are unavailable

**When in doubt:** Use `debian:bookworm-slim` for a good balance of size and compatibility.

---

## 3. Strategy 2: Reduce Layers

Every instruction in a Dockerfile creates a new layer. Each layer adds to the image size.

### Combine Related Commands

```dockerfile
# ❌ BAD — 3 layers, middle layers are wasted space
FROM ubuntu:24.04
RUN apt-get update
RUN apt-get install -y curl git vim
RUN apt-get clean

# ✅ GOOD — 1 layer, cache cleaned in same layer
FROM ubuntu:24.04
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      git \
      vim \
    && rm -rf /var/lib/apt/lists/*
```

### Why This Works

```
Layer analysis of the BAD example:
Layer 1: apt-get update          → 25 MB (package lists)
Layer 2: apt-get install ...     → 100 MB (packages + cached .deb files)
Layer 3: apt-get clean           → 0 MB (can't remove previous layers' data!)
Total: 125 MB

Layer analysis of the GOOD example:
Layer 1: update + install + clean → 100 MB (packages only, cache cleaned before commit)
Total: 100 MB
```

**Key rule:** Cleanup must happen in the **same `RUN` instruction** as the installation.

### Package Manager Cleanup Cheatsheet

```dockerfile
# apt (Debian/Ubuntu)
RUN apt-get update && apt-get install -y package \
    && rm -rf /var/lib/apt/lists/*

# apk (Alpine)
RUN apk add --no-cache package
# --no-cache flag skips downloading the index, saving ~5-10 MB

# yum (CentOS/RHEL)
RUN yum install -y package \
    && yum clean all

# dnf (Fedora)
RUN dnf install -y package \
    && dnf clean all
```

---

## 4. Strategy 3: Multi-Stage Builds

Multi-stage builds are the most powerful optimization technique in Docker. They let you use one image to **build** your app and a completely different (smaller) image to **run** it.

### The Problem

```dockerfile
# Traditional approach: one stage
FROM node:20                    ← 1 GB image with Node, npm, compilers, etc.
WORKDIR /app
COPY package*.json ./
RUN npm install                 ← Builds native modules
COPY . .
RUN npm run build               ← Produces dist/ directory
CMD ["node", "dist/server.js"]

# Final image: ~1.2 GB
# Contains: Node.js, npm, build tools, source code, dev dependencies
# Only needs: Node.js runtime + dist/ directory
```

### The Multi-Stage Solution

```dockerfile
# Stage 1: Build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
# This stage has all the build tools, source code, dev dependencies
# Final artifact: /app/dist/

# Stage 2: Run
FROM node:20-alpine
WORKDIR /app

# Copy ONLY the built application from the builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Runtime doesn't have source code, build tools, or dev dependencies
EXPOSE 3000
CMD ["node", "dist/server.js"]

# Final image: ~180 MB (vs 1.2 GB before!)
```

**Visual:**
```
Stage 1 (builder):              Stage 2 (production):
┌─────────────────────┐         ┌─────────────────────┐
│ node:20 (1 GB)      │         │ node:20-alpine (180MB)│
│ source code         │         │ dist/ (from builder) │
│ dev dependencies    │    ───▶ │ node_modules (prod)  │
│ build tools         │  COPY   │                      │
│ node_modules (all)  │         │ CMD ["node", ...]    │
│                     │         │                      │
│ npm run build       │         │ Final: 180 MB        │
│                     │         └─────────────────────┘
└─────────────────────┘
```

### Go Application — From 800 MB to 10 MB

```dockerfile
# Stage 1: Build
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /myapp .

# Stage 2: Run (scratch = empty image, 0 bytes)
FROM scratch
# For Go, we need SSL certs (statically linked binary)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/myapp /myapp

EXPOSE 8080
ENTRYPOINT ["/myapp"]

# Final image: ~10 MB (just the binary + SSL certs)
# Previous single-stage: ~800 MB (full Go toolchain + source code)
```

### Python + Multi-Stage

```dockerfile
# Stage 1: Install dependencies
FROM python:3.12-slim AS dependencies
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: Runtime
FROM python:3.12-slim
WORKDIR /app

# Copy installed packages from dependencies stage
COPY --from=dependencies /install /usr/local

# Copy only application code (not requirements.txt, not build cache)
COPY app.py .

EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

### Naming Stages

You can name stages for clarity:

```dockerfile
FROM golang:1.22 AS builder
# ...

FROM node:20-alpine AS production
# ...

FROM alpine:3.19 AS test
# ...

# Build a specific stage only:
docker build --target builder -t myapp:build .
```

### Build Only Up to a Specific Stage

```dockerfile
FROM golang:1.22 AS builder
# Build stage...

FROM builder AS test
# Run tests...

FROM alpine AS production
# Final image...
```

```bash
# Build only the builder stage (for debugging)
docker build --target builder -t myapp:debug .

# Build the test stage
docker build --target test -t myapp:test .

# Build the production image (default)
docker build -t myapp:prod .
```

---

## 5. Strategy 4: .dockerignore (Revisited)

We covered `.dockerignore` on Day 2, but its impact on image size is worth emphasizing.

### What Happens Without `.dockerignore`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
# This copies EVERYTHING in the build context:
# - .git/ (could be 100+ MB)
# - node_modules/ (could be 500+ MB)
# - __pycache__/ (megabytes of compiled Python)
# - .venv/ (hundreds of MB of installed packages)
# - *.pyc files (compiled bytecode)
# - Build artifacts, temp files, logs
```

### Essential `.dockerignore`

```gitignore
# Version control
.git
.github/
.gitignore

# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
.venv/
venv/
*.egg-info/
.pytest_cache/
.mypy_cache/

# Node
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Docker
Dockerfile
docker-compose*.yml
.dockerignore

# Build artifacts
dist/
build/
*.tar.gz
*.zip

# Environment files (secrets!)
.env
.env.*
*.pem
*.key
```

---

## 6. Strategy 5: Distroless Images

Google's **distroless** images contain only your application and its runtime dependencies. No package manager, no shell, no tools.

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app .

FROM gcr.io/distroless/static
COPY --from=builder /app /app
ENTRYPOINT ["/app"]

# Image size: ~5 MB
# No shell, no package manager, no tools
# Very small attack surface
```

**Available distroless images:**

| Image | Contents | Size |
|-------|----------|------|
| `distroless/static` | Nothing extra | ~2 MB |
| `distroless/base` | glibc, SSL certs | ~17 MB |
| `distroless/cc` | + C/C++ runtime | ~20 MB |
| `distroless/java` | JRE only | ~80 MB |
| `distroless/python3` | Python 3 runtime | ~55 MB |
| `distroless/nodejs` | Node.js runtime | ~60 MB |

**Trade-off:** No shell means you can't `docker exec` into the container for debugging. You need to rely on logs and external monitoring.

---

## 7. Analyzing Image Size

### `docker history` — See What Takes Space

```bash
docker history myapp:latest

# IMAGE          CREATED        CREATED BY                                       SIZE
# a1b2c3d4e5f6   2 hours ago    CMD ["node" "dist/server.js"]                    0B
# <missing>      2 hours ago    COPY --from=builder /app/dist ./dist             2.1MB
# <missing>      2 hours ago    COPY --from=builder /app/node_modules ./node_…   45MB
# <missing>      2 hours ago    WORKDIR /app                                     0B
# <missing>      3 days ago     node:20-alpine                                   130MB
```

### `dive` — Interactive Layer Explorer

[dive](https://github.com/wagoodman/dive) is a fantastic tool for exploring Docker images.

```bash
# Install dive
# macOS: brew install dive
# Linux: Download from GitHub releases

# Analyze an image
dive myapp:latest

# CI mode — fail build if inefficiency is too high
dive myapp:latest --ci --threshold 0.90
```

### Docker Scout — Vulnerability Scanning

```bash
# Scan image for vulnerabilities
docker scout cves myapp:latest

# Show recommendations
docker scout recommendations myapp:latest
```

---

## 8. Cleaning Up Docker

```bash
# Remove dangling images (untagged)
docker image prune

# Remove ALL unused images
docker image prune -a

# Remove everything: stopped containers, unused networks, dangling images, build cache
docker system prune

# Remove everything including unused images
docker system prune -a

# Remove everything including volumes (⚠️ this deletes persistent data!)
docker system prune -a --volumes

# Check what you'll remove before doing it:
docker system df
docker system df -v
```

### Automated Cleanup in CI/CD

```bash
# After a build pipeline:
docker image prune -f          # Remove dangling images
docker builder prune -f        # Remove build cache
docker container prune -f      # Remove stopped containers
```

---

## 9. Complete Optimized Dockerfile Examples

### Node.js Application (Production)

```dockerfile
# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Production
FROM node:20-alpine AS production
WORKDIR /app

# Copy production dependencies
COPY --from=deps /app/node_modules ./node_modules

# Copy built application
COPY --from=builder /app/dist ./dist

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/server.js"]

# Total size: ~180 MB (vs ~1.2 GB unoptimized)
```

### Python Flask Application

```dockerfile
FROM python:3.12-slim

# System deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Python deps first (better caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application code
COPY app.py .
COPY templates/ ./templates/
COPY static/ ./static/

# Non-root user
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "app:app"]

# Total size: ~150 MB (vs ~300+ MB unoptimized)
```

---

## 📝 Hands-On Exercises

### Exercise 1: Compare Base Image Sizes

```bash
# Pull three different base images
docker pull ubuntu:24.04
docker pull debian:bookworm-slim
docker pull alpine:3.19

# Compare sizes
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | grep -E "ubuntu|debian|alpine"

# Expected sizes:
# ubuntu:24.04          ~78 MB
# debian:bookworm-slim  ~75 MB
# alpine:3.19           ~7 MB
```

### Exercise 2: Optimize a Bad Dockerfile

Create this "bad" Dockerfile first:

```bash
mkdir ~/optimize-exercise && cd ~/optimize-exercise

cat > app.py << 'EOF'
print("Hello from optimized container!")
EOF

cat > requirements.txt << 'EOF'
flask==3.0.0
EOF
```

**Bad Dockerfile (`Dockerfile.bad`):**
```dockerfile
FROM ubuntu:24.04
WORKDIR /app
RUN apt-get update
RUN apt-get install -y python3 python3-pip
COPY . .
RUN pip3 install -r requirements.txt
CMD ["python3", "app.py"]
```

**Build it:**
```bash
docker build -f Dockerfile.bad -t myapp:bad .
docker images myapp:bad
```

Now create an optimized version (`Dockerfile.good`):

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system deps and clean in one layer
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps first (cache benefit)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Then copy app
COPY app.py .

# Non-root
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

```bash
docker build -f Dockerfile.good -t myapp:good .
docker images myapp:bad myapp:good
```

**Compare the sizes!** The good version should be 30-50% smaller.

### Exercise 3: Multi-Stage Build for a Go App

```bash
mkdir ~/go-multistage && cd ~/go-multstage

# Create a simple Go app
cat > main.go << 'EOF'
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello from Go!")
}

func main() {
    http.HandleFunc("/", handler)
    fmt.Println("Server running on :8080")
    http.ListenAndServe(":8080", nil)
}
EOF

cat > go.mod << 'EOF'
module myapp

go 1.22
EOF
```

**Single-stage Dockerfile:**
```dockerfile
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o myapp .
CMD ["./myapp"]
```

```bash
docker build -f Dockerfile.single -t mygo:single .
docker images mygo:single
# Should be ~800 MB
```

**Multi-stage Dockerfile:**
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /myapp .

FROM alpine:3.19
COPY --from=builder /myapp /myapp
EXPOSE 8080
CMD ["/myapp"]
```

```bash
docker build -f Dockerfile.multi -t mygo:multi .
docker images mygo:multi
# Should be ~10-15 MB!
```

### Exercise 4: Clean Up

```bash
# Check current disk usage
docker system df

# Remove all exercise images
docker rmi myapp:bad myapp:good mygo:single mygo:multi

# Clean up everything unused
docker system prune -f

# Verify
docker system df
```

---

## 🧠 Common Questions

**Q: Should I always use Alpine?**
A: Not always. Use Alpine when your app works with musl libc. Use `debian:bookworm-slim` when you need glibc compatibility. Use `scratch` or `distroless` for statically compiled binaries (Go, Rust).

**Q: Do multi-stage builds increase build time?**
A: The first build takes the same time. But subsequent builds benefit from caching in each stage independently. The production stage is often very fast to rebuild.

**Q: Can I debug containers with no shell (distroless)?**
A: Yes, but you need different tools:
- Use `kubectl debug` (Kubernetes) with an ephemeral debug container
- Rely on application logs (stdout/stderr)
- Use profiling endpoints and external monitoring tools

**Q: What's the smallest possible Docker image?**
A: `FROM scratch` with a statically compiled Go binary can be as small as 2-10 MB. The world record for a useful container is around 100 KB (a statically compiled "Hello World" in C).

**Q: Should I remove apt/apk caches?**
A: Yes, always. But only if you do it in the same `RUN` instruction. For Alpine, use `--no-cache` with `apk add` instead of manual cleanup.

---

## ✅ Day 4 Summary — Week 2 Complete!

**What you learned today:**

- ✅ Why image size matters (speed, cost, security)
- ✅ Choosing the right base image (Ubuntu vs Debian vs Alpine vs scratch vs distroless)
- ✅ Reducing layers by combining commands and cleaning caches
- ✅ Multi-stage builds for dramatically smaller images
- ✅ `.dockerignore` best practices
- ✅ Distroless images for minimal attack surface
- ✅ Analyzing image size with `docker history`, `dive`, and `docker scout`
- ✅ Cleaning up Docker disk usage

**🎉 Week 2 Complete!**

You now know how to:
- ✅ Write efficient Dockerfiles
- ✅ Build images quickly with optimal caching
- ✅ Persist data with volumes and bind mounts
- ✅ Optimize images for size, speed, and security

**Coming up in Week 3:** Docker Compose — managing multi-container applications with a single configuration file.

---

*💡 Tip: Image optimization is a balance. Don't sacrifice readability and maintainability for a few megabytes. Focus on the big wins: base image choice, multi-stage builds, and layer ordering.*