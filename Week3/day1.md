# Week 3 • Day 1 • Intermediate
# Introduction to Docker Compose

## Progress Checklist
- [x] Day 1 — Introduction to Compose
- [ ] Day 2 — Multi-container Apps
- [ ] Day 3 — Networking in Compose
- [ ] Day 4 — Advanced Compose Patterns

---

← [Roadmap](../roadmap.md) | [Day 2 →](day2.md)

---

## 1. The Multi-Container Problem

So far, you've been running single containers with long `docker run` commands. But real applications have **multiple services**:

```
A typical web application:
┌──────────┐    ┌──────────┐    ┌──────────┐
│  Frontend│───▶│   API    │───▶│ Database │
│  (nginx) │    │  (Node)  │    │(Postgres)│
└──────────┘    └──────────┘    └──────────┘
                      │
                      ▼
                ┌──────────┐
                │   Redis  │
                │  (Cache) │
                └──────────┘
```

### Without Compose

Starting all these services manually:

```bash
# Start PostgreSQL
docker run -d \
  --name myapp-db \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_USER=myapp \
  -e POSTGRES_PASSWORD=secret123 \
  -e POSTGRES_DB=myapp \
  --network myapp-net \
  postgres:16-alpine

# Start Redis
docker run -d \
  --name myapp-redis \
  --network myapp-net \
  redis:7-alpine

# Build and start the API
docker build -t myapp-api .
docker run -d \
  --name myapp-api \
  -p 3000:3000 \
  -e DATABASE_URL=postgres://myapp:secret123@myapp-db:5432/myapp \
  -e REDIS_URL=redis://myapp-redis:6379 \
  --network myapp-net \
  myapp-api

# Start the frontend
docker run -d \
  --name myapp-frontend \
  -p 80:80 \
  -e API_URL=http://myapp-api:3000 \
  --network myapp-net \
  nginx:alpine

# Now hope you got everything right...
```

**Problems:**
- 4 separate commands to remember
- Easy to make a typo in any flag
- Hard to share with your team
- Stopping everything requires 4 `docker stop` commands
- No single source of truth for the infrastructure

### With Compose

One file, one command:

```bash
docker compose up -d
```

---

## 2. What is Docker Compose?

**Docker Compose** is a tool for defining and running multi-container Docker applications. You describe your entire application in a single YAML file, then start everything with one command.

```
docker-compose.yml
├── Service: web (nginx)
├── Service: api (Node.js)
├── Service: db (PostgreSQL)
├── Service: redis (Redis)
├── Network: default
└── Volumes: pgdata
```

### Compose vs Docker CLI

| Feature | Docker CLI | Docker Compose |
|---------|-----------|----------------|
| Containers | One at a time | All at once |
| Configuration | Command-line flags | YAML file |
| Networking | Manual setup | Automatic |
| Volumes | Manual creation | Defined in YAML |
| Sharing | Share commands | Share YAML file |
| Reproducibility | Error-prone | Guaranteed |

---

## 3. Compose File Structure

A `docker-compose.yml` file has three main sections:

```yaml
# Services: the containers you want to run
services:
  web:
    ...
  api:
    ...
  db:
    ...

# Networks: how services communicate
networks:
  ...

# Volumes: persistent data storage
volumes:
  ...
```

### Services

Each service is a container (or a group of containers running the same image):

```yaml
services:
  service_name:        # Your chosen name (e.g., web, api, db)
    image: nginx       # Image to use
    ports:
      - "80:80"        # Host:Container port mapping
    environment:       # Environment variables
      - KEY=value
    volumes:           # Mount points
      - data:/app/data
```

### Networks (Optional)

```yaml
networks:
  frontend:           # Custom network name
    driver: bridge    # Default: bridge network
  backend:
    driver: bridge
```

### Volumes (Optional)

```yaml
volumes:
  pgdata:            # Named volume
  redis-data:
```

---

## 4. Your First Compose File

### Example 1: Two nginx Containers

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro

  api:
    image: nginx:alpine
    ports:
      - "8081:80"
    volumes:
      - ./api-html:/usr/share/nginx/html:ro
```

**Directory structure:**
```
my-compose-project/
├── docker-compose.yml
├── html/
│   └── index.html     ← "Hello from web!"
└── api-html/
    └── index.html     ← "Hello from API!"
```

### Running It

```bash
# Navigate to the directory with docker-compose.yml
cd my-compose-project

# Start all services in detached mode (background)
docker compose up -d

# Output:
# [+] Running 3/3
#  ✔ Network my-compose-project_default    Created
#  ✔ Container my-compose-project-web-1     Started
#  ✔ Container my-compose-project-api-1     Started

# Check what's running
docker compose ps

# Output:
# NAME                        IMAGE           STATUS          PORTS
# my-compose-project-api-1    nginx:alpine    Up 10 seconds   0.0.0.0:8081->80/tcp
# my-compose-project-web-1    nginx:alpine    Up 10 seconds   0.0.0.0:8080->80/tcp

# Visit both:
curl http://localhost:8080    # Web page
curl http://localhost:8081    # API page
```

### Stopping

```bash
# Stop and remove all containers, networks
docker compose down

# Stop and remove containers, networks, AND volumes
docker compose down --volumes

# Stop and remove containers, networks, volumes, AND images
docker compose down --volumes --rmi all
```

---

## 5. Essential Compose Commands

### Lifecycle Commands

```bash
# Start all services (creates if needed)
docker compose up

# Start in background (detached)
docker compose up -d

# Start specific services only
docker compose up -d web api

# Start and force rebuild (if using build:)
docker compose up -d --build

# Start without recreating existing containers
docker compose up -d --no-recreate

# Stop all services
docker compose stop

# Start stopped services
docker compose start

# Restart services
docker compose restart

# Stop and remove everything
docker compose down
```

### Monitoring Commands

```bash
# List services
docker compose ps

# List all services (including stopped)
docker compose ps -a

# View logs from all services
docker compose logs

# View logs and follow (like tail -f)
docker compose logs -f

# View last 50 lines of logs
docker compose logs --tail=50

# View logs for a specific service
docker compose logs web
docker compose logs -f api

# View resource usage
docker compose top

# Show port bindings
docker compose port web 80
```

### Build Commands

```bash
# Build images for services that use `build:`
docker compose build

# Build without cache
docker compose build --no-cache

# Build a specific service
docker compose build api

# Pull latest images for services that use `image:`
docker compose pull
```

### Other Commands

```bash
# Execute a command in a running service
docker compose exec api bash
docker compose exec db psql -U postgres

# Run a one-off command (new container)
docker compose run --rm api python manage.py migrate

# Validate compose file syntax
docker compose config

# Show the resolved compose config (with defaults filled in)
docker compose config --format json

# Pause all services
docker compose pause

# Unpause
docker compose unpause

# Show images used by services
docker compose images
```

---

## 6. Compose File Versions

You may see different `version` numbers in compose files:

```yaml
version: '3.8'    # Most common
version: '2.4'    # Legacy
# No version key  # Modern Compose (recommended)
```

### The Current State (2024+)

**The `version` key is deprecated** in the Compose specification. Modern Docker uses the Compose Specification by default:

```yaml
# Modern approach (recommended)
# No version key needed — just start with services:
services:
  web:
    image: nginx
```

**Why?** Docker now uses a unified specification that supports all features. Old version numbers were tied to specific Docker releases.

| Version | Docker Engine | Notes |
|---------|--------------|-------|
| `3.8` | 19.03+ | Most common in existing projects |
| `3.7` | 18.06+ | Legacy |
| `2.4` | 17.12+ | Legacy, different syntax |
| **None** | **20.10+** | **Compose Specification (recommended)** |

---

## 7. Compose File: `image` vs `build`

### Using `image` — Pull from Registry

```yaml
services:
  web:
    image: nginx:alpine          # Pull from Docker Hub

  redis:
    image: redis:7-alpine        # Pull from Docker Hub

  private-app:
    image: ghcr.io/myorg/myapp:v1  # Pull from private registry
```

### Using `build` — Build from Dockerfile

```yaml
services:
  api:
    build: .                     # Build from Dockerfile in current directory

  worker:
    build: ./worker              # Build from Dockerfile in ./worker directory

  frontend:
    build:
      context: ./frontend        # Build context
      dockerfile: Dockerfile.dev # Use a different Dockerfile name
      args:                      # Build arguments
        NODE_ENV: development
```

### Using Both — Build and Tag

```yaml
services:
  api:
    build: .
    image: myregistry.com/myapp:latest

# This builds the image AND tags it with the given name
# Useful for: build locally, then push to registry
docker compose build
docker compose push
```

---

## 8. Real-World Example: Full Stack Application

Here's a complete compose file for a typical web application:

```yaml
services:
  # Frontend: Nginx serving static files
  frontend:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./frontend/dist:/usr/share/nginx/html:ro
    depends_on:
      - api

  # API: Node.js application
  api:
    build: ./api
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/appdb
      - REDIS_URL=redis://redis:6379
      - NODE_ENV=production
    depends_on:
      - db
      - redis

  # Database: PostgreSQL
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=appdb
    ports:
      - "5432:5432"

  # Cache: Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  # Worker: Background job processor
  worker:
    build: ./api
    command: npm run worker
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/appdb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

volumes:
  pgdata:
```

**What this gives you with `docker compose up -d`:**
- 5 containers started in the right order
- Automatic network so they can talk to each other
- Persistent PostgreSQL data in a named volume
- All port mappings configured
- Environment variables set everywhere

---

## 9. Project Naming

By default, Compose uses the **directory name** as the project name:

```bash
# In directory: /home/user/my-webapp
docker compose up -d

# Containers are named:
# my-webapp-web-1
# my-webapp-api-1
# my-webapp-db-1
```

Override the project name:

```bash
# Using -p flag
docker compose -p production up -d

# Using environment variable
COMPOSE_PROJECT_NAME=staging docker compose up -d

# In compose file (modern spec)
name: my-custom-project

services:
  web:
    image: nginx
```

With a custom name:
```
# Containers: my-custom-project-web-1, my-custom-project-api-1, etc.
```

---

## 📝 Hands-On Exercises

### Exercise 1: Your First Compose File

Create a new directory:

```bash
mkdir ~/compose-exercise && cd ~/compose-exercise
```

Create `docker-compose.yml`:

```yaml
services:
  web1:
    image: nginx:alpine
    ports:
      - "8080:80"

  web2:
    image: nginx:alpine
    ports:
      - "8081:80"
```

```bash
# Start both containers
docker compose up -d

# Verify both are running
docker compose ps

# Check logs
docker compose logs

# Visit both in browser or curl:
curl http://localhost:8080
curl http://localhost:8081

# Stop everything
docker compose down
```

### Exercise 2: Add Custom HTML

```bash
# Create HTML directories
mkdir -p html1 html2

# Create different pages
echo '<h1>Web 1 - Compose!</h1>' > html1/index.html
echo '<h1>Web 2 - Compose!</h1>' > html2/index.html
```

Update `docker-compose.yml`:

```yaml
services:
  web1:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html1:/usr/share/nginx/html:ro

  web2:
    image: nginx:alpine
    ports:
      - "8081:80"
    volumes:
      - ./html2:/usr/share/nginx/html:ro
```

```bash
# Start
docker compose up -d

# Each serves different content!
curl http://localhost:8080
curl http://localhost:8081

# Clean up
docker compose down
```

### Exercise 3: Practice All Compose Commands

With the two-nginx setup from Exercise 2:

```bash
# Start
docker compose up -d

# List services
docker compose ps
docker compose ps -a

# View logs
docker compose logs
docker compose logs web1
docker compose logs -f web2  # Ctrl+C to stop following

# Restart one service
docker compose restart web1

# Stop all
docker compose stop

# Start again
docker compose start

# Full cleanup
docker compose down --volumes
```

---

## 🧠 Common Questions

**Q: Is it `docker-compose` or `docker compose`?**
A: `docker compose` (with a space) is the modern V2 plugin. `docker-compose` (with a hyphen) is the legacy V1 standalone tool. Use `docker compose` on Docker 20.10+.

**Q: Do all services need a `ports` mapping?**
A: No. Only services you need to access from the host. Services that only talk to each other internally don't need ports exposed.

**Q: Can I have multiple compose files?**
A: Yes! Use `-f` flag: `docker compose -f base.yml -f overrides.yml up`. Useful for dev/prod overrides.

**Q: What happens if I run `docker compose up` again?**
A: Compose checks if services are already running. If nothing changed, it does nothing. If you changed the config, it recreates affected services.

**Q: Do I need a `networks` section?**
A: No. Compose creates a default network automatically and puts all services on it. Only add custom networks if you need segmentation (covered Day 3).

---

## ✅ Day 1 Summary

**What you learned today:**

- ✅ The problem Docker Compose solves (multi-container management)
- ✅ Compose file structure (services, networks, volumes)
- ✅ Your first compose YAML file
- ✅ Essential commands: `up`, `down`, `ps`, `logs`, `stop`, `start`, `restart`, `build`, `exec`, `run`
- ✅ The `image` vs `build` distinction
- ✅ Real-world full-stack compose example
- ✅ Project naming conventions

**Coming up tomorrow:** We'll build multi-container applications with dependencies, environment variables, and proper startup ordering.

---

*💡 Tip: Compose is infrastructure as code. Your docker-compose.yml file is the single source of truth for your application's infrastructure. Treat it like code — version control it, review it, test it.*