# Systemd — Service Management

Systemd is the init system on modern Linux. It manages services (daemons), controls startup order, and provides logging via journald.

## Why You Need to Know Systemd

Every non-dockerised backend service needs systemd to:
- Start on boot
- Restart on crash
- Run as a specific user
- Manage environment variables
- Provide log access

## Essential Commands

```bash
# Start a service
sudo systemctl start nginx

# Stop a service
sudo systemctl stop nginx

# Restart (full stop + start)
sudo systemctl restart nginx

# Reload (graceful — re-read config without dropping connections)
sudo systemctl reload nginx

# Check status
sudo systemctl status nginx

# Enable (start on boot)
sudo systemctl enable nginx

# Disable (don't start on boot)
sudo systemctl disable nginx

# Check if a service is active
systemctl is-active nginx

# Check if a service is enabled (starts on boot)
systemctl is-enabled nginx

# List all running services
systemctl list-units --type=service --state=running

# List all services (including stopped)
systemctl list-units --type=service --all

# Reload systemd after changing a service file
sudo systemctl daemon-reload
```

## Writing a Service File

Service files live in `/etc/systemd/system/`. Each file defines how to run a service.

### Basic structure

```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/home/deploy/myapp
ExecStart=/usr/bin/node app.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Creating a service

```bash
sudo nano /etc/systemd/system/myapp.service
# Paste the service file content

sudo systemctl daemon-reload       # Tell systemd about the new file
sudo systemctl enable myapp        # Start on boot
sudo systemctl start myapp         # Start now
sudo systemctl status myapp        # Verify
```

## [Unit] Section

Controls when and how the service starts.

```ini
[Unit]
Description=My Node.js API            # Human-readable name
Documentation=https://github.com/...  # Optional docs URL
After=network.target                  # Start after network is up
After=network.target postgresql.service  # Start after network AND postgres
Requires=postgresql.service           # Fail if postgres isn't running
Wants=redis.service                   # Start redis if available, but don't fail
```

| Directive | Meaning |
|-----------|---------|
| `After` | Start this service after the listed ones |
| `Before` | Start this service before the listed ones |
| `Requires` | This service NEEDS the listed ones (fail if they fail) |
| `Wants` | This service WANTS the listed ones (soft dependency) |

## [Service] Section

### Type

```ini
Type=simple        # Default. The process runs in foreground.
Type=forking       # Process forks into background (legacy daemons)
Type=oneshot       # Runs once and exits (scripts, migrations)
Type=notify        # Process signals systemd when ready
```

Use `simple` for Node.js, Python, Go. Use `oneshot` for scripts and migrations.

### ExecStart

```ini
# The main command to run
ExecStart=/usr/bin/node /home/deploy/myapp/app.js

# Go binary
ExecStart=/usr/local/bin/myapp

# Python with venv
ExecStart=/home/deploy/myapp/venv/bin/gunicorn --bind 127.0.0.1:8000 app:app

# Java
ExecStart=/usr/bin/java -jar /opt/myapp/app.jar
```

**Must be an absolute path.** No `cd` — use `WorkingDirectory` instead.

### Pre/Post commands

```ini
ExecStartPre=/usr/bin/node /home/deploy/myapp/migrate.js   # Run before start
ExecStartPost=/usr/bin/curl -s http://localhost:8080/health # Run after start
ExecStopPost=/usr/bin/echo "Service stopped"                # Run after stop
```

### Restart policies

```ini
Restart=no              # Never restart (default)
Restart=on-failure      # Restart if exit code != 0
Restart=on-abnormal     # Restart on signal, timeout, or watchdog
Restart=always          # Always restart (except systemctl stop)

RestartSec=5            # Wait 5 seconds before restarting
StartLimitBurst=5       # Max restarts in a time window
StartLimitIntervalSec=60  # Time window for burst limit
```

For production: `Restart=on-failure` with `RestartSec=5`.

### User and group

```ini
User=deploy            # Run as this user
Group=deploy           # Run as this group

# Never run services as root unless absolutely necessary
```

### Working directory

```ini
WorkingDirectory=/home/deploy/myapp
```

### Environment variables

```ini
# Inline
Environment=NODE_ENV=production
Environment=PORT=3000
Environment=DATABASE_URL=postgres://user:pass@localhost/mydb

# From a file (one VAR=value per line)
EnvironmentFile=/home/deploy/myapp/.env
```

**Important:** The `.env` file format for systemd is:
```
NODE_ENV=production
PORT=3000
DATABASE_URL=postgres://user:pass@localhost/mydb
```

No quotes, no `export`, no spaces around `=`.

### Resource limits

```ini
# Memory limit
MemoryMax=512M

# CPU limit (100% = 1 core)
CPUQuota=100%

# File descriptor limit
LimitNOFILE=65536

# Process limit
LimitNPROC=4096
```

### Standard output/error

```ini
StandardOutput=journal         # Default — logs to journald
StandardError=journal          # Default — errors to journald

# Or log to a file
StandardOutput=append:/var/log/myapp/output.log
StandardError=append:/var/log/myapp/error.log
```

Prefer `journal` — journald handles rotation, searching, and filtering.

## [Install] Section

```ini
[Install]
WantedBy=multi-user.target     # Standard for services (normal boot)
WantedBy=graphical.target      # After GUI loads (desktop systems)
```

Almost always use `multi-user.target`.

## Complete Examples

### Node.js

```ini
[Unit]
Description=Node.js API
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/home/deploy/my-api
ExecStart=/usr/bin/node app.js
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production
EnvironmentFile=/home/deploy/my-api/.env

[Install]
WantedBy=multi-user.target
```

### Go

```ini
[Unit]
Description=Go API Server
After=network.target

[Service]
Type=simple
User=deploy
ExecStart=/usr/local/bin/my-go-api
Restart=on-failure
RestartSec=5
EnvironmentFile=/home/deploy/my-go-api/.env

[Install]
WantedBy=multi-user.target
```

### Python (Gunicorn)

```ini
[Unit]
Description=Flask App (Gunicorn)
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/home/deploy/my-flask-app
ExecStart=/home/deploy/my-flask-app/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 app:app
Restart=on-failure
RestartSec=5
EnvironmentFile=/home/deploy/my-flask-app/.env

[Install]
WantedBy=multi-user.target
```

### Java (Spring Boot)

```ini
[Unit]
Description=Spring Boot Application
After=network.target

[Service]
Type=simple
User=deploy
ExecStart=/usr/bin/java -Xms256m -Xmx512m -jar /opt/myapp/app.jar
Restart=on-failure
RestartSec=10
EnvironmentFile=/opt/myapp/.env

[Install]
WantedBy=multi-user.target
```

### One-shot (Database migration)

```ini
[Unit]
Description=Run database migrations
After=postgresql.service

[Service]
Type=oneshot
User=deploy
WorkingDirectory=/home/deploy/my-api
ExecStart=/usr/bin/node migrate.js
EnvironmentFile=/home/deploy/my-api/.env
```

```bash
sudo systemctl start db-migrate   # Runs once and exits
```

## Journald — Viewing Logs

Systemd services log to journald by default.

```bash
# View logs for a service
sudo journalctl -u myapp

# Follow logs in real time
sudo journalctl -u myapp -f

# Last 50 lines
sudo journalctl -u myapp -n 50

# Logs since a time
sudo journalctl -u myapp --since "1 hour ago"
sudo journalctl -u myapp --since "2024-01-15 10:00" --until "2024-01-15 12:00"
sudo journalctl -u myapp --since today

# Logs from the last boot
sudo journalctl -u myapp -b

# Only errors
sudo journalctl -u myapp -p err

# Output as JSON (for piping to other tools)
sudo journalctl -u myapp -o json

# Disk usage of journal logs
journalctl --disk-usage

# Clean old logs (keep last 2 weeks)
sudo journalctl --vacuum-time=2weeks

# Clean old logs (keep under 500MB)
sudo journalctl --vacuum-size=500M
```

## Systemd Timers (Cron Alternative)

Systemd timers are a modern alternative to cron jobs.

### Create a timer

**The service (what to run):**

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup database

[Service]
Type=oneshot
User=deploy
ExecStart=/home/deploy/scripts/backup.sh
```

**The timer (when to run it):**

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily

[Timer]
OnCalendar=daily              # Every day at midnight
# OnCalendar=*-*-* 02:00:00  # Every day at 2 AM
# OnCalendar=Mon *-*-* 00:00 # Every Monday at midnight
Persistent=true               # Run immediately if missed (server was off)

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable backup.timer
sudo systemctl start backup.timer

# Check timer status
systemctl list-timers
sudo systemctl status backup.timer
```

### Timer syntax

| Expression | Meaning |
|-----------|---------|
| `minutely` | Every minute |
| `hourly` | Every hour |
| `daily` | Every day at midnight |
| `weekly` | Every Monday at midnight |
| `monthly` | First day of month at midnight |
| `*-*-* 02:00:00` | Every day at 2 AM |
| `Mon *-*-* 09:00` | Every Monday at 9 AM |
| `*-*-01 00:00:00` | First of every month |

## Troubleshooting

**Service won't start**
```bash
sudo systemctl status myapp       # Quick status
sudo journalctl -u myapp -n 50    # Recent logs
```

**"Failed to start: Unit not found"**
```bash
sudo systemctl daemon-reload      # Did you forget this?
ls /etc/systemd/system/myapp.service  # Does the file exist?
```

**Service starts but crashes immediately**
- Run the `ExecStart` command manually to see the error
- Check `User` has permission to access files and ports
- Check `WorkingDirectory` exists and is accessible

**"Permission denied"**
- The service `User` can't access the files
- Check: `sudo -u deploy ls /home/deploy/myapp/`
- Fix ownership: `sudo chown -R deploy:deploy /home/deploy/myapp`

**Environment variables not loading**
- Check `.env` file format (no quotes, no `export`)
- Check file permissions: `chmod 600 .env`

---

**Back to:** [Table of Contents](../README.md)
