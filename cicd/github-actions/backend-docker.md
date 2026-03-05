# CI/CD for Backend Services (With Docker)

Automate building Docker images of your backend services and deploying containers to your server.

## Prerequisites

- Docker installed on your server
- Nginx on the host configured as reverse proxy (see [Dockerised Backend](../../manual-server-deployment/backend/dockerised.md))
- GitHub Secrets: `SERVER_HOST`, `SERVER_USER`, `SSH_PRIVATE_KEY`
- A `Dockerfile` in your repo

## Compiled Languages

### Go — deploy.yml

```yaml
name: Deploy Go Service (Docker)

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go test ./...

  build-and-deploy:
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to GHCR
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

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          envs: REGISTRY,IMAGE_NAME
          script: |
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

            docker pull $REGISTRY/$IMAGE_NAME:latest

            docker stop my-go-api || true
            docker rm my-go-api || true

            docker run -d \
              --name my-go-api \
              --restart unless-stopped \
              -p 8080:8080 \
              --env-file /home/deploy/my-go-api/.env \
              $REGISTRY/$IMAGE_NAME:latest

            docker image prune -f

      - name: Health check
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            sleep 3
            curl -f http://localhost:8080/health || exit 1
        env:
          REGISTRY: ghcr.io
          IMAGE_NAME: ${{ github.repository }}
```

### Java (Spring Boot) — deploy.yml

```yaml
name: Deploy Spring Boot (Docker)

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
          cache: 'maven'
      - run: mvn test

  build-and-deploy:
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to GHCR
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

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          envs: REGISTRY,IMAGE_NAME
          script: |
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

            docker pull $REGISTRY/$IMAGE_NAME:latest

            docker stop my-java-api || true
            docker rm my-java-api || true

            docker run -d \
              --name my-java-api \
              --restart unless-stopped \
              -p 8080:8080 \
              --env-file /home/deploy/my-java-api/.env \
              $REGISTRY/$IMAGE_NAME:latest

            docker image prune -f

            # Java needs more time to start
            sleep 15
            curl -f http://localhost:8080/actuator/health || exit 1
        env:
          REGISTRY: ghcr.io
          IMAGE_NAME: ${{ github.repository }}
```

## Interpreted Languages

### Node.js — deploy.yml

```yaml
name: Deploy Node.js Service (Docker)

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm test

  build-and-deploy:
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to GHCR
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

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          envs: REGISTRY,IMAGE_NAME
          script: |
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

            docker pull $REGISTRY/$IMAGE_NAME:latest

            docker stop my-node-api || true
            docker rm my-node-api || true

            docker run -d \
              --name my-node-api \
              --restart unless-stopped \
              -p 3000:3000 \
              --env-file /home/deploy/my-node-api/.env \
              $REGISTRY/$IMAGE_NAME:latest

            docker image prune -f

            sleep 3
            curl -f http://localhost:3000/health || exit 1
        env:
          REGISTRY: ghcr.io
          IMAGE_NAME: ${{ github.repository }}
```

### Python (Flask / FastAPI) — deploy.yml

```yaml
name: Deploy Python Service (Docker)

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      - run: |
          pip install -r requirements.txt
          pytest

  build-and-deploy:
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to GHCR
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

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          envs: REGISTRY,IMAGE_NAME
          script: |
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

            docker pull $REGISTRY/$IMAGE_NAME:latest

            docker stop my-python-api || true
            docker rm my-python-api || true

            docker run -d \
              --name my-python-api \
              --restart unless-stopped \
              -p 8000:8000 \
              --env-file /home/deploy/my-python-api/.env \
              $REGISTRY/$IMAGE_NAME:latest

            docker image prune -f

            sleep 3
            curl -f http://localhost:8000/health || exit 1
        env:
          REGISTRY: ghcr.io
          IMAGE_NAME: ${{ github.repository }}
```

## Build on Server (Simple Alternative)

If you don't want a registry, just build on the server:

```yaml
name: Deploy (Build on Server)

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
            cd /home/deploy/my-api
            git pull origin main

            docker compose down
            docker compose up -d --build
            docker image prune -f

            sleep 5
            curl -f http://localhost:8080/health || exit 1
```

## Docker Compose — Full Stack Deploy

Deploy frontend + backend + database together:

```yaml
name: Deploy Full Stack

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm test

  deploy:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/my-fullstack-app
            git pull origin main

            # Rebuild only changed services
            docker compose up -d --build

            # Clean up
            docker image prune -f

            # Health checks
            sleep 10
            curl -f http://localhost:3000 || exit 1     # Frontend
            curl -f http://localhost:8080/health || exit 1  # Backend
```

The `docker-compose.yml` on the server:

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
    env_file: .env
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    env_file: .env.db
    restart: unless-stopped

volumes:
  pgdata:
```

## Rollback

Every image is tagged with the git SHA, so rollback is straightforward:

```bash
# On the server
docker stop my-go-api
docker rm my-go-api
docker run -d \
  --name my-go-api \
  --restart unless-stopped \
  -p 8080:8080 \
  --env-file /home/deploy/my-go-api/.env \
  ghcr.io/myuser/my-api:abc123def456   # Previous working SHA
```

You can also automate this in a separate workflow triggered manually:

```yaml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      sha:
        description: 'Git SHA to roll back to'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Rollback on server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull ghcr.io/${{ github.repository }}:${{ github.event.inputs.sha }}

            docker stop my-api || true
            docker rm my-api || true

            docker run -d \
              --name my-api \
              --restart unless-stopped \
              -p 8080:8080 \
              --env-file /home/deploy/my-api/.env \
              ghcr.io/${{ github.repository }}:${{ github.event.inputs.sha }}
```

Trigger from GitHub UI: **Actions** → **Rollback** → **Run workflow** → enter the SHA.

## Troubleshooting

**Image push fails**
- Check `permissions.packages: write` is set
- Ensure `GITHUB_TOKEN` is not restricted in repo settings

**Container starts but exits immediately**
- `docker logs my-api` — check for startup errors
- Common: missing env vars, wrong port, bad CMD

**Health check fails after deploy**
- App might need more startup time — increase `sleep`
- Check if the health endpoint exists and returns 200
- Test manually: `docker exec my-api curl localhost:8080/health`

**"No space left on device"**
- Old images accumulating: `docker system prune -a`
- Add `docker image prune -f` to the deploy script (already included in examples above)

**Database connection refused in Docker**
- Containers can't reach the host's `localhost`
- Use Docker Compose service names, or `host.docker.internal` for host services
- Or use `--network host` (simple but less isolated)

---

**Back to:** [Table of Contents](../../README.md)
