# CI/CD for Frontend Apps (With Docker)

Automate building Docker images of your frontend app and deploying containers to your server.

## Prerequisites

- Docker installed on your server
- Nginx on the host configured as reverse proxy (see [Dockerised Frontend](../../manual-server-deployment/frontend/dockerised-with-nginx.md))
- GitHub Secrets: `SERVER_HOST`, `SERVER_USER`, `SSH_PRIVATE_KEY`
- A `Dockerfile` in your repo (see [Dockerised Frontend](../../manual-server-deployment/frontend/dockerised-with-nginx.md))

## Strategy: Build on Server

The simplest approach — push code, server pulls and builds the image locally.

```
Push to main → GitHub Actions SSHes into server → Server pulls code, builds image, restarts container
```

### React / Angular — deploy.yml

```yaml
name: Deploy Frontend (Docker)

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/my-react-app

            # Pull latest code
            git pull origin main

            # Build new image
            docker build -t my-react-app .

            # Stop and remove old container
            docker stop my-react-app || true
            docker rm my-react-app || true

            # Start new container
            docker run -d \
              --name my-react-app \
              --restart unless-stopped \
              -p 3000:80 \
              my-react-app

            # Clean up old images
            docker image prune -f
```

**Pros:** Simple, no registry needed.
**Cons:** Build uses server CPU/memory, code must be cloned on server.

## Strategy: Build on Runner, Push to Registry

Build the image on the GitHub Actions runner, push to a container registry, then pull on the server. Better for production.

```
Push to main → Runner builds image → Push to registry → Server pulls image → Restart container
```

### Using GitHub Container Registry (GHCR)

```yaml
name: Deploy Frontend (Docker + Registry)

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push

    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            # Login to registry
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

            # Pull new image
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

            # Stop and remove old container
            docker stop my-react-app || true
            docker rm my-react-app || true

            # Start new container
            docker run -d \
              --name my-react-app \
              --restart unless-stopped \
              -p 3000:80 \
              ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

            # Clean up old images
            docker image prune -f
        env:
          REGISTRY: ghcr.io
          IMAGE_NAME: ${{ github.repository }}
```

### Using Docker Hub

Add secrets: `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`

```yaml
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/my-react-app:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/my-react-app:${{ github.sha }}
```

## Strategy: Docker Compose on Server

If your app uses `docker-compose.yml`:

```yaml
name: Deploy Frontend (Docker Compose)

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/my-react-app
            git pull origin main
            docker compose up -d --build
            docker image prune -f
```

`docker compose up -d --build` rebuilds the image and restarts only if something changed.

## Zero-Downtime Deployment

The basic approach has a brief downtime between stopping the old container and starting the new one. To avoid this:

```yaml
      - name: Zero-downtime deploy
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/my-react-app
            git pull origin main

            # Build new image
            docker build -t my-react-app:new .

            # Start new container on a different port
            docker run -d --name my-react-app-new -p 3001:80 my-react-app:new

            # Wait for it to be healthy
            sleep 5
            curl -f http://localhost:3001 || exit 1

            # Switch Nginx to new container
            sudo sed -i 's/localhost:3000/localhost:3001/' /etc/nginx/sites-available/my-react-app
            sudo nginx -t && sudo systemctl reload nginx

            # Stop old container
            docker stop my-react-app || true
            docker rm my-react-app || true

            # Rename new container
            docker rename my-react-app-new my-react-app

            # Update Nginx back to standard port for next deploy
            sudo sed -i 's/localhost:3001/localhost:3000/' /etc/nginx/sites-available/my-react-app
            # (Next deploy will use 3001 again as the "new" port)

            docker image prune -f
```

This is a simplified approach. For true zero-downtime, consider using Docker Swarm or Kubernetes.

## Rollback

Tag images with the git SHA so you can roll back:

```bash
# On the server — roll back to a specific version
docker stop my-react-app
docker rm my-react-app
docker run -d --name my-react-app --restart unless-stopped -p 3000:80 \
  ghcr.io/myuser/myapp:abc123def   # Use the git SHA of the working version
```

This is why we push with both `latest` and `${{ github.sha }}` tags.

## Troubleshooting

**"Cannot connect to the Docker daemon"**
- Ensure Docker is installed and running on the server
- Ensure the deploy user is in the `docker` group: `sudo usermod -aG docker deploy`

**Image push fails with 403**
- For GHCR: ensure `permissions.packages: write` is set in the workflow
- For Docker Hub: check that `DOCKERHUB_TOKEN` has push access

**Container starts but site is unreachable**
- Check container is running: `docker ps`
- Check container logs: `docker logs my-react-app`
- Check host Nginx is proxying to the right port

**Old image being used**
- Docker caches layers aggressively
- Use `docker build --no-cache -t my-react-app .` if needed
- Or use `docker compose up -d --build --force-recreate`

---

**Next:** [Backend CI/CD](backend.md)
