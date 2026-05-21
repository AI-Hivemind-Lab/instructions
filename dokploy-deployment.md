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

---

## Quick Path

> **📖 Full walkthrough →** For step-by-step instructions with screenshots, see the [Dokploy User Guide on Notion](https://www.notion.so/anduin/Draft-Dokploy-User-Guide-2fa3f653b1df80248273e5de25ae1d6a). The Notion page is the source of truth for the click-by-click flow — this section is the high-level shape so you know what you're about to do.

1. **Prepare your app.** Add a `Dockerfile` at the repo root (or rely on Nixpacks for Node/Python/Go projects) and create a `.env.example` listing every environment variable your app reads.

2. **Create a service in Dokploy.** Log into Dokploy, pick or create a project, and add either an **Application** service (single container) or a **Compose** service (multi-container) pointing at your Git repo.

3. **Set environment variables.** In the service settings, paste each variable from `.env.example` with its production value. Secrets like LiteLLM API keys come from 1Password — never check them into git.

4. **Add a domain.** Assign a subdomain (typically under `anduin.center`) in the service's Domains tab. Dokploy provisions a Let's Encrypt SSL certificate automatically.

5. **Deploy and watch logs.** Click Deploy and tail the build + runtime logs in the Logs tab. First builds typically take 2–5 minutes.

6. **Connect to internal Supabase (optional).** If your app uses self-hosted Supabase, set `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY` from the Supabase guide.

✅ **You're done when:** Your app is reachable at the assigned domain and returns responses (HTTP 200, not 502 or 5xx).

---

## Troubleshooting

**Build fails with "no Dockerfile found" or "no build pack matched"**
Dokploy didn't detect your `Dockerfile` or a Nixpacks-compatible project. Make sure a `Dockerfile` exists at the repo root, or that your project has a recognized entrypoint (`package.json`, `requirements.txt`, `go.mod`, etc.) so Nixpacks can pick it up.

**App builds but crashes on startup**
Almost always a missing or wrong environment variable. Check the Logs tab — the runtime error will print there. Compare what your app reads (anything in code that touches `process.env` or `os.environ`) against what you set in the service's Environment tab.

**Domain returns `502 Bad Gateway`**
Your app isn't listening on the port Dokploy is forwarding to. Defaults are `3000` for Node/Next.js and `8000` for Python apps. Check the `EXPOSE` line in your `Dockerfile` and your app's actual listen port — they must match the port set in Dokploy's General tab.

**SSL certificate fails to provision**
DNS hasn't propagated yet, or the domain doesn't resolve to Dokploy's IP. Wait 5–10 minutes after adding the domain. If it still fails, ask the infra team to verify the DNS record.

**Can't log into Dokploy**
You're not on the Anduin network. Connect to VPN and retry. If you still can't reach the URL, ask in `#vibecoding` on Slack.
