# Week 3 • Day 2 • Intermediate
# Multi-container Applications

## Progress Checklist
- [x] Day 1 — Introduction to Compose
- [ ] Day 2 — Multi-container Apps
- [ ] Day 3 — Networking in Compose
- [ ] Day 4 — Advanced Compose Patterns

---

← [Day 1](day1.md) | [Roadmap](../roadmap.md) | [Day 3 →](day3.md)

---

## 1. Architecture of Multi-Container Apps

Most real-world applications follow a layered architecture:

```
┌─────────────────────────────────────────────────────────┐
│                    User's Browser                       │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│  Tier 1: Frontend / Reverse Proxy                       │
│  ┌─────────────┐  ┌─────────────┐                      │
│  │   Nginx     │  │    CDN      │                      │
│  │  (static)   │  │             │                      │
│  └──────┬──────┘  └─────────────┘                      │
│         │                                               │
├─────────┼───────────────────────────────────────────────┤
│         ▼                                               │
│  Tier 2: Application / API                              │
│  ┌─────────────┐  ┌─────────────┐                      │
│  │  Node.js    │  │   Celery    │                      │
│  │  / Flask    │  │  Workers    │                      │
│  └──────┬──────┘  └──────┬──────┘                      │
│         │                │                              │
├─────────┼────────────────┼──────────────────────────────┤
│         ▼                ▼                              │
│  Tier 3: Data Layer                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ PostgreSQL  │  │   Redis     │  │  Elasticsearch│    │
│  │  (primary)  │  │  (cache)    │  │  (search)     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

Each tier is one or more containers. Compose orchestrates all of them.

---

## 2. Service Dependencies: `depends_on`

Containers start in **parallel** by default. But some services need others to be ready first.

### Basic `depends_on`

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7-alpine
```

**What this does:** Docker Compose starts `db` and `redis` **before** `web`.

**What it does NOT do:** It does NOT wait for PostgreSQL to be ready to accept connections. It only waits for the container to be running.

```
Timeline with basic depends_on:
db:      [starting...][running (but not ready)]     ← depends_on satisfied
redis:   [starting...][running]
web:                          [starts NOW]         ← but PostgreSQL isn't ready yet!
                                       │
                                       ▼
                              web tries to connect to PostgreSQL
                              → Connection refused!
                              → App crashes or errors
```

### `depends_on` with Health Checks (The Right Way)

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
```

```
Timeline with health-based depends_on:
db:      [starting...][running][healthcheck...][healthy]  ← depends_on satisfied
redis:   [starting...][running][healthcheck...][healthy]  ← depends_on satisfied
web:                                       [starts NOW]   ← Both are actually ready!
                                                │
                                                ▼
                                       web connects to PostgreSQL
                                       → Connection succeeds!
                                       → App starts properly
```

### Alternative: Retry Scripts

If you can't use healthchecks, use a wait script:

```yaml
services:
  web:
    build: .
    command: >
      sh -c "
        until pg_isready -h db -p 5432; do
          echo 'Waiting for PostgreSQL...';
          sleep 2;
        done;
        echo 'DB is ready, starting app!';
        python app.py
      "
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
```

---

## 3. Environment Variables — Configuration

Services need configuration. Compose provides several ways to pass it.

### Inline Variables

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: super-secret-123
      POSTGRES_DB: myapp_db
```

**⚠️ Problem:** Secrets are stored directly in the YAML file (visible in version control).

### Environment File

```yaml
services:
  db:
    image: postgres:16-alpine
    env_file:
      - .env.db

  web:
    build: .
    env_file:
      - .env.web
```

**`.env.db`:**
```bash
POSTGRES_USER=myapp
POSTGRES_PASSWORD=super-secret-123
POSTGRES_DB=myapp_db
```

**`.env.web`:**
```bash
DATABASE_URL=postgres://myapp:super-secret-123@db:5432/myapp_db
SECRET_KEY=abc123
DEBUG=false
```

### `.env` File (Compose's Default)

Compose automatically loads a `.env` file from the same directory:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}

  web:
    build: .
    environment:
      DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
```

**`.env` file (not committed to Git):**
```bash
DB_USER=myapp
DB_PASSWORD=super-secret-123
DB_NAME=myapp_db
```

**`.env.example` (committed to Git):**
```bash
DB_USER=
DB_PASSWORD=
DB_NAME=
```

### Precedence

Environment variables are resolved in this order (last wins):

1. Compose file default values
2. `.env` file
3. Shell environment (variables set in your terminal)
4. `--env-file` flag

```bash
# Shell variable overrides .env file
DB_PASSWORD=production-secret docker compose up -d
```

---

## 4. Real-World Example: Python App + PostgreSQL

### Project Structure

```
flask-postgres-app/
├── docker-compose.yml
├── .env
├── .env.example
├── Dockerfile
├── requirements.txt
├── app.py
└── init.sql
```

### `app.py`
```python
from flask import Flask, jsonify
import psycopg2
import os

app = Flask(__name__)

def get_db():
    return psycopg2.connect(
        host=os.environ.get("DB_HOST", "db"),
        database=os.environ.get("DB_NAME", "myapp"),
        user=os.environ.get("DB_USER", "myapp"),
        password=os.environ.get("DB_PASSWORD", "secret"),
    )

@app.route("/health")
def health():
    try:
        conn = get_db()
        conn.close()
        return jsonify({"status": "healthy", "db": "connected"})
    except Exception as e:
        return jsonify({"status": "unhealthy", "error": str(e)}), 503

@app.route("/users")
def get_users():
    conn = get_db()
    cur = conn.cursor()
    cur.execute("SELECT id, name FROM users;")
    users = [{"id": row[0], "name": row[1]} for row in cur.fetchall()]
    cur.close()
    conn.close()
    return jsonify(users)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### `requirements.txt`
```
flask==3.0.0
psycopg2-binary==2.9.9
gunicorn==21.2.0
```

### `Dockerfile`
```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

### `docker-compose.yml`
```yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      DB_NAME: ${DB_NAME:-myapp}
      DB_USER: ${DB_USER:-myapp}
      DB_PASSWORD: ${DB_PASSWORD:-secret}
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      POSTGRES_USER: ${DB_USER:-myapp}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
      POSTGRES_DB: ${DB_NAME:-myapp}
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-myapp}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### `.env`
```bash
DB_USER=flaskuser
DB_PASSWORD=f1ask_p@ssw0rd!
DB_NAME=flaskapp
```

### `init.sql`
```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com')
ON CONFLICT DO NOTHING;
```

### Running It

```bash
cd flask-postgres-app

# Start everything
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f web

# Test the API
curl http://localhost:5000/health
# {"status":"healthy","db":"connected"}

curl http://localhost:5000/users
# [{"id":1,"name":"Alice"},{"id":2,"name":"Bob"},{"id":3,"name":"Charlie"}]

# Connect to PostgreSQL directly
docker compose exec db psql -U flaskuser -d flaskapp -c "SELECT * FROM users;"

# Clean up
docker compose down --volumes
```

---

## 5. Commands and Entry Points

Override the default command of an image:

```yaml
services:
  # Use default CMD from Dockerfile
  web:
    build: .

  # Override the command
  migrate:
    build: .
    command: python manage.py db upgrade

  # Override both entrypoint and command
  shell:
    build: .
    entrypoint: /bin/bash
    command: ""

  # Run a worker process (same image, different command)
  worker:
    build: .
    command: celery -A app.celery worker --loglevel=info

  # Run a scheduler
  scheduler:
    build: .
    command: celery -A app.celery beat --loglevel=info
```

---

## 6. Multiple Instances of the Same Service

Scale services directly (Docker Compose v2):

```bash
# Start 3 instances of the worker service
docker compose up -d --scale worker=3

# Check
docker compose ps
# worker-1, worker-2, worker-3 all running
```

Or in the compose file:

```yaml
services:
  worker:
    build: .
    command: celery -A app.celery worker --loglevel=info
    deploy:
      replicas: 3
```

**Important:** Scaled services share the same service name for DNS resolution. Other services reaching `worker` will round-robin between instances.

---

## 7. Restart Policies

```yaml
services:
  web:
    build: .
    restart: unless-stopped    # Always restart except when manually stopped

  db:
    image: postgres:16-alpine
    restart: always            # Always restart, even after docker compose stop

  worker:
    build: .
    restart: on-failure        # Only restart if it exits with non-zero code

  one-off-job:
    build: .
    restart: "no"              # Never restart (default)
```

| Policy | Behavior |
|--------|----------|
| `no` | Never restart (default) |
| `always` | Always restart |
| `on-failure[:max-retries]` | Restart on error only |
| `unless-stopped` | Always restart unless manually stopped |
| `no` (for one-off) | Run once and exit |

---

## 📝 Hands-On Exercises

### Exercise: Full Python + PostgreSQL Stack

Create the complete application from the example above.

```bash
mkdir ~/flask-postgres && cd ~/flask-postgres

# Create all the files: app.py, Dockerfile, requirements.txt, 
# docker-compose.yml, .env, init.sql

# Then:
docker compose up -d

# Wait for health checks to pass:
docker compose ps  # Wait for (healthy) status

# Test endpoints
curl http://localhost:5000/health
curl http://localhost:5000/users

# Try database operations directly
docker compose exec db psql -U flaskuser -d flaskapp -c "INSERT INTO users (name, email) VALUES ('Dave', 'dave@example.com');"
curl http://localhost:5000/users  # Should include Dave now

# Clean up
docker compose down --volumes
```

### Bonus: Add a Redis Cache

Extend the compose file:

```yaml
services:
  # ... existing web and db ...

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
```

Update `app.py` to use Redis for caching.

---

## 🧠 Common Questions

**Q: Why doesn't `depends_on` wait for the database to be ready?**
A: By default, `depends_on` only waits for the container to be started, not for the service inside it to be ready. Use `condition: service_healthy` to wait for actual readiness.

**Q: Can I run database migrations automatically on startup?**
A: Yes! Use a separate service or a shell command wrapper:

```yaml
services:
  migrate:
    build: .
    command: python manage.py db upgrade
    depends_on:
      db:
        condition: service_healthy

  web:
    build: .
    depends_on:
      migrate:
        condition: service_completed_successfully
```

**Q: How do I pass secrets securely?**
A: Use Docker secrets (Swarm mode), external secret managers (Vault), or ensure `.env` files are never committed to version control (add to `.gitignore`).

**Q: Can I use one `.env` file for all environments?**
A: Use multiple env files:

```yaml
services:
  web:
    env_file:
      - .env.${ENVIRONMENT:-development}
```

```bash
ENVIRONMENT=production docker compose up -d  # Loads .env.production
```

---

## ✅ Day 2 Summary

**What you learned today:**

- ✅ Multi-container architecture patterns (frontend → API → data layer)
- ✅ `depends_on` and its limitations (basic vs health-based)
- ✅ Environment variable strategies (inline, env_file, `.env`)
- ✅ Complete real-world example (Flask + PostgreSQL)
- ✅ Overriding commands and entry points
- ✅ Scaling services with `--scale`
- ✅ Restart policies

**Coming up tomorrow:** We'll dive deep into Compose networking — how services discover and communicate with each other.

---

*💡 Tip: Health-based depends_on is the key to reliable multi-container startups. Always add healthchecks to your database and cache services.*