# Serving Static Files with Nginx

This is the most basic and common use of Nginx — serving HTML, CSS, JavaScript, and image files directly to the browser.

## Basic Static File Server

```nginx
server {
    listen 80;
    server_name mysite.com;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### How `try_files` works

When a request comes in for `/about`:

1. Try `/var/www/mysite/about` (exact file)
2. Try `/var/www/mysite/about/` (directory → look for index.html inside)
3. If neither exists → return 404

## Setting Up a Static Site Step by Step

```bash
# Create directory
sudo mkdir -p /var/www/mysite

# Create a simple HTML file
sudo tee /var/www/mysite/index.html > /dev/null <<'HTML'
<!DOCTYPE html>
<html>
<head>
    <title>My Site</title>
    <link rel="stylesheet" href="/css/style.css">
</head>
<body>
    <h1>Hello World</h1>
    <script src="/js/app.js"></script>
</body>
</html>
HTML

# Create CSS and JS directories
sudo mkdir -p /var/www/mysite/{css,js}
echo "body { font-family: sans-serif; }" | sudo tee /var/www/mysite/css/style.css
echo "console.log('loaded');" | sudo tee /var/www/mysite/js/app.js

# Set permissions
sudo chown -R www-data:www-data /var/www/mysite
```

Create the Nginx config:

```bash
sudo tee /etc/nginx/sites-available/mysite > /dev/null <<'NGINX'
server {
    listen 80;
    server_name mysite.com;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
NGINX

sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## Caching Static Assets

Browsers can cache static files to reduce load times. Use the `expires` directive:

```nginx
server {
    listen 80;
    server_name mysite.com;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # Cache CSS, JS, and images for 30 days
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2|ttf)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

## Gzip Compression

Compress text-based files before sending them to the browser:

```nginx
server {
    listen 80;
    server_name mysite.com;

    root /var/www/mysite;
    index index.html;

    # Enable gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 256;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

## Custom Error Pages

```nginx
server {
    listen 80;
    server_name mysite.com;

    root /var/www/mysite;
    index index.html;

    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Create the error pages:

```bash
echo "<h1>404 - Page Not Found</h1>" | sudo tee /var/www/mysite/404.html
echo "<h1>Server Error</h1>" | sudo tee /var/www/mysite/50x.html
```

## Serving a React / Angular Build

When you build a React or Angular app, you get a `dist/` or `build/` folder with static files. The deployment is identical — just copy the build output to the web root.

```bash
# Example: React
cd /path/to/your/react-app
npm run build

# Copy build output to web root
sudo cp -r build/* /var/www/mysite/
sudo chown -R www-data:www-data /var/www/mysite
```

**Important for SPAs (Single Page Applications):** React and Angular use client-side routing. If a user navigates to `/dashboard` and refreshes, Nginx will look for a file called `dashboard` — which doesn't exist. You need to fall back to `index.html`:

```nginx
server {
    listen 80;
    server_name myapp.com;

    root /var/www/myapp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2|ttf)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

The key difference: `try_files $uri $uri/ /index.html;` — instead of returning 404, it serves `index.html` and lets the JavaScript router handle the URL.

## Hosting Multiple Sites on One Server

Each site gets its own server block:

```bash
# Site A
sudo tee /etc/nginx/sites-available/site-a > /dev/null <<'NGINX'
server {
    listen 80;
    server_name site-a.com;
    root /var/www/site-a;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
NGINX

# Site B
sudo tee /etc/nginx/sites-available/site-b > /dev/null <<'NGINX'
server {
    listen 80;
    server_name site-b.com;
    root /var/www/site-b;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
NGINX

# Enable both
sudo ln -s /etc/nginx/sites-available/site-a /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/site-b /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Nginx uses the `server_name` directive to decide which block handles each request based on the `Host` header.

## Debugging

```bash
# Check what Nginx is actually serving
curl -I http://localhost  # Show response headers

# Check access logs in real time
sudo tail -f /var/log/nginx/access.log

# Check error logs
sudo tail -f /var/log/nginx/error.log

# Verify file permissions
ls -la /var/www/mysite/
namei -l /var/www/mysite/index.html  # Check entire path permissions
```

---

**Next:** [Reverse Proxy](reverse-proxy.md)
