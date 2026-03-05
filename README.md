# DevOps Documentation

A hands-on, beginner-friendly guide to DevOps — starting from bare metal fundamentals and progressively building up to modern practices.

This documentation is structured to help freshers understand how things work under the hood before introducing abstractions and automation.

---

## Prerequisites

- Basic understanding of Linux commands
- Access to a Linux server (VM, cloud instance, or bare metal)
- A terminal / SSH client

---

## Table of Contents

### Linux Fundamentals

Start here if you're new to Linux servers.

- [Users, Groups & Permissions](linux-fundamentals/users-and-permissions.md) — Users, groups, sudo, chmod, chown
- [File System](linux-fundamentals/file-system.md) — Directory structure, essential commands, pipes, processes
- [Package Management](linux-fundamentals/package-management.md) — apt, adding repos, installing software

### SSH

How to connect to and manage remote servers securely.

- [SSH](ssh/ssh.md) — Key-based auth, config, SCP, rsync, tunneling, agent forwarding, hardening

### Networking

Understand how traffic flows to and from your server.

- [Networking Basics](networking/networking-basics.md) — IPs, ports, DNS, firewalls (UFW), troubleshooting
- [Domain & DNS Setup](networking/dns-and-domains.md) — Buying domains, DNS records, Cloudflare, connecting to Nginx

### Git

Version control essentials for deployment workflows.

- [Git for DevOps](git/git-for-devops.md) — Branching strategies, tags, deploy keys, hooks, .gitignore

### Nginx

The foundation for serving applications, reverse proxying, and load balancing.

- [Introduction to Nginx](nginx/introduction.md) — What is Nginx and why it matters
- [Installation](nginx/installation.md) — Installing Nginx on Ubuntu/Debian
- [Configuration Basics](nginx/configuration-basics.md) — Understanding nginx.conf, directives, and contexts
- [Serving Static Files](nginx/serving-static-files.md) — Hosting HTML, CSS, JS with Nginx
- [Reverse Proxy](nginx/reverse-proxy.md) — Forwarding requests to backend services
- [SSL/TLS with Certbot](nginx/ssl-tls.md) — Enabling HTTPS on your server

### Docker

Deep dive into containerisation — from basics to production patterns.

- [Introduction to Docker](docker/introduction.md) — What is Docker, containers vs VMs, core concepts
- [Images & Containers](docker/images-and-containers.md) — Pulling, building, running, Dockerfile reference
- [Volumes](docker/volumes.md) — Persisting data, bind mounts, database volumes, backup/restore
- [Networking](docker/networking.md) — Bridge networks, DNS, port mapping, container communication, UFW gotchas
- [Docker Compose](docker/compose.md) — Multi-container apps, depends_on, health checks, profiles, dev vs prod
- [Best Practices](docker/best-practices.md) — Image optimization, security, caching, tagging, cleanup

### Systemd

Service management, logging, and scheduled tasks.

- [Systemd](systemd/systemd.md) — Writing service files, journalctl, timers, restart policies

### Manual Server Deployment

Deploy real applications on a bare metal / VM server without any CI/CD or orchestration.

#### Frontend

- [Client-Side Rendered Apps](manual-server-deployment/frontend/client-side-rendering.md) — Deploying static sites (HTML/CSS/JS, React, Angular) with Nginx
- [Server-Side Rendered Apps](manual-server-deployment/frontend/server-side-rendering.md) — Deploying SSR apps (Next.js, Nuxt, Angular Universal) with Nginx as reverse proxy
- [Dockerised Frontend with Nginx](manual-server-deployment/frontend/dockerised-with-nginx.md) — Containerising frontend apps and serving via Nginx

#### Backend

- [Backend Services](manual-server-deployment/backend/services.md) — Deploying compiled (Go, Java) and interpreted (Python, Node.js) services with Nginx
- [Dockerised Backend Services](manual-server-deployment/backend/dockerised.md) — Containerising backend services with Nginx as reverse proxy

### CI/CD Setup

Automate the deployment workflows above using GitHub Actions.

- [Introduction to GitHub Actions](cicd/github-actions/introduction.md) — Workflows, triggers, secrets, and SSH setup
- [Frontend CI/CD](cicd/github-actions/frontend.md) — Deploying CSR and SSR apps without Docker
- [Frontend CI/CD with Docker](cicd/github-actions/frontend-docker.md) — Building and deploying containerised frontends
- [Backend CI/CD](cicd/github-actions/backend.md) — Deploying compiled and interpreted services without Docker
- [Backend CI/CD with Docker](cicd/github-actions/backend-docker.md) — Building and deploying containerised backends with registry

### Security

Protect your server and applications.

- [Server Hardening](security/server-hardening.md) — SSH hardening, firewall, fail2ban, Nginx security headers, rate limiting

### Secrets Management

Handle environment variables and sensitive data properly.

- [Environment & Secrets](secrets/environment-and-secrets.md) — .env files, GitHub Actions secrets, permissions, best practices

### Monitoring & Logging

Keep your servers and apps observable.

- [Monitoring & Logging](monitoring/monitoring-and-logging.md) — Server metrics, log analysis, rotation, health checks, alerting

---

## How to Use This Repo

- Follow the sections in order if you're a beginner
- Each document is self-contained with step-by-step instructions
- Commands are written for **Ubuntu/Debian** — adapt as needed for other distros
- Look for `TODO` markers for exercises you should try yourself

## Contributing

This is a living document. Topics will be added and expanded over time. If you find errors or want to suggest improvements, open an issue or PR.
