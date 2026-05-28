# Cloudflare Workers Deployment Playbook

*Last updated: 2026-05-28*

---

## 1. Is Your App a Fit for Cloudflare Workers?

Cloudflare Workers is a serverless edge runtime: your code lives on Cloudflare's network and spins up only when a request comes in. Cheap and fast — free tier handles most apps (bundle under 3 MiB gzipped, 100k requests/day, 10ms CPU per request); larger apps need the Workers Paid plan at $5/month. Two real constraints:

1. **Short time budget per request.** Each request must finish in seconds. Parsing a 50-page PDF or transcoding a video in one go will get cut off. (Waiting on an external API for 30 seconds is fine — that's idle time, not work time.)
2. **Not using Node-only libraries.** Workers runs a stripped-down JavaScript runtime — anything touching the filesystem, opening raw network sockets, or relying on native binaries (`sharp`, `puppeteer`, `playwright`, `bcrypt`, `canvas`, `pdf-parse`, `pg`, `prisma`, heavy PDF libraries) will fail. The Next.js framework itself deploys fine through an adapter (`@opennextjs/cloudflare`) — it's the heavy dependencies inside your Next.js code that break things, not Next.js itself. Ask your AI to audit `package.json` before committing.

Most modern web apps fit these constraints naturally — especially apps where an AI API does the heavy lifting instead of your own code.

### Your app fits Cloudflare Workers if:

- **Each request finishes in a few seconds.** "Call an API, wait, return the result" is the sweet spot.
- **Files are read by an AI API, not your code.** Shuttling a PDF to Claude or OpenAI for parsing is fine; parsing it yourself is not.
- **The database is reached over HTTPS** (Supabase REST API, Supabase Edge Functions, Firebase). You don't connect directly to Postgres or MySQL. Even if your database is hosted on Supabase, code that uses `pg` / `prisma` / `drizzle` with a `postgresql://` connection string will fail on Workers.
- **It's a frontend plus a thin API proxy layer.**
- **Dependencies are Workers-compatible.** No Node-only libraries in `package.json` (see constraint #2).
- **It's an Anduin internal app.** Anduin's Cloudflare Zero Trust gate covers wildcards on assigned root domains, so your app sits behind company SSO for free.

### Your app does NOT fit Cloudflare Workers if:

- **Your code parses large files itself** (PDFs, slides, Excel, video).
- **It runs background jobs lasting minutes** (batch processing, ETL, model training).
- **It needs persistent connections** (websockets, long-lived SSE).

If any of these describe your app, **Cloudflare Workers is the wrong platform.** Use **Dokploy** for heavy backends or **Vercel** for public-facing Next.js apps. The rest of this document doesn't apply.

---
## 2. How to Deploy to Cloudflare Workers

> [!IMPORTANT]
> There are 3 main ways to deploy to Cloudflare Workers:
> 
> 1. **Wrangler CLI** — run `wrangler deploy` from your terminal. The original and most flexible method. Requires Node.js + project setup. AI can drive this end-to-end.
> 2. **Cloudflare Dashboard** — edit code directly in the Cloudflare web UI and click Deploy. Good for tiny one-file Workers and quick edits. Not practical for real apps with frameworks (Next.js, etc.) or dependencies.
> 3. **Workers Builds** — connect your GitHub/GitLab repo to Cloudflare. Every push to main auto-deploys (like Vercel or Netlify). Great for teams once the app is stable, but requires initial setup that's still Wrangler-based under the hood.
> 
> **This document covers the Wrangler CLI path** because it's the only one where AI can do the full deploy unassisted — install packages, write config, run commands, set secrets, all from the terminal. The Dashboard path requires manual clicking through UI screens that AI can't see, and Workers Builds is best added later as automation on top of a working Wrangler setup.

### 2.1 Before You Start — Get These 4 Things Ready

1. **Of course, code in a GitHub repo.** Private, under the Anduin org if internal.

2. **The right Cloudflare account.** If you have multiple accounts (personal + Anduin master), pick deliberately. Internal apps deploy to the master account. Check with `npx wrangler whoami`. **Deploying to the wrong account is the #1 mistake** — custom domains won't attach if the Worker isn't on the same account as the DNS zone.

3. **A subdomain decision.** InfoSec assigns root domains for internal apps (e.g. `fin.anduin.center`, `dev.anduin.center`, etc.). If you have one assigned, pick a subdomain underneath for your app (e.g. `*.fin.anduin.center`) — the Zero Trust gate covers the wildcard at this level, so any subdomain you create is automatically behind company SSO. DNS auto-provisioned by Cloudflare when you set `custom_domain: true`. If you don't have a root domain assigned yet, or need a new one, contact InfoSec before committing — DNS changes are not easily reversible.

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

4. **Secrets in `.env.local`.** Make sure every API key, OAuth ID, and database credential your app needs is in `.env.local` before you start. AI can then run secret commands that pipe values from the file straight to Cloudflare — without ever seeing the values themselves. **Never paste secrets directly into the AI chat** — once they're in chat history, they're considered compromised and must be rotated.

5. **Make sure your app is not using Node-only libraries** Ask AI to audit `package.json` for Node-only libraries before starting (see Section 1, constraint #2). If any are found, fix them before deploying — the build will fail otherwise.

> [!NOTE]
> **Cloudflare One Client**
> 
> You'll also need **Cloudflare One Client enrolled in Anduin Zero Trust** on your machine (formerly Cloudflare WARP). Follow [this link](https://www.notion.so/anduin/Anduin-VPN-af7e70a8e35f494eb6125cda5087cbee) for setup instructions. Must be enrolled in the Anduin team, not the consumer version — they look identical but only the Anduin enrollment will pass the gate.
> 
> The client is **not required to run `wrangler deploy`** — the deploy command itself talks to Cloudflare's API over normal HTTPS and works from any network. You only need the client to **access internal apps** behind the Zero Trust gate, which includes:
> 
> - Opening your app's URL in a browser to test after deploy
> - Other Anduin users opening the app in their browsers to use it day-to-day
> 
> So: every developer who deploys, and every user who uses the app, needs the client enrolled in the Anduin team. If a teammate reports "the app won't load," first thing to check is whether they have the client enabled with Anduin enrollment (not consumer mode).

---

### 2.2 The Deploy Itself — 6 Checkpoints

AI will run the commands. Your job is to review the output at each checkpoint and confirm it's correct before AI moves on.

**Overview:**

1. [Verify your Cloudflare account](#checkpoint-1--verify-your-cloudflare-account)
2. [Install the Cloudflare adapter and update `.gitignore`](#checkpoint-2--install-the-cloudflare-adapter-and-update-gitignore)
3. [Write the deploy config in `wrangler.jsonc`](#checkpoint-3--write-the-deploy-config-in-wranglerjsonc)
4. [Pre-deploy sanity check via `cf:preview`](#checkpoint-4--pre-deploy-sanity-check-via-cfpreview)
5. [Pre-flight validation via dry-run](#checkpoint-5--pre-flight-validation-via-dry-run)
6. [Real deploy and set secrets](#checkpoint-6--real-deploy-and-set-secrets)

---

#### Checkpoint 1 — Verify your Cloudflare account

**What AI does:** Runs `npx wrangler whoami` to display the currently authenticated Cloudflare account.

**What you verify:** The output should show the account that owns the domain you'll deploy to. For internal Anduin apps, that's the Anduin master account (since InfoSec-assigned root domains live there). If a different account is shown, tell AI to run `wrangler logout` then `wrangler login` — the login command opens a browser where you pick the right account. Your code and your domain must live on the same Cloudflare account. If they don't match, the custom domain won't attach.

> [!IMPORTANT]
> Internal company apps should never live on personal Cloudflare accounts. Organizationally this creates real problems: if you leave the company, the app goes with you; admins can't monitor or take over. Even if a deploy technically succeeds to a personal account, it's the wrong place for company work.

---

#### Checkpoint 2 — Install the Cloudflare adapter and update `.gitignore`

**What AI does:** 
- Installs the OpenNext Cloudflare adapter and CLI tools (`@opennextjs/cloudflare`, `wrangler`, `esbuild`).
- Adds three npm scripts to `package.json`:
  - `cf:build` — test the build
  - `cf:preview` — run the Worker locally on the production runtime — pre-deploy sanity check (Checkpoint 4)
  - `cf:deploy` — build and push to Cloudflare (Checkpoint 6)
- Updates `next.config.mjs` to set `images.unoptimized: true` — Workers can't run the Next.js image optimizer (which requires a Node.js runtime). Without this flag, any page using `<Image>` from `next/image` will throw a runtime error.
- Adds `.open-next/`, `.wrangler/`, and `dist/` to `.gitignore`. These are build artifacts (regenerate-able) and per-machine state — committing them bloats the repo and can leak credentials in `.wrangler/`.

**What you verify:**
- Confirm `.env.local` is already in `.gitignore` — if it isn't, **this is a blocker**. Stop and fix it before continuing, otherwise your next `git add .` will commit secrets to the repo.
- Run `npm run cf:build` and confirm it completes without errors. Output goes to `.open-next/`.
  
> [!IMPORTANT]
> If `npm run cf:build` runs before `.gitignore` is updated, artifacts (`.open-next/`, `.wrangler/`, `dist/`) will be sitting in the working tree. These folders are large and will bloat commits or cause merge conflicts if accidentally added. Worse, if `.env.local` had not already been gitignored, the same oversight would commit secrets and expose them in Git history forever — even if you delete them later, they remain in commit history and must be rotated. **Update `.gitignore` before the first build, not after.**

---

#### Checkpoint 3 — Write the deploy config in `wrangler.jsonc`

**What AI does:** Creates `wrangler.jsonc` in the project root with these required fields:
- `name` (your Worker name)
- `main` (entry point — typically `.open-next/worker.js`)
- `compatibility_date` (pins Workers runtime behavior to a specific date for consistency)
- `account_id`
- `routes` (with `custom_domain: true` for your chosen subdomain)
- `vars` (non-sensitive config only)

```jsonc
//Example skeleton//
   {
     "name": "your-app",
     "main": ".open-next/worker.js",
     "compatibility_date": "2026-05-28",
     "account_id": "<from wrangler whoami>",
     "routes": [
       { "pattern": "your-app.your-root.com", "custom_domain": true }
     ],
     "vars": {
       "NEXT_PUBLIC_APP_URL": "https://your-app.your-root.com"
     }
   }
```

**What you verify:**
- `account_id` matches the account shown by `wrangler whoami` from Checkpoint 1
- `routes[].pattern` is your intended subdomain (e.g. `your-app.your-assigned-root.com`), not the root domain and not `*.workers.dev`
- `custom_domain: true` is set on the route — this is what auto-provisions DNS
- `vars` contains only non-sensitive config (feature flags, public URLs)

> [!IMPORTANT]
> `wrangler.jsonc` is the source of truth for the deploy. AI may pull `account_id` from a stale env variable or guess it from a previous project — always cross-check against `wrangler whoami` output. Putting sensitive values in `vars` is a common mistake because `vars` and secrets look similar; `vars` are visible in the dashboard while secrets are encrypted at rest. If something feels like a credential, treat it as a secret.

---

#### Checkpoint 4 — Pre-deploy sanity check via `cf:preview`

**What AI does:** Runs `npm run cf:preview`. This builds the Worker and serves it on `localhost:8787` using the same Cloudflare Workers runtime that production uses.

**What you verify:**
- Open `localhost:8787` in your browser — the app should load without errors
- Click through 2-3 main flows (sign-in, fetch data, navigate between pages)
- Watch the terminal output — any runtime errors will print here immediately

If anything fails in `cf:preview`, it will fail in production too. Fix it now, before deploying.

> [!IMPORTANT]
> Don't confuse `npm run dev` with `npm run cf:preview` — they're not the same test.
>  - `npm run dev` runs on **Node.js** at `localhost:3000` (Next.js default)
>  - `npm run cf:preview` runs on the **Cloudflare Workers runtime** at `localhost:8787` 
>
> Code that works in `npm run dev` can still fail when deployed to Cloudflare — most commonly because of Node-only library imports (see Section 1, constraint #2) that the dev server tolerates but Workers doesn't. **`cf:preview` is the only local test that mirrors production.** Always run it before Checkpoint 5.

---

#### Checkpoint 5 — Pre-flight validation via dry-run

**What AI does:** Runs `npx wrangler deploy --dry-run --outdir dist`. This simulates the full deploy process — building the package, validating the config, checking account permissions, resolving routes and DNS — but stops short of actually pushing to Cloudflare. AI then runs `du -sh dist/` to show the final bundle size.

**What you verify:**
- Dry-run completes with **no errors or warnings**
- Output shows the correct account name and your custom domain
- No route conflicts reported (subdomain isn't already claimed by another Worker)
- Bundle size is within plan limits:
  - **Free plan:** 3 MiB ceiling — bundle larger than this requires upgrading to Paid
  - **Paid plan:** 10 MiB ceiling (Anduin is on Paid)
- For best cold-start performance, aim for under 2 MiB even on Paid

> [!IMPORTANT]
> Dry-run is a full pre-flight check, not just a bundle size test. It validates:
> - **Config correctness:** `wrangler.jsonc` syntax, required fields, value types
> - **Account permissions:** your account can actually deploy to this Worker
> - **Route conflicts:** the subdomain you chose isn't claimed elsewhere
> - **DNS readiness:** custom domain can be auto-provisioned
> - **Bundle size:** within plan limits
> 
> Skipping dry-run means *any* of these issues only surface during real deploy, where failures cost more — they clutter version history and risk leaving the Worker in an inconsistent state.

---

#### Checkpoint 6 — Real deploy and set secrets

**What AI does:**

- Runs `npm run cf:deploy` (which calls `wrangler deploy` under the hood). Your app becomes live at your custom subdomain.
- Pipes secrets from `.env.local` to Cloudflare in batch — using a shell loop that reads each `KEY=value` line, pipes the value through stdin to `npx wrangler secret put KEY`. All secrets upload in one command, **without prompts and without exposing values in the terminal or chat**.
- Runs `npx wrangler secret list` to confirm all secrets are set.
- **Disables the default `*.workers.dev` URLs** that Cloudflare auto-creates for every Worker. AI does this via wrangler API or by guiding you through dashboard toggles — but it may happen silently. **Always verify yourself (see below).**

**What you verify:**

- Deploy output ends with `Current Version ID: <some-id>` and your custom domain. Note the version ID — it's your rollback handle if something breaks later.
- After all secrets are set, `wrangler secret list` shows the same names as keys in your `.env.local`.
- Missing any secret will cause runtime crashes when the app tries to read them at startup.
- **Open Cloudflare dashboard → Workers & Pages → your Worker → Settings.** Confirm:
  - **Worker URL** shows `Inactive` (under "Domains & Routes")
  - **Preview URLs** show `Inactive`
  - **Custom Domain** shows `Production` with your subdomain
  If any of the `*.workers.dev` URLs are still Active, tell AI to disable them — do not deploy further until this is confirmed.

> [!IMPORTANT]
> Deploy and secrets are **2 separate operations**. Your app can deploy successfully but then crash on the first request because a secret is missing — and the dashboard won't tell you which one. Cross-checking `wrangler secret list` against `.env.local` catches this before users hit it.
> 
> The batch pipe approach is **safer** than typing secrets manually:
> - Values read directly from `.env.local` → no copy-paste into terminal or chat
> - AI sees only the file path, never the values
> - One command sets all secrets, no human in the loop for individual values
> 
> **If AI asks you to paste a secret value into chat** (instead of using the pipe), that's a setup bug — abort and have AI redo this checkpoint using the batch approach. Secrets pasted into chat are considered compromised and must be rotated.

> [!IMPORTANT]
> **The `*.workers.dev` URLs bypass your Zero Trust gate.** Cloudflare auto-creates two default URLs for every Worker on `*.workers.dev` — the Worker URL and Preview URLs. These are **not** behind Anduin Zero Trust — anyone on the Internet who discovers the URL can reach your app and any data it exposes, bypassing company SSO entirely.
>
> AI usually disables these as part of the deploy flow, but it may do so silently without telling you. **Always check the dashboard yourself after every deploy.** The custom domain you set up via `wrangler.jsonc` is the only URL that should remain active for internal apps.

---

### 2.3 Wire External Services

After Checkpoint 6, the app is deployed but external services still point at development URLs (or aren't configured at all for production). Three things commonly need wiring after every deploy:

#### 1. OAuth provider (Google, Microsoft, Auth0, etc.)

Add your production URL to the **Authorized redirect URIs** in the OAuth provider console. For Google OAuth, the callback URL typically looks like `https://your-app.your-assigned-root.com/api/auth/callback/google`. AI cannot do this step — provider consoles require manual UI clicks that AI can't see.

Until this is added, sign-in will fail with a redirect URI mismatch error.

#### 2. Database CORS / allowed origins

If your app calls Supabase Edge Functions, Firebase, or any database service with CORS protection, add your production URL to the allowed origins. For Supabase Edge Functions, ask AI to run:

```bash
supabase secrets set ALLOWED_ORIGIN=https://your-app.your-assigned-root.com
```

Without this, your app loads but every API call from the browser gets blocked by CORS and returns a network error.

#### 3. Webhooks and third-party callbacks

If your app receives webhooks (Stripe, GitHub, Slack, etc.) or sends callbacks to third-party services, update those services with your production URL. Each provider has its own console — AI cannot touch them. Make a list of every external service you wired during development and update each one.

> [!TIP]
> After wiring, sanity-check by signing in and clicking through the main flows in your browser. If anything fails, ask AI to run `wrangler tail` — this streams live logs from your Worker so you can see exactly what's happening on each request. Most post-deploy failures show up here within seconds.

---

## 3. Security Non-Negotiables

These rules apply regardless of what AI suggests, what the docs say, or how convenient a shortcut feels. Breaking any of them creates a real exposure that's expensive to clean up.

1. **Never paste secrets into AI chat.** Chat history is considered a compromise vector — even private chats, even "just to debug." Any secret that touches chat must be rotated.

2. **Never bypass the Zero Trust gate for convenience.** Not "just for 5 minutes of testing," not "only on my laptop." The gate is the only thing between your app and the public Internet. If you genuinely need an exception, contact InfoSec — don't disable it yourself.

3. **Never deploy a company app to a personal Cloudflare account.** Even if it technically works. Personal accounts mean the app leaves when you leave the company, admins can't monitor or take over, and billing ends up on personal cards.

4. **Always verify `*.workers.dev` URLs are disabled after every deploy.** AI usually handles this, but silently. Open the Cloudflare dashboard → your Worker → Settings, and confirm both Worker URL and Preview URLs show `Inactive`. Until they are, your app is reachable from the public Internet bypassing the Zero Trust gate.

5. **Sync secrets across all platforms when rotating.** If you rotate `SUPABASE_ANON_KEY` in Supabase but forget to update it in the Worker, the app breaks. If you rotate it in the Worker but forget Supabase, the old key still works elsewhere and isn't actually rotated. Rotation must happen everywhere the secret lives, in the same change.

---
