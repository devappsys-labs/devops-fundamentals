# Monitoring & Logging

You can't fix what you can't see. Monitoring and logging are how you know your servers and apps are healthy.

## Server Monitoring (Command Line)

### CPU and Memory

```bash
# Real-time system overview
htop                           # Interactive (install: apt install htop)
top                            # Built-in

# Memory usage
free -h
# Output:
#               total   used   free   shared  buff/cache  available
# Mem:           4Gi    1.2Gi  500Mi  100Mi   2.3Gi       2.5Gi
# Swap:          2Gi    0Bi    2Gi

# CPU info
nproc                          # Number of CPU cores
lscpu                          # Detailed CPU info

# Load average
uptime
# 10:30:00 up 45 days, load average: 0.15, 0.10, 0.05
# 1min, 5min, 15min averages
# Rule: load > number of CPU cores = overloaded
```

### Disk

```bash
# Disk usage by filesystem
df -h
# Filesystem      Size  Used  Avail  Use%  Mounted on
# /dev/sda1       50G   15G   33G    31%   /

# Directory sizes
du -sh /var/log
du -sh /var/lib/docker
du -h --max-depth=1 /

# Find large files
find / -type f -size +100M 2>/dev/null | head -20

# inode usage (running out of inodes = can't create files even with free space)
df -i
```

### Network

```bash
# Active connections
ss -tlnp                       # Listening TCP ports
ss -tunp                       # All active connections

# Bandwidth usage
sudo apt install -y iftop
sudo iftop                     # Real-time bandwidth per connection

# Network interface stats
ip -s link show
```

### Processes

```bash
# Top memory consumers
ps aux --sort=-%mem | head -10

# Top CPU consumers
ps aux --sort=-%cpu | head -10

# Count processes
ps aux | wc -l

# Find a specific process
pgrep -la nginx
```

## Application Logging

### Where logs live

| Service | Log location |
|---------|-------------|
| Nginx access | `/var/log/nginx/access.log` |
| Nginx errors | `/var/log/nginx/error.log` |
| systemd services | `journalctl -u servicename` |
| Docker containers | `docker logs containername` |
| System auth | `/var/log/auth.log` |
| System messages | `/var/log/syslog` |
| Kernel | `/var/log/kern.log` |

### Reading logs

```bash
# Nginx — follow access log
sudo tail -f /var/log/nginx/access.log

# Nginx — search for errors
sudo grep "500" /var/log/nginx/access.log
sudo grep "error" /var/log/nginx/error.log

# Systemd service logs
sudo journalctl -u myapp -f
sudo journalctl -u myapp --since "1 hour ago"
sudo journalctl -u myapp -p err       # Errors only

# Docker container logs
docker logs -f myapp --tail 100

# System authentication logs (SSH logins, sudo, etc.)
sudo tail -f /var/log/auth.log
```

### Analyzing Nginx access logs

```bash
# Top 10 IP addresses
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Requests by status code
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Top requested URLs
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Requests per minute (recent)
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f1-3 | uniq -c | tail -10

# All 5xx errors
grep '" 5[0-9][0-9] ' /var/log/nginx/access.log
```

## Log Rotation

Logs grow forever if not managed. `logrotate` handles this automatically.

### Default config

```bash
cat /etc/logrotate.d/nginx
```

```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

This keeps 14 days of compressed logs and rotates daily.

### Custom log rotation for your app

```bash
sudo nano /etc/logrotate.d/myapp
```

```
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    create 0644 deploy deploy
    postrotate
        systemctl restart myapp
    endscript
}
```

### Docker log size limits

Docker logs can grow huge. Limit them in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
sudo systemctl restart docker
```

Or per container in Compose:

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

## Health Check Scripts

### Simple bash health check

```bash
#!/bin/bash
# /home/deploy/scripts/healthcheck.sh

check_service() {
    if systemctl is-active --quiet $1; then
        echo "[OK] $1 is running"
    else
        echo "[FAIL] $1 is NOT running"
    fi
}

check_url() {
    if curl -sf -o /dev/null "$1"; then
        echo "[OK] $1 is responding"
    else
        echo "[FAIL] $1 is NOT responding"
    fi
}

echo "=== Server Health Check ==="
echo "Date: $(date)"
echo ""

# Services
check_service nginx
check_service myapp

# Endpoints
check_url http://localhost:80
check_url http://localhost:8080/health

# Disk
DISK_USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
if [ "$DISK_USAGE" -gt 90 ]; then
    echo "[WARN] Disk usage at ${DISK_USAGE}%"
else
    echo "[OK] Disk usage at ${DISK_USAGE}%"
fi

# Memory
MEM_AVAILABLE=$(free -m | awk 'NR==2 {printf "%.0f", $7/$2*100}')
if [ "$MEM_AVAILABLE" -lt 10 ]; then
    echo "[WARN] Memory available: ${MEM_AVAILABLE}%"
else
    echo "[OK] Memory available: ${MEM_AVAILABLE}%"
fi
```

### Run with a systemd timer

```ini
# /etc/systemd/system/healthcheck.service
[Unit]
Description=Health Check

[Service]
Type=oneshot
ExecStart=/home/deploy/scripts/healthcheck.sh
```

```ini
# /etc/systemd/system/healthcheck.timer
[Unit]
Description=Run health check every 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
chmod +x /home/deploy/scripts/healthcheck.sh
sudo systemctl enable healthcheck.timer
sudo systemctl start healthcheck.timer
```

## Disk Space Alerts

```bash
#!/bin/bash
# /home/deploy/scripts/disk-alert.sh

THRESHOLD=90
USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$USAGE" -gt "$THRESHOLD" ]; then
    echo "ALERT: Disk usage is ${USAGE}% on $(hostname)" | \
    mail -s "Disk Alert: $(hostname)" your-email@example.com
fi
```

## What to Monitor in Production

| Metric | Why | Alert when |
|--------|-----|-----------|
| **CPU usage** | App performance | Sustained > 80% |
| **Memory usage** | App stability | Available < 10% |
| **Disk usage** | Logs, data growth | > 85% |
| **Disk I/O** | Database bottleneck | Sustained high wait |
| **Network** | Traffic spikes, DDoS | Unusual patterns |
| **HTTP 5xx rate** | App errors | Any increase |
| **Response time** | User experience | > 2 seconds |
| **Service status** | Uptime | Service not running |
| **SSL expiry** | HTTPS | < 14 days before expiry |
| **Failed logins** | Security | Unusual spike |

## When You Need More

For single servers, command-line tools and scripts are enough. When you scale:

| Tool | Purpose |
|------|---------|
| **Prometheus + Grafana** | Metrics collection and dashboards |
| **Loki** | Log aggregation (pairs with Grafana) |
| **Uptime Kuma** | Self-hosted uptime monitoring |
| **Netdata** | Real-time server monitoring (easy to set up) |
| **Datadog / New Relic** | Full observability platforms (paid) |

### Quick win: Uptime Kuma

Self-hosted monitoring with a nice UI:

```bash
docker run -d \
  --name uptime-kuma \
  --restart unless-stopped \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  louislam/uptime-kuma
```

Visit `http://your-server:3001`, add your endpoints, get alerts via email/Slack/Telegram.

---

**Back to:** [Table of Contents](../README.md)
