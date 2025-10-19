# 🐳 Docker Foundations Guide

A **comprehensive and detailed reference** covering everything you’ve learned so far about Docker — including images, containers, volumes, networks, and Docker Compose — with commands, code blocks, and real-world examples. Perfect for GitHub README or offline study.

---

## 🧱 1. Docker Overview

### What Is Docker?

Docker is a **containerization platform** that packages applications and their dependencies into lightweight, portable containers. Containers ensure the app runs the same way everywhere — on your laptop, staging, or production.

### How It Works Internally

Docker uses a **client–server architecture**:

* **Docker Client (CLI):** Sends commands like `docker build`, `docker run` to the daemon.
* **Docker Daemon (dockerd):** Builds, runs, and manages containers.
* **Docker Engine API:** REST interface between client and daemon.
* **Registry (e.g., Docker Hub):** Stores and distributes images.

```
+---------------------+
|     Docker CLI      |
+---------------------+
          |
          v
+---------------------+
|   Docker Daemon     |
|  (Builds, Runs)     |
+---------------------+
          |
          v
+---------------------+
|   Container Runtime |
+---------------------+
```

### Docker vs Virtual Machines

| Feature      | Docker        | Virtual Machine            |
| ------------ | ------------- | -------------------------- |
| Startup time | Seconds       | Minutes                    |
| Isolation    | Process-level | Full OS                    |
| Size         | MBs           | GBs                        |
| Performance  | Near-native   | Overhead due to hypervisor |
| Portability  | High          | Medium                     |

**Conclusion:** Docker containers share the host OS kernel, making them faster and more resource-efficient than VMs.

---

## 🧩 2. Images & Containers

### Understanding Images

A **Docker image** is a *read-only blueprint* for creating containers. Images are built in layers — each command in a Dockerfile adds a new layer.

Example Dockerfile:

```Dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]
```

**Build and run:**

```bash
docker build -t myapp:1.0 .
docker run -d -p 3000:3000 myapp:1.0
```

### Layers & UnionFS

Docker uses a **Union File System (OverlayFS)** to combine multiple image layers into one filesystem view. Layers are cached — speeding up rebuilds.

* Each line in Dockerfile creates a layer.
* Layers are immutable.
* Docker reuses unchanged layers to speed up builds.

### Containers

Containers are **running instances of images.**

```bash
docker run -d -p 8080:80 nginx
```

* `docker ps` — list running containers.
* `docker exec -it <id> bash` — access the shell.
* `docker logs <id>` — view logs.
* `docker stop/start/rm` — manage lifecycle.

**Lifecycle:**

```
create → start → pause → stop → remove
```

### Entrypoint vs CMD

| Directive    | Purpose                                   |
| ------------ | ----------------------------------------- |
| `CMD`        | Default command when container starts     |
| `ENTRYPOINT` | Always runs, even if arguments are passed |

Example:

```Dockerfile
ENTRYPOINT ["python3", "app.py"]
CMD ["--port=8080"]
```

---

## 💾 3. Volumes & Persistence

### Ephemeral Filesystem

By default, data written inside a container disappears when the container stops or is deleted. This is because each container has a temporary filesystem.

To persist data, Docker provides **volumes** and **bind mounts.**

### Types of Volumes

1. **Named Volumes** — Managed by Docker.
2. **Anonymous Volumes** — Created automatically without explicit names.
3. **Bind Mounts** — Directly link a host directory to the container.

### Creating and Using Volumes

```bash
docker volume create todo-db
docker run -dp 127.0.0.1:3000:3000 \
  --mount type=volume,src=todo-db,target=/etc/todos getting-started
```

Here:

* **src**: `todo-db` (volume name)
* **target**: `/etc/todos` (path inside container)

You can view volume data here:

```bash
ls /var/lib/docker/volumes/todo-db/_data
```

### Bind Mounts Example

```bash
mkdir ~/test-bind
echo "hello" > ~/test-bind/hello.txt

docker run -it --mount type=bind,src=$HOME/test-bind,target=/app ubuntu bash
```

Now `hello.txt` appears inside `/app` in the container.

Use bind mounts during development for **live code reload**.

### Volumes in Docker Compose

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos
    volumes:
      - todo-mysql-data:/var/lib/mysql

volumes:
  todo-mysql-data:
```

#### Common Pitfall

```yaml
volumes:
  todo-mysql-data:
    external: true
```

If `external: true` is used, Docker expects the volume to already exist. Otherwise, it fails to start.

### Inspecting Volumes

```bash
docker volume inspect todo-db
docker volume ls
docker volume rm unused-volume
```

---

## 🌐 4. Docker Networking

### Why Networking Matters

Docker networking allows containers to communicate with each other and with the outside world.

### Network Types

| Type        | Description                                       |
| ----------- | ------------------------------------------------- |
| **bridge**  | Default network for containers; uses internal NAT |
| **host**    | Shares host network stack (no isolation)          |
| **none**    | No network access                                 |
| **overlay** | Cross-host network (used by Swarm)                |
| **macvlan** | Assigns MAC address to container                  |

### Default Bridge Network

Containers connected to the default bridge can access each other **via IP**, not by name.

```bash
docker run -d --name web nginx
docker run -it --rm --network bridge alpine ping web
```

➡️ This fails, because default bridge doesn’t provide DNS.

### Custom Bridge Network (Recommended)

```bash
docker network create my-app-net

docker run -d --name mysql --network my-app-net mysql:8
docker run -d --name backend --network my-app-net node-app
```

Now the backend can resolve `mysql` by name automatically using Docker’s internal DNS.

### Inspecting Networks

```bash
docker network ls
docker network inspect my-app-net
```

This shows connected containers, subnet, and IPs.

### Host Networking

```bash
docker run --network host nginx
```

Container uses the host network directly — no port mapping.

### NAT & Masquerading

Bridge networks use **iptables** to perform NAT (Network Address Translation):

```bash
sudo iptables -t nat -L
```

You’ll see rules mapping container IPs to host ports. This is how `-p 3000:3000` works.

### Overlay Network (Swarm)

Overlay networks allow containers on **different hosts** to communicate securely. Kubernetes uses a similar concept called **Pod networking (CNI)**.

---

## ⚙️ 5. Docker Compose — Multi-Container Setup

### Why Use Compose

Docker Compose lets you define and run multi-container applications using a single `docker-compose.yml` file.

### Syntax Overview

```yaml
version: "3.9"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
    networks:
      - app-net

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos
    volumes:
      - todo-mysql-data:/var/lib/mysql
    networks:
      - app-net

networks:
  app-net:
    driver: bridge

volumes:
  todo-mysql-data:
```

### Key Sections

* **services:** Defines containers.
* **depends_on:** Controls startup order.
* **networks:** Defines communication bridge.
* **volumes:** Persists data.

### Commands

```bash
docker compose up -d
docker compose ps
docker compose down
docker compose logs app
```

### Custom Networks

```yaml
networks:
  frontend:
  backend:
```

Containers connected to multiple networks can communicate across layers securely.

### Example: Node.js + MySQL

```yaml
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos
    volumes:
      - todo-mysql-data:/var/lib/mysql
    networks:
      - app-net

  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=db
    depends_on:
      - db
    networks:
      - app-net

volumes:
  todo-mysql-data:

networks:
  app-net:
```

### Common Troubleshooting Cases

* Container exits immediately → check logs: `docker logs <id>`
* Database data lost → volume misconfigured.
* `external: true` → use only if volume already exists.

---

## 🧠 6. Dockerfile Deep Dive

### Dockerfile Instructions

| Instruction  | Purpose                                 |
| ------------ | --------------------------------------- |
| `FROM`       | Sets base image                         |
| `WORKDIR`    | Sets working directory inside container |
| `COPY`       | Copies files from host to image         |
| `RUN`        | Executes commands during build          |
| `EXPOSE`     | Documents port the container listens on |
| `CMD`        | Default command on container start      |
| `ENTRYPOINT` | Configures a fixed command              |
| `ENV`        | Sets environment variables              |
| `ARG`        | Defines build-time variables            |

### Example Optimized Node.js Dockerfile

```Dockerfile
FROM node:lts-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:lts-alpine
WORKDIR /app
COPY --from=builder /app .
EXPOSE 3000
CMD ["npm", "start"]
```

### Docker Ignore

Always include `.dockerignore`:

```
node_modules
.git
Dockerfile
.dockerignore
.env
```

This prevents unnecessary files from being copied into the image.

---

## 🔒 7. Security, Logs & Cleanup

### Security Best Practices

* Use **official images** from Docker Hub.
* Avoid running as `root`: use the `USER` directive.
* Keep images small (use alpine).
* Don’t store secrets in images — use environment variables or secret managers.

### Logging

```bash
docker logs -f <container>
docker inspect <container>
docker diff <container>
```

### Cleanup

```bash
docker system prune -a -f
docker volume prune
docker network prune
```

---

## 🧰 8. Commands Cheat Sheet

### Containers

```bash
docker ps -a
docker run -d -p 8080:80 nginx
docker stop/start/rm <container>
docker exec -it <container> bash
docker logs -f <container>
```

### Images

```bash
docker build -t myapp:1.0 .
docker images
docker rmi <image>
```

### Volumes

```bash
docker volume create myvol
docker volume ls
docker volume inspect myvol
docker volume rm myvol
```

### Networks

```bash
docker network create app-net
docker network inspect app-net
docker network connect app-net <container>
```

### Compose

```bash
docker compose up -d
docker compose down
docker compose logs
docker compose ps
```

---


# 🚀 THE BRIDGE ROADMAP — “From Docker to Kubernetes”

---

## 🧩 1. Containers & Images (Deep Clarity)

**Goal:** Know how containers are built, layered, stored, and shipped.

You should be able to:
- Explain the difference between **container**, **image**, and **layer**.
- Run containers with `docker run` and flags (`-d`, `-p`, `--name`, `--env-file`, `--mount`, etc.).
- Inspect containers and images (`docker inspect`, `docker logs`, `docker exec`, `docker diff`).

**Why it matters:**  
Kubernetes runs **containers**, not raw code.  
If you don’t know how images and layers work, debugging Pods later will feel like black magic.

---

## 🧱 2. Volumes & Persistence

**Goal:** Understand how containers retain data beyond lifecycle.

**Learn:**
- `docker volume create`, `--mount` vs `-v`, bind mounts vs named volumes.
- Where volumes live: `/var/lib/docker/volumes/...`
- Difference between **ephemeral** and **persistent** storage.

**Why it matters:**  
Kubernetes **PersistentVolumes (PVs)** and **PersistentVolumeClaims (PVCs)** are the same idea — just abstracted and automated.

---

## 🧰 3. Docker Compose (Multi-Container Orchestration)

**Goal:** Learn how to define services that work together.

**Learn:**
- `docker-compose.yml` syntax (`services`, `volumes`, `networks`, `depends_on`).
- Compose networking: automatic DNS, internal network isolation.
- Environment injection, secrets, restart policies, and scaling (`docker compose up --scale app=3`).

**Why it matters:**  
Docker Compose is the **mini-Kubernetes**.  
Everything you do here — services, ports, env vars, networks — maps directly to Kubernetes manifests:
- `Deployment` → service definitions  
- `Service` → network routing  
- `ConfigMap/Secret` → env files  

---

## 🌐 4. Docker Networking (Critical)

**Goal:** See how containers talk to each other.

**Learn:**
- Bridge, host, and overlay networks.
- Inspect container IPs and routes: `docker network inspect`, `ip a`, `iptables -t nat -L`.
- Difference between `127.0.0.1`, `0.0.0.0`, and container-to-container DNS (e.g., app resolves automatically).

**Why it matters:**  
Kubernetes uses a **flat networking model**.  
If you don’t grasp how Docker isolates and connects containers, K8s `ClusterIP`, `NodePort`, and `Ingress` will confuse you.

---

## 🧱 5. Dockerfiles (Production-Ready)

**Goal:** Write optimized, secure, and portable images.

**Learn:**
- Multi-stage builds to reduce size.
- `.dockerignore` to avoid sending junk.
- Non-root users, `CMD` vs `ENTRYPOINT`, environment variables.
- Use Alpine base vs Ubuntu, layer caching.

**Why it matters:**  
Every Kubernetes Pod uses an image.  
Efficient Dockerfiles = faster pulls + smaller deploys.

---

## 🔒 6. Image Registry Management

**Goal:** Push/pull from private/public registries.

**Learn:**
- Tagging (`docker tag`), pushing (`docker push`), pulling (`docker pull`).
- Auth with Docker Hub or ECR.
- Versioning images (`:latest`, semantic tags).

**Why it matters:**  
Kubernetes doesn’t **build** images — it **pulls** them from registries.  
Understanding image distribution avoids `ImagePullBackOff` errors.

---

## ⚙️ 7. Container Lifecycle & Logs

**Goal:** Debug and manage running containers effectively.

**Learn:**
- `docker ps`, `docker logs`, `docker exec`, `docker inspect`.
- Restart policies (`always`, `unless-stopped`).
- Container exit codes, signals, and healthchecks (`HEALTHCHECK CMD`).

**Why it matters:**  
Kubernetes Pods restart on crashes automatically — same logic applies here.

---

## 🧱 8. Build → Run → Ship (Full CI/CD Flow)

**Goal:** Automate container builds and deployments.

**Learn:**
- Automate builds with Dockerfile + GitHub Actions (build & push to registry).
- Understand image tagging conventions in pipelines.

**Why it matters:**  
Kubernetes CI/CD pipelines depend on a strong **automated Docker foundation**.

---

## 🧠 9. Security & Best Practices

**Goal:** Container hardening and scanning.

**Learn:**
- Avoid root users in containers.
- Use `.env` or Secrets (not hardcoded values).
- Scan images (`docker scan`, **Trivy**).

**Why it matters:**  
Kubernetes extends security with **RBAC**, **NetworkPolicies**, and **Secrets** — but your base image must be clean first.

---

## 🪜 Then You’re Ready for Kubernetes

Once you finish those **9 pillars**, you’ll fully understand how Docker concepts map directly to Kubernetes:

| Kubernetes Concept | Docker Equivalent |
|---------------------|------------------|
| **Pod** | Container |
| **Deployment** | `docker compose up` |
| **Service** | Docker bridge network |
| **PV/PVC** | Docker volume |
| **ConfigMap/Secret** | `.env` files |
| **Ingress** | Exposed ports with routing |
| **ReplicaSet** | `--scale` |

---

**🎯 Summary:**  
Master Docker fundamentals → Abstract them in Kubernetes.  
Once this bridge is crossed, Kubernetes becomes **intuitive**, not intimidating.
```markdown
