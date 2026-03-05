# SSL/TLS with Certbot

HTTPS encrypts traffic between the browser and your server. In production, this is not optional — browsers mark HTTP sites as "Not Secure" and many features (service workers, geolocation, etc.) require HTTPS.

## How It Works

```
Browser ──HTTPS──> Nginx (terminates SSL) ──HTTP──> Backend (localhost:3000)
```

Nginx handles the encryption/decryption. Your backend application doesn't need to know anything about SSL.

## Prerequisites

- A domain name pointing to your server's IP address (A record in DNS)
- Nginx installed and serving your site on port 80
- Ports 80 and 443 open in your firewall

## Installing Certbot

Certbot is a free tool from Let's Encrypt that automatically obtains and renews SSL certificates.

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

## Getting a Certificate

Make sure your site is already configured and working on port 80:

```bash
# Certbot will automatically modify your Nginx config
sudo certbot --nginx -d mysite.com -d www.mysite.com
```

Certbot will:
1. Verify you own the domain (via an HTTP challenge on port 80)
2. Obtain a certificate from Let's Encrypt
3. Modify your Nginx config to use HTTPS
4. Set up an HTTP → HTTPS redirect

### What Certbot adds to your config

Before (your original config):
```nginx
server {
    listen 80;
    server_name mysite.com www.mysite.com;
    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

After Certbot runs:
```nginx
server {
    server_name mysite.com www.mysite.com;
    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/mysite.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mysite.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

# HTTP → HTTPS redirect (added by Certbot)
server {
    listen 80;
    server_name mysite.com www.mysite.com;

    if ($host = www.mysite.com) {
        return 301 https://$host$request_uri;
    }
    if ($host = mysite.com) {
        return 301 https://$host$request_uri;
    }

    return 404;
}
```

## Certificate Auto-Renewal

Let's Encrypt certificates expire every 90 days. Certbot sets up a systemd timer to auto-renew:

```bash
# Check the renewal timer
sudo systemctl status certbot.timer

# Test renewal (dry run)
sudo certbot renew --dry-run
```

If the dry run succeeds, your certificates will auto-renew without any action from you.

## Manual SSL Config (Without Certbot)

If you prefer to understand what's happening or need to configure SSL manually:

```nginx
server {
    listen 443 ssl;
    server_name mysite.com;

    ssl_certificate /etc/letsencrypt/live/mysite.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mysite.com/privkey.pem;

    # SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;

    # HSTS — tell browsers to always use HTTPS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name mysite.com;
    return 301 https://$server_name$request_uri;
}
```

## SSL with Reverse Proxy

Combining SSL termination with reverse proxy:

```nginx
server {
    listen 443 ssl;
    server_name myapp.com;

    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name myapp.com;
    return 301 https://$server_name$request_uri;
}
```

## Verifying SSL

```bash
# Check certificate details
sudo certbot certificates

# Test SSL from command line
curl -I https://mysite.com

# Check certificate expiry
echo | openssl s_client -connect mysite.com:443 2>/dev/null | openssl x509 -noout -dates
```

## Troubleshooting

**"Could not automatically find a matching server block"**
- Ensure `server_name` in your Nginx config matches the domain you're requesting a cert for

**"Challenge failed"**
- Ensure port 80 is open and accessible from the internet
- Ensure your domain's DNS A record points to your server's IP
- Check with: `curl http://yourdomain.com` from another machine

**Certificate not renewing**
- Check the timer: `sudo systemctl status certbot.timer`
- Check Certbot logs: `sudo cat /var/log/letsencrypt/letsencrypt.log`
- Manual renew: `sudo certbot renew`

---

**Back to:** [Table of Contents](../README.md)
