# Environment Variables & Secrets Management

## The Golden Rule

**Never commit secrets to Git.** Not in code, not in config files, not in Dockerfiles.

Secrets include: database passwords, API keys, JWT secrets, private keys, tokens.

## .env Files

The simplest approach — store secrets in a `.env` file on the server.

### Format

```
DATABASE_URL=postgres://user:password@localhost:5432/mydb
SECRET_KEY=a1b2c3d4e5f6g7h8i9j0
API_KEY=sk-prod-abc123
REDIS_URL=redis://localhost:6379
PORT=8080
```

No quotes. No `export`. No spaces around `=`.

### .gitignore

Always exclude `.env` from version control:

```
# .gitignore
.env
.env.local
.env.production
*.pem
*.key
```

### Provide a template

Commit a template so developers know what variables are needed:

```
# .env.example (committed to git)
DATABASE_URL=postgres://user:password@localhost:5432/mydb
SECRET_KEY=change-me
API_KEY=your-api-key-here
REDIS_URL=redis://localhost:6379
PORT=8080
```

New developers copy this and fill in real values:

```bash
cp .env.example .env
nano .env
```

## Using .env in Different Contexts

### In systemd services

```ini
[Service]
EnvironmentFile=/home/deploy/myapp/.env
ExecStart=/usr/bin/node app.js
```

systemd reads the `.env` file and sets the variables for the process.

### In Docker

```bash
# From a file
docker run --env-file .env myapp

# Inline (for non-sensitive values)
docker run -e NODE_ENV=production -e PORT=3000 myapp
```

### In Docker Compose

```yaml
services:
  api:
    env_file:
      - .env
    environment:
      NODE_ENV: production    # Non-sensitive can be inline
```

Compose also auto-loads `.env` for variable substitution in the YAML:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}    # Read from .env
```

### In GitHub Actions

Secrets are stored in GitHub and injected at runtime:

```yaml
- name: Deploy
  uses: appleboy/ssh-action@v1
  with:
    host: ${{ secrets.SERVER_HOST }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      echo "DATABASE_URL=${{ secrets.DATABASE_URL }}" > /home/deploy/myapp/.env
```

## File Permissions for Secrets

```bash
# .env files — only readable by the owner
chmod 600 /home/deploy/myapp/.env

# Private keys
chmod 600 ~/.ssh/id_ed25519

# Verify
ls -la /home/deploy/myapp/.env
# -rw------- 1 deploy deploy 256 Mar 06 10:00 .env
```

## Managing Secrets Across Environments

### Directory structure

```
/home/deploy/myapp/
├── .env              # Currently active environment
├── .env.production   # Production values (backup)
└── app.js
```

### On the server

Create the `.env` file manually on the server (never transfer via insecure channels):

```bash
# SSH into the server
ssh deploy@server

# Create .env
nano /home/deploy/myapp/.env
# Paste the values

# Lock it down
chmod 600 /home/deploy/myapp/.env
chown deploy:deploy /home/deploy/myapp/.env
```

## GitHub Actions Secrets

### Setting up

1. Go to your repo → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add each secret

### Using in workflows

```yaml
env:
  NODE_ENV: production              # Non-sensitive — visible in logs

jobs:
  deploy:
    steps:
      - name: Create .env on server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cat > /home/deploy/myapp/.env << 'EOF'
            DATABASE_URL=${{ secrets.DATABASE_URL }}
            SECRET_KEY=${{ secrets.SECRET_KEY }}
            API_KEY=${{ secrets.API_KEY }}
            EOF
            chmod 600 /home/deploy/myapp/.env
```

Secrets are **masked** in GitHub Actions logs — they show as `***`.

### Environment-specific secrets

Use GitHub Environments for different configs:

```yaml
jobs:
  deploy-staging:
    environment: staging
    steps:
      - run: echo "Using staging secrets"
        env:
          DB: ${{ secrets.DATABASE_URL }}    # Staging DB

  deploy-production:
    environment: production
    steps:
      - run: echo "Using production secrets"
        env:
          DB: ${{ secrets.DATABASE_URL }}    # Production DB
```

## Accessing Env Vars in Code

### Node.js

```javascript
const dbUrl = process.env.DATABASE_URL;
const port = process.env.PORT || 3000;
```

### Python

```python
import os
db_url = os.environ.get('DATABASE_URL')
port = int(os.environ.get('PORT', 8000))
```

### Go

```go
dbURL := os.Getenv("DATABASE_URL")
port := os.Getenv("PORT")
if port == "" {
    port = "8080"
}
```

### Java (Spring Boot)

```properties
# application.properties
spring.datasource.url=${DATABASE_URL}
server.port=${PORT:8080}
```

## Common Mistakes

**Committing .env to git**
```bash
# If you accidentally committed it
git rm --cached .env
echo ".env" >> .gitignore
git commit -m "Remove .env from tracking"
# IMPORTANT: The secret is still in git history
# Rotate all secrets that were exposed
```

**Hardcoding secrets in Dockerfiles**
```dockerfile
# NEVER do this
ENV DATABASE_PASSWORD=mysecret

# Anyone can see it with: docker inspect
```

**Logging secrets**
```javascript
// NEVER do this
console.log('Config:', process.env);
console.log('DB URL:', process.env.DATABASE_URL);
```

**Using the same secrets across environments**
- Production and staging should have different secrets
- If staging is compromised, production stays safe

## When You Outgrow .env Files

For more complex setups, consider:

| Tool | Use case |
|------|----------|
| **HashiCorp Vault** | Centralized secret management, dynamic secrets, rotation |
| **AWS Secrets Manager** | If you're on AWS |
| **AWS SSM Parameter Store** | Simpler/cheaper AWS option |
| **GCP Secret Manager** | If you're on GCP |
| **Azure Key Vault** | If you're on Azure |
| **SOPS** | Encrypted files committed to git |
| **doppler** | Cloud-hosted secret management |

For bare metal deployments, `.env` files with proper permissions are perfectly fine.

---

**Back to:** [Table of Contents](../README.md)
