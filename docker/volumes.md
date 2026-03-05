# Docker Volumes

Containers are **ephemeral** — when you remove a container, all data inside it is lost. Volumes solve this by persisting data outside the container's lifecycle.

## The Problem

```bash
# Run a PostgreSQL container
docker run -d --name mydb -e POSTGRES_PASSWORD=secret postgres:16

# Create some data...
docker exec mydb psql -U postgres -c "CREATE TABLE users (id serial, name text);"
docker exec mydb psql -U postgres -c "INSERT INTO users (name) VALUES ('Alice');"

# Remove the container
docker rm -f mydb

# Start a new one
docker run -d --name mydb -e POSTGRES_PASSWORD=secret postgres:16

# Data is GONE
docker exec mydb psql -U postgres -c "SELECT * FROM users;"
# ERROR: relation "users" does not exist
```

## Three Ways to Persist Data

### Named Volumes (Recommended)

Docker manages the storage location. Best for databases and application data.

```bash
# Create a named volume
docker volume create mydata

# Run container with the volume
docker run -d \
  --name mydb \
  -v mydata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# Now if you remove and recreate the container, data survives
docker rm -f mydb

docker run -d \
  --name mydb \
  -v mydata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# Data is still there!
```

### Bind Mounts

Map a specific host directory into the container. Best for development and config files.

```bash
# Mount a host directory into the container
docker run -d \
  --name my-nginx \
  -p 80:80 \
  -v /var/www/mysite:/usr/share/nginx/html:ro \
  nginx

# :ro = read-only (container can't modify host files)
# Without :ro, the container can write to the host directory
```

### tmpfs Mounts

In-memory storage. Fast, but data is lost when the container stops. Good for sensitive data that shouldn't be written to disk.

```bash
docker run -d \
  --name myapp \
  --tmpfs /app/tmp \
  myapp
```

## Volume Commands

```bash
# Create a volume
docker volume create mydata

# List all volumes
docker volume ls

# Inspect a volume (see where it's stored on disk)
docker volume inspect mydata
# "Mountpoint": "/var/lib/docker/volumes/mydata/_data"

# Remove a volume
docker volume rm mydata

# Remove ALL unused volumes
docker volume prune

# Remove a volume with its container
docker rm -v mycontainer
```

## Volume Syntax: -v vs --mount

Two syntaxes that do the same thing. `--mount` is more explicit.

```bash
# -v syntax (shorter)
docker run -v mydata:/app/data myapp
docker run -v /host/path:/container/path myapp
docker run -v /host/path:/container/path:ro myapp

# --mount syntax (more explicit)
docker run --mount type=volume,source=mydata,target=/app/data myapp
docker run --mount type=bind,source=/host/path,target=/container/path myapp
docker run --mount type=bind,source=/host/path,target=/container/path,readonly myapp
```

**Key difference:** `-v` auto-creates directories if they don't exist. `--mount` fails if the source doesn't exist (safer).

## Common Volume Patterns

### Database persistence

```bash
# PostgreSQL
docker run -d \
  --name postgres \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  postgres:16-alpine

# MySQL
docker run -d \
  --name mysql \
  -v mysqldata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -p 3306:3306 \
  mysql:8

# MongoDB
docker run -d \
  --name mongo \
  -v mongodata:/data/db \
  -p 27017:27017 \
  mongo:7

# Redis (with persistence)
docker run -d \
  --name redis \
  -v redisdata:/data \
  -p 6379:6379 \
  redis:7-alpine redis-server --appendonly yes
```

### Application logs

```bash
docker run -d \
  --name myapp \
  -v /var/log/myapp:/app/logs \
  myapp
```

Logs are written to the host filesystem, so they survive container restarts and can be processed by log tools.

### Configuration files

```bash
# Mount Nginx config
docker run -d \
  --name nginx \
  -v /home/deploy/nginx.conf:/etc/nginx/conf.d/default.conf:ro \
  -v /var/www/mysite:/usr/share/nginx/html:ro \
  -p 80:80 \
  nginx

# Mount app config
docker run -d \
  --name myapp \
  -v /home/deploy/config.yml:/app/config.yml:ro \
  myapp
```

Use `:ro` for config files — the container should read them, not modify them.

### Development: Live code reloading

```bash
# Mount source code so changes appear immediately in the container
docker run -d \
  --name dev-app \
  -v $(pwd):/app \
  -v /app/node_modules \
  -p 3000:3000 \
  myapp npm run dev

# The second -v (/app/node_modules) creates an anonymous volume
# This prevents the host's node_modules from overriding the container's
```

## Volumes in Docker Compose

```yaml
services:
  app:
    build: .
    volumes:
      - ./src:/app/src                     # Bind mount (development)
      - app-logs:/app/logs                 # Named volume

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data    # Named volume
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  pgdata:                                   # Declare named volumes
  app-logs:
```

### Volume with specific driver options

```yaml
volumes:
  pgdata:
    driver: local
    driver_opts:
      type: none
      device: /data/postgres             # Specific host path
      o: bind
```

## Backup and Restore

### Backup a volume

```bash
# Create a backup of the pgdata volume
docker run --rm \
  -v pgdata:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata-backup.tar.gz -C /source .

# This:
# 1. Mounts the volume as read-only at /source
# 2. Mounts current directory at /backup
# 3. Creates a tar.gz archive
```

### Restore a volume

```bash
# Create a new volume and restore
docker volume create pgdata-restored

docker run --rm \
  -v pgdata-restored:/target \
  -v $(pwd):/backup:ro \
  alpine tar xzf /backup/pgdata-backup.tar.gz -C /target
```

### Database-specific backups (better approach)

```bash
# PostgreSQL dump
docker exec mydb pg_dump -U postgres mydb > backup.sql

# PostgreSQL restore
docker exec -i mydb psql -U postgres mydb < backup.sql

# MySQL dump
docker exec mydb mysqldump -u root -p mydb > backup.sql
```

## Volume Permissions

A common problem: the container process can't write to a mounted volume because of UID mismatches.

```bash
# Check what user the container runs as
docker exec myapp id
# uid=1000(node) gid=1000(node)

# The host directory might be owned by a different UID
ls -la /host/data
# drwxr-xr-x root root /host/data

# Fix: set ownership to match the container's user
sudo chown -R 1000:1000 /host/data
```

### In Dockerfile

```dockerfile
FROM node:20-alpine

# Create and own the app directory
RUN mkdir -p /app/data && chown -R node:node /app/data

USER node
WORKDIR /app
# ...
```

## Volume Gotchas

**Volume data persists even after `docker compose down`**
```bash
docker compose down           # Containers removed, volumes kept
docker compose down -v        # Containers AND volumes removed
```

**Anonymous volumes accumulate**
```bash
docker volume ls               # Lists all volumes including anonymous ones
docker volume prune            # Clean up unused volumes
```

**Bind mount hides container files**
If you mount `/host/empty-dir` to `/app` in the container, the container's `/app` contents are hidden (not merged). The mount "wins."

**SELinux (if applicable)**
On systems with SELinux, add `:z` or `:Z`:
```bash
docker run -v /host/data:/app/data:z myapp
```

---

**Next:** [Docker Networking](networking.md)
