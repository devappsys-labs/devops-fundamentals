# Deploying Client-Side Rendered Apps

Client-side rendered (CSR) apps run entirely in the browser. The server just serves static files — HTML, CSS, JavaScript — and the JavaScript takes over to render the UI.

Examples: plain HTML/CSS/JS sites, React (CRA/Vite), Angular, Vue.

## How CSR Works

```mermaid
flowchart LR
    A[Browser requests\nmyapp.com] --> B[Nginx serves\nindex.html + bundle.js] --> C[Browser executes\nJavaScript] --> D[JavaScript\nrenders the UI]
```

The server does no processing. It's just a file server.

## Deploying a Plain HTML/CSS/JS Site

### Prepare the files

```bash
# On your local machine
scp -r ./my-website/* user@your-server:/tmp/mysite/
```

### Set up on the server

```bash
# Create the web root
sudo mkdir -p /var/www/mysite

# Copy files
sudo cp -r /tmp/mysite/* /var/www/mysite/

# Set permissions
sudo chown -R www-data:www-data /var/www/mysite
sudo chmod -R 755 /var/www/mysite
```

### Configure Nginx

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

    # Cache static assets
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2|ttf)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Enable gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 256;
}
NGINX

sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## Deploying a React App (Vite / CRA)

### Build on your local machine

```bash
# Using Vite
npm run build    # Output: dist/

# Using Create React App
npm run build    # Output: build/
```

### Transfer to server

```bash
# Vite
scp -r ./dist/* user@your-server:/tmp/myapp/

# CRA
scp -r ./build/* user@your-server:/tmp/myapp/
```

### Set up on the server

```bash
sudo mkdir -p /var/www/myapp
sudo cp -r /tmp/myapp/* /var/www/myapp/
sudo chown -R www-data:www-data /var/www/myapp
```

### Configure Nginx for SPA

React uses client-side routing. The key difference from a plain HTML site is the `try_files` fallback to `index.html`:

```bash
sudo tee /etc/nginx/sites-available/myapp > /dev/null <<'NGINX'
server {
    listen 80;
    server_name myapp.com;

    root /var/www/myapp;
    index index.html;

    # SPA fallback — all routes serve index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets (Vite/CRA adds hashes to filenames)
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Don't cache index.html itself (it references hashed assets)
    location = /index.html {
        add_header Cache-Control "no-cache";
    }

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 256;
}
NGINX

sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### Why `try_files $uri $uri/ /index.html`?

Without this, refreshing on `/dashboard` would return a 404 because there's no `dashboard` file on disk. The fallback serves `index.html`, and React Router reads the URL and renders the correct component.

## Deploying an Angular App

### Build

```bash
ng build --configuration production
# Output: dist/<project-name>/browser/
```

### Transfer and set up

```bash
scp -r ./dist/my-angular-app/browser/* user@your-server:/tmp/myapp/

# On the server
sudo mkdir -p /var/www/myapp
sudo cp -r /tmp/myapp/* /var/www/myapp/
sudo chown -R www-data:www-data /var/www/myapp
```

### Nginx config

Same as React — Angular also uses client-side routing:

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

## Updating the Deployment

When you push a new version:

```bash
# Build locally
npm run build

# Transfer new build
scp -r ./dist/* user@your-server:/tmp/myapp-new/

# On the server — swap the files
sudo rm -rf /var/www/myapp/*
sudo cp -r /tmp/myapp-new/* /var/www/myapp/
sudo chown -R www-data:www-data /var/www/myapp

# No need to reload Nginx — it serves files from disk on each request
```

No Nginx restart needed. The new files are served immediately.

## Troubleshooting

**Blank page after deploy**
- Check browser console for errors
- Verify all files were copied: `ls -la /var/www/myapp/`
- Check if paths in `index.html` are correct (absolute vs relative)

**404 on page refresh (SPA)**
- Ensure `try_files` falls back to `/index.html`, not `=404`

**Old version still showing**
- Hard refresh: `Ctrl + Shift + R`
- Clear browser cache
- Check if `index.html` is being cached (it shouldn't be)

**Permission denied in Nginx logs**
- `sudo namei -l /var/www/myapp/index.html` — check every directory in the path is readable by `www-data`

---

**Back to:** [Table of Contents](../../README.md)
