# Docker Best Practices

## Image Best Practices

### Use specific tags, not `latest`

```dockerfile
# Bad — can break without warning
FROM node:latest

# Good — predictable and reproducible
FROM node:20-alpine
```

### Use small base images

```dockerfile
# node:20        ~1GB
# node:20-slim   ~200MB
# node:20-alpine ~130MB  ← Use this

FROM node:20-alpine
```

### Order Dockerfile instructions for caching

Put things that change least first, things that change most last:

```dockerfile
FROM node:20-alpine
WORKDIR /app

# These rarely change → cached
COPY package.json package-lock.json ./
RUN npm ci --production

# This changes every deploy → only this layer rebuilds
COPY . .

CMD ["node", "app.js"]
```

If you put `COPY . .` before `RUN npm install`, every code change triggers a full `npm install` — very slow.

### Use multi-stage builds

```dockerfile
# Stage 1: Build (big image with build tools)
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production (small image)
FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY package.json .
CMD ["node", "dist/index.js"]
```

For Go:

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM scratch
COPY --from=build /app/server /server
CMD ["/server"]
```

Final Go image: ~5-15MB.

### Minimize layers

```dockerfile
# Bad — 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Good — 1 layer
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### Don't run as root

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY --chown=node:node . .
RUN npm ci --production
USER node
CMD ["node", "app.js"]
```

If the container is compromised, the attacker gets `node` privileges, not `root`.

### Use .dockerignore

```
node_modules
.git
.env
*.md
.vscode
.idea
dist
build
__pycache__
*.pyc
.pytest_cache
coverage
```

Without this, `COPY . .` copies everything — including `node_modules` (which gets overwritten by `npm install` anyway) and `.git` (potentially huge).

### Don't store secrets in images

```dockerfile
# NEVER do this
ENV DATABASE_PASSWORD=mysecret
COPY .env /app/.env

# Do this instead — pass at runtime
# docker run -e DATABASE_PASSWORD=mysecret myapp
# docker run --env-file .env myapp
```

Anyone who pulls your image can see `ENV` values with `docker inspect`.

## Container Best Practices

### One process per container

```yaml
# Good — separate containers
services:
  api:
    build: ./api
  worker:
    build: ./worker
  db:
    image: postgres:16
```

```yaml
# Bad — cramming everything into one container
services:
  app:
    build: .
    command: "supervisord"  # Running multiple processes
```

### Use health checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

Or in Compose:

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

Docker uses health checks to know if your app is actually working, not just running.

### Handle signals properly

Your app should handle `SIGTERM` to shut down gracefully:

```javascript
// Node.js
process.on('SIGTERM', () => {
  console.log('Received SIGTERM, shutting down...');
  server.close(() => process.exit(0));
});
```

`docker stop` sends `SIGTERM`, waits 10 seconds, then sends `SIGKILL`. If your app doesn't handle SIGTERM, it gets killed hard.

### Use `exec` form for CMD

```dockerfile
# Good — exec form (PID 1, receives signals properly)
CMD ["node", "app.js"]

# Bad — shell form (runs as /bin/sh -c, signals may not reach your app)
CMD node app.js
```

### Log to stdout/stderr

```dockerfile
# Don't log to files inside the container
# Log to stdout — Docker captures it automatically

# In your app:
# console.log() → stdout → docker logs
# console.error() → stderr → docker logs
```

```bash
docker logs myapp          # See the logs
docker logs -f myapp       # Follow in real time
```

## Security Best Practices

### Don't run as root (repeated because it's critical)

```dockerfile
USER node       # Node.js
USER nobody     # Go (static binary)
USER 1001       # Specific UID
```

### Scan images for vulnerabilities

```bash
# Docker Scout (built into Docker)
docker scout cve myapp

# Trivy (popular open-source scanner)
trivy image myapp
```

### Use read-only filesystem where possible

```bash
docker run --read-only --tmpfs /tmp myapp
```

Or in Compose:

```yaml
services:
  api:
    read_only: true
    tmpfs:
      - /tmp
```

### Don't expose unnecessary ports

```yaml
services:
  db:
    image: postgres:16
    # No 'ports:' — only accessible within Docker network
    # DON'T do: ports: ["5432:5432"]

  api:
    ports:
      - "127.0.0.1:8080:8080"  # Localhost only, not 0.0.0.0
```

## Performance Best Practices

### Use Alpine images

Alpine Linux is ~7MB vs Ubuntu's ~77MB. Smaller images = faster pulls, less disk, smaller attack surface.

### Cache dependency installation

```dockerfile
# Copy dependency files FIRST
COPY package.json package-lock.json ./
RUN npm ci

# THEN copy source code
COPY . .
```

If dependencies haven't changed, Docker uses the cached layer.

### Use BuildKit

```bash
# Enable BuildKit (faster, parallel builds)
DOCKER_BUILDKIT=1 docker build -t myapp .

# Or set it globally in /etc/docker/daemon.json:
# { "features": { "buildkit": true } }
```

### Clean up in the same RUN layer

```dockerfile
RUN apt-get update && \
    apt-get install -y build-essential && \
    npm run build && \
    apt-get remove -y build-essential && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
```

If you remove packages in a separate `RUN`, the files still exist in the previous layer — the image doesn't get smaller.

## Tagging Strategy

```bash
# Tag with git SHA for traceability
docker build -t myapp:$(git rev-parse --short HEAD) .
docker build -t myapp:latest .

# Tag with version
docker build -t myapp:v1.2.3 .

# Full tagging for a release
docker build \
  -t myapp:latest \
  -t myapp:v1.2.3 \
  -t myapp:$(git rev-parse --short HEAD) \
  .
```

- `latest` — convenience, always points to newest
- `v1.2.3` — semantic version for releases
- `abc123` — git SHA for exact traceability

## Cleanup

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune           # Dangling only
docker image prune -a        # All unused

# Remove unused volumes (careful — deletes data!)
docker volume prune

# Remove unused networks
docker network prune

# Nuclear option: remove EVERYTHING unused
docker system prune -a --volumes

# Check disk usage
docker system df
```

---

**Back to:** [Table of Contents](../README.md)
