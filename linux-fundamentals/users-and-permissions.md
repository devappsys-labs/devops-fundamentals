# Users, Groups, and Permissions

## Users

Every process on Linux runs as a user. Every file is owned by a user. Understanding users is foundational.

### Key users

| User | Purpose |
|------|---------|
| `root` | Superuser — can do anything. UID 0. |
| `www-data` | Nginx/Apache runs as this user |
| `deploy` | Common name for a deployment user you create |
| `nobody` | Unprivileged user for processes that need minimal access |

### Viewing users

```bash
# Current user
whoami

# Current user's ID and groups
id

# All users on the system
cat /etc/passwd

# Format: username:x:UID:GID:comment:home:shell
# Example: deploy:x:1001:1001:Deploy User:/home/deploy:/bin/bash
```

### Creating users

```bash
# Create a user with home directory and bash shell
sudo adduser deploy
# Interactive — asks for password, name, etc.

# Non-interactive version
sudo useradd -m -s /bin/bash deploy
sudo passwd deploy

# Create a system user (no home, no login — for running services)
sudo useradd -r -s /usr/sbin/nologin myapp
```

### Deleting users

```bash
sudo userdel deploy          # Remove user, keep home directory
sudo userdel -r deploy       # Remove user AND home directory
```

### Switching users

```bash
su - deploy           # Switch to deploy user (needs their password)
sudo su - deploy      # Switch using your sudo privilege
sudo -u deploy whoami # Run a single command as another user
```

## Groups

Groups let you give multiple users the same permissions.

```bash
# See your groups
groups

# See a user's groups
groups deploy

# Create a group
sudo groupadd webapps

# Add user to a group
sudo usermod -aG webapps deploy
# -a = append (don't remove from other groups)
# -G = supplementary group

# Remove user from a group
sudo gpasswd -d deploy webapps
```

**Important:** After adding a user to a group, they need to log out and back in for it to take effect.

### Common groups

| Group | Purpose |
|-------|---------|
| `sudo` | Can run commands with `sudo` |
| `docker` | Can run Docker without `sudo` |
| `www-data` | Web server group |
| `adm` | Can read log files |

## sudo

`sudo` lets a permitted user run commands as root.

```bash
sudo apt update                    # Run as root
sudo -u www-data ls /var/www       # Run as www-data
sudo su -                          # Become root
```

### Granting sudo access

```bash
# Add user to sudo group
sudo usermod -aG sudo deploy

# Or edit sudoers for fine-grained control
sudo visudo
```

### Restricting sudo to specific commands

Edit `/etc/sudoers.d/deploy`:

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

```
# Allow deploy to restart services without password
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl stop myapp, /usr/bin/systemctl start myapp
```

**Never edit `/etc/sudoers` directly.** Always use `visudo` — it validates syntax before saving.

## File Permissions

Every file and directory has three permission sets:

```
-rwxr-xr-- 1 deploy webapps 4096 Mar 06 10:00 script.sh
│├──┤├──┤├──┤   │      │
│ │   │   │    owner  group
│ │   │   └── Others: read only
│ │   └────── Group: read + execute
│ └────────── Owner: read + write + execute
└──────────── - = file, d = directory, l = symlink
```

### Permission values

| Symbol | Number | Meaning |
|--------|--------|---------|
| `r` | 4 | Read |
| `w` | 2 | Write |
| `x` | 1 | Execute (for files) / Enter (for directories) |
| `-` | 0 | No permission |

### Reading permissions

```
rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4

So: rwxr-xr-- = 754
```

### Changing permissions

```bash
# Numeric (most common)
chmod 755 script.sh    # Owner: rwx, Group: r-x, Others: r-x
chmod 644 index.html   # Owner: rw-, Group: r--, Others: r--
chmod 600 .env         # Owner: rw-, Group: ---, Others: ---
chmod 700 .ssh         # Owner: rwx, Group: ---, Others: ---

# Symbolic
chmod u+x script.sh    # Add execute for owner
chmod g-w file.txt     # Remove write for group
chmod o-r secret.txt   # Remove read for others
chmod a+r public.html  # Add read for all (a = all)

# Recursive (directories)
chmod -R 755 /var/www/myapp
```

### Changing ownership

```bash
# Change owner
sudo chown deploy file.txt

# Change owner and group
sudo chown deploy:webapps file.txt

# Recursive
sudo chown -R www-data:www-data /var/www/myapp
```

### Common permission patterns

| Scenario | Permission | Numeric |
|----------|-----------|---------|
| Web files served by Nginx | `rw-r--r--` | `644` |
| Web directories | `rwxr-xr-x` | `755` |
| Scripts | `rwxr-xr-x` | `755` |
| .env / secrets | `rw-------` | `600` |
| SSH private keys | `rw-------` | `600` |
| SSH directory | `rwx------` | `700` |
| SSH authorized_keys | `rw-r--r--` | `644` |

## Special Permissions

### setuid (SUID)

When set on an executable, it runs as the file's owner, not the user who runs it.

```bash
# passwd runs as root even when a normal user calls it
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ...
#    ^ 's' instead of 'x' = SUID is set
```

### setgid (SGID)

On a directory, new files inherit the directory's group:

```bash
# Create a shared directory
sudo mkdir /opt/shared
sudo chgrp webapps /opt/shared
sudo chmod 2775 /opt/shared
# The '2' = SGID. New files will belong to 'webapps' group.
```

### Sticky bit

On a directory, only the file owner can delete their files (even if others have write permission):

```bash
ls -ld /tmp
# drwxrwxrwt    ← 't' at the end = sticky bit
#                  Anyone can write to /tmp but can only delete their own files
```

## Debugging Permission Issues

```bash
# Check the full path permissions
namei -l /var/www/myapp/index.html

# Output shows permissions for EVERY directory in the path:
# f: /var/www/myapp/index.html
# drwxr-xr-x root     root     /
# drwxr-xr-x root     root     var
# drwxr-xr-x root     root     www
# drwxr-xr-x www-data www-data myapp
# -rw-r--r-- www-data www-data index.html

# If ANY directory in the path blocks access, Nginx can't read the file

# Check what user a process runs as
ps aux | grep nginx

# Test access as another user
sudo -u www-data cat /var/www/myapp/index.html
```

---

**Next:** [File System](file-system.md)
