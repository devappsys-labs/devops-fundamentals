# Dockerised Backend Services

This covers containerising backend services and running them with Nginx as a reverse proxy on the host.

## The Pattern

```
Internet → Host Nginx (port 80/443) → Docker Container (port 8080)
```

Your backend runs inside a Docker container. Host Nginx handles SSL and public traffic.

## Compiled Languages

### Go — Dockerfile

Go produces a static binary, so you can use a minimal base image:

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/main.go

# Stage 2: Run
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=build /app/server .
EXPOSE 8080
CMD ["./server"]
```

**Why multi-stage?** The Go toolchain is ~500MB. The final image with just the binary is ~10-20MB.

**Why `CGO_ENABLED=0`?** Produces a fully static binary that doesn't need glibc — runs on `alpine` or even `scratch`.

#### Build and run

```bash
docker build -t my-go-api .
docker run -d --name my-go-api -p 8080:8080 my-go-api

curl http://localhost:8080
```

#### Using scratch (smallest possible image)

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/main.go

FROM scratch
COPY --from=build /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

The `scratch` image is literally empty — 0 bytes. Your final image is just the binary (~5-15MB).

### Java (Spring Boot) — Dockerfile

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21-alpine AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

#### With Gradle

```dockerfile
FROM gradle:8-jdk21-alpine AS build
WORKDIR /app
COPY build.gradle settings.gradle ./
COPY src ./src
RUN gradle bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

#### Build and run

```bash
docker build -t my-java-api .
docker run -d --name my-java-api -p 8080:8080 my-java-api
```

## Interpreted Languages

### Node.js — Dockerfile

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

#### Build and run

```bash
docker build -t my-node-api .
docker run -d --name my-node-api -p 3000:3000 my-node-api
```

### Python (Flask/FastAPI/Django) — Dockerfile

**Flask + Gunicorn:**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "3", "app:app"]
```

**FastAPI + Uvicorn:**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Build and run

```bash
docker build -t my-python-api .
docker run -d --name my-python-api -p 8000:8000 my-python-api
```

## Host Nginx Configuration

Regardless of language, the host Nginx config is always the same — reverse proxy to the container's mapped port:

```bash
sudo tee /etc/nginx/sites-available/my-api > /dev/null <<'NGINX'
server {
    listen 80;
    server_name api.myapp.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts for long-running requests
        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
    }
}
NGINX

sudo ln -s /etc/nginx/sites-available/my-api /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Then add SSL with Certbot:

```bash
sudo certbot --nginx -d api.myapp.com
```

## .dockerignore

Always include this to keep images small:

```
.git
.env
node_modules
__pycache__
*.pyc
venv
dist
build
target
*.md
.vscode
.idea
```

## Environment Variables

Pass environment variables to containers:

```bash
# Inline
docker run -d \
  --name my-api \
  -p 8080:8080 \
  -e DATABASE_URL=postgres://user:pass@host/db \
  -e SECRET_KEY=mysecret \
  my-go-api

# From a file
docker run -d \
  --name my-api \
  -p 8080:8080 \
  --env-file .env \
  my-go-api
```

## Docker Compose — Full Stack Example

Running frontend + backend + database together:

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    restart: unless-stopped

  backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=myapp
    restart: unless-stopped

volumes:
  pgdata:
```

```bash
docker compose up -d
docker compose logs -f backend    # Follow backend logs
docker compose down               # Stop everything
```

Host Nginx then proxies to `localhost:3000` (frontend) and `localhost:8080` (backend).

## Auto-Restart on Server Reboot

Docker containers don't restart by default after a server reboot. Use the `--restart` flag:

```bash
docker run -d --restart unless-stopped --name my-api -p 8080:8080 my-go-api
```

Or in docker-compose.yml:
```yaml
restart: unless-stopped
```

Also make sure Docker itself starts on boot:
```bash
sudo systemctl enable docker
```

## Updating the Deployment

```bash
# Rebuild with new code
docker build -t my-api .

# Stop and remove old container
docker stop my-api
docker rm my-api

# Start with new image
docker run -d --restart unless-stopped --name my-api -p 8080:8080 my-api

# Clean up old images
docker image prune -f
```

Or with Compose:
```bash
docker compose up -d --build
```

## Troubleshooting

**Container exits immediately**
- Check logs: `docker logs my-api`
- Run interactively to debug: `docker run -it my-api sh`

**Cannot connect to database**
- Containers communicate via Docker network, not `localhost`
- Use the service name as hostname in Compose (e.g., `db` not `localhost`)

**Port conflict**
- `docker ps` to see what's running
- `sudo lsof -i :8080` to find what's using the port

**Image too large**
- Use Alpine base images
- Use multi-stage builds
- Add `.dockerignore`
- Check image size: `docker images`

**Permission issues inside container**
- Don't run as root: add `USER` directive in Dockerfile
- For Go: `USER nobody` (since it's a static binary)
- For Node: `USER node` (built into node image)

---

**Back to:** [Table of Contents](../../README.md)
