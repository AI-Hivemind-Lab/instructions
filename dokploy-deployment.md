# Dokploy Deployment Guide

## What is this?

**What is Dokploy?**
Dokploy is a self-hosted deployment platform — think Vercel or Heroku, but running on Anduin's own servers. You point it at a Git repo and it builds and runs your app in containers, handling routing, SSL, and rollbacks for you.

**Why Dokploy and not Vercel, Render, or Railway?**
Cost control, data residency, and independence from third-party pricing changes. Self-hosting means our deployment platform doesn't change pricing or features without our consent, and customer data never leaves Anduin infrastructure.

**Who manages the Dokploy server?**
The infra team. You don't install or maintain anything — you just use the Dokploy web UI. Server upgrades, SSL cert renewals, and outage response are handled centrally.

**What is a Compose service?**
A Compose service is a deployment based on a `docker-compose.yml` file, where you describe one or more containers together (an app plus its sidecars, for example). For a single container built from a `Dockerfile` or Nixpacks, use the simpler **Application** service instead.

**Do I need Docker on my laptop?**
Not for deployment — Dokploy builds your image on the server. You may want Docker locally to test your `Dockerfile` before pushing, but it's optional.

---

## Prerequisites

- App code in a Git repo (GitHub or GitLab) that Dokploy can pull from
- A `Dockerfile` at the repo root, OR a project structure Nixpacks can auto-detect (Node, Python, Go, Rust, etc.)
- All environment variables your app reads listed in a `.env.example` file
- Access to Anduin's Dokploy instance (ask your manager for the URL and login)
