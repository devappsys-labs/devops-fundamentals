# Server Hardening

The first things you should do on a fresh server before deploying anything.

## Initial Setup Checklist

1. Create a deploy user (don't use root)
2. Set up SSH key authentication
3. Disable root login and password auth
4. Configure the firewall
5. Install fail2ban
6. Enable automatic security updates
7. Set up basic monitoring

## Create a Deploy User

Never run applications as root.

```bash
# As root on a fresh server
adduser deploy
usermod -aG sudo deploy

# Switch to the new user
su - deploy

# Verify sudo works
sudo whoami
# root
```

## SSH Hardening

### Copy your SSH key to the server

```bash
# From your local machine
ssh-copy-id deploy@your-server-ip
```

### Harden sshd_config

```bash
sudo nano /etc/ssh/sshd_config
```

Change these settings:

```
# Change default port (reduces automated attacks)
Port 2222

# Disable root login
PermitRootLogin no

# Disable password authentication (key-only)
PasswordAuthentication no
PubkeyAuthentication yes

# Limit authentication attempts
MaxAuthTries 3

# Disconnect idle sessions
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable empty passwords
PermitEmptyPasswords no

# Disable X11 forwarding (not needed for servers)
X11Forwarding no
```

```bash
# Validate the config
sudo sshd -t

# Restart SSH
sudo systemctl restart sshd
```

**Test with a new terminal before closing your current session.** If you mess up, you'll be locked out.

```bash
# From your local machine (new terminal)
ssh -p 2222 deploy@your-server-ip
```

## Firewall (UFW)

```bash
# Allow SSH on your custom port
sudo ufw allow 2222/tcp

# Allow web traffic
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Set defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Enable
sudo ufw enable

# Verify
sudo ufw status verbose
```

**Order matters:** Allow SSH BEFORE enabling the firewall, or you'll lock yourself out.

### Restrict database access

```bash
# PostgreSQL — only from specific IP (your app server)
sudo ufw allow from 10.0.0.5 to any port 5432

# Block from everywhere else (default deny handles this)
```

## fail2ban

Monitors log files and bans IPs that show malicious behavior (brute force SSH, etc.).

```bash
sudo apt install -y fail2ban
```

### Configure

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Key settings:

```ini
[DEFAULT]
bantime = 3600          # Ban for 1 hour
findtime = 600          # Look at the last 10 minutes
maxretry = 3            # 3 failures = ban

[sshd]
enabled = true
port = 2222             # Your SSH port
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check status
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Unban an IP
sudo fail2ban-client set sshd unbanip 203.0.113.100
```

### Nginx jail (block bad HTTP requests)

```ini
[nginx-http-auth]
enabled = true

[nginx-badbots]
enabled = true

[nginx-botsearch]
enabled = true
```

## Automatic Security Updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

This auto-installs security patches. Application updates are NOT included — only security fixes.

Verify:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
# APT::Periodic::Update-Package-Lists "1";
# APT::Periodic::Unattended-Upgrade "1";
```

## File Permissions Hardening

```bash
# Protect SSH directory
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Protect environment files
chmod 600 /home/deploy/myapp/.env

# Web files — readable by Nginx, not writable by the world
sudo chown -R www-data:www-data /var/www/myapp
sudo chmod -R 755 /var/www/myapp
sudo chmod -R 644 /var/www/myapp/*.html

# Application binaries — executable by owner only
chmod 750 /usr/local/bin/myapp
```

## Disable Unused Services

```bash
# List all enabled services
systemctl list-unit-files --state=enabled

# Disable services you don't need
sudo systemctl disable cups           # Printing
sudo systemctl disable avahi-daemon   # mDNS (Bonjour)
sudo systemctl disable bluetooth      # Bluetooth
```

## Restrict sudo

Don't give full sudo access. Limit to specific commands:

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

```
# Only allow specific systemctl commands
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl stop myapp, /usr/bin/systemctl start myapp, /usr/bin/systemctl status myapp, /usr/bin/systemctl reload nginx
```

## Nginx Security Headers

Add to your server blocks:

```nginx
server {
    # ...

    # Prevent clickjacking
    add_header X-Frame-Options "SAMEORIGIN" always;

    # Prevent MIME type sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # XSS protection
    add_header X-XSS-Protection "1; mode=block" always;

    # Referrer policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Content Security Policy (adjust to your needs)
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';" always;

    # Hide Nginx version
    server_tokens off;
}
```

### Hide Nginx version globally

In `/etc/nginx/nginx.conf`:

```nginx
http {
    server_tokens off;
    # ...
}
```

## Rate Limiting in Nginx

Prevent brute force and DDoS:

```nginx
# In http context (nginx.conf)
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
}

# In server/location context
server {
    # API: 10 requests/sec with burst of 20
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://localhost:8080;
    }

    # Login: 1 request/sec (strict)
    location /api/login {
        limit_req zone=login burst=3;
        proxy_pass http://localhost:8080;
    }
}
```

## Server Audit Checklist

Run periodically:

```bash
# Check for users with empty passwords
sudo awk -F: '($2 == "") {print $1}' /etc/shadow

# Check for users with UID 0 (root privileges)
awk -F: '($3 == "0") {print}' /etc/passwd

# Check listening ports (are any unexpected?)
sudo ss -tlnp

# Check failed login attempts
sudo lastb | head -20

# Check successful logins
last | head -20

# Check disk usage
df -h

# Check running processes
ps aux --sort=-%mem | head -20

# Check for pending security updates
sudo apt list --upgradable
```

---

**Back to:** [Table of Contents](../README.md)
