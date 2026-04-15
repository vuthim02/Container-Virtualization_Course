# Week 2 • Day 2 • Beginner
# Building Images

## Progress Checklist
- [x] Day 1 — Dockerfile Basics
- [ ] Day 2 — Building Images
- [ ] Day 3 — Volume Management
- [ ] Day 4 — Image Optimization

---

← [Day 1](day1.md) | [Roadmap](../roadmap.md) | [Day 3 →](day3.md)

---

## 1. The `docker build` Command

You've written a Dockerfile. Now let's turn it into an image.

```bash
docker build [OPTIONS] PATH | URL | -
```

### The Anatomy of a Build

```bash
# Basic build
docker build -t myapp:v1 .

# What each part means:
#   -t myapp:v1  → Tag the image as "myapp" with tag "v1"
#   .            → Use the current directory as build context
```

### What Happens During Build

```
$ docker build -t myapp:v1 .

[+] Building 3/5 (5 steps)
 => [1/5] FROM python:3.12-slim                  ← Pulls base image (or uses cached)
 => [2/5] WORKDIR /app                           → Creates directory layer
 => [3/5] COPY app.py .                          → Copies file into image
 => [4/5] EXPOSE 8080                            → Sets metadata
 => [5/5] CMD ["python", "app.py"]               → Sets metadata
 => exporting to image                           → Finalizes image
 => => naming to docker.io/library/myapp:v1      ← Applies tag
 => => DONE
```

**Key insight:** Docker processes instructions **top to bottom**. Each instruction creates a layer. If a layer hasn't changed since the last build, Docker reuses the **cached** version.

---

## 2. Build Context — The Hidden Cost

When you run `docker build .`, Docker does this:

1. **Collects** all files in the current directory (recursively)
2. **Sends** them to the Docker daemon (can be megabytes or gigabytes)
3. **Builds** the image using only the files referenced by `COPY`/`ADD`

```bash
# Watch the build context size:
$ docker build -t myapp:v1 .

[+] Building 0.0s (1/1)
 => [internal] load build definition from Dockerfile        0.0s
 => => transferring dockerfile: 154B                        0.0s
 => [internal] load .dockerignore                           0.0s
 => [internal] load metadata for docker.io/library/python   0.5s
 => [1/3] FROM python:3.12-slim                             0.0s
 => [internal] load build context                           0.1s
 => => transferring context: 45.2MB                         ← Whoa! 45 MB?
```

**Why is the context 45 MB when my Dockerfile only copies one file?**

Because Docker sent **everything** in the directory — including `node_modules/`, `__pycache__/`, `.git/`, virtual environments, build artifacts, etc.

### The Solution: `.dockerignore`

Create a `.dockerignore` file in the same directory as your Dockerfile:

```gitignore
# Version control
.git
.gitignore

# Python
__pycache__
*.pyc
*.pyo
.venv
venv/
*.egg-info/
.pytest_cache/

# Node
node_modules/
npm-debug.log

# IDE
.vscode/
.idea/
*.swp
*.swo

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
```

**After adding `.dockerignore`:**
```bash
$ docker build -t myapp:v1 .
 => => transferring context: 2.1MB     ← Much better!
```

**Best Practice:** Always create a `.dockerignore` file. It makes builds faster, smaller, and more secure (no accidentally copying `.env` files with secrets).

---

## 3. Tagging Images

### Tag During Build

```bash
# Single tag
docker build -t myapp:v1 .

# Multiple tags at once
docker build -t myapp:v1 -t myapp:latest .

# Tag with registry
docker build -t registry.mycompany.com/myapp:v1 .

# Tag with organization
docker build -t myorg/myapp:v1 .

# Full path: registry/org/image:tag
docker build -t ghcr.io/myorg/myapp:v1.2.3 .
```

### Tag After Build

```bash
# Build first, tag later
docker build -t myapp:v1 .
docker tag myapp:v1 myapp:latest
docker tag myapp:v1 registry.mycompany.com/myapp:v1
```

### Image Naming Conventions

| Pattern | Example | When to Use |
|---------|---------|-------------|
| `image:tag` | `myapp:v1.2.3` | Local development |
| `org/image:tag` | `mycompany/myapp:v1` | Docker Hub |
| `registry/org/image:tag` | `ghcr.io/mycompany/myapp:v1` | Private registries |

**Tag naming best practices:**

```
# ✅ Good tags
v1.0.0          # Semantic versioning
1.0.0-alpine    # Version + base image
2024-01-15      # Date-based (for CI/CD)
abc123          # Git commit SHA
latest          # Only for convenience, never production

# ❌ Bad tags
new             # Meaningless
final           # There's always another "final"
test            # Too vague
latest-only     # Confusing
```

---

## 4. Understanding the Build Cache

Docker caches every layer. If a layer hasn't changed, Docker reuses the cached version instead of rebuilding it.

### How Cache Works

```dockerfile
FROM python:3.12-slim       ← Layer 1: cached (base image rarely changes)
WORKDIR /app                ← Layer 2: cached (always the same)
COPY requirements.txt .     ← Layer 3: cached (if requirements.txt unchanged)
RUN pip install ...         ← Layer 4: cached (if layer 3 is cached)
COPY . .                    ← Layer 5: rebuilt (source code changes often)
CMD ["python", "app.py"]   ← Layer 6: cached (metadata)
```

**The cache breaks at the FIRST changed layer, and everything after it rebuilds.**

```
Build 1:
COPY requirements.txt .   ← No change → CACHED
RUN pip install ...       ← No change → CACHED
COPY . .                  ← Changed → REBUILT
CMD ["python", "app.py"]  ← After rebuild → REBUILT

Build 2:
COPY requirements.txt .   ← Changed → REBUILT
RUN pip install ...       ← After rebuild → REBUILT (reinstalls all packages!)
COPY . .                  ← After rebuild → REBUILT
CMD ["python", "app.py"]  ← After rebuild → REBUILT
```

**This is why order matters!** Copy dependency files first, then install, then copy source code.

```dockerfile
# ✅ OPTIMAL ORDER (fast builds)
COPY requirements.txt .     ← Rarely changes → cache hits
RUN pip install ...         ← Rarely changes → cache hits
COPY . .                    ← Often changes → cache miss (but only this layer)

# ❌ WORST ORDER (slow builds)
COPY . .                    ← Often changes → cache miss EVERY time
COPY requirements.txt .     ← After rebuild → always rebuilds
RUN pip install ...         ← After rebuild → always reinstalls everything
```

### Controlling the Cache

```bash
# Build without using any cache (rebuild everything)
docker build --no-cache -t myapp:v1 .

# Use cache from a specific image
docker build --cache-from myapp:previous -t myapp:v1 .

# BuildKit: only invalidate specific stages
# (Add this to the instruction you want to rebuild)
RUN --no-cache pip install some-new-package
```

---

## 5. Viewing and Managing Images

### List Images

```bash
# Basic list
docker images

# Include intermediate images (build artifacts)
docker images -a

# Show digests
docker images --digests

# Quiet mode (IDs only, for scripting)
docker images -q

# Custom format
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedAt}}"
```

**Example output:**
```
REPOSITORY    TAG         IMAGE ID       CREATED          SIZE
myapp         v1          a1b2c3d4e5f6   30 seconds ago   145MB
myapp         latest      a1b2c3d4e5f6   30 seconds ago   145MB
python        3.12-slim   4b99c4cc0f27   3 days ago       123MB
```

### Image History

```bash
# See all layers in an image
docker history myapp:v1

# Human-readable, no truncation
docker history --no-trunc myapp:v1

# Example output:
# IMAGE          CREATED              CREATED BY                                      SIZE
# a1b2c3d4e5f6   30 seconds ago       CMD ["python" "app.py"]                         0B
# <missing>      30 seconds ago       COPY . . # buildkit                             2.1MB
# <missing>      2 minutes ago        RUN /bin/sh -c pip install -r requirements.txt  45MB
# <missing>      2 minutes ago        COPY requirements.txt . # buildkit              154B
# <missing>      2 minutes ago        WORKDIR /app                                    0B
# <missing>      3 days ago           CMD ["python3"]                                 0B
# <missing>      3 days ago           # (base image layers...)                        123MB
```

### Inspect Images

```bash
# Full JSON metadata
docker image inspect myapp:v1

# Get specific info
docker inspect myapp:v1 --format '{{.Size}}'
docker inspect myapp:v1 --format '{{.Config.Env}}'
docker inspect myapp:v1 --format '{{.ContainerConfig.Cmd}}'
```

### Remove Images

```bash
# Remove by name:tag
docker rmi myapp:v1

# Remove by image ID
docker rmi a1b2c3d4e5f6

# Remove all dangling images (untagged)
docker image prune

# Remove ALL unused images
docker image prune -a

# Force remove (even if containers reference it)
docker rmi -f myapp:v1
```

---

## 6. Build Arguments and Variables

### Build-Time Variables (`ARG`)

```dockerfile
FROM python:3.12-slim

# Define build argument with default value
ARG APP_VERSION=1.0
ARG BUILD_ENV=production

# Use during build
LABEL version=$APP_VERSION
ENV APP_ENV=$BUILD_ENV

RUN echo "Building version $APP_VERSION for $BUILD_ENV"

COPY . /app
```

```bash
# Build with defaults
docker build -t myapp:v1 .

# Build with custom values
docker build \
  --build-arg APP_VERSION=2.0 \
  --build-arg BUILD_ENV=staging \
  -t myapp:v2 .
```

### Important: ARG Scope

```dockerfile
# ❌ ARG before FROM is only available for the FROM line
ARG BASE_IMAGE_TAG=3.12-slim
FROM python:$BASE_IMAGE_TAG    # ✅ Works here

WORKDIR /app
# ❌ $BASE_IMAGE_TAG is NOT available here (scope ends after FROM)

# ✅ ARG after FROM is available for all subsequent instructions
FROM python:3.12-slim
ARG APP_VERSION=1.0            # Available for everything below
ENV VERSION=$APP_VERSION
```

### Passing Secrets During Build

For sensitive build-time data (API keys, tokens), use **BuildKit secrets**:

```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1

# Pass secret as a file
echo "my-secret-token" > /tmp/token.txt
docker build --secret id=mytoken,src=/tmp/token.txt -t myapp .

# In Dockerfile (syntax directive at top):
# syntax=docker/dockerfile:1
FROM python:3.12-slim
RUN --mount=type=secret,id=mytoken \
    TOKEN=$(cat /run/secrets/mytoken) && echo "Token: $TOKEN"
```

---

## 7. Build Platforms (Cross-Platform Builds)

Build images for different CPU architectures:

```bash
# Build for ARM64 (e.g., Apple Silicon, Raspberry Pi)
docker build --platform linux/arm64 -t myapp:arm64 .

# Build for AMD64 (standard x86_64)
docker build --platform linux/amd64 -t myapp:amd64 .

# Build for multiple platforms
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myapp:latest \
  --push .
```

Check your current platform:
```bash
docker info --format '{{.OperatingSystem}} {{.Architecture}}'
uname -m
# x86_64 = amd64, aarch64 = arm64
```

---

## 8. BuildKit — The Modern Build Engine

BuildKit is Docker's next-generation build engine. It's enabled by default in Docker 23.0+.

### Benefits of BuildKit
- **Faster builds** (better caching, parallel execution)
- **Advanced features** (secrets, SSH forwarding, conditional builds)
- **Better output** (cleaner progress display)
- **Smaller images** (better layer optimization)

### Enable BuildKit

```bash
# Check if it's enabled
docker build --help | grep buildkit

# Enable per-build
DOCKER_BUILDKIT=1 docker build -t myapp .

# Enable globally (add to ~/.docker/config.json)
{
  "features": {
    "buildkit": true
  }
}

# Or set environment variable
export DOCKER_BUILDKIT=1
```

### BuildKit Syntax Extensions

Add this to the top of your Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim

# Now you can use BuildKit features:
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

The `--mount=type=cache` creates a persistent cache that survives between builds. Pip packages are cached, making subsequent builds much faster.

---

## 📝 Hands-On Exercises

### Exercise 1: Build the Python Dockerfile from Day 1

```bash
# Navigate to your exercise directory
cd ~/docker-exercise

# Make sure you have the Dockerfile and app.py from Day 1

# Build the image
docker build -t my-python-app:v1 .

# Watch the build output carefully:
# - Check the context size (should be small if .dockerignore is used)
# - Watch each step execute
# - Note which layers are cached vs rebuilt

# Verify the build
docker images | grep my-python-app

# Run the container
docker run -d -p 8080:8080 --name test-app my-python-app:v1

# Test it
curl http://localhost:8080

# View logs
docker logs test-app

# Clean up
docker rm -f test-app
```

### Exercise 2: Understand the Cache

```bash
# Build again (should be fast — all cached)
docker build -t my-python-app:v2 .

# Now modify app.py
echo '# Modified' >> app.py

# Build again — which layers rebuild?
docker build -t my-python-app:v3 .

# Now add a .dockerignore if you don't have one
cat > .dockerignore << 'EOF'
*.pyc
__pycache__
.venv
.git
EOF

# Build again — check the context size
docker build -t my-python-app:v4 .
```

### Exercise 3: Build with Build Arguments

Create this Dockerfile:

```dockerfile
FROM python:3.12-slim
ARG APP_VERSION=1.0
ARG APP_NAME=myapp
LABEL version=$APP_VERSION
LABEL name=$APP_NAME
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

```bash
# Build with default values
docker build -t myapp:default .

# Build with custom values
docker build \
  --build-arg APP_VERSION=2.0 \
  --build-arg APP_NAME=custom-app \
  -t myapp:custom .

# Verify labels
docker inspect myapp:custom --format '{{json .Config.Labels}}'
```

### Exercise 4: Practice Image Management

```bash
# List all your images
docker images

# See history of your image
docker history my-python-app:v1

# Tag it multiple ways
docker tag my-python-app:v1 my-python-app:latest
docker tag my-python-app:v1 my-python-app:stable

# List again — note same IMAGE ID with different tags
docker images

# Remove one tag
docker rmi my-python-app:stable

# Remove all unused images
docker image prune -a
```

---

## 🧠 Common Questions

**Q: Why is my build so slow?**
A: Most likely, you're copying all source code before installing dependencies. Reorder your Dockerfile: copy requirements → install → copy source.

**Q: Why is my image so large?**
A: Check with `docker history`. Look for large layers (package installations, copied files). See Day 4 for optimization techniques.

**Q: Can I use a Dockerfile from a different directory?**
A: Yes: `docker build -f /path/to/Dockerfile -t myapp .` (The `.` is still the build context.)

**Q: What's the difference between `docker build` and `docker image build`?**
A: They're the same command. `docker image build` is the more explicit form.

**Q: How do I see all intermediate layers?**
A: `docker images -a` shows them. `docker history` shows them for a specific image.

---

## ✅ Day 2 Summary

**What you learned today:**

- ✅ How `docker build` works and what happens during a build
- ✅ Build context and why `.dockerignore` is essential
- ✅ Image tagging strategies and naming conventions
- ✅ How Docker's layer caching works and how to optimize for cache hits
- ✅ Viewing and managing images (`images`, `history`, `inspect`, `rmi`)
- ✅ Build arguments (`ARG`) and passing secrets
- ✅ Cross-platform builds and BuildKit

**Coming up tomorrow:** We'll learn about Docker volumes — how to persist data beyond the container lifecycle.

---

*💡 Tip: The build cache is your friend. Structure your Dockerfile so that frequently-changing instructions come last. This makes rebuilds nearly instant.*