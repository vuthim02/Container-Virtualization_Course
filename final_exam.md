# 🎓 Container Completely Course: Final Exam

This exam covers the core concepts from all 6 weeks of the course. It is divided into three sections: **Multiple Choice**, **Scenario-Based Questions**, and **Hands-On Challenges**.

---

## Section 1: Multiple Choice (Fundamentals)

1. **Which Linux kernel feature is responsible for isolating what a process can *see* (e.g., its own PID list or network stack)?**
   - A) Cgroups
   - B) Namespaces
   - C) Hypervisor
   - D) OverlayFS

2. **In a Dockerfile, what is the difference between `CMD` and `ENTRYPOINT`?**
   - A) `CMD` is mandatory, `ENTRYPOINT` is optional.
   - B) `ENTRYPOINT` sets the command that always runs; `CMD` provides default arguments that can be overridden.
   - C) `CMD` sets the command that always runs; `ENTRYPOINT` provides default arguments.
   - D) There is no difference; they are interchangeable.

3. **Which Docker Compose command stops and *removes* containers, networks, and images?**
   - A) `docker compose stop`
   - B) `docker compose rm`
   - C) `docker compose down --rmi all`
   - D) `docker compose kill`

4. **What is the primary purpose of a Kubernetes `Service`?**
   - A) To run a persistent background process.
   - B) To provide a stable IP address and DNS name for a set of Pods.
   - C) To manage the lifecycle of a Pod.
   - D) To store sensitive configuration data.

5. **Which Kubernetes resource is used to ensure that at least a certain number of Pods remain available during a voluntary disruption (like a node drain)?**
   - A) HorizontalPodAutoscaler (HPA)
   - B) PodAntiAffinity
   - C) PodDisruptionBudget (PDB)
   - D) ResourceQuota

---

## Section 2: Scenario-Based Questions

### Scenario A: The "Works on My Machine" Mystery
A developer reports that their Node.js application works perfectly locally in a container, but fails in the staging Kubernetes cluster with a "Permission Denied" error when trying to write to `/app/logs`. 

**Question 6:** Based on Week 6's production best practices, what is the most likely cause of this discrepancy, and how would you fix it in the Kubernetes manifest?

### Scenario B: The Throttling App
You notice that your Python API container is intermittently slow. After checking the logs, you see no errors, but `kubectl top pods` shows the container is hitting its CPU limit of `200m`.

**Question 7:** Should you increase the `limit`, the `request`, or both? Explain the difference in how Kubernetes handles a Pod that exceeds its CPU request versus one that exceeds its CPU limit.

### Scenario C: The Database Connection
You have a `docker-compose.yml` with a `web` service and a `db` service. The `web` app keeps failing to connect to the database because it tries to connect before the database is fully initialized.

**Question 8:** Name two ways to solve this "startup race condition" (one in Compose, one in the application code).

---

## Section 3: Hands-On Challenges

### Challenge 1: Optimize the Dockerfile
Take the following "naive" Dockerfile and rewrite it using **multi-stage builds** and **non-root user** best practices.

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Challenge 2: Kubernetes High Availability
Write a snippet of a Kubernetes `Deployment` manifest that ensures:
1. Exactly 3 replicas are running.
2. No two replicas are scheduled on the same node (using anti-affinity).
3. The container has a liveness probe checking `/healthz` on port 8080.

### Challenge 3: Network Security
Write a Kubernetes `NetworkPolicy` that:
1. Applies to Pods with the label `app: database`.
2. Allows incoming traffic *only* from Pods with the label `app: api`.
3. Denies all other incoming traffic.

---

## 🏆 Answer Key & Grading

- **16-20 points:** Container Master 🐳
- **11-15 points:** Intermediate Navigator ⛴️
- **0-10 points:** Keep Learning! 📚

*(Answers are provided in the [Answers.md](Answers.md) file)*
