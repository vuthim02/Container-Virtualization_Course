# 🐳 Container Mastery Labs

This directory contains practical examples and companion files for the hands-on exercises in the **Container Completely Course**.

## Directory Structure

- `week3-multi-container/`: A full-stack application setup (Web, API, DB, Redis)
- `week4-load-balancing/`: An nginx-based load balancer with 3 replicas
- `week6-production/`: Production-ready Kubernetes manifests

## How to Use These Labs

1.  Navigate to the specific week's lab directory.
2.  Read the `README.md` (if provided) or examine the `docker-compose.yml` file.
3.  Run `docker compose up -d` to start the lab.
4.  Follow the hands-on exercise instructions from the corresponding day's lesson.

---

### Lab 1: Week 3 - Multi-container Application
**Location:** `labs/week3-multi-container/docker-compose.yml`
**Features:**
- PostgreSQL for persistent data.
- Redis for caching.
- Node.js API.
- Nginx Frontend.
- Custom network for isolation.

### Lab 2: Week 4 - Load Balancing
**Location:** `labs/week4-load-balancing/docker-compose.yml`
**Features:**
- Nginx configured as a Round-Robin load balancer.
- Scaled backend with 3 replicas.
- Embedded Docker DNS for service discovery.
