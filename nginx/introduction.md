# Introduction to Nginx

## What is Nginx?

Nginx (pronounced "engine-x") is a high-performance web server, reverse proxy, and load balancer. It was created by Igor Sysoev in 2004 to solve the C10K problem — handling 10,000+ concurrent connections efficiently.

## Why Nginx?

As a DevOps engineer, Nginx will be one of your most-used tools. Here's why:

**Web Server** — Serves static files (HTML, CSS, JS, images) directly to browsers. This is how most frontend applications are deployed.

**Reverse Proxy** — Sits in front of your backend application and forwards client requests to it. Your backend runs on a local port (e.g., `localhost:3000`), and Nginx handles the public-facing HTTP/HTTPS traffic.

**Load Balancer** — Distributes incoming traffic across multiple instances of your application for high availability.

**SSL/TLS Termination** — Handles HTTPS encryption so your application doesn't have to.

## How Does Nginx Fit in a Deployment?

Here's a typical flow when a user visits your website:

```
User's Browser
      |
      v
   Internet
      |
      v
   Nginx (port 80/443)
      |
      ├── Static files? → Serve directly from disk
      |
      └── Dynamic request? → Forward to backend (localhost:3000, :8080, etc.)
```

### Scenario: Static Website (Client-Side Rendered)

```
Browser → Nginx → /var/www/html/index.html
```

Nginx directly serves the HTML, CSS, and JS files. No backend needed.

### Scenario: Backend API

```
Browser → Nginx (port 80) → proxy_pass → Node.js app (port 3000)
```

Nginx receives the request on port 80 and forwards it to your Node.js/Go/Python application running on a local port.

### Scenario: Full-Stack Application

```
Browser → Nginx (port 80/443)
              |
              ├── /           → Serve React build from /var/www/app/
              ├── /api/*      → proxy_pass to backend on port 8080
              └── /static/*   → Serve static assets from /var/www/static/
```

A single Nginx instance handles both frontend and backend routing.

## Nginx vs Apache

| Feature | Nginx | Apache |
|---------|-------|--------|
| Architecture | Event-driven, async | Process/thread per connection |
| Performance | Better for static content and high concurrency | Good for dynamic content with mod_php |
| Config style | Centralized (nginx.conf) | Distributed (.htaccess supported) |
| Memory usage | Lower | Higher under load |

For most modern deployments, Nginx is the default choice. This documentation focuses entirely on Nginx.

## Key Terminology

| Term | Meaning |
|------|---------|
| **Upstream** | A backend server that Nginx forwards requests to |
| **Directive** | A configuration instruction (e.g., `listen 80;`) |
| **Context/Block** | A section in the config enclosed in `{}` (e.g., `server { }`) |
| **Virtual Host** | A server block that lets you host multiple sites on one server |
| **Reverse Proxy** | Nginx sitting between the client and your app, forwarding requests |
| **Static Files** | Files served as-is (HTML, CSS, JS, images) without processing |

---

**Next:** [Installation](installation.md)
