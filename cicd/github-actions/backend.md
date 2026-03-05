# CI/CD for Backend Services (Without Docker)

Automate building and deploying backend services to your server using GitHub Actions.

## Prerequisites

- Server with Nginx configured as reverse proxy (see [Reverse Proxy](../../nginx/reverse-proxy.md))
- systemd service set up for your app (see [Backend Services](../../manual-server-deployment/backend/services.md))
- GitHub Secrets: `SERVER_HOST`, `SERVER_USER`, `SSH_PRIVATE_KEY`

## Compiled Languages

Compiled languages are built on the GitHub Actions runner and the binary is copied to the server. The server doesn't need the build toolchain.

### Go — deploy.yml

```yaml
name: Deploy Go Service

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go test ./...

  deploy:
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Build for Linux
        run: |
          CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o myapp ./cmd/main.go

      - name: Copy binary to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "myapp"
          target: "/home/deploy/"

      - name: Restart service
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            # Stop the service
            sudo systemctl stop myapp

            # Replace binary
            sudo mv /home/deploy/myapp /usr/local/bin/myapp
            sudo chmod +x /usr/local/bin/myapp

            # Start the service
            sudo systemctl start myapp

            # Verify it's running
            sleep 2
            sudo systemctl is-active myapp
```

### Why build on the runner?

- Go cross-compilation is trivial (`GOOS=linux GOARCH=amd64`)
- No Go toolchain needed on the server
- Doesn't use server resources for building
- The binary is a single file — easy to copy

### Java (Spring Boot) — deploy.yml

```yaml
name: Deploy Spring Boot Service

on:
  push:
    branches: [main]

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

  deploy:
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
          cache: 'maven'

      - name: Build JAR
        run: mvn clean package -DskipTests

      - name: Copy JAR to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "target/*.jar"
          target: "/home/deploy/"
          strip_components: 1

      - name: Restart service
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            # Move JAR to app directory
            sudo mv /home/deploy/*.jar /opt/myapp/app.jar
            sudo chown www-data:www-data /opt/myapp/app.jar

            # Restart
            sudo systemctl restart myapp

            # Wait and verify
            sleep 10
            sudo systemctl is-active myapp
```

Java needs a longer `sleep` before verification — the JVM takes time to start up.

### Java with Gradle

Replace the Maven steps:

```yaml
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
          cache: 'gradle'

      - name: Build JAR
        run: ./gradlew bootJar

      - name: Copy JAR to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "build/libs/*.jar"
          target: "/home/deploy/"
          strip_components: 2
```

## Interpreted Languages

Interpreted languages need the runtime on the server. You either copy the source code and install dependencies on the server, or copy everything from the runner.

### Node.js (Express / Fastify) — deploy.yml

```yaml
name: Deploy Node.js Service

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
            cd /home/deploy/my-node-app

            # Pull latest code
            git pull origin main

            # Install production dependencies
            npm ci --production

            # Restart
            sudo systemctl restart nodeapp

            # Verify
            sleep 2
            sudo systemctl is-active nodeapp
```

### Alternative: Copy source from runner

If you don't want git on the server:

```yaml
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Copy source to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "*.js,*.json,src/*"
          target: "/home/deploy/my-node-app"

      - name: Install deps and restart
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/my-node-app
            npm ci --production
            sudo systemctl restart nodeapp
```

### Python (Flask / FastAPI / Django) — deploy.yml

```yaml
name: Deploy Python Service

on:
  push:
    branches: [main]

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
            cd /home/deploy/my-python-app

            # Pull latest code
            git pull origin main

            # Update dependencies in virtualenv
            source venv/bin/activate
            pip install -r requirements.txt

            # Run migrations (if Django)
            # python manage.py migrate

            # Restart
            sudo systemctl restart flask-app

            # Verify
            sleep 2
            sudo systemctl is-active flask-app
```

### Django with Migrations

Django deployments often need database migrations:

```yaml
      - name: Deploy and migrate
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/my-django-app
            git pull origin main

            source venv/bin/activate
            pip install -r requirements.txt

            # Collect static files
            python manage.py collectstatic --noinput

            # Run migrations
            python manage.py migrate --noinput

            # Restart
            sudo systemctl restart django-app
```

## Sudoers Configuration

The deploy user needs passwordless sudo for `systemctl restart`. Add this on your server:

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

Add:

```
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl stop myapp, /usr/bin/systemctl start myapp, /usr/bin/systemctl restart myapp, /usr/bin/systemctl is-active myapp
```

This gives the `deploy` user permission to only manage the specific service — not full sudo access.

## Health Check After Deploy

Add a health check to confirm the deployment actually works:

```yaml
      - name: Health check
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            sleep 5
            curl -f http://localhost:8080/health || exit 1
            echo "Health check passed"
```

If the health check fails, the workflow fails — alerting you that something went wrong.

## Deployment with Environment Variables

Pass secrets as environment variables to the service:

```yaml
      - name: Update env file and restart
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            # Update environment file
            cat > /home/deploy/myapp/.env <<'EOF'
            DATABASE_URL=${{ secrets.DATABASE_URL }}
            API_KEY=${{ secrets.API_KEY }}
            EOF
            chmod 600 /home/deploy/myapp/.env

            sudo systemctl restart myapp
```

## Troubleshooting

**Tests pass but deploy fails**
- Check SSH connectivity: can you SSH manually?
- Check the deploy user has the right permissions on the server

**Service fails after restart**
- Check logs: `sudo journalctl -u myapp -n 50`
- The binary might not be executable: `chmod +x`
- Missing environment variables in the service file

**Go binary: "exec format error"**
- Wrong `GOARCH` — check your server architecture: `uname -m`
- `x86_64` → `GOARCH=amd64`, `aarch64` → `GOARCH=arm64`

**Python: "No module named..."**
- systemd isn't using the venv — check `ExecStart` uses the full venv path
- Dependencies not installed: re-run `pip install -r requirements.txt` inside the venv

---

**Next:** [Backend CI/CD with Docker](backend-docker.md)
