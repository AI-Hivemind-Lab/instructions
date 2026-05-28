# Cloudflare Workers Deployment Playbook

*Last updated: 2026-05-28*

---

## 1. Is Your App a Fit for Cloudflare Workers?

Cloudflare Workers is a serverless edge runtime: your code lives on Cloudflare's network and spins up only when a request comes in. Cheap and fast — free tier handles small apps (bundle under 1 MB, up to 100k requests/day); larger apps need the Workers Paid plan at $5/month. Two real constraints:

1. **Short time budget per request.** Each request must finish in seconds. Parsing a 50-page PDF or transcoding a video in one go will get cut off. (Waiting on an external API for 30 seconds is fine — that's idle time, not work time.)
2. **Not using Node-only libraries.** Workers runs a stripped-down JavaScript runtime — anything touching the filesystem, opening raw network sockets, or relying on native binaries (`sharp`, `puppeteer`, `playwright`, `bcrypt`, `canvas`, `pdf-parse`, `pg`, `prisma`, heavy PDF libraries) will fail. The Next.js framework itself deploys fine through an adapter (`@opennextjs/cloudflare`) — it's the heavy dependencies inside your Next.js code that break things, not Next.js itself. Ask your AI to audit `package.json` before committing.

Most modern web apps fit these constraints naturally — especially apps where an AI API does the heavy lifting instead of your own code.

### Your app fits Cloudflare Workers if:

- **Each request finishes in a few seconds.** "Call an API, wait, return the result" is the sweet spot.
- **Files are read by an AI API, not your code.** Shuttling a PDF to Claude or OpenAI for parsing is fine; parsing it yourself is not.
- **The database is reached over HTTPS** (Supabase REST API, Supabase Edge Functions, Firebase). You don't connect directly to Postgres or MySQL. Even if your database is hosted on Supabase, code that uses `pg` / `prisma` / `drizzle` with a `postgresql://` connection string will fail on Workers.
- **It's a frontend plus a thin API proxy layer.**
- **Dependencies are Workers-compatible.** No Node-only libraries in `package.json` (see constraint #2).
- **It's an Anduin internal app.** Anduin's Cloudflare Zero Trust gate covers `*.fin.anduin.center` and similar subdomains, so your app sits behind company SSO for free.

### Your app does NOT fit Workers if:

- **Your code parses large files itself** (PDFs, slides, Excel, video).
- **It runs background jobs lasting minutes** (batch processing, ETL, model training).
- **It needs persistent connections** (websockets, long-lived SSE).

If any of these describe your app, **Workers is the wrong platform.** Use **Dokploy** for heavy backends or **Vercel** for public-facing Next.js apps. The rest of this document doesn't apply.

---
## 2. How to Deploy to Cloudflare Workers

> [!IMPORTANT]
> There are 3 main ways to deploy a Worker to Cloudflare:
> 
> 1. **Wrangler CLI** — run `wrangler deploy` from your terminal. The original and most flexible method. Requires Node.js + project setup. AI can drive this end-to-end.
> 2. **Cloudflare Dashboard** — edit code directly in the Cloudflare web UI and click Deploy. Good for tiny one-file Workers and quick edits. Not practical for real apps with frameworks (Next.js, etc.) or dependencies.
> 3. **Workers Builds** — connect your GitHub/GitLab repo to Cloudflare. Every push to main auto-deploys (like Vercel or Netlify). Great for teams once the app is stable, but requires initial setup that's still Wrangler-based under the hood.
> 
> **This document covers the Wrangler CLI path** because it's the only one where AI can do the full deploy unassisted — install packages, write config, run commands, set secrets, all from the terminal. The Dashboard path requires manual clicking through UI screens that AI can't see, and Workers Builds is best added later as automation on top of a working Wrangler setup.

### 2.1 Before You Start — Get These 4 Things Ready

**1. Of course, code in a GitHub repo.** Private, under the Anduin org if internal.

**2. The right Cloudflare account.** If you have multiple accounts (personal + Anduin master), pick deliberately. Internal apps deploy to the master account. Check with `npx wrangler whoami`. **Deploying to the wrong account is the #1 mistake** — custom domains won't attach if the Worker isn't on the same account as the DNS zone.

**3. A subdomain decision.** InfoSec assigns root domains for internal apps (e.g. fin.anduin.center, dev.anduin.center, etc.). If you have one assigned, pick a subdomain underneath for your app (e.g. *.fin.anduin.center) — the Zero Trust gate covers the wildcard at this level, so any subdomain you create is automatically behind company SSO. DNS auto-provisioned by Cloudflare when you set `custom_domain: true`. If you don't have a root domain assigned yet, or need a new one, contact InfoSec before committing — DNS changes are not easily reversible.

   The domain hierarchy looks like this:

```
   Root zone: company-wide domains owned by Anduin and managed by InfoSec
              (e.g. anduintransact.com, anduin.center, etc.)
   │
   └── Your assigned root domain: provisioned by InfoSec for your app or scope
       Do NOT deploy your app directly here — it's the public-facing parent.
       │
       └── Your app subdomain: create during deployment, behind SSO
           Deploy your app here. Zero Trust gate covers this level automatically.
```

**4. Secrets in `.env.local`.** Make sure every API key, OAuth ID, and database credential your app needs is in `.env.local` before you start. AI can then run secret commands that pipe values from the file straight to Cloudflare — without ever seeing the values themselves. **Never paste secrets directly into the AI chat** — once they're in chat history, they're considered compromised and must be rotated.

Also ask AI to audit `package.json` for Node-only libraries before starting (see Section 1, constraint #2). If any are found, fix them before deploying — the build will fail otherwise.

> [!NOTE]
> **Cloudflare One Client**
> 
> You'll also need **Cloudflare One Client enrolled in Anduin Zero Trust on your machine** (formerly Cloudflare WARP). Follow [this link](https://www.notion.so/anduin/Anduin-VPN-af7e70a8e35f494eb6125cda5087cbee) for setup instructions. Must be enrolled in the Anduin team, not the consumer version — they look identical but only the Anduin enrollment will pass the gate.
> 
> The client is **not required to run `wrangler deploy`** — the deploy command itself talks to Cloudflare's API over normal HTTPS and works from any network. You only need the client to **access internal apps** behind the Zero Trust gate, which includes:
> 
> - Opening your app's URL in a browser to test after deploy
> - Other Anduin users opening the app in their browsers to use it day-to-day
> 
> So: every developer who deploys, and every user who uses the app, needs the client enrolled in the Anduin team. If a teammate reports "the app won't load," first thing to check is whether they have the client enabled with Anduin enrollment (not consumer mode).

---
