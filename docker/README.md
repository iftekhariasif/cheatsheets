# Docker

Containerization fundamentals, Node.js best practices, and production-ready patterns.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Dockerfile Best Practices](#dockerfile-best-practices)
- [Multi-Stage Builds](#multi-stage-builds)
- [Docker Compose V2](#docker-compose-v2)
- [Networking](#networking)
- [Volumes and Persistence](#volumes-and-persistence)
- [Health Checks](#health-checks)
- [Node.js in Docker](#nodejs-in-docker)
- [Security](#security)
- [Production Patterns](#production-patterns)

---

## Core Concepts

### Images vs containers

| Concept | What it is | Analogy |
|---|---|---|
| **Image** | Read-only template with app code, runtime, dependencies | A class definition |
| **Container** | Running instance of an image | An object instance |
| **Layer** | Each instruction in a Dockerfile creates a cached layer | Git commits |
| **Registry** | Storage for images (Docker Hub, GHCR, ECR) | npm registry for packages |

### Essential commands

```bash
# Images
docker build -t myapp:1.0 .          # Build image from Dockerfile
docker images                          # List images
docker rmi myapp:1.0                   # Remove image
docker pull node:22-alpine             # Pull from registry
docker push myuser/myapp:1.0           # Push to registry

# Containers
docker run -d -p 3000:3000 myapp:1.0  # Run detached, map ports
docker run -it myapp:1.0 /bin/sh      # Run interactive shell
docker ps                              # List running containers
docker ps -a                           # List all containers (including stopped)
docker stop <container_id>             # Stop container
docker rm <container_id>               # Remove container
docker logs <container_id>             # View logs
docker logs -f <container_id>          # Follow logs (tail)
docker exec -it <container_id> sh     # Shell into running container

# Cleanup
docker system prune                    # Remove unused data
docker system prune -a                 # Remove ALL unused data (including images)
docker volume prune                    # Remove unused volumes
```

### Tags

```bash
# Tag format: registry/repository:tag
docker build -t myapp:latest .
docker build -t myapp:1.0.0 .
docker build -t ghcr.io/myuser/myapp:1.0.0 .

# Tag an existing image
docker tag myapp:latest myapp:1.0.0

# Always pin versions in production — never rely on :latest
```

### Build context

Everything in the directory you pass to `docker build`:

```bash
docker build -t myapp .          # current directory is the build context
docker build -t myapp -f deploy/Dockerfile .   # custom Dockerfile location
```

Files in the build context are sent to the Docker daemon. Use `.dockerignore` to exclude unnecessary files.

---

## Dockerfile Best Practices

### Layer caching — order matters

Docker caches layers. If a layer changes, all subsequent layers are rebuilt.

```dockerfile
# BAD — copying all files first busts cache on ANY file change
COPY . .
RUN npm ci

# GOOD — copy package files first, install deps, then copy source
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
# Now npm ci is cached unless package files change
```

### .dockerignore

```dockerignore
node_modules
.next
.git
.gitignore
*.md
.env*
.DS_Store
dist
coverage
.vscode
.idea
Dockerfile
docker-compose*.yml
```

### Pin base image versions

```dockerfile
# BAD — could change anytime
FROM node:latest
FROM node:alpine

# GOOD — pinned major + variant
FROM node:22-alpine

# BEST — pinned exact version
FROM node:22.12.0-alpine3.21
```

### Reduce image size

```dockerfile
# Use alpine variants (5MB vs 350MB for full Debian)
FROM node:22-alpine

# Combine RUN commands to reduce layers
RUN apk add --no-cache curl \
    && rm -rf /var/cache/apk/*

# Use npm ci instead of npm install (clean, reproducible)
RUN npm ci --omit=dev

# Remove unnecessary files after build
RUN npm run build && npm prune --production
```

### Common Dockerfile for a Node.js app

```dockerfile
FROM node:22-alpine

WORKDIR /app

# Dependencies first (cached layer)
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# Application code
COPY . .

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

---

## Multi-Stage Builds

Separate build-time dependencies from the production image.

### Node.js API (Express/Fastify)

```dockerfile
# Stage 1: Install all dependencies and build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production image with only runtime deps
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --from=builder /app/dist ./dist

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Next.js with standalone output

```dockerfile
FROM node:22-alpine AS base

# Dependencies
FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Build
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Production — minimal image
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy only what's needed
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

### Size comparison

| Approach | Typical Size |
|---|---|
| `node:22` (no multi-stage) | ~1.2 GB |
| `node:22-alpine` (no multi-stage) | ~400 MB |
| Multi-stage + alpine + standalone | ~100-150 MB |
| Multi-stage + distroless | ~80-120 MB |

---

## Docker Compose V2

### Basic compose.yaml

```yaml
# compose.yaml (preferred filename over docker-compose.yml)
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    volumes:
      - .:/app          # bind mount for development
      - /app/node_modules  # anonymous volume to preserve node_modules

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

### Commands

```bash
# Start all services
docker compose up

# Start in detached mode
docker compose up -d

# Rebuild images
docker compose up --build

# Stop all services
docker compose down

# Stop and remove volumes
docker compose down -v

# View logs
docker compose logs
docker compose logs -f web    # follow specific service

# Run a one-off command
docker compose exec web sh
docker compose run web npm test

# Scale a service
docker compose up -d --scale worker=3
```

### Profiles

Run certain services only when needed:

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"

  db:
    image: postgres:16-alpine

  # Only runs with: docker compose --profile debug up
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    profiles:
      - debug

  # Only runs with: docker compose --profile monitoring up
  prometheus:
    image: prom/prometheus
    profiles:
      - monitoring
```

```bash
docker compose --profile debug up
docker compose --profile debug --profile monitoring up
```

### Watch mode (development)

Auto-rebuild/sync on file changes:

```yaml
services:
  web:
    build: .
    develop:
      watch:
        # Sync source files without rebuilding
        - action: sync
          path: ./src
          target: /app/src

        # Rebuild when package.json changes
        - action: rebuild
          path: ./package.json

        # Sync and restart when config changes
        - action: sync+restart
          path: ./config
          target: /app/config
```

```bash
docker compose watch
```

### Environment files

```yaml
services:
  web:
    build: .
    env_file:
      - .env
      - .env.local  # overrides .env
    environment:
      - NODE_ENV=production  # overrides env_file
```

---

## Networking

### Network types

| Type | Use Case |
|---|---|
| `bridge` (default) | Containers on the same host communicate |
| `host` | Container shares the host's network (no isolation) |
| `none` | No networking |
| `overlay` | Multi-host networking (Docker Swarm) |

### Service discovery

In Compose, services can reach each other by name:

```yaml
services:
  web:
    build: .
    # Access db at "db:5432" and cache at "cache:6379"

  db:
    image: postgres:16-alpine

  cache:
    image: redis:7-alpine
```

```js
// In your app code
const dbUrl = "postgresql://user:pass@db:5432/mydb"; // "db" is the service name
const redisUrl = "redis://cache:6379";               // "cache" is the service name
```

### Custom networks

```yaml
services:
  web:
    build: .
    networks:
      - frontend
      - backend

  api:
    build: ./api
    networks:
      - backend

  db:
    image: postgres:16-alpine
    networks:
      - backend  # not accessible from frontend network

  nginx:
    image: nginx
    networks:
      - frontend

networks:
  frontend:
  backend:
```

### Port mapping

```yaml
services:
  web:
    ports:
      - "3000:3000"         # host:container
      - "127.0.0.1:3001:3000"  # bind to localhost only
      - "3000"               # random host port → container 3000
    expose:
      - "3000"               # expose to other containers only (not host)
```

---

## Volumes and Persistence

### Types of mounts

```yaml
services:
  web:
    volumes:
      # Named volume — managed by Docker, persists across restarts
      - pgdata:/var/lib/postgresql/data

      # Bind mount — maps host directory into container
      - ./src:/app/src

      # tmpfs — in-memory, not persisted
      - type: tmpfs
        target: /tmp

  db:
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:                    # Named volume declaration
```

### When to use what

| Mount Type | Persistence | Performance | Use Case |
|---|---|---|---|
| Named volume | Yes | Best | Database data, uploads |
| Bind mount | Yes (host) | Good | Development source code |
| tmpfs | No (RAM) | Fastest | Temp files, secrets |
| Anonymous volume | Container lifetime | Best | node_modules in dev |

### Common patterns

```yaml
# Development — bind mount source, anonymous volume for node_modules
services:
  web:
    volumes:
      - .:/app
      - /app/node_modules  # prevents host node_modules from overwriting container's

# Production — named volumes only
services:
  web:
    volumes:
      - uploads:/app/uploads
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  uploads:
  pgdata:
```

### Backup and restore

```bash
# Backup a volume
docker run --rm -v pgdata:/data -v $(pwd):/backup alpine \
  tar czf /backup/pgdata-backup.tar.gz -C /data .

# Restore a volume
docker run --rm -v pgdata:/data -v $(pwd):/backup alpine \
  tar xzf /backup/pgdata-backup.tar.gz -C /data
```

---

## Health Checks

### In Dockerfile

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

### In compose.yaml

```yaml
services:
  web:
    build: .
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      start_period: 10s
      retries: 3

  # For images without curl — use wget
  api:
    image: node:22-alpine
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # For databases
  db:
    image: postgres:16-alpine
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

### Using health checks for startup ordering

```yaml
services:
  web:
    build: .
    depends_on:
      db:
        condition: service_healthy   # waits for db to be healthy
      cache:
        condition: service_started   # just waits for container to start

  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### Custom health check endpoint

```js
// health.js — in your Node.js app
app.get("/health", async (req, res) => {
  try {
    // Check database connection
    await db.query("SELECT 1");
    // Check Redis
    await redis.ping();

    res.status(200).json({ status: "healthy" });
  } catch (error) {
    res.status(503).json({ status: "unhealthy", error: error.message });
  }
});
```

---

## Node.js in Docker

### Choosing a base image

| Image | Size | Use Case |
|---|---|---|
| `node:22` | ~350 MB | Full Debian — when you need native build tools |
| `node:22-slim` | ~200 MB | Minimal Debian — good default for production |
| `node:22-alpine` | ~50 MB | Smallest — watch for musl libc compatibility |
| `gcr.io/distroless/nodejs22` | ~40 MB | No shell, no package manager — most secure |

### Production Dockerfile

```dockerfile
FROM node:22-alpine

# Security: install tini for proper signal handling
RUN apk add --no-cache tini

WORKDIR /app

# Dependencies (cached layer)
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Application code
COPY . .

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 3000

# Use tini as init system
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

### Why tini?

Node.js doesn't handle signals properly as PID 1:

```dockerfile
# WITHOUT tini:
# - SIGTERM not forwarded to Node.js
# - Zombie processes not reaped
# - Container takes 10s to stop (Docker sends SIGKILL after timeout)

# WITH tini:
# - Signals forwarded correctly
# - Zombies reaped
# - Graceful shutdown works
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

### npm ci vs npm install

```dockerfile
# BAD — non-deterministic, modifies lockfile
RUN npm install

# GOOD — clean install from lockfile, faster, deterministic
RUN npm ci

# PRODUCTION — skip devDependencies
RUN npm ci --omit=dev
```

### .dockerignore for Node.js

```dockerignore
node_modules
npm-debug.log
.next
dist
coverage
.git
.gitignore
.env*
*.md
.vscode
.idea
Dockerfile
docker-compose*.yml
.DS_Store
```

### Development Dockerfile

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

EXPOSE 3000
CMD ["npm", "run", "dev"]
```

```yaml
# compose.yaml for development
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
```

---

## Security

### Run as non-root

```dockerfile
# Alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Debian/Ubuntu
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

# Or use built-in node user (available in official Node images)
USER node
```

### Read-only filesystem

```yaml
services:
  web:
    build: .
    read_only: true
    tmpfs:
      - /tmp
      - /app/.next/cache  # Next.js needs write access to cache
```

### Secrets management

```yaml
# compose.yaml — never put secrets in environment variables in production
services:
  web:
    build: .
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    environment: API_KEY  # reads from host env
```

```js
// Read secret from file in Node.js
import { readFileSync } from "fs";
const dbPassword = readFileSync("/run/secrets/db_password", "utf8").trim();
```

### Image scanning

```bash
# Docker Scout (built into Docker Desktop)
docker scout cves myapp:latest
docker scout recommendations myapp:latest

# Trivy
trivy image myapp:latest

# Snyk
snyk container test myapp:latest
```

### Security checklist

```dockerfile
# 1. Pin base image version
FROM node:22.12.0-alpine3.21

# 2. Don't run as root
USER node

# 3. Use npm ci (deterministic)
RUN npm ci --omit=dev

# 4. No secrets in image
# Use runtime secrets, not build-time ARGs for sensitive data

# 5. Minimal image (multi-stage + alpine/distroless)
# 6. Scan images in CI
# 7. Use .dockerignore
# 8. Set read-only filesystem where possible
```

---

## Production Patterns

### Logging — always use stdout/stderr

```js
// GOOD — Docker captures stdout/stderr
console.log("info: server started on port 3000");
console.error("error: database connection failed");

// BAD — writing to files inside the container
fs.writeFileSync("/var/log/app.log", message); // logs lost when container dies
```

```bash
# View logs
docker logs <container_id>
docker logs -f <container_id>          # follow
docker logs --since 1h <container_id>  # last hour
docker compose logs -f web             # follow specific service
```

### Graceful shutdown

```js
// server.js
const server = app.listen(3000);

function gracefulShutdown(signal) {
  console.log(`${signal} received. Shutting down gracefully...`);

  server.close(() => {
    console.log("HTTP server closed.");

    // Close database connections
    db.end()
      .then(() => {
        console.log("Database connections closed.");
        process.exit(0);
      })
      .catch((err) => {
        console.error("Error closing database:", err);
        process.exit(1);
      });
  });

  // Force shutdown after 30s
  setTimeout(() => {
    console.error("Forced shutdown after timeout.");
    process.exit(1);
  }, 30000);
}

process.on("SIGTERM", () => gracefulShutdown("SIGTERM"));
process.on("SIGINT", () => gracefulShutdown("SIGINT"));
```

### Resource limits

```yaml
services:
  web:
    build: .
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
    # Also set Node.js memory limit
    environment:
      - NODE_OPTIONS=--max-old-space-size=384
```

### Restart policies

```yaml
services:
  web:
    restart: unless-stopped   # restart unless manually stopped

  # Options:
  # - no              (default — don't restart)
  # - always          (always restart)
  # - on-failure      (restart only on non-zero exit)
  # - unless-stopped  (restart unless manually stopped)

  worker:
    restart: on-failure
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
```

### Container orchestration (overview)

When you outgrow single-host Docker Compose:

| Tool | Complexity | Use Case |
|---|---|---|
| **Docker Compose** | Low | Single host, development, small production |
| **Docker Swarm** | Medium | Multi-host, built into Docker, simple orchestration |
| **Kubernetes (K8s)** | High | Enterprise, auto-scaling, self-healing, cloud-native |
| **AWS ECS / Fargate** | Medium | AWS-native, serverless containers |
| **Google Cloud Run** | Low | Serverless, auto-scaling, pay-per-request |

### Full production compose.yaml example

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  pgdata:
```

---

## References

- [Docker Docs](https://docs.docker.com)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Node.js Docker Best Practices](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md)
- [Docker Security Best Practices](https://docs.docker.com/build/building/best-practices/)
