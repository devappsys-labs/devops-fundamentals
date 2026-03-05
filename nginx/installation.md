# Installing Nginx

## Ubuntu / Debian

### Update package list and install

```bash
sudo apt update
sudo apt install nginx -y
```

### Verify installation

```bash
nginx -v
# Expected output: nginx version: nginx/1.x.x
```

### Check if Nginx is running

```bash
sudo systemctl status nginx
```

You should see `active (running)`. If not:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx   # start on boot
```

### Test in browser

Open your browser and navigate to your server's IP address:

```
http://<your-server-ip>
```

You should see the **"Welcome to nginx!"** default page. This confirms Nginx is installed and serving on port 80.

## Important File Locations

| Path | Purpose |
|------|---------|
| `/etc/nginx/nginx.conf` | Main configuration file |
| `/etc/nginx/sites-available/` | Where you define site configurations |
| `/etc/nginx/sites-enabled/` | Symlinks to active site configs |
| `/var/www/html/` | Default web root directory |
| `/var/log/nginx/access.log` | Request logs |
| `/var/log/nginx/error.log` | Error logs |

## Essential Commands

```bash
# Start Nginx
sudo systemctl start nginx

# Stop Nginx
sudo systemctl stop nginx

# Restart Nginx (full restart — drops connections)
sudo systemctl restart nginx

# Reload Nginx (graceful — applies config changes without dropping connections)
sudo systemctl reload nginx

# Test configuration for syntax errors (ALWAYS do this before reload/restart)
sudo nginx -t
```

**Golden rule:** Always run `sudo nginx -t` before reloading or restarting Nginx. A bad config will take down your server.

## Firewall Configuration

If you have UFW (Uncomplicated Firewall) enabled:

```bash
# Allow HTTP traffic
sudo ufw allow 'Nginx HTTP'

# Allow HTTPS traffic
sudo ufw allow 'Nginx HTTPS'

# Allow both
sudo ufw allow 'Nginx Full'

# Verify
sudo ufw status
```

## Removing the Default Site

The default site config is symlinked at `/etc/nginx/sites-enabled/default`. You'll typically remove this when setting up your own sites:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl reload nginx
```

The original config remains in `/etc/nginx/sites-available/default` if you ever need to reference it.

---

**Next:** [Configuration Basics](configuration-basics.md)
