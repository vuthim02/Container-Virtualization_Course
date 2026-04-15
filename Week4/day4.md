# Week 4 • Day 4 • Intermediate
# Container Security

## Progress Checklist
- [ ] Day 1
- [ ] Day 2
- [ ] Day 3
- [ ] Day 4

---

← Day 3 | Roadmap | Week 5 Day 1 →

---

## Container Security Best Practices

### Run as Non-root
```dockerfile
FROM node:18-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### Read-only Filesystem
```bash
docker run --read-only nginx
```

### Limit Capabilities
```bash
docker run --cap-drop ALL --cap-add CHOWN myimage
```

### Scan Images
```bash
docker scan myimage
trivy image myimage
```

### Secrets Management
- Never bake secrets into images
- Use Docker secrets or environment variables
- Use secrets management tools (Vault)

---

## Your Task
Create a Dockerfile that runs as a non-root user.