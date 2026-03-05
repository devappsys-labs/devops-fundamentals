# Deploying Backend Services

This covers deploying backend services directly on a server (no Docker) with Nginx as a reverse proxy. We'll cover both **compiled** (Go, Java) and **interpreted** (Python, Node.js) languages.

## The Pattern

Regardless of language, the deployment pattern is the same:

```mermaid
flowchart LR
    A[Internet] --> B["Nginx\n(port 80/443)"] -->|proxy_pass| C["Your app\n(localhost:8080)"]
```

1. Get your app running on a local port
2. Create a systemd service to keep it alive
3. Configure Nginx to reverse proxy to it

## Compiled Languages

Compiled languages produce a binary that runs directly — no runtime needed on the server (for Go) or just a runtime (for Java).

### Go

#### Build on your local machine

```bash
# Cross-compile for Linux (if building from macOS/Windows)
GOOS=linux GOARCH=amd64 go build -o myapp ./cmd/main.go
```

#### Transfer to server

```bash
scp myapp user@your-server:/home/user/myapp
```

#### Set up on the server

```bash
# Make it executable
chmod +x /home/user/myapp

# Test it runs
/home/user/myapp
# Should start listening on a port (e.g., :8080)
# Ctrl+C to stop
```

#### Systemd service

```bash
sudo tee /etc/systemd/system/myapp.service > /dev/null <<'SERVICE'
[Unit]
Description=My Go Application
After=network.target

[Service]
Type=simple
User=www-data
ExecStart=/home/user/myapp
Restart=on-failure
RestartSec=5
Environment=PORT=8080
Environment=GIN_MODE=release

[Install]
WantedBy=multi-user.target
SERVICE

sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
sudo systemctl status myapp
```

#### Nginx config

```bash
sudo tee /etc/nginx/sites-available/myapp > /dev/null <<'NGINX'
server {
    listen 80;
    server_name api.myapp.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
NGINX

sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### Java (Spring Boot)

#### Build

```bash
# On your local machine
./mvnw clean package -DskipTests
# or with Gradle
./gradlew bootJar

# Output: target/myapp-0.0.1-SNAPSHOT.jar
```

#### Transfer

```bash
scp target/myapp-0.0.1-SNAPSHOT.jar user@your-server:/home/user/myapp.jar
```

#### Install Java on the server

```bash
sudo apt update
sudo apt install -y openjdk-21-jre-headless

java -version
```

#### Systemd service

```bash
sudo tee /etc/systemd/system/myapp.service > /dev/null <<'SERVICE'
[Unit]
Description=My Spring Boot Application
After=network.target

[Service]
Type=simple
User=www-data
ExecStart=/usr/bin/java -jar /home/user/myapp.jar
Restart=on-failure
RestartSec=5
Environment=SPRING_PROFILES_ACTIVE=production
Environment=SERVER_PORT=8080

[Install]
WantedBy=multi-user.target
SERVICE

sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
```

Nginx config is the same as the Go example — just `proxy_pass` to `localhost:8080`.

## Interpreted Languages

Interpreted languages need their runtime installed on the server.

### Node.js (Express, Fastify, etc.)

#### Install Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
```

#### Transfer code

```bash
scp -r ./my-node-app user@your-server:/home/user/my-node-app
```

#### Install dependencies on server

```bash
cd /home/user/my-node-app
npm install --production
```

#### Test

```bash
node app.js
# Should listen on port 3000
curl http://localhost:3000
```

#### Systemd service

```bash
sudo tee /etc/systemd/system/nodeapp.service > /dev/null <<'SERVICE'
[Unit]
Description=Node.js Application
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/home/user/my-node-app
ExecStart=/usr/bin/node app.js
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
SERVICE

sudo systemctl daemon-reload
sudo systemctl enable nodeapp
sudo systemctl start nodeapp
```

#### Nginx config

```nginx
server {
    listen 80;
    server_name api.myapp.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Python (Flask / FastAPI / Django)

#### Install Python and pip

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
```

#### Transfer and set up

```bash
scp -r ./my-python-app user@your-server:/home/user/my-python-app

# On the server
cd /home/user/my-python-app

# Create virtual environment
python3 -m venv venv

# Activate and install dependencies
source venv/bin/activate
pip install -r requirements.txt
```

#### Running Python web apps

Python web apps need a WSGI/ASGI server in production. **Never use the built-in development server in production.**

**Flask with Gunicorn:**
```bash
pip install gunicorn

# Test
gunicorn --bind 0.0.0.0:8000 app:app
```

**FastAPI with Uvicorn:**
```bash
pip install uvicorn

# Test
uvicorn main:app --host 0.0.0.0 --port 8000
```

**Django with Gunicorn:**
```bash
pip install gunicorn

# Test
gunicorn --bind 0.0.0.0:8000 myproject.wsgi:application
```

#### Systemd service (Flask + Gunicorn example)

```bash
sudo tee /etc/systemd/system/flask-app.service > /dev/null <<'SERVICE'
[Unit]
Description=Flask Application (Gunicorn)
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/home/user/my-python-app
ExecStart=/home/user/my-python-app/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 app:app
Restart=on-failure
RestartSec=5
Environment=FLASK_ENV=production

[Install]
WantedBy=multi-user.target
SERVICE

sudo systemctl daemon-reload
sudo systemctl enable flask-app
sudo systemctl start flask-app
```

#### Nginx config

```nginx
server {
    listen 80;
    server_name api.myapp.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Serve Django/Flask static files directly via Nginx
    location /static/ {
        alias /home/user/my-python-app/static/;
        expires 30d;
    }
}
```

## Environment Variables

Never hardcode secrets. Use environment variables:

### In systemd

```ini
Environment=DATABASE_URL=postgres://user:pass@localhost/mydb
Environment=SECRET_KEY=your-secret-key
```

### Using an environment file

```bash
# Create /home/user/myapp/.env
DATABASE_URL=postgres://user:pass@localhost/mydb
SECRET_KEY=your-secret-key
```

```ini
[Service]
EnvironmentFile=/home/user/myapp/.env
```

**Security:** Make sure `.env` files are only readable by the service user:
```bash
chmod 600 /home/user/myapp/.env
```

## Compiled vs Interpreted — Key Differences

| Aspect | Compiled (Go, Java) | Interpreted (Python, Node.js) |
|--------|---------------------|-------------------------------|
| Runtime on server | None (Go) / JRE (Java) | Node.js / Python |
| Dependencies | Bundled in binary (Go) / JAR (Java) | `node_modules` / `venv` on server |
| Deploy artifact | Single binary / JAR file | Entire source code + dependencies |
| Memory usage | Generally lower (Go) | Higher (especially Java) |
| Startup time | Fast (Go) / Slow (Java) | Medium |
| Update process | Replace binary, restart | Pull code, install deps, restart |

## Troubleshooting

**502 Bad Gateway**
- Backend not running: `sudo systemctl status myapp`
- Wrong port: `curl http://localhost:<port>` from the server

**Permission denied**
- systemd user can't access files: check ownership and permissions
- Use `namei -l /path/to/app` to trace permission issues

**App crashes on start**
- Check logs: `sudo journalctl -u myapp -n 50`
- Run the command manually to see the error

**Python: ModuleNotFoundError**
- systemd isn't using your venv — use full path to venv's Python/Gunicorn in `ExecStart`

---

**Back to:** [Table of Contents](../../README.md)
