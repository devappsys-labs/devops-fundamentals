# Dockerised Frontend with Nginx

This covers running your frontend app inside a Docker container with Nginx serving the files. This is how most production frontends are deployed.

## Why Dockerise?

- **Consistency** — Same environment on every server
- **Isolation** — App dependencies don't conflict with the host system
- **Portability** — Build once, run anywhere
- **Easy rollbacks** — Just run the previous image

## Prerequisites

Install Docker on your server:

```bash
# Install Docker
curl -fsSL https://get.docker.com | sudo sh

# Add your user to the docker group (so you don't need sudo)
sudo usermod -aG docker $USER

# Log out and back in for group change to take effect
# Then verify
docker --version
```

## Pattern: Multi-Stage Build

The standard approach is a **multi-stage Docker build**:
1. **Stage 1 (Build):** Install dependencies and build the app
2. **Stage 2 (Serve):** Copy the build output into an Nginx image

This keeps the final image small — it only contains Nginx and your built files, not Node.js or `node_modules`.

## Dockerising a React App (Vite)

### Project structure

```
my-react-app/
├── src/
├── public/
├── package.json
├── vite.config.js
├── Dockerfile
└── nginx.conf
```

### Dockerfile

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### nginx.conf (for the container)

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location = /index.html {
        add_header Cache-Control "no-cache";
    }

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 256;
}
```

### Build and run

```bash
# Build the Docker image
docker build -t my-react-app .

# Run it
docker run -d --name my-react-app -p 3000:80 my-react-app

# Verify
curl http://localhost:3000
```

The container's Nginx listens on port 80 inside the container, mapped to port 3000 on the host.

## Dockerising an Angular App

### Dockerfile

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx ng build --configuration production

# Stage 2: Serve
FROM nginx:alpine
COPY --from=build /app/dist/my-angular-app/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

The `nginx.conf` is the same as for React (SPA fallback to `index.html`).

## Host Nginx as Reverse Proxy to Docker Container

In production, you typically have **Nginx on the host** in front of the **Nginx inside Docker**:

```mermaid
flowchart LR
    A[Internet] --> B["Host Nginx\n(port 80/443, SSL)"] --> C["Docker Container\n(port 3000 → internal 80)"]
```

### Host Nginx config

```bash
sudo tee /etc/nginx/sites-available/my-react-app > /dev/null <<'NGINX'
server {
    listen 80;
    server_name myapp.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
NGINX

sudo ln -s /etc/nginx/sites-available/my-react-app /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Now `myapp.com` → host Nginx → Docker container → container Nginx → serves files.

You can then add SSL with Certbot on the host Nginx as covered in [SSL/TLS with Certbot](../../nginx/ssl-tls.md).

## .dockerignore

Always include a `.dockerignore` to keep your image small and avoid copying unnecessary files:

```
node_modules
dist
build
.git
.env
*.md
.vscode
```

## Docker Compose (Optional)

For convenience, use `docker-compose.yml`:

```yaml
services:
  frontend:
    build: .
    ports:
      - "3000:80"
    restart: unless-stopped
```

```bash
# Start
docker compose up -d

# Stop
docker compose down

# Rebuild and restart
docker compose up -d --build
```

## Updating the Deployment

```bash
# Rebuild the image with new code
docker build -t my-react-app .

# Stop and remove the old container
docker stop my-react-app
docker rm my-react-app

# Start with the new image
docker run -d --name my-react-app -p 3000:80 my-react-app
```

Or with Docker Compose:

```bash
docker compose up -d --build
```

## Troubleshooting

**Container starts but page is blank**
- Check if the build output path in the Dockerfile matches your framework
- React Vite: `dist/`, CRA: `build/`, Angular: `dist/<name>/browser/`
- Enter the container and check: `docker exec -it my-react-app ls /usr/share/nginx/html`

**Container exits immediately**
- Check logs: `docker logs my-react-app`
- Usually a build failure or bad nginx.conf

**404 on refresh (SPA)**
- Check that `nginx.conf` is being copied into the container correctly
- Verify: `docker exec -it my-react-app cat /etc/nginx/conf.d/default.conf`

**Port conflict**
- Something else is using port 3000: `sudo lsof -i :3000`
- Change the host port: `-p 3001:80`

---

**Back to:** [Table of Contents](../../README.md)
