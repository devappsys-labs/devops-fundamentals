# Domain & DNS Setup

A practical guide to buying a domain, pointing it to your server, and making it work with Nginx and SSL.

## Getting a Domain

### Domain registrars

Popular registrars:
- **Cloudflare Registrar** — At-cost pricing, free DNS, free proxy/CDN
- **Namecheap** — Affordable, good UI
- **GoDaddy** — Popular but watch for upsells
- **Google Domains** (now Squarespace Domains)

Pick one, search for your domain, buy it. Annual renewal (~$10-15/year for .com).

## Setting Up DNS Records

After buying a domain, you need to point it to your server's IP address.

### Step-by-step

**1. Find your server's public IP:**

```bash
curl ifconfig.me
# Example: 203.0.113.50
```

**2. Go to your registrar's DNS settings and add records:**

```
Type    Name    Value              TTL
A       @       203.0.113.50       300
A       www     203.0.113.50       300
```

- `@` means the root domain (myapp.com)
- `www` is the subdomain (www.myapp.com)
- TTL 300 = 5 minutes (good for initial setup; increase later)

**3. Verify DNS is working:**

```bash
# Wait a few minutes, then check
dig myapp.com +short
# Should return: 203.0.113.50

dig www.myapp.com +short
# Should return: 203.0.113.50
```

### Using CNAME for www

Instead of two A records, you can use a CNAME for www:

```
Type    Name    Value              TTL
A       @       203.0.113.50       300
CNAME   www     myapp.com          300
```

The CNAME says "www.myapp.com points to wherever myapp.com points." If you change the server IP, you only update the A record.

### Subdomains

```
Type    Name      Value              TTL
A       api       203.0.113.50       300
A       staging   203.0.113.51       300
A       admin     203.0.113.50       300
```

Each subdomain can point to the same or different servers.

## Nginx Configuration for Your Domain

### Single domain

```nginx
server {
    listen 80;
    server_name myapp.com www.myapp.com;

    root /var/www/myapp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Multiple subdomains

```nginx
# Main site
server {
    listen 80;
    server_name myapp.com www.myapp.com;
    root /var/www/myapp;
    index index.html;
    location / {
        try_files $uri $uri/ /index.html;
    }
}

# API
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

# Admin panel
server {
    listen 80;
    server_name admin.myapp.com;
    root /var/www/admin;
    index index.html;
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Redirect www to non-www (or vice versa)

```nginx
# Redirect www → non-www
server {
    listen 80;
    server_name www.myapp.com;
    return 301 http://myapp.com$request_uri;
}

server {
    listen 80;
    server_name myapp.com;
    root /var/www/myapp;
    # ...
}
```

## Adding SSL After DNS Setup

Once DNS is pointing to your server and Nginx is configured:

```bash
sudo certbot --nginx -d myapp.com -d www.myapp.com
```

For subdomains:

```bash
sudo certbot --nginx -d api.myapp.com
sudo certbot --nginx -d admin.myapp.com
```

Or all at once:

```bash
sudo certbot --nginx -d myapp.com -d www.myapp.com -d api.myapp.com -d admin.myapp.com
```

## Using Cloudflare as DNS + Proxy

Cloudflare provides free DNS hosting with optional proxying (CDN, DDoS protection).

### Setup

1. Add your domain to Cloudflare
2. Cloudflare gives you nameservers (e.g., `ns1.cloudflare.com`)
3. Update your registrar's nameservers to Cloudflare's
4. Manage DNS records in Cloudflare dashboard

### Proxy modes

| Mode | Icon | What happens |
|------|------|-------------|
| DNS only | Grey cloud | Traffic goes directly to your server |
| Proxied | Orange cloud | Traffic goes through Cloudflare (CDN, protection) |

**With Cloudflare proxy enabled:**
- Your server's real IP is hidden
- Cloudflare provides free SSL (between user and Cloudflare)
- You still want SSL between Cloudflare and your server ("Full (Strict)" mode)

## Common DNS Mistakes

**DNS not resolving**
- Did you update the nameservers at your registrar?
- Propagation can take up to 48 hours (usually minutes)
- Check: `dig myapp.com @8.8.8.8 +short` (bypass local cache)

**"Server IP address could not be found"**
- No A record set
- Domain not pointed to correct nameservers

**Certbot fails with "Challenge failed"**
- DNS not pointing to your server yet
- Check: `dig myapp.com +short` — does it return your server's IP?

**Wrong server responding**
- `server_name` in Nginx doesn't match the domain
- Nginx default server is catching the request: remove `/etc/nginx/sites-enabled/default`

---

**Back to:** [Table of Contents](../README.md)
