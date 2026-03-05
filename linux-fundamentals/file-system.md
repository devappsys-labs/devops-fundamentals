# Linux File System

## Directory Structure

Everything in Linux is a file. The directory tree starts at `/` (root).

```
/
├── bin/          → Essential binaries (ls, cp, mv, cat)
├── boot/         → Bootloader files, kernel
├── dev/          → Device files (disks, terminals)
├── etc/          → System configuration files
│   ├── nginx/
│   ├── systemd/
│   ├── ssh/
│   └── ...
├── home/         → User home directories
│   ├── deploy/
│   └── adarsh/
├── lib/          → Shared libraries
├── opt/          → Optional/third-party software
├── proc/         → Virtual filesystem for process info
├── root/         → Root user's home directory
├── run/          → Runtime data (PID files, sockets)
├── sbin/         → System binaries (systemctl, iptables)
├── srv/          → Service data (sometimes used for web)
├── sys/          → Virtual filesystem for kernel/hardware info
├── tmp/          → Temporary files (cleared on reboot)
├── usr/          → User programs, libraries, docs
│   ├── bin/      → User binaries
│   ├── lib/      → Libraries
│   ├── local/    → Locally installed software
│   └── share/    → Shared data
└── var/          → Variable data
    ├── log/      → Log files
    ├── www/      → Web server files (Nginx default)
    ├── lib/      → State data (databases, Docker)
    └── run/      → Runtime data
```

### Directories you'll use most in DevOps

| Directory | You'll use it for |
|-----------|------------------|
| `/etc/nginx/` | Nginx configuration |
| `/etc/systemd/system/` | Custom service files |
| `/var/www/` | Web application files |
| `/var/log/` | Application and system logs |
| `/home/deploy/` | Deployment user's files |
| `/opt/` | Installing third-party apps |
| `/tmp/` | Temporary build artifacts |

## Essential Commands

### Navigating

```bash
pwd                    # Print working directory
cd /var/www            # Change directory
cd ~                   # Go to home directory
cd -                   # Go to previous directory
cd ..                  # Go up one level
```

### Listing files

```bash
ls                     # List files
ls -l                  # Long format (permissions, owner, size, date)
ls -la                 # Include hidden files (starting with .)
ls -lh                 # Human-readable file sizes (KB, MB, GB)
ls -lt                 # Sort by modification time (newest first)
ls -lS                 # Sort by size (largest first)
```

### Creating and deleting

```bash
# Files
touch newfile.txt              # Create empty file
mkdir mydir                    # Create directory
mkdir -p path/to/nested/dir    # Create nested directories

# Deleting
rm file.txt                    # Delete file
rm -r directory/               # Delete directory recursively
rm -rf directory/              # Force delete (no prompts) — BE CAREFUL
rmdir emptydir/                # Delete empty directory only
```

### Copying and moving

```bash
cp file.txt backup.txt                    # Copy file
cp -r source_dir/ dest_dir/              # Copy directory
mv file.txt /new/location/              # Move file
mv oldname.txt newname.txt               # Rename file
```

### Viewing files

```bash
cat file.txt                   # Print entire file
less file.txt                  # Paginated view (q to quit)
head -n 20 file.txt            # First 20 lines
tail -n 20 file.txt            # Last 20 lines
tail -f /var/log/nginx/access.log  # Follow log in real time
```

### Searching

```bash
# Find files by name
find /var/www -name "*.html"
find / -name "nginx.conf"
find . -type d -name "node_modules"    # Directories only

# Search file contents
grep "error" /var/log/nginx/error.log
grep -r "TODO" /var/www/myapp/         # Recursive search
grep -rn "proxy_pass" /etc/nginx/      # With line numbers
grep -i "error" logfile.txt            # Case insensitive
```

### Disk usage

```bash
df -h                          # Disk space on all filesystems
du -sh /var/www/myapp          # Size of a directory
du -sh /var/log/*              # Size of each item in /var/log
du -h --max-depth=1 /          # Size of top-level directories

# Find large files
find / -type f -size +100M 2>/dev/null    # Files > 100MB
```

## Working with Text

### Pipes and redirection

```bash
# Pipe: send output of one command as input to another
cat access.log | grep "404" | wc -l       # Count 404 errors

# Redirect output to a file
echo "hello" > file.txt                    # Overwrite
echo "world" >> file.txt                   # Append

# Redirect stderr
command 2> error.log                       # Errors to file
command > output.log 2>&1                  # Both stdout and stderr
command > /dev/null 2>&1                   # Discard all output
```

### Useful text tools

```bash
wc -l file.txt                 # Count lines
sort file.txt                  # Sort lines
uniq                           # Remove duplicate lines (use with sort)
cut -d: -f1 /etc/passwd        # Extract first field (delimiter = :)
awk '{print $1}' access.log    # Print first column
sed 's/old/new/g' file.txt     # Replace text
```

### Real-world example: Analyze Nginx access log

```bash
# Top 10 IP addresses hitting your server
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Count requests by status code
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Find all 500 errors
grep '" 500 ' /var/log/nginx/access.log
```

## Process Management

```bash
# View running processes
ps aux                         # All processes
ps aux | grep nginx            # Filter for nginx
top                            # Real-time process monitor
htop                           # Better version of top (install: apt install htop)

# Find what's using a port
sudo lsof -i :80               # What's on port 80
sudo ss -tlnp                  # All listening ports

# Kill a process
kill <PID>                     # Graceful stop
kill -9 <PID>                  # Force kill (last resort)
killall nginx                  # Kill all nginx processes

# Run in background
./long-task.sh &               # Run in background
nohup ./long-task.sh &         # Survives terminal close
```

## Symlinks

A symbolic link (symlink) is a pointer to another file or directory. Used extensively by Nginx (`sites-enabled` → `sites-available`).

```bash
# Create a symlink
ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/myapp
#     ^^^ target (real file)           ^^^ link name (pointer)

# View symlinks
ls -l /etc/nginx/sites-enabled/
# lrwxrwxrwx 1 root root 34 Mar 06 myapp -> /etc/nginx/sites-available/myapp

# Remove a symlink (NOT the target)
rm /etc/nginx/sites-enabled/myapp

# Check if a file is a symlink
file /etc/nginx/sites-enabled/myapp
```

## Environment Variables

```bash
# View all
env
printenv

# View one
echo $HOME
echo $PATH
echo $USER

# Set temporarily (current session only)
export MY_VAR="hello"

# Set permanently (add to ~/.bashrc or ~/.profile)
echo 'export MY_VAR="hello"' >> ~/.bashrc
source ~/.bashrc    # Reload without logging out

# Common variables
echo $HOME         # /home/deploy
echo $USER         # deploy
echo $PATH         # Where the shell looks for commands
echo $SHELL        # /bin/bash or /bin/zsh
```

## Archiving and Compression

```bash
# Create a tar.gz archive
tar -czf backup.tar.gz /var/www/myapp/

# Extract
tar -xzf backup.tar.gz

# Extract to specific directory
tar -xzf backup.tar.gz -C /tmp/

# View contents without extracting
tar -tzf backup.tar.gz

# Flags:
# c = create, x = extract, t = list
# z = gzip compression
# f = filename follows
```

---

**Next:** [Package Management](package-management.md)
