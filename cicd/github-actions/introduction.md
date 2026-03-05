# Introduction to GitHub Actions

## What is GitHub Actions?

GitHub Actions is a CI/CD platform built into GitHub. It lets you automate tasks whenever something happens in your repository — like pushing code, opening a PR, or creating a release.

In the context of this guide, we'll use GitHub Actions to **automatically deploy your application to a server** whenever you push to the `main` branch.

## Key Concepts

### Workflow

A YAML file in `.github/workflows/` that defines what to automate. Each repository can have multiple workflows.

### Event (Trigger)

What causes the workflow to run:

```yaml
on:
  push:
    branches: [main]        # Run when code is pushed to main
  pull_request:
    branches: [main]        # Run when a PR targets main
  workflow_dispatch:         # Manual trigger from GitHub UI
```

### Job

A set of steps that run on the same machine. Jobs run in parallel by default.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest   # The machine type
    steps:
      - ...
  deploy:
    runs-on: ubuntu-latest
    needs: build              # This job waits for 'build' to finish
    steps:
      - ...
```

### Step

A single task within a job — either a shell command or a reusable action:

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4         # Reusable action

  - name: Install dependencies
    run: npm install                   # Shell command
```

### Action

A reusable unit of code. You can use community actions from the GitHub Marketplace or write your own. Common ones:

| Action | Purpose |
|--------|---------|
| `actions/checkout@v4` | Clone your repo |
| `actions/setup-node@v4` | Install Node.js |
| `actions/setup-go@v5` | Install Go |
| `actions/setup-java@v4` | Install Java |
| `actions/setup-python@v5` | Install Python |

### Runner

The machine that executes your workflow. GitHub provides hosted runners (Ubuntu, Windows, macOS) or you can use self-hosted runners.

## Minimal Workflow Example

Create `.github/workflows/deploy.yml` in your repo:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to server
        run: echo "Deploying..."
```

## How We'll Deploy

The deployment pattern across all our workflows:

```mermaid
flowchart LR
    A[Push to main] --> B[GitHub Actions runner\nbuilds the app] --> C[Copies files to\nyour server via SSH]
```

### SSH Setup

Every workflow will SSH into your server to deploy. You'll need to store your SSH credentials as **GitHub Secrets**.

#### Generate an SSH key pair (if you don't have one for deployment)

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key -N ""
```

#### Add the public key to your server

```bash
# On your server
cat ~/.ssh/deploy_key.pub >> ~/.ssh/authorized_keys
```

#### Add secrets to your GitHub repo

Go to your repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret Name | Value |
|-------------|-------|
| `SERVER_HOST` | Your server's IP or domain |
| `SERVER_USER` | SSH username (e.g., `deploy`) |
| `SSH_PRIVATE_KEY` | Contents of `~/.ssh/deploy_key` (the private key) |

#### Using SSH in a workflow

```yaml
- name: Deploy to server
  uses: appleboy/ssh-action@v1
  with:
    host: ${{ secrets.SERVER_HOST }}
    username: ${{ secrets.SERVER_USER }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      cd /var/www/myapp
      git pull origin main
      # ... restart app, etc.
```

Alternatively, using `scp` to copy files:

```yaml
- name: Copy files to server
  uses: appleboy/scp-action@v0.1.7
  with:
    host: ${{ secrets.SERVER_HOST }}
    username: ${{ secrets.SERVER_USER }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "dist/*"
    target: "/var/www/myapp"
```

## Workflow File Location

Workflows **must** be in `.github/workflows/` directory:

```
my-project/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── src/
├── package.json
└── ...
```

## Viewing Workflow Runs

- Go to your repo on GitHub → **Actions** tab
- Click on a workflow run to see logs
- Each step shows its output — useful for debugging

## Environment Variables vs Secrets

```yaml
env:
  NODE_ENV: production              # Visible in logs — for non-sensitive values

steps:
  - name: Deploy
    env:
      API_KEY: ${{ secrets.API_KEY }}  # Masked in logs — for sensitive values
    run: ./deploy.sh
```

**Rule:** Anything sensitive (passwords, keys, tokens) goes in **Secrets**. Everything else can be a plain environment variable.

---

**Next:** [Frontend CI/CD](frontend.md)
