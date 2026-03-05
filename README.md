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

### Nginx

Start here. Nginx is the foundation for serving applications, reverse proxying, and load balancing.

- [Introduction to Nginx](nginx/introduction.md) — What is Nginx and why it matters
- [Installation](nginx/installation.md) — Installing Nginx on Ubuntu/Debian
- [Configuration Basics](nginx/configuration-basics.md) — Understanding nginx.conf, directives, and contexts
- [Serving Static Files](nginx/serving-static-files.md) — Hosting HTML, CSS, JS with Nginx
- [Reverse Proxy](nginx/reverse-proxy.md) — Forwarding requests to backend services
- [SSL/TLS with Certbot](nginx/ssl-tls.md) — Enabling HTTPS on your server

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

> Coming soon — Will cover setting up CI/CD pipelines to automate the deployment workflows above.

---

## How to Use This Repo

- Follow the sections in order if you're a beginner
- Each document is self-contained with step-by-step instructions
- Commands are written for **Ubuntu/Debian** — adapt as needed for other distros
- Look for `TODO` markers for exercises you should try yourself

## Contributing

This is a living document. Topics will be added and expanded over time. If you find errors or want to suggest improvements, open an issue or PR.
