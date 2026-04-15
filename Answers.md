# 🎓 Final Exam: Answer Key & Explanations

---

## Section 1: Multiple Choice

1. **B) Namespaces.** Namespaces isolate what a process can see (e.g., `PID`, `NET`, `UTS`, `MNT`). Cgroups control what a process can *use* (resources like CPU/RAM).
2. **B) `ENTRYPOINT` sets the command that always runs; `CMD` provides default arguments.** `CMD` can be overridden by arguments passed to `docker run`. `ENTRYPOINT` is much harder to override (`--entrypoint` flag).
3. **C) `docker compose down --rmi all`.** `down` removes containers and networks. `--rmi all` removes all images used by the services.
4. **B) To provide a stable IP address and DNS name.** Since Pods are ephemeral and their IPs change, a Service provides a stable endpoint for discovery.
5. **C) PodDisruptionBudget (PDB).** This ensures that even during voluntary disruptions like `kubectl drain`, the cluster maintains a minimum number of available Pods.

---

## Section 2: Scenario-Based Questions

6. **Cause:** The Pod is likely running in a **Restricted** namespace or with `readOnlyRootFilesystem: true` in its `securityContext`. Locally, the container might run as root or with a writable filesystem.
   - **Fix:** Add a volume (e.g., `emptyDir`) to the `/app/logs` mount point and ensure the `securityContext` allows the specific user ID to write to that volume.

7. **Increase the `request`.** 
   - **Difference:** When a Pod exceeds its CPU **request**, it only gets throttled *if there is contention* on the node. When it hits its CPU **limit**, it is throttled *immediately and consistently* regardless of node availability. Increasing the request ensures the scheduler places the Pod on a node with enough guaranteed CPU.

8. **Compose Solution:** Use `depends_on` with a `condition: service_healthy` (requires a `healthcheck` in the `db` service).
   - **Code Solution:** Implement a **retry loop** or "wait-for-it" script in the application's startup logic to wait for the database connection to become available.

---

## Section 3: Hands-On Challenges

### Challenge 1: Optimized Dockerfile
```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
USER appuser
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Challenge 2: Kubernetes HA Snippet
```yaml
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - my-app
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: my-app
          image: my-app:1.0
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
```

### Challenge 3: Network Security Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
```
