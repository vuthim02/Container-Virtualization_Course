# Week 3 • Day 4 • Intermediate
# Advanced Compose Patterns

## Progress Checklist
- [x] Day 1 — Introduction to Compose
- [x] Day 2 — Multi-container Apps
- [x] Day 3 — Networking in Compose
- [ ] Day 4 — Advanced Compose Patterns

---

← [Day 3](day3.md) | [Roadmap](../roadmap.md) | [Week 4 Day 1 →](../Week4/day1.md)

---

## 1. Environment Variables — Advanced Patterns

We covered the basics of environment variables on Day 2. Now let's go deeper.

### Variable Substitution with Defaults

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}          # Default: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}    # Default: secret
      POSTGRES_DB: ${DB_NAME:-myapp}               # Default: myapp
```

```bash
# Uses defaults (no .env file set):
docker compose up -d
# POSTGRES_USER=postgres, POSTGRES_PASSWORD=secret, POSTGRES_DB=myapp

# Override with .env file:
# DB_USER=admin
# DB_PASSWORD=production-secret
docker compose up -d
# POSTGRES_USER=admin, POSTGRES_PASSWORD=production-secret

# Override from shell:
DB_USER=root DB_PASSWORD=override docker compose up -d
# POSTGRES_USER=root, POSTGRES_PASSWORD=override
```

### Mandatory Variables (No Default)

```yaml
services:
  app:
    build: .
    environment:
      # Will cause error if not set
      API_KEY: ${API_KEY:?API_KEY is required}
      DATABASE_URL: ${DATABASE_URL:?DATABASE_URL must be set}
```

```bash
# Fails with error message:
docker compose up -d
# Error: API_KEY is required
```

### Variable from Host Environment

```yaml
services:
  app:
    build: .
    environment:
      CURRENT_USER: ${USER}          # Takes $USER from your shell
      HOSTNAME: ${HOSTNAME}
```

### Interpolation in Values

```yaml
services:
  web:
    image: nginx
    environment:
      GREETING: "Hello, ${USER}!"
      FULL_URL: "http://${DOMAIN:-localhost}:${PORT:-8080}"
```

---

## 2. Extension Fields (Reuse Configuration)

Avoid repeating yourself with YAML anchors and Compose extensions.

### YAML Anchors (Traditional)

```yaml
x-common-vars: &common-vars
  DATABASE_URL: postgres://app:secret@db:5432/myapp
  REDIS_URL: redis://redis:6379
  LOG_LEVEL: info
  NODE_ENV: production

services:
  api:
    build: ./api
    environment:
      <<: *common-vars
      PORT: 3000

  worker:
    build: ./api
    command: npm run worker
    environment:
      <<: *common-vars

  scheduler:
    build: ./api
    command: npm run scheduler
    environment:
      <<: *common-vars
```

### Extension Fields (Modern Compose)

```yaml
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

x-common-config: &common-config
  restart: unless-stopped
  logging: *default-logging

services:
  web:
    <<: *common-config
    image: nginx:alpine
    ports:
      - "80:80"

  api:
    <<: *common-config
    build: .
    ports:
      - "3000:3000"

  db:
    <<: *common-config
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
```

### Reusing Service Definitions

```yaml
services:
  app-base: &app-base
    build: .
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/myapp
    networks:
      - backend

  api:
    <<: *app-base
    ports:
      - "3000:3000"
    command: npm start

  worker:
    <<: *app-base
    command: npm run worker
    # No ports — worker doesn't need external access

  scheduler:
    <<: *app-base
    command: npm run scheduler
    # No ports either
```

---

## 3. Volumes in Compose — Advanced

### Named Volumes (Top-Level Declaration)

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  pgdata:
  redis-data:
```

### Bind Mounts for Development

```yaml
services:
  web:
    build: .
    volumes:
      # Source code — live editing
      - .:/app
      # Node modules — named volume overrides bind mount
      - node_modules:/app/node_modules
    ports:
      - "3000:3000"

  # Watcher service — auto-rebuild on file changes
  watcher:
    build: .
    volumes:
      - .:/app
    command: npm run watch

volumes:
  node_modules:
```

**Why the `node_modules` volume?** Without it, the bind mount of `.` would shadow the container's `node_modules` directory. The named volume preserves the installed packages.

### tmpfs in Compose

```yaml
services:
  app:
    build: .
    tmpfs:
      - /tmp
      - /app/cache
```

### Read-Only Container

```yaml
services:
  web:
    image: nginx:alpine
    read_only: true    # Container filesystem is read-only
    tmpfs:
      - /tmp           # Writable tmp for temp files
      - /var/cache/nginx
      - /var/run
    volumes:
      - ./html:/usr/share/nginx/html:ro
```

**Security benefit:** If an attacker compromises the container, they can't modify any files in the image.

### Volume Configuration

```yaml
volumes:
  pgdata:
    driver: local
    driver_opts:
      type: none
      device: /mnt/data/postgres
      o: bind
    labels:
      com.example.project: "myapp"
      com.example.environment: "production"
```

---

## 4. Healthchecks — Production-Ready

We've seen basic healthchecks. Let's go deeper.

### Healthcheck Anatomy

```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s        # Check every 30 seconds
      timeout: 10s         # Fail if check takes > 10s
      retries: 3           # Mark unhealthy after 3 failures
      start_period: 10s    # Grace period before checks start
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `test` | None | The command to run |
| `interval` | 30s | Time between checks |
| `timeout` | 30s | Maximum time for a single check |
| `retries` | 3 | Consecutive failures before unhealthy |
| `start_period` | 0s | Startup grace period (no penalty during this time) |

### Healthcheck Test Formats

```yaml
# CMD form (preferred — no shell)
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]

# CMD-SHELL form (when you need shell features)
healthcheck:
  test: ["CMD-SHELL", "curl -f http://localhost || exit 1"]

# Inline string (implicitly uses CMD-SHELL)
healthcheck:
  test: curl -f http://localhost || exit 1
```

### Database Healthchecks

```yaml
# PostgreSQL
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

# MySQL
  mysql:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 5

# MongoDB
  mongo:
    image: mongo:7
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

# Redis
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### Application Healthchecks

```yaml
# HTTP-based health
  api:
    build: .
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

# Process-based health (check if process is running)
  worker:
    build: .
    command: celery -A app worker
    healthcheck:
      test: ["CMD-SHELL", "celery inspect ping -A app"]
      interval: 30s
      timeout: 10s
      retries: 3

# File-based health (check if a file exists)
  app:
    build: .
    healthcheck:
      test: ["CMD-SHELL", "test -f /tmp/app-ready"]
      interval: 5s
      timeout: 5s
      retries: 10
```

### Checking Health Status

```bash
# See health status in docker compose ps
docker compose ps
# NAME          STATUS
# myapp-web-1   Up 5 minutes (healthy)
# myapp-db-1    Up 5 minutes (healthy)
# myapp-api-1   Up 2 minutes (health: starting)
# myapp-w-1     Up 1 minute (unhealthy)

# Detailed health info
docker inspect myapp-web-1 --format '{{.State.Health.Status}}'
# healthy

docker inspect myapp-web-1 --format '{{json .State.Health}}' | python3 -m json.tool
# {
#   "Status": "healthy",
#   "FailingStreak": 0,
#   "Log": [
#     {
#       "Start": "2024-01-15T10:00:00Z",
#       "End": "2024-01-15T10:00:01Z",
#       "ExitCode": 0,
#       "Output": "  % Total...OK\n"
#     }
#   ]
# }
```

### Disable Inherited Healthchecks

```yaml
services:
  # Base image has a healthcheck, but we want to disable it
  web:
    image: some-image-with-healthcheck
    healthcheck:
      disable: true
```

---

## 5. Profiles — Conditional Services

Profiles let you define services that only start when you explicitly request them.

```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"

  db:
    image: postgres:16-alpine

  # Only starts when --profile monitoring is used
  prometheus:
    image: prom/prometheus
    profiles:
      - monitoring
    ports:
      - "9090:9090"

  # Only starts when --profile monitoring is used
  grafana:
    image: grafana/grafana
    profiles:
      - monitoring
    ports:
      - "3000:3000"

  # Only starts when --profile debug is used
  debug-tools:
    image: nicolaka/netshoot
    profiles:
      - debug
    command: sleep infinity
```

```bash
# Start only non-profiled services (web, db)
docker compose up -d

# Start with monitoring
docker compose --profile monitoring up -d
# Now includes: web, db, prometheus, grafana

# Start with debugging
docker compose --profile debug up -d
# Now includes: web, db, debug-tools

# Start all profiles
docker compose --profile monitoring --profile debug up -d
```

**Use cases:**
- `monitoring` profile: Prometheus, Grafana, Jaeger
- `debug` profile: Debug containers, network tools
- `ci` profile: Test runners, linters
- `dev` profile: Hot-reload watchers, debug proxies

---

## 6. Multiple Compose Files

Split your configuration by environment:

### Base Configuration

`docker-compose.yml` (common to all environments):
```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### Production Overrides

`docker-compose.prod.yml`:
```yaml
services:
  web:
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
    restart: always
    environment:
      NODE_ENV: production

  db:
    restart: always
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
```

### Development Overrides

`docker-compose.dev.yml`:
```yaml
services:
  web:
    volumes:
      - .:/app        # Live code editing
    environment:
      NODE_ENV: development
      DEBUG: "true"

  db:
    ports:
      - "5432:5432"   # Expose for local debugging
```

```bash
# Development (base + dev overrides)
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# Production (base + prod overrides)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Shorthand: Compose automatically loads docker-compose.override.yml
# (Rename docker-compose.dev.yml to docker-compose.override.yml)
docker compose up -d   # Uses base + override
```

**Merge behavior:** Later files override earlier ones. Services are merged (not replaced), so you only need to specify what changes.

---

## 7. Deploy — Resource Limits

```yaml
services:
  web:
    build: .
    deploy:
      resources:
        limits:
          cpus: "0.50"      # Max 50% of one CPU
          memory: 256M       # Max 256 MB RAM
        reservations:
          cpus: "0.25"      # Guaranteed 25% of one CPU
          memory: 128M       # Guaranteed 128 MB RAM

  db:
    image: postgres:16-alpine
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

**Limits vs Reservations:**
- **Limits:** Maximum resources the container can use
- **Reservations:** Guaranteed minimum resources (important for scheduling in Swarm)

**Legacy syntax (still works):**
```yaml
services:
  web:
    build: .
    mem_limit: 256m        # Same as deploy.resources.limits.memory
    cpus: 0.5              # Same as deploy.resources.limits.cpus
```

---

## 8. Logging Configuration

```yaml
services:
  web:
    image: nginx
    logging:
      driver: json-file
      options:
        max-size: "10m"    # Rotate log file at 10 MB
        max-file: "5"      # Keep 5 rotated files

  db:
    image: postgres:16-alpine
    logging:
      driver: syslog       # Send logs to syslog
      options:
        syslog-address: "tcp://192.168.0.1:514"
        tag: "postgres"

  worker:
    build: .
    logging:
      driver: none         # Disable logging entirely
```

### Global Logging (in daemon.json or Compose)

```yaml
# Apply to all services in compose file
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  web:
    image: nginx
    logging: *default-logging

  api:
    build: .
    logging: *default-logging
```

---

## 9. Complete Production-Ready Example

```yaml
name: production-app

services:
  # Reverse proxy / Load balancer
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      api:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    networks:
      - frontend
    deploy:
      resources:
        limits:
          cpus: "0.25"
          memory: 128M
    logging: &default-logging
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  # Application API
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
    environment:
      DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      REDIS_URL: redis://redis:6379
      NODE_ENV: production
      LOG_LEVEL: info
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    restart: unless-stopped
    networks:
      - frontend
      - backend
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    logging: *default-logging

  # Background workers
  worker:
    build:
      context: ./api
      dockerfile: Dockerfile
    command: npm run worker
    environment:
      DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      REDIS_URL: redis://redis:6379
      NODE_ENV: production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - backend
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
    logging: *default-logging

  # PostgreSQL Database
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - backend
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    logging: *default-logging

  # Redis Cache
  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - backend
    deploy:
      resources:
        limits:
          cpus: "0.25"
          memory: 128M
    logging: *default-logging

  # Monitoring (optional profile)
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    profiles:
      - monitoring
    restart: unless-stopped
    networks:
      - backend
    logging: *default-logging

  grafana:
    image: grafana/grafana
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    profiles:
      - monitoring
    depends_on:
      - prometheus
    restart: unless-stopped
    networks:
      - backend
    logging: *default-logging

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true     # No external access to backend network

volumes:
  pgdata:
  redis-data:
  prometheus-data:
  grafana-data:
```

**Features demonstrated:**
- ✅ Health-based dependencies
- ✅ Network segmentation (frontend/backend)
- ✅ Internal network (backend is isolated)
- ✅ Resource limits
- ✅ Structured logging
- ✅ Environment variables with `.env`
- ✅ Named volumes
- ✅ Read-only config mounts
- ✅ Profiles for optional services
- ✅ Restart policies
- ✅ Extension fields for DRY config

---

## 📝 Hands-On Exercises

### Exercise: Production-Ready Compose File

Build a production-ready compose file for a WordPress + MySQL setup:

```bash
mkdir ~/compose-advanced && cd ~/compose-advanced
```

```yaml
services:
  wordpress:
    image: wordpress:6-apache
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: ${DB_USER:-wordpress}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD:-wordpress}
      WORDPRESS_DB_NAME: ${DB_NAME:-wordpress}
    volumes:
      - wp-data:/var/www/html
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  db:
    image: mysql:8.0
    environment:
      MYSQL_USER: ${DB_USER:-wordpress}
      MYSQL_PASSWORD: ${DB_PASSWORD:-wordpress}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-rootsecret}
      MYSQL_DATABASE: ${DB_NAME:-wordpress}
    volumes:
      - db-data:/var/lib/mysql
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  wp-data:
  db-data:
```

```bash
# Create .env file
cat > .env << 'EOF'
DB_USER=wpuser
DB_PASSWORD=wp_s3cret_p@ss
DB_NAME=wordpress_db
DB_ROOT_PASSWORD=r00t_s3cret
EOF

# Start
docker compose up -d

# Wait for health checks
docker compose ps
# Wait for both (healthy)

# Visit http://localhost:8080
# Complete the WordPress installation

# Check logs
docker compose logs -f wordpress
docker compose logs -f db

# Clean up
docker compose down --volumes
```

---

## 🧠 Common Questions

**Q: What's the difference between `env_file` and `environment`?**
A: `env_file` loads variables from a file. `environment` sets them directly in the compose file. Both end up as environment variables in the container. Use `env_file` for many variables or sharing across services.

**Q: Can I use `.env` files for secrets?**
A: For development, yes. For production, use Docker Secrets (Swarm mode), HashiCorp Vault, or your cloud provider's secret manager. Never commit `.env` files with real secrets to Git.

**Q: Does `internal: true` on a network mean no internet access?**
A: No. `internal: true` means no **external** (host or other networks) access. Containers can still reach the internet through NAT. To block internet, you need firewall rules.

**Q: What happens if a healthcheck fails?**
A: After `retries` consecutive failures, the container is marked `unhealthy`. Docker does NOT automatically restart the container (unless you have a restart policy that triggers on failure).

**Q: Can I restart an unhealthy container automatically?**
A: Not natively with Compose. Options:
- Use an orchestrator (Kubernetes, Docker Swarm)
- Use a watchtower or health-monitor sidecar
- Use `restart: on-failure` (only triggers on process exit, not unhealthy status)

---

## ✅ Day 4 Summary — Week 3 Complete!

**What you learned today:**

- ✅ Advanced environment variable patterns (defaults, required, interpolation)
- ✅ Extension fields and YAML anchors for DRY configuration
- ✅ Advanced volume patterns (tmpfs, read-only, named volume overrides)
- ✅ Production healthchecks (database, application, process-based)
- ✅ Profiles for conditional services
- ✅ Multiple compose files for environment-specific configuration
- ✅ Resource limits and reservations
- ✅ Logging configuration
- ✅ Complete production-ready compose file example

**🎉 Week 3 Complete!**

You now know how to:
- ✅ Define multi-container applications with Compose
- ✅ Manage service dependencies and startup ordering
- ✅ Design secure network topologies
- ✅ Write production-ready compose files with healthchecks, resource limits, and logging

**Coming up in Week 4:** Container Networking — deeper dive into Docker network drivers, service discovery, load balancing, and container security.

---

*💡 Tip: A production-ready compose file is a living document. Start simple, add healthchecks, limits, and logging as you move toward production. Don't add everything at once — earn each feature.*