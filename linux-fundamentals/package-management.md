# Package Management

Package managers install, update, and remove software on your server. On Ubuntu/Debian, the package manager is `apt`.

## apt — The Package Manager

### Updating package lists

```bash
sudo apt update
```

This downloads the latest list of available packages from the repositories. **Always run this before installing anything.**

It does NOT upgrade installed packages — it only refreshes the list.

### Installing packages

```bash
sudo apt install nginx -y          # -y auto-confirms
sudo apt install nginx curl git    # Multiple packages
```

### Removing packages

```bash
sudo apt remove nginx              # Remove package, keep config files
sudo apt purge nginx               # Remove package AND config files
sudo apt autoremove                # Remove unused dependencies
```

### Upgrading packages

```bash
sudo apt upgrade                   # Upgrade all installed packages
sudo apt full-upgrade              # Upgrade with dependency changes (adds/removes)
```

### Searching for packages

```bash
apt search nginx                   # Search available packages
apt show nginx                     # Show package details
apt list --installed               # List installed packages
apt list --installed | grep nginx  # Check if nginx is installed
```

### Checking package versions

```bash
apt policy nginx                   # Installed and available versions
nginx -v                           # Check running version directly
```

## Adding Third-Party Repositories

Not everything is in the default Ubuntu repos. You'll often add external repos for newer versions.

### Node.js (via NodeSource)

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

### Docker

```bash
# Add Docker's official GPG key and repo
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### General pattern

```bash
# 1. Download and add the GPG key
# 2. Add the repository to sources
# 3. apt update
# 4. apt install
```

## Automatic Security Updates

Keep your server patched without manual intervention:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

This automatically installs security updates. It will NOT upgrade application packages — only security patches.

Check the config:

```bash
cat /etc/apt/apt.conf.d/50unattended-upgrades
```

## Installing from Source

Sometimes a package isn't available via `apt`. You'll compile it from source:

```bash
# General pattern
sudo apt install -y build-essential    # Compiler and tools
wget https://example.com/software-1.0.tar.gz
tar -xzf software-1.0.tar.gz
cd software-1.0
./configure
make
sudo make install
```

This is rare in modern DevOps — prefer `apt`, Docker, or pre-built binaries.

## Installing Pre-Built Binaries

Many tools ship as a single binary (Go tools, Terraform, etc.):

```bash
# Example: installing a Go-based tool
wget https://example.com/tool-linux-amd64.tar.gz
tar -xzf tool-linux-amd64.tar.gz
sudo mv tool /usr/local/bin/
tool --version
```

`/usr/local/bin/` is the standard location for manually installed binaries.

## Useful Packages to Install on a New Server

```bash
sudo apt update
sudo apt install -y \
  curl \
  wget \
  git \
  vim \
  htop \
  tree \
  unzip \
  net-tools \
  ufw \
  fail2ban
```

| Package | Purpose |
|---------|---------|
| `curl` / `wget` | Download files from the web |
| `git` | Version control |
| `vim` | Text editor |
| `htop` | Interactive process viewer |
| `tree` | Display directory structure as tree |
| `unzip` | Extract ZIP files |
| `net-tools` | `ifconfig`, `netstat` (legacy but useful) |
| `ufw` | Simple firewall |
| `fail2ban` | Block brute force SSH attempts |

---

**Back to:** [Table of Contents](../README.md)
