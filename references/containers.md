# Containers Reference

## Table of Contents
1. Core Concepts
2. Dockerfile Best Practices
3. Docker Compose (local dev)
4. Container Registries
5. Multi-stage Builds
6. Networking
7. File Structure

---

## 1. Core Concepts

A **container** packages your app + its dependencies into an isolated, portable unit.
A **Docker image** is the blueprint. A **container** is a running instance of that image.

```
[Source Code] → [Dockerfile] → [docker build] → [Image]
                                                     ↓
                                             [docker run] → [Container]
```

---

## 2. Dockerfile — Production Best Practices

```dockerfile
# Use a specific version tag, never "latest" in production
FROM node:20-alpine AS base

# Set working directory
WORKDIR /app

# Copy dependency files FIRST (for layer caching)
# Docker only re-installs when package.json changes
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Don't run as root — create a non-privileged user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Document which port the app uses (informational only)
EXPOSE 3000

# Use exec form (not shell form) to handle signals correctly
CMD ["node", "src/index.js"]
```

⚠️ **Common mistakes:**
- Copying everything before installing deps (breaks caching)
- Running as root
- Using `latest` tag (non-deterministic builds)
- Storing secrets in ENV — use runtime secrets instead

---

## 3. Multi-stage Build (smaller final image)

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build           # Compiles TypeScript, etc.

# Stage 2: Production (only the compiled output)
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/index.js"]
# Result: image is ~5x smaller than single-stage
```

---

## 4. Docker Compose — Local Dev Environment

```yaml
# docker-compose.yml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app                    # Mount source for hot reload
      - /app/node_modules         # Prevent host node_modules override
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy  # Wait for DB to be ready

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data   # Persist data between restarts
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:    # Named volume persists across container restarts
```

**Useful commands:**
```bash
docker compose up -d          # Start in background
docker compose logs -f app    # Follow app logs
docker compose exec app sh    # Shell into running container
docker compose down -v        # Stop and remove volumes
```

---

## 5. Container Registries

| Registry | Use Case | Command |
|---|---|---|
| Docker Hub | Public images | `docker pull nginx` |
| GHCR | GitHub-integrated | `docker pull ghcr.io/org/image:tag` |
| ECR (AWS) | AWS deployments | `aws ecr get-login-password \| docker login` |
| GCR (GCP) | GCP deployments | `gcloud auth configure-docker` |
| ACR (Azure) | Azure deployments | `az acr login --name myregistry` |

---

## 6. File Structure

```
docker/
├── Dockerfile              # Production image
├── Dockerfile.dev          # Dev image (includes dev tools, hot reload)
└── .dockerignore           # Files to exclude from build context

docker-compose.yml          # Local dev stack
docker-compose.prod.yml     # Production overrides (optional)

.dockerignore contents:
  node_modules
  .git
  .env
  *.log
  dist/
  coverage/
```