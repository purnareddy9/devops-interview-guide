---
layout: default
title: "🐳 Docker — DevOps Interview Guide"
render_with_liquid: false
---

# 🐳 Docker — DevOps Interview Guide

> [← Git](./Git.md) | [Main Index](./README.md) | [Kubernetes →](./Kubernetes.md)

---

## Table of Contents

1. [Docker Architecture](#docker-architecture)
2. [Images & Containers](#images--containers)
3. [Dockerfile Best Practices](#dockerfile-best-practices)
4. [Networking](#networking)
5. [Volumes & Storage](#volumes--storage)
6. [Docker Compose](#docker-compose)
7. [Registry & Image Management](#registry--image-management)
8. [Security](#security)
9. [Interview Questions](#interview-questions)
10. [Scenario-Based Questions](#scenario-based-questions)
11. [Hands-On Labs](#hands-on-labs)
12. [Cheat Sheet](#cheat-sheet)

---

## Docker Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Docker Client (CLI)                  │
│            docker build / run / push / pull           │
└────────────────────┬────────────────────────────────┘
                     │  REST API (Unix socket / TCP)
┌────────────────────▼────────────────────────────────┐
│                 Docker Daemon (dockerd)               │
│                                                       │
│  ┌──────────────┐  ┌─────────────────────────────┐  │
│  │  Image Store │  │    Container Runtime         │  │
│  │  (layers)    │  │    (containerd → runc)       │  │
│  └──────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                     │
         Linux Kernel (namespaces + cgroups)
```

### Container vs VM

| Feature | Container | Virtual Machine |
|---------|-----------|----------------|
| Boot time | Seconds | Minutes |
| Overhead | Minimal (shares kernel) | High (full OS) |
| Isolation | Process-level | Hardware-level |
| Portability | High | Medium |
| Size | MB | GB |
| Security boundary | Weaker | Stronger |
| Use case | Microservices, apps | Full OS isolation |

---

## Images & Containers

### Essential Commands

```bash
# ─── IMAGES ──────────────────────────────────────────
docker images                       # List local images
docker pull nginx:1.25              # Pull specific version
docker inspect nginx                # Full metadata JSON
docker history nginx                # Show image layers
docker image prune -a               # Remove unused images
docker save nginx > nginx.tar       # Export to tarball
docker load < nginx.tar             # Import from tarball

# ─── CONTAINERS ──────────────────────────────────────
docker ps                           # Running containers
docker ps -a                        # All containers
docker run -d --name web nginx      # Detached named container
docker run -it ubuntu bash          # Interactive shell
docker run --rm alpine echo hello   # Auto-remove after run

# Run with resource limits
docker run -d \
  --name myapp \
  --memory="512m" \
  --cpus="0.5" \
  --restart=unless-stopped \
  -p 8080:80 \
  -e ENV=production \
  -v /data:/app/data \
  myapp:latest

# Container lifecycle
docker stop myapp                   # Graceful stop (SIGTERM → SIGKILL after 10s)
docker kill myapp                   # Immediate SIGKILL
docker start myapp                  # Start stopped container
docker restart myapp                # Stop + start
docker rm myapp                     # Remove stopped container
docker rm -f myapp                  # Force remove running container

# Container interaction
docker exec -it myapp bash          # Exec into running container
docker exec myapp env               # Run non-interactive command
docker logs myapp                   # View logs
docker logs -f --tail 100 myapp     # Follow logs
docker cp myapp:/app/logs /tmp/     # Copy files from container
docker stats myapp                  # Live resource usage
docker top myapp                    # Processes in container
```

---

## Dockerfile Best Practices

### Multi-Stage Build (Production Pattern)

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Production image
FROM node:20-alpine AS production
# Security: run as non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app

# Copy only what's needed from builder
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules

USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

ENTRYPOINT ["node"]
CMD ["dist/server.js"]
```

### Dockerfile Optimization Rules

```dockerfile
# ✅ DO: Order layers from least to most frequently changing
COPY package.json ./      # Rarely changes → cache hit
RUN npm install
COPY . .                  # Changes often → cache miss here only

# ✅ DO: Combine RUN commands to reduce layers
RUN apt-get update && \
    apt-get install -y curl wget git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# ✅ DO: Use specific tags, not 'latest'
FROM python:3.11.7-slim-bookworm

# ✅ DO: Use .dockerignore
# .dockerignore content:
# .git
# node_modules
# *.log
# .env
# tests/
# docs/

# ✅ DO: COPY specific files instead of context
COPY src/ ./src/
COPY config/ ./config/

# ❌ DON'T: COPY . . at the start (invalidates all cache)
# ❌ DON'T: Store secrets in layers (they persist even if deleted)
# ❌ DON'T: Run as root
# ❌ DON'T: Use latest tag in production
```

### Python App Dockerfile

```dockerfile
FROM python:3.11-slim

# Security and best practices
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# System dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application code
COPY --chown=nobody:nogroup . .

USER nobody
EXPOSE 8000

CMD ["gunicorn", "app:create_app()", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

---

## Networking

### Network Modes

| Mode | Description | Use Case |
|------|-------------|---------|
| **bridge** (default) | Isolated virtual network on host | Multi-container apps |
| **host** | Container uses host network directly | Performance, host-level access |
| **none** | No network | Isolated compute tasks |
| **overlay** | Multi-host networking (Swarm) | Distributed applications |
| **macvlan** | Container gets its own MAC/IP | Legacy apps needing direct LAN access |

```bash
# Create custom bridge network
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --ip-range 172.20.240.0/20 \
  mynet

# Run containers in same network (they can resolve by name)
docker run -d --network mynet --name db postgres
docker run -d --network mynet --name app myapp
# app can connect to: postgres://db:5432/mydb

# Inspect network
docker network inspect mynet
docker network ls
docker network connect mynet existing-container
```

### Container DNS

```
Container 'app' can reach container 'db' by:
- Container name: db
- Service name (in Compose): db
- Full DNS: db.mynet (custom bridge)
- IP address: 172.20.x.x
```

---

## Volumes & Storage

### Storage Types

```bash
# ─── VOLUMES (managed by Docker, preferred) ──────────
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker run -v mydata:/app/data myapp
docker volume rm mydata
docker volume prune                # Remove unused volumes

# ─── BIND MOUNTS (host path → container) ─────────────
docker run -v /host/path:/container/path myapp
docker run -v $(pwd):/app myapp   # Mount current directory

# ─── TMPFS (in-memory, not persisted) ────────────────
docker run --tmpfs /tmp:rw,size=100m myapp

# Volume in docker run
docker run -d \
  -v db_data:/var/lib/postgresql/data \   # Named volume
  -v /etc/ssl/certs:/etc/ssl/certs:ro \  # Read-only bind mount
  postgres:15
```

### When to Use Each

| Type | Persistence | Performance | Use Case |
|------|-------------|-------------|---------|
| Volume | Yes | Good | Database data, app state |
| Bind mount | Yes (host) | Best | Dev: hot-reload, config files |
| tmpfs | No | Best | Sensitive data, caches |

---

## Docker Compose

### Production-Grade Compose File

```yaml
version: "3.9"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
      args:
        - BUILD_VERSION=${BUILD_VERSION:-latest}
    image: myapp:${BUILD_VERSION:-latest}
    restart: unless-stopped
    ports:
      - "8080:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://db/mydb
    env_file:
      - .env.production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 512M
    volumes:
      - app_logs:/app/logs
    networks:
      - frontend
      - backend
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    secrets:
      - db_password

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - backend

  nginx:
    image: nginx:1.25-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - frontend

volumes:
  db_data:
  redis_data:
  app_logs:

networks:
  frontend:
  backend:
    internal: true       # Not accessible from outside

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Compose Commands

```bash
docker compose up -d                   # Start in background
docker compose up --build              # Rebuild and start
docker compose down                    # Stop and remove containers
docker compose down -v                 # Also remove volumes
docker compose ps                      # Status
docker compose logs -f app             # Follow service logs
docker compose exec app bash           # Shell in running service
docker compose scale app=3             # Scale service (legacy)
docker compose run --rm app pytest     # One-off command
docker compose config                  # Validate + show config
```

---

## Registry & Image Management

### Working with Registries

```bash
# Docker Hub
docker login
docker tag myapp:latest username/myapp:1.0
docker push username/myapp:1.0

# AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

docker tag myapp:latest \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# GCR / Artifact Registry
gcloud auth configure-docker
docker push gcr.io/project-id/myapp:1.0

# Scan image for vulnerabilities
docker scout cves nginx:latest
trivy image nginx:latest
```

---

## Security

### Container Security Best Practices

```dockerfile
# 1. Non-root user
RUN useradd -r -u 1001 appuser
USER 1001

# 2. Read-only filesystem
docker run --read-only --tmpfs /tmp myapp

# 3. Drop capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# 4. No privileged containers
# NEVER: docker run --privileged myapp (gives root on host!)

# 5. Seccomp profile
docker run --security-opt seccomp=default.json myapp

# 6. AppArmor
docker run --security-opt apparmor=docker-default myapp

# 7. Resource limits (prevent DoS)
docker run --memory=512m --cpu-quota=50000 myapp
```

### Docker Bench Security

```bash
# Run Docker Bench for Security
docker run -it --net host --pid host --userns host --cap-add audit_control \
    -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
    -v /etc:/etc:ro \
    -v /lib/systemd/system:/lib/systemd/system:ro \
    -v /usr/bin/containerd:/usr/bin/containerd:ro \
    -v /usr/bin/runc:/usr/bin/runc:ro \
    -v /usr/lib/systemd:/usr/lib/systemd:ro \
    -v /var/lib:/var/lib:ro \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    docker/docker-bench-security
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between an image and a container?**

> An **image** is an immutable, read-only template — a snapshot of a filesystem and configuration. A **container** is a running instance of an image — it adds a writable layer on top. Multiple containers can run from the same image. When the container is deleted, its writable layer is gone (unless using volumes).

**Q2: What is a Dockerfile `ENTRYPOINT` vs `CMD`?**

> `ENTRYPOINT` defines the **command that always runs** (the executable). `CMD` provides **default arguments** to ENTRYPOINT or a default command if no ENTRYPOINT.
> ```dockerfile
> ENTRYPOINT ["nginx"]     # Always runs nginx
> CMD ["-g", "daemon off;"] # Default arg; can be overridden
> ```
> `docker run myimage -c /etc/nginx/custom.conf` would override CMD.
> If both are shell form, CMD args are ignored by shell form ENTRYPOINT.

**Q3: Explain Docker layer caching.**

> Each Dockerfile instruction creates a layer. Docker caches layers; if nothing changed in a layer or its dependencies, it reuses the cache. Cache is **invalidated** when instruction content changes, a `COPY`/`ADD` source file changes, or a parent layer is invalidated. Order instructions from least-changed to most-changed to maximize cache hits.

---

### 🟡 Intermediate

**Q4: How do you reduce Docker image size?**

> - Use **minimal base images** (alpine, distroless, scratch for static binaries)
> - **Multi-stage builds** — don't ship build tools in production
> - **Combine RUN commands** to squash layers
> - **Remove caches** in same RUN: `apt-get clean && rm -rf /var/lib/apt/lists/*`
> - Use `.dockerignore` to exclude unnecessary files
> - Don't install documentation or extras: `--no-install-recommends`

**Q5: What is Docker's overlay filesystem?**

> Docker uses OverlayFS (or overlay2 storage driver) to implement image layers. Each layer is a directory of changes. When a container starts, Docker creates a merged view: all image layers (read-only) + container layer (read-write) appear as one filesystem. Copy-on-write: when a container modifies a file, it's copied to the writable layer first.

**Q6: How does Docker networking work for containers on the same host?**

> Default **bridge network** (`docker0`): Each container gets a veth pair — one end in container namespace (eth0), one end on the docker0 bridge. The bridge handles intra-host routing. NAT (iptables MASQUERADE) handles outbound internet. Port publishing (`-p 8080:80`) adds an iptables DNAT rule. Custom bridge networks add DNS resolution by container name.

---

### 🔴 Advanced

**Q7: How would you implement zero-downtime deployments with Docker?**

```bash
# Strategy 1: Blue-Green with nginx upstream swap
# Deploy new version as new container
docker run -d --name app-green myapp:v2

# Health check
until curl -f http://localhost:8081/health; do sleep 2; done

# Update nginx upstream
sed -i 's/app-blue/app-green/g' /etc/nginx/sites-enabled/app
nginx -s reload

# Remove old version
docker stop app-blue && docker rm app-blue

# Strategy 2: Docker Compose rolling update
docker compose up -d --no-deps --scale app=2 app
# Wait for new containers to be healthy
docker compose up -d --no-deps app
```

**Q8: How do you handle secrets in Docker securely?**

> - **Docker Secrets** (Swarm): secrets mounted as files in `/run/secrets/`, never in env vars or layers
> - **External secret managers**: HashiCorp Vault, AWS Secrets Manager — inject at runtime
> - **Environment files** (`.env`): acceptable for dev, not production — not committed to git
> - **Never**: hardcode in Dockerfile, pass via `--build-arg` (visible in `docker history`), put in environment variables that end up in logs

---

## Scenario-Based Questions

### 🔵 Scenario 1: Container Keeps Restarting

```bash
# 1. Check restart count and last exit code
docker ps -a                          # Exit code in STATUS column
docker inspect myapp | jq '.[0].State'

# 2. Check logs
docker logs myapp --tail 50
docker logs --since 5m myapp

# 3. Exit code meanings:
# 0  = Success
# 1  = App error
# 137 = OOM killed (128 + 9)
# 139 = Segfault (128 + 11)
# 143 = SIGTERM (128 + 15)

# 4. If OOM: check memory
docker stats myapp
docker inspect myapp | jq '.[0].HostConfig.Memory'
# Fix: increase memory limit or fix memory leak

# 5. Run interactively to see error
docker run --rm -it myapp bash
```

### 🔵 Scenario 2: Image Vulnerabilities in CI

```bash
# In CI pipeline, scan before push
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:latest

# Generate SBOM
docker sbom myapp:latest

# Policy: fail on HIGH+ for production
# Allow LOW/MEDIUM with review

# Add to Dockerfile for known-good base
FROM node:20.10.0-alpine3.18@sha256:abc123...  # Pin digest
```

---

## Hands-On Labs

### Lab 1: Multi-Stage Build Optimization

```bash
# Create a simple Go app
mkdir docker-lab && cd docker-lab
cat > main.go << 'EOF'
package main
import (
    "fmt"
    "net/http"
)
func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from Docker!")
    })
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(200)
    })
    http.ListenAndServe(":8080", nil)
}
EOF

# Naive Dockerfile (large image)
cat > Dockerfile.large << 'EOF'
FROM golang:1.21
WORKDIR /app
COPY . .
RUN go build -o server .
EXPOSE 8080
CMD ["./server"]
EOF

# Multi-stage Dockerfile (tiny image)
cat > Dockerfile << 'EOF'
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o server .

FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
EOF

docker build -t myapp-large -f Dockerfile.large .
docker build -t myapp-small .

echo "Size comparison:"
docker images | grep myapp
```

---

## Cheat Sheet

```bash
# ─── BUILD ────────────────────────────────────────────
docker build -t name:tag .
docker build --build-arg KEY=val -t name:tag .
docker build --target stage_name -t name:tag .

# ─── RUN ──────────────────────────────────────────────
docker run -d -p 8080:80 --name c1 nginx
docker run -it --rm ubuntu bash
docker run -v vol:/data -e KEY=val image

# ─── DEBUG ────────────────────────────────────────────
docker logs -f --tail 100 c1
docker exec -it c1 sh
docker inspect c1
docker stats c1
docker top c1

# ─── CLEANUP ──────────────────────────────────────────
docker system prune -a          # Remove all unused
docker container prune          # Remove stopped containers
docker image prune -a           # Remove unused images
docker volume prune             # Remove unused volumes

# ─── MULTI-ARCH ───────────────────────────────────────
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t user/app:latest --push .
```

---

> **Cross-links:** [← Git](./Git.md) | [Kubernetes →](./Kubernetes.md) | [CI/CD (Docker in pipelines) →](./CICD.md) | [Security (container security) →](./Security.md)