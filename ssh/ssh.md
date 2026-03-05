# SSH (Secure Shell)

SSH is how you connect to and manage remote servers. Every DevOps task starts with SSH.

## Basic Connection

```bash
ssh user@server-ip
ssh deploy@192.168.1.100
ssh deploy@myserver.com

# Specify port (default is 22)
ssh -p 2222 deploy@192.168.1.100
```

## SSH Key-Based Authentication

Passwords are slow and insecure. Use SSH keys instead.

### How it works

```
Your Machine                          Server
┌──────────┐                    ┌──────────────┐
│ Private  │───authenticates──→│  Public Key   │
│   Key    │                    │ (authorized)  │
│ (secret) │                    │               │
└──────────┘                    └──────────────┘
~/.ssh/id_ed25519               ~/.ssh/authorized_keys
```

The private key stays on your machine. The public key goes on the server. When you connect, SSH proves you have the private key without ever sending it.

### Generate a key pair

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

- `-t ed25519` — Algorithm (ed25519 is modern and secure; use `rsa` for older systems)
- `-C` — Comment to identify the key

You'll be asked:
1. **File location** — Press Enter for default (`~/.ssh/id_ed25519`)
2. **Passphrase** — Optional but recommended. Encrypts the private key on disk.

This creates two files:
- `~/.ssh/id_ed25519` — Private key (NEVER share this)
- `~/.ssh/id_ed25519.pub` — Public key (goes on the server)

### Copy public key to the server

```bash
# Easiest method
ssh-copy-id deploy@192.168.1.100

# Manual method (if ssh-copy-id isn't available)
cat ~/.ssh/id_ed25519.pub | ssh deploy@192.168.1.100 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Test it

```bash
ssh deploy@192.168.1.100
# Should connect without asking for a password
```

### Permissions matter

SSH is strict about file permissions. Wrong permissions = key rejected.

```bash
# On your local machine
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519          # Private key
chmod 644 ~/.ssh/id_ed25519.pub      # Public key

# On the server
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## SSH Config File

Instead of typing `ssh -p 2222 deploy@192.168.1.100` every time, create `~/.ssh/config`:

```
Host myserver
    HostName 192.168.1.100
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

Host production
    HostName prod.myapp.com
    User deploy
    IdentityFile ~/.ssh/deploy_key

Host staging
    HostName staging.myapp.com
    User deploy
    IdentityFile ~/.ssh/deploy_key
```

Now you can just type:

```bash
ssh myserver
ssh production
ssh staging
```

### Wildcard patterns

```
# Apply settings to all hosts
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes

# Apply to all hosts in a domain
Host *.myapp.com
    User deploy
    IdentityFile ~/.ssh/deploy_key
```

`ServerAliveInterval 60` — Sends a keepalive every 60 seconds to prevent timeouts.

## Multiple SSH Keys

Use different keys for different purposes:

```bash
# Personal GitHub
ssh-keygen -t ed25519 -C "personal@email.com" -f ~/.ssh/id_github

# Work server
ssh-keygen -t ed25519 -C "deploy-key" -f ~/.ssh/deploy_key

# CI/CD deployment
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/ci_deploy
```

Configure in `~/.ssh/config`:

```
Host github.com
    IdentityFile ~/.ssh/id_github

Host production
    HostName prod.myapp.com
    User deploy
    IdentityFile ~/.ssh/deploy_key
```

## SCP — Copying Files Over SSH

```bash
# Copy file to server
scp file.txt deploy@server:/home/deploy/

# Copy file from server
scp deploy@server:/var/log/nginx/error.log ./

# Copy directory recursively
scp -r ./my-app deploy@server:/home/deploy/

# With specific port
scp -P 2222 file.txt deploy@server:/home/deploy/

# Using SSH config host
scp file.txt production:/home/deploy/
```

## rsync — Better File Sync

`rsync` is smarter than `scp` — it only transfers changed files.

```bash
# Sync a directory to server
rsync -avz ./dist/ deploy@server:/var/www/myapp/

# Flags:
# -a = archive (preserves permissions, timestamps, symlinks)
# -v = verbose
# -z = compress during transfer

# Delete files on server that don't exist locally
rsync -avz --delete ./dist/ deploy@server:/var/www/myapp/

# Dry run (see what would change without doing it)
rsync -avzn ./dist/ deploy@server:/var/www/myapp/

# Exclude files
rsync -avz --exclude='node_modules' --exclude='.git' ./ deploy@server:/home/deploy/myapp/
```

## SSH Tunneling (Port Forwarding)

Access remote services through an SSH connection.

### Local port forwarding

Access a remote service as if it were local:

```bash
# Access server's PostgreSQL (port 5432) via localhost:5433
ssh -L 5433:localhost:5432 deploy@server

# Now connect to the database:
psql -h localhost -p 5433 -U myuser mydb
```

```
Your Machine (localhost:5433) ──SSH──→ Server (localhost:5432)
```

### Use case: Access a database that's only listening on localhost

The database is on the server but only accepts connections from `127.0.0.1`. You can't connect directly. SSH tunneling lets you access it through the SSH connection.

### Remote port forwarding

Expose a local service on the remote server:

```bash
# Make your local port 3000 available on the server's port 8080
ssh -R 8080:localhost:3000 deploy@server
```

### Background tunnels

```bash
# Run tunnel in the background
ssh -fN -L 5433:localhost:5432 deploy@server
# -f = background after authentication
# -N = don't execute any command (just the tunnel)

# Find and kill the tunnel
ps aux | grep ssh
kill <PID>
```

## SSH Agent

The SSH agent holds your decrypted private keys in memory so you don't have to type your passphrase repeatedly.

```bash
# Start the agent
eval "$(ssh-agent -s)"

# Add your key
ssh-add ~/.ssh/id_ed25519

# List loaded keys
ssh-add -l

# On macOS, use the keychain
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

### Agent forwarding

Use your local SSH keys on a remote server (e.g., to pull from GitHub on the server without putting keys on it):

```bash
# Enable agent forwarding
ssh -A deploy@server

# Or in ~/.ssh/config
Host server
    ForwardAgent yes
```

On the server, `git pull` will use your local machine's GitHub key.

**Security note:** Only use agent forwarding on servers you trust. A compromised server could use your forwarded keys.

## Hardening SSH

### Disable password authentication

Once key-based auth works, disable passwords:

```bash
sudo nano /etc/ssh/sshd_config
```

```
PasswordAuthentication no
PubkeyAuthentication yes
```

```bash
sudo systemctl restart sshd
```

**Test with a second terminal before closing your current session** — if something goes wrong, you'll be locked out.

### Disable root login

```
PermitRootLogin no
```

### Change default port

```
Port 2222
```

Reduces automated brute force attempts (they target port 22).

### Full recommended sshd_config changes

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

After editing:

```bash
# Validate config
sudo sshd -t

# Restart
sudo systemctl restart sshd
```

### fail2ban — Block brute force attacks

```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check banned IPs
sudo fail2ban-client status sshd
```

fail2ban monitors SSH login attempts and bans IPs that fail too many times.

## Troubleshooting

**"Permission denied (publickey)"**
```bash
# Check key permissions
ls -la ~/.ssh/
# Private key must be 600, directory must be 700

# Verbose mode to see what's happening
ssh -vv deploy@server
```

**"Connection refused"**
- SSH server not running: `sudo systemctl status sshd`
- Wrong port: check `/etc/ssh/sshd_config`
- Firewall blocking: `sudo ufw status`

**"Connection timed out"**
- Server is down or unreachable
- Firewall blocking the port
- Wrong IP address

**Key not being used**
```bash
# Check which key SSH is trying
ssh -vv deploy@server 2>&1 | grep "Offering"

# Explicitly specify the key
ssh -i ~/.ssh/deploy_key deploy@server
```

---

**Back to:** [Table of Contents](../README.md)
