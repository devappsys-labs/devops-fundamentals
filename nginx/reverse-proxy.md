# Nginx as a Reverse Proxy

A reverse proxy sits between the client and your backend application. The client talks to Nginx, and Nginx forwards the request to your app.

## Why Use a Reverse Proxy?

Your backend application (Node.js, Go, Python, Java) typically listens on a high port like `3000` or `8080`. You don't want users to visit `http://yoursite.com:3000`. Instead:

- Nginx listens on port **80** (HTTP) and **443** (HTTPS)
- Nginx forwards requests to your app on `localhost:3000`
- Your app never touches the public internet directly

**Benefits:**
- Single entry point for HTTP/HTTPS
- SSL termination handled by Nginx
- Serve static files directly without hitting your app
- Load balance across multiple app instances
- Add security headers, rate limiting, etc.

## Basic Reverse Proxy Config

Your app runs on `localhost:3000`. Nginx proxies all traffic to it:

```nginx
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
```

### What each `proxy_set_header` does

| Header | Purpose |
|--------|---------|
| `Host` | Passes the original domain name to your app |
| `X-Real-IP` | Passes the client's real IP (otherwise your app sees `127.0.0.1`) |
| `X-Forwarded-For` | Chain of IPs if there are multiple proxies |
| `X-Forwarded-Proto` | Whether the original request was HTTP or HTTPS |

Without these headers, your backend won't know the real client IP or whether the connection was HTTPS.

## Proxying to Specific Paths

A common pattern: serve the frontend statically and proxy API requests to the backend.

```nginx
server {
    listen 80;
    server_name myapp.com;

    # Frontend — serve static files
    root /var/www/myapp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # API — proxy to backend
    location /api/ {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

With this config:
- `myapp.com/` → serves React/Angular build from `/var/www/myapp/`
- `myapp.com/api/users` → proxied to `localhost:8080/api/users`

## Proxy Pass Trailing Slash Behavior

This is a common source of bugs. Pay close attention:

```nginx
# With trailing slash — strips the matched location prefix
location /api/ {
    proxy_pass http://localhost:8080/;
}
# Request: /api/users → Backend receives: /users

# Without trailing slash — passes the full path
location /api/ {
    proxy_pass http://localhost:8080;
}
# Request: /api/users → Backend receives: /api/users
```

Choose based on how your backend routes are set up.

## WebSocket Support

If your app uses WebSockets (e.g., Socket.io, real-time features):

```nginx
location /ws/ {
    proxy_pass http://localhost:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
}
```

The `Upgrade` and `Connection` headers are required for the WebSocket handshake.

## Timeouts

If your backend takes time to respond (e.g., processing a large file):

```nginx
location /api/ {
    proxy_pass http://localhost:8080;

    proxy_connect_timeout 60s;    # Time to establish connection to backend
    proxy_send_timeout 60s;       # Time to send request to backend
    proxy_read_timeout 60s;       # Time to wait for backend response

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

## Load Balancing

If you're running multiple instances of your app:

```nginx
upstream myapp_backend {
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
}

server {
    listen 80;
    server_name myapp.com;

    location / {
        proxy_pass http://myapp_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Load balancing methods

```nginx
upstream myapp_backend {
    # Round robin (default) — each request goes to next server
    server localhost:3001;
    server localhost:3002;

    # Least connections — send to the server with fewest active connections
    # least_conn;

    # IP hash — same client IP always goes to same server (sticky sessions)
    # ip_hash;
}
```

## Health Checks

Mark a server as down or set backup servers:

```nginx
upstream myapp_backend {
    server localhost:3001;
    server localhost:3002;
    server localhost:3003 backup;         # Only used if others are down
    server localhost:3004 down;           # Temporarily disabled
}
```

## Full Example: Frontend + Backend + API

```nginx
upstream api_backend {
    server localhost:8080;
}

server {
    listen 80;
    server_name myapp.com;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Frontend (SPA)
    root /var/www/myapp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # API reverse proxy
    location /api/ {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Static asset caching
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2|ttf)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

## Debugging Reverse Proxy Issues

```bash
# Check if your backend is actually running
curl http://localhost:3000

# Check Nginx error log for proxy errors
sudo tail -f /var/log/nginx/error.log

# Common errors:
# "connect() failed (111: Connection refused)" → Backend is not running
# "upstream timed out" → Backend is too slow, increase proxy_read_timeout
# "no resolver defined" → Using a hostname instead of IP, add resolver directive
```

---

**Next:** [SSL/TLS with Certbot](ssl-tls.md)
