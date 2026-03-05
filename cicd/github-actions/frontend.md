# CI/CD for Frontend Apps (Without Docker)

Automate building and deploying frontend applications to your server using GitHub Actions.

## Prerequisites

- Server with Nginx configured (see [Serving Static Files](../../nginx/serving-static-files.md) or [Reverse Proxy](../../nginx/reverse-proxy.md))
- GitHub Secrets configured: `SERVER_HOST`, `SERVER_USER`, `SSH_PRIVATE_KEY` (see [Introduction](introduction.md))

## Client-Side Rendered Apps (React, Angular, Plain HTML)

CSR apps produce static files. The workflow builds them and copies the output to the server.

### React (Vite) — deploy.yml

```yaml
name: Deploy Frontend

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Copy build to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "dist/*"
          target: "/var/www/myapp"
          strip_components: 1

      - name: Set permissions
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            sudo chown -R www-data:www-data /var/www/myapp
            sudo chmod -R 755 /var/www/myapp
```

### What each step does

| Step | Purpose |
|------|---------|
| `actions/checkout@v4` | Clones your repo on the runner |
| `actions/setup-node@v4` | Installs Node.js 20 with npm caching |
| `npm ci` | Clean install — faster and stricter than `npm install` |
| `npm run build` | Produces `dist/` folder with static files |
| `scp-action` | Copies `dist/*` to `/var/www/myapp` on your server |
| `strip_components: 1` | Removes the `dist/` prefix so files land directly in target |
| `ssh-action` | SSHes in to fix file ownership |

### React (CRA)

Same workflow, just the build output is `build/` instead of `dist/`:

```yaml
      - name: Build
        run: npm run build

      - name: Copy build to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "build/*"
          target: "/var/www/myapp"
          strip_components: 1
```

### Angular

```yaml
      - name: Build
        run: npx ng build --configuration production

      - name: Copy build to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "dist/my-angular-app/browser/*"
          target: "/var/www/myapp"
          strip_components: 3
```

`strip_components: 3` removes `dist/my-angular-app/browser/` prefix.

### Plain HTML/CSS/JS

No build step needed — just copy the files:

```yaml
name: Deploy Static Site

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Copy files to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "*.html,*.css,*.js,css/*,js/*,images/*"
          target: "/var/www/mysite"

      - name: Set permissions
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            sudo chown -R www-data:www-data /var/www/mysite
```

## Server-Side Rendered Apps (Next.js, Nuxt)

SSR apps are running processes. The workflow builds the app on the server and restarts the service.

### Next.js — deploy.yml

```yaml
name: Deploy Next.js

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Copy source to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "."
          target: "/home/deploy/my-nextjs-app"

      - name: Build and restart on server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/my-nextjs-app
            npm ci --production
            npm run build
            sudo systemctl restart nextjs-app
```

### Alternative: Build on runner, copy the output

Building on the server uses server resources. If your server is small, build on the GitHub runner instead:

```yaml
name: Deploy Next.js (Build on Runner)

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install and build
        run: |
          npm ci
          npm run build

      - name: Copy build output to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: ".next/*,package.json,package-lock.json,public/*,next.config.*"
          target: "/home/deploy/my-nextjs-app"

      - name: Install production deps and restart
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/my-nextjs-app
            npm ci --production
            sudo systemctl restart nextjs-app
```

### Nuxt.js — deploy.yml

```yaml
name: Deploy Nuxt

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install and build
        run: |
          npm ci
          npm run build

      - name: Copy output to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: ".output/*"
          target: "/home/deploy/my-nuxt-app"

      - name: Restart service
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            sudo systemctl restart nuxt-app
```

## Adding a Test Step

Always run tests before deploying:

```yaml
      - name: Run tests
        run: npm test

      # Deploy steps only run if tests pass
      - name: Copy build to server
        uses: appleboy/scp-action@v0.1.7
        ...
```

## Deploy Only on Main, Test on PRs

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run build

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run build

      - name: Deploy to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "dist/*"
          target: "/var/www/myapp"
          strip_components: 1
```

Key parts:
- `needs: test` — deploy waits for test job to pass
- `if: github.event_name == 'push' && github.ref == 'refs/heads/main'` — only deploy on push to main, not on PRs

## Troubleshooting

**"Permission denied (publickey)"**
- Check `SSH_PRIVATE_KEY` secret — it must be the full private key including `-----BEGIN/END-----` lines
- Ensure the corresponding public key is in `~/.ssh/authorized_keys` on the server

**Build fails on runner**
- Check the Node.js version matches your local version
- Ensure `package-lock.json` is committed

**Files not appearing on server**
- Check `strip_components` value — wrong value puts files in a nested directory
- SSH into the server and check: `ls -la /var/www/myapp/`

**Service won't restart**
- The deploy user needs sudo permission for `systemctl restart`
- Add to sudoers: `deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nextjs-app`

---

**Next:** [Frontend CI/CD with Docker](frontend-docker.md)
