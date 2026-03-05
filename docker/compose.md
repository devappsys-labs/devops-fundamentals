# Docker Compose

Docker Compose lets you define and run multi-container applications with a single YAML file instead of long `docker run` commands.

## Why Compose?

Without Compose, running a full-stack app means:

```bash
docker network create myapp
docker volume create pgdata
docker run -d --name db --network myapp -v pgdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres:16
docker run -d --name cache --network myapp redis:7-alpine
docker run -d --name api --network myapp -p 8080:8080 -e DATABASE_URL=postgres://postgres:secret@db:5432/myapp -e REDIS_URL=redis://cache:6379 my-backend
docker run -d --name frontend --network myapp -p 80:80 my-frontend
```

With Compose, it's one file and one command:

```yaml
# docker-compose.yml
services:
  frontend:
    build: ./frontend
    ports:
      - "80:80"

  api:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/myapp
      REDIS_URL: redis://cache:6379
    depends_on:
      - db
      - cache

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

  cache:
    image: redis:7-alpine

volumes:
  pgdata:
```

```bash
docker compose up -d
```

## Essential Commands

```bash
# Start all services (detached)
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# Stop all services
docker compose down

# Stop and remove volumes too (destroys data!)
docker compose down -v

# View running services
docker compose ps

# View logs
docker compose logs
docker compose logs -f              # Follow
docker compose logs api             # Specific service
docker compose logs -f --tail 50 api

# Restart a specific service
docker compose restart api

# Stop a specific service
docker compose stop api

# Execute a command in a running service
docker compose exec api sh
docker compose exec db psql -U postgres

# Run a one-off command
docker compose run --rm api npm test

# Rebuild a specific service
docker compose build api

# Pull latest images
docker compose pull
```

## docker-compose.yml Reference

### Building from source

```yaml
services:
  api:
    # Simple build
    build: ./backend

    # With more options
    build:
      context: ./backend
      dockerfile: Dockerfile.production
      args:
        NODE_ENV: production
```

### Using pre-built images

```yaml
services:
  db:
    image: postgres:16-alpine

  cache:
    image: redis:7-alpine

  api:
    image: ghcr.io/myuser/my-api:latest
```

### Port mapping

```yaml
services:
  api:
    ports:
      - "8080:8080"                    # host:container
      - "127.0.0.1:8080:8080"         # localhost only
      - "8080"                         # random host port
```

### Environment variables

```yaml
services:
  api:
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://user:pass@db:5432/myapp

    # Or from a file
    env_file:
      - .env
      - .env.production
```

### Volumes

```yaml
services:
  api:
    volumes:
      - ./src:/app/src                 # Bind mount
      - app-data:/app/data             # Named volume
      - /app/node_modules              # Anonymous volume

  db:
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
  app-data:
```

### Depends on

Controls startup order:

```yaml
services:
  api:
    depends_on:
      - db
      - cache
    # api starts AFTER db and cache

  db:
    image: postgres:16-alpine

  cache:
    image: redis:7-alpine
```

**Important:** `depends_on` only waits for the container to start, not for the service inside to be ready. PostgreSQL might still be initializing when your app tries to connect.

### Depends on with health checks

Wait for a service to actually be ready:

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/myapp

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

Now `api` only starts after `db` passes its health check.

### Restart policies

```yaml
services:
  api:
    restart: unless-stopped    # Restart unless manually stopped

  db:
    restart: always            # Always restart

  worker:
    restart: on-failure        # Only restart on crash
```

### Resource limits

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
```

### Networks

```yaml
services:
  frontend:
    networks:
      - frontend-net

  api:
    networks:
      - frontend-net
      - backend-net

  db:
    networks:
      - backend-net

networks:
  frontend-net:
  backend-net:
```

### Healthchecks

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s     # Grace period before health checks start
```

## Compose Profiles

Run different subsets of services for different environments:

```yaml
services:
  api:
    build: ./backend
    ports:
      - "8080:8080"

  frontend:
    build: ./frontend
    ports:
      - "80:80"

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data

  # Only run in development
  adminer:
    image: adminer
    ports:
      - "8081:8080"
    profiles:
      - dev

  # Only run for debugging
  debug-tools:
    image: alpine
    profiles:
      - debug

volumes:
  pgdata:
```

```bash
docker compose up -d                        # api, frontend, db only
docker compose --profile dev up -d          # Include adminer
docker compose --profile debug up -d        # Include debug-tools
```

## Common Compose Patterns

### Full-stack web app

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    restart: unless-stopped

  api:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://postgres:${DB_PASSWORD}@db:5432/myapp
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    restart: unless-stopped

volumes:
  pgdata:
```

With a `.env` file:

```
DB_PASSWORD=mysecretpassword
```

### Development vs Production

```yaml
# docker-compose.yml (base)
services:
  api:
    build: ./backend
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/myapp
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```yaml
# docker-compose.override.yml (auto-loaded in dev)
services:
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    volumes:
      - ./backend/src:/app/src     # Live reload
    ports:
      - "8080:8080"
      - "9229:9229"               # Debug port
    environment:
      NODE_ENV: development

  db:
    ports:
      - "5432:5432"               # Expose DB in dev
```

```yaml
# docker-compose.production.yml
services:
  api:
    image: ghcr.io/myuser/my-api:latest
    restart: unless-stopped
    environment:
      NODE_ENV: production

  db:
    restart: unless-stopped
    # No ports exposed!
```

```bash
# Development (uses docker-compose.yml + docker-compose.override.yml)
docker compose up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.production.yml up -d
```

## Compose with Multiple Dockerfiles

```
my-project/
├── docker-compose.yml
├── frontend/
│   ├── Dockerfile
│   ├── src/
│   └── package.json
├── backend/
│   ├── Dockerfile
│   ├── src/
│   └── package.json
└── .env
```

```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:80"

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
```

## Troubleshooting Compose

**Service won't start**
```bash
docker compose logs <service>         # Check logs
docker compose ps -a                   # Check exit codes
```

**"Port already in use"**
```bash
docker compose down                    # Stop everything
sudo lsof -i :<port>                  # Find what's using it
```

**Changes not reflected after rebuild**
```bash
docker compose up -d --build --force-recreate
```

**Database data lost after `docker compose down`**
- `docker compose down` preserves volumes
- `docker compose down -v` deletes volumes — don't use `-v` unless you want to lose data

**Container can't reach another service**
- Use the service name as hostname (not `localhost`)
- They must be on the same network (Compose handles this automatically)

---

**Next:** [Docker Best Practices](best-practices.md)
