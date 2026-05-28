# Cloudflare Workers Deployment Playbook

*Last updated: 2026-05-28*

---

## 1. Is Your App a Fit for Workers?

Cloudflare Workers is a serverless edge runtime: your code lives on Cloudflare's network and spins up only when a request comes in. Cheap and fast — free tier handles small apps (bundle under 1 MB, up to 100k requests/day); larger apps need the Workers Paid plan at $5/month. Two real constraints:

1. **Short time budget per request.** Each request must finish in seconds. Parsing a 50-page PDF or transcoding a video in one go will get cut off. (Waiting on an external API for 30 seconds is fine — that's idle time, not work time.)
2. **Not using Node-only libraries.** Workers runs a stripped-down JavaScript runtime — anything touching the filesystem, opening raw network sockets, or relying on native binaries (`sharp`, `puppeteer`, `playwright`, `bcrypt`, `canvas`, `pdf-parse`, `pg`, `prisma`, heavy PDF libraries) will fail. The Next.js framework itself deploys fine through an adapter (`@opennextjs/cloudflare`) — it's the heavy dependencies inside your Next.js code that break things, not Next.js itself. Ask your AI to audit `package.json` before committing.

Most modern web apps fit these constraints naturally — especially apps where an AI API does the heavy lifting instead of your own code.

### Your app fits Workers if:

- **Each request finishes in a few seconds.** "Call an API, wait, return the result" is the sweet spot.
- **Files are read by an AI API, not your code.** Shuttling a PDF to Claude or OpenAI for parsing is fine; parsing it yourself is not.
- **The database is reached over HTTPS** (Supabase REST API, Supabase Edge Functions, Firebase). You don't connect directly to Postgres or MySQL — even if your database is hosted on Supabase, using pg / prisma / drizzle with a postgresql:// connection string will fail on Workers.
- **It's a frontend plus a thin API proxy layer.**
- **Dependencies are Workers-compatible.** No Node-only libraries in `package.json` (see constraint #2).
- **It's an Anduin internal app.** Anduin's Cloudflare Zero Trust gate covers `*.fin.anduin.center` and similar subdomains, so your app sits behind company SSO for free.

### Your app does NOT fit Workers if:

- **Your code parses large files itself** (PDFs, slides, Excel, video).
- **It runs background jobs lasting minutes** (batch processing, ETL, model training).
- **It needs persistent connections** (websockets, long-lived SSE).

If any of these describe your app, **Workers is the wrong platform.** Use **Dokploy** for heavy backends or **Vercel** for public-facing Next.js apps. The rest of this document doesn't apply.

---
