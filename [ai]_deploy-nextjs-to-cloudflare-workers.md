# Deploy Next.js App to Cloudflare Workers (Anduin)

For human-readable explanation of every step (why, what to verify, common pitfalls), see [\[Human\] Cloudflare Workers Deployment Playbook](https://github.com/AI-Hivemind-Lab/instructions/blob/main/%5Bhuman%5D_cloudflare_workers_deployment_playbook.md).

## Scope and assumptions

- App is Next.js (any version supported by `@opennextjs/cloudflare`)
- Target is the Anduin master Cloudflare account
- Custom domain is a subdomain under an InfoSec-assigned root (e.g. `*.fin.anduin.center`)
- All database access goes over HTTPS (Supabase REST, Supabase Edge Functions, Firebase) — never raw `postgresql://` connections
- Secrets live in `.env.local` and are never pasted into chat

## Pre-flight checks

Run these in order. Stop and follow recovery if any fails.

### 1. Project structure

Confirm project root contains `next.config.{js,mjs,ts}` and `package.json`.

- If `next.config.*` missing → ABORT. Tell user: "This skill only deploys Next.js apps. For other frameworks, deployment differs."

### 2. Wrangler tooling

Run `npx wrangler --version`.

- If not found or errors → run `npm install -D wrangler @opennextjs/cloudflare esbuild`. Re-run version check. If still fails, ABORT with the error message.

### 3. Cloudflare account

Run `npx wrangler whoami`.

- If output shows a non-Anduin master account → STOP. Tell user: "Currently logged into [account]. Internal Anduin apps deploy to the Anduin master account. Run `wrangler logout && wrangler login`, then pick the Anduin master account in the browser." Wait for user confirmation before continuing.
- If `wrangler whoami` errors with not-logged-in → run `wrangler login`. Wait for browser auth to complete.

### 4. Dependency audit

Read `package.json`. Sort findings into THREE buckets — they are not all the same severity:

**Bucket A — Native / Node-only libs (hard blockers).** These rely on native binaries or filesystem APIs Workers doesn't have:

- `sharp`, `puppeteer`, `playwright`, `bcrypt`, `canvas`, `pdf-parse`

**Bucket B — ORMs (investigate, do NOT auto-block).** These CAN run on Workers when paired with an HTTP/serverless driver, so their mere presence is not a blocker:

- `prisma`, `drizzle-orm`

**Bucket C — Raw TCP database drivers (hard blockers).** These open raw TCP sockets to a DB, which Workers can't do:

- `pg`, `postgres`, `mysql2`, `mongodb`

Also grep source files for `postgresql://`, `postgres://`, `mysql://` connection strings.

- If a **Bucket A** lib found → STOP. Tell user: "Found [lib]. This needs a native runtime and won't run on Workers. Options: (a) replace with a Workers-compatible alternative, (b) move that work to a Supabase Edge Function, (c) deploy to Dokploy instead. Which do you want?" Wait for decision.
- If a **Bucket B** ORM found → do NOT stop. Ask: "Found [orm]. This works on Workers only with an HTTP/serverless driver (e.g. Supabase REST, Neon serverless, Prisma Accelerate, D1) — not with a raw `pg`/`mysql2` driver. Which driver does this app use?" Continue only once the driver is confirmed HTTP-based. If it turns out to use a raw TCP driver or `postgresql://` string, treat as Bucket C.
- If a **Bucket C** driver OR a `postgresql://`/`mysql://` connection string found → STOP. Tell user: "Found a direct database connection. Workers doesn't support raw TCP to databases. Migrate to HTTPS-based access (Supabase REST API / Edge Functions, Firebase) or deploy to Dokploy instead."

### 5. Secrets readiness

Check `.env.local` exists in project root.

- If missing → STOP. Tell user: "No `.env.local` found. Create it with all API keys, OAuth IDs, and database credentials this app needs. Each line should be `KEY=value`. Tell me when ready."

Check `.env.local` is in `.gitignore`.

- If not → add `.env.local` to `.gitignore` automatically. No need to ask user.

### 6. Manual prerequisites

Ask user to confirm both:

- "Do you have a root domain assigned by InfoSec for this app (e.g. `your-app.fin.anduin.center`)? If yes, what subdomain do you want to use?"
- "Is Cloudflare One Client enrolled in the Anduin Zero Trust team on your machine? (formerly WARP — consumer mode won't work.)"

If user says no to either → tell them to resolve (contact InfoSec for domain; see Notion VPN setup link for client) and ABORT.

## Deploy steps

> Numbering note: the human-facing playbook groups the deploy into **6 Checkpoints**. When narrating progress to the user, announce the matching checkpoint name so they can follow along in their doc:
>
> | This file | Human playbook checkpoint |
> |---|---|
> | Pre-flight Step 3 (Cloudflare account) | Checkpoint 1 — Verify your Cloudflare account |
> | Step 1 (install adapter + configs) | Checkpoint 2 — Install adapter & update `.gitignore` |
> | Step 2 (`wrangler.jsonc`) | Checkpoint 3 — Write the deploy config |
> | Step 3 (local preview) | Checkpoint 4 — Pre-deploy sanity check (`cf:preview`) |
> | Step 4 (dry-run) | Checkpoint 5 — Pre-flight validation (dry-run) |
> | Step 5 + Step 6 (deploy, secrets, verify URLs) | Checkpoint 6 — Real deploy & set secrets |

### Step 1: Install adapter and update configs

Run:

```bash
npm install -D @opennextjs/cloudflare wrangler esbuild
```

Add scripts to `package.json`:

```json
{
  "scripts": {
    "cf:build": "opennextjs-cloudflare build",
    "cf:preview": "opennextjs-cloudflare build && wrangler dev",
    "cf:deploy": "opennextjs-cloudflare build && wrangler deploy"
  }
}
```

Update `next.config.{js,mjs,ts}` to set `images.unoptimized: true`.

Add to `.gitignore`:

```
.open-next/
.wrangler/
dist/
```

Verify by running `npm run cf:build`. If errors, paste them and ABORT — likely a Node-only lib slipped past Step 4.

### Step 2: Write `wrangler.jsonc`

Create `wrangler.jsonc` at project root:

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "<app-name>",
  "main": ".open-next/worker.js",
  "compatibility_date": "<today YYYY-MM-DD>",
  "compatibility_flags": ["nodejs_compat", "global_fetch_strictly_public"],
  "account_id": "<from wrangler whoami>",
  "assets": { "directory": ".open-next/assets", "binding": "ASSETS" },
  "workers_dev": false,
  "preview_urls": false,
  "routes": [
    { "pattern": "<user-confirmed-subdomain>", "custom_domain": true }
  ],
  "vars": {},
  "observability": { "enabled": true }
}
```

Field meanings:
- `<app-name>`: derive from `package.json` `name` field
- `compatibility_date`: today's date. MUST be `2024-09-23` or later or `@opennextjs/cloudflare` will fail at runtime. (Some runtime APIs like `FinalizationRegistry` need `2025-05-05` or later — today's date is always safe.)
- `compatibility_flags`: **REQUIRED.** `nodejs_compat` enables the Node.js APIs that the OpenNext adapter depends on — without it the Worker errors at runtime. `global_fetch_strictly_public` makes the app's outbound `fetch()` calls go to the public Internet instead of being intercepted by same-zone routing; recommended for proxy-style apps that call external APIs (Supabase, AI providers). Drop `global_fetch_strictly_public` only if the app makes no outbound HTTPS calls.
- `<account_id>`: extract from `wrangler whoami` output
- `assets`: **REQUIRED.** Tells Workers where OpenNext put the static frontend files (`.open-next/assets`) and binds them as `ASSETS`. Without this, CSS/JS/images won't be served and the app renders blank.
- `workers_dev: false`: disables the auto-generated `*.workers.dev` URL (bypasses Zero Trust gate if left enabled)
- `preview_urls: false`: disables Preview URLs on `*.workers.dev` (same reason)
- `<user-confirmed-subdomain>`: from pre-flight Step 6
- `vars`: leave empty unless user explicitly provides non-sensitive **server-side** config (see note below)
- `observability: { enabled: true }`: turns on Worker logs/metrics from day one. Cheap and worth it.

NOTE on `vars`: only server-side runtime config belongs here (e.g. `NEXTAUTH_URL`, `AUTH_TRUST_HOST`, feature flags). Do NOT put `NEXT_PUBLIC_*` variables here expecting them to reach the browser — Next.js inlines `NEXT_PUBLIC_*` at **build time**, so they must be present when `opennextjs-cloudflare build` runs, not added afterward via `vars`.

DO NOT put any secrets in `vars`. Secrets go through Step 5.

> Reference: this is the same config shape as real deployed Anduin Workers (e.g. HiveFin uses exactly these `compatibility_flags`, `assets` binding, and `observability` block — plus `NEXTAUTH_URL` / `AUTH_TRUST_HOST` in `vars`).

### Step 3: Local preview

Run `npm run cf:preview`.

Tell user: "App is running locally on `localhost:8787` using the Cloudflare Workers runtime. Open it in your browser, click through main flows (sign-in, fetch data, navigate). Tell me when it works, or paste any errors."

Wait for user confirmation. If errors:

- Paste the error
- Diagnose: most common causes are Node-only imports slipping through, missing env vars, runtime API mismatches
- Fix and re-run preview
- Do NOT proceed to Step 4 until preview is clean

### Step 4: Dry-run validation

Run:

```bash
npx wrangler deploy --dry-run --outdir dist
```

Read the size line wrangler prints, which looks like:

```
Total Upload: 8570.03 KiB / gzip: 1617.01 KiB
```

The **`gzip:` figure is the one that counts** — Cloudflare enforces Worker size limits on the compressed size, not the uncompressed bundle. Do NOT use `du -sh dist/`: that measures the uncompressed bundle and will overstate the size 2–4×, causing false ABORTs.

Check output for:

- Errors or warnings → ABORT, paste output, diagnose
- Account name matches Anduin master
- Custom domain in output matches user-confirmed subdomain
- Route conflicts → ABORT, the subdomain is taken
- Bundle size (the **gzip** number):
  - Under 3 MiB → OK on both Free and Paid
  - 3 MiB to 10 MiB → OK on Paid (Anduin is on Paid)
  - Over 10 MiB → ABORT, tell user to audit dependencies. Common culprit: beta/RC versions (e.g. `next-auth@5.0.0-beta.*`) — suggest downgrade to stable.

### Step 5: Real deploy and secrets

Run `npm run cf:deploy`. Expect output ending with `Current Version ID: <id>`. Note the version ID.

Then pipe secrets from `.env.local` to Cloudflare in batch:

```bash
while IFS='=' read -r key value || [[ -n "$key" ]]; do
  [[ -z "$key" || "$key" =~ ^[[:space:]]*# ]] && continue
  value="${value%$'\r'}"                      # strip trailing CR (Windows/CRLF files)
  value="${value#\"}"; value="${value%\"}"    # strip one layer of surrounding double quotes
  printf '%s' "$value" | npx wrangler secret put "$key"
done < .env.local
```

This reads each `KEY=value` line and pipes the value via stdin. AI never sees the values. DO NOT prompt the user to paste individual secret values into chat.

Why the extra lines (they prevent silent secret corruption):
- `|| [[ -n "$key" ]]` — also processes the **last line** even if the file has no trailing newline (otherwise the last secret is silently skipped).
- `${value%$'\r'}` — strips the carriage return that Windows/CRLF files add, which would otherwise be uploaded as a hidden character inside the secret and cause auth failures that are very hard to debug.
- quote-stripping — handles `.env.local` lines written as `KEY="value"`, which would otherwise upload the literal quotes.

CAVEAT — multi-line secrets: this loop is line-based, so it cannot handle values that span multiple lines (e.g. a PEM private key like `FIREBASE_PRIVATE_KEY`). If `.env.local` contains a multi-line value, set that one manually with `npx wrangler secret put <KEY>` (which prompts for the value) and skip it in the loop, OR ensure it's stored on a single line with literal `\n` escapes.

Verify by running `npx wrangler secret list`. Confirm count and names match keys in `.env.local`.

### Step 6: Verify workers.dev URLs are disabled

The `workers_dev: false` and `preview_urls: false` fields you set in Step 2 should have disabled the public `*.workers.dev` URLs during deploy. Verify they took effect.

Tell user: "Open Cloudflare dashboard → Workers & Pages → [worker name] → Settings → Domains & Routes. Confirm:
- **Worker URL** shows `Inactive`
- **Preview URLs** show `Inactive`
- **Custom Domain** shows `Production` with your subdomain

Tell me when confirmed."

Wait for user confirmation. If any `*.workers.dev` URL is still Active:

1. Re-check `wrangler.jsonc` — confirm `workers_dev: false` and `preview_urls: false` are present
2. Redeploy: `npm run cf:deploy`
3. Verify again
4. If still Active after redeploy → tell user to toggle off manually in dashboard. ABORT the skill until confirmed.

Until both URLs show Inactive, the app is reachable from the public Internet bypassing the Anduin Zero Trust gate.

## Post-deploy manual steps (notify user)

Generate a checklist for the user. AI cannot do these — they require manual access to third-party consoles:

```
Production URL: https://<user-confirmed-subdomain>

External services to wire:

1. OAuth provider (Google/Microsoft/etc.)
   - Add to Authorized redirect URIs:
     https://<user-confirmed-subdomain>/api/auth/callback/<provider>

2. Database CORS (if using Supabase Edge Functions or similar)
   - Run: supabase secrets set ALLOWED_ORIGIN=https://<user-confirmed-subdomain>

3. Webhooks and third-party callbacks
   - List every external service this app integrates with
   - Update each one with the production URL
```

Tell user to wire each, then sanity-check by signing in and clicking through main flows. If anything fails, suggest `npx wrangler tail` to see live Worker logs.

## Recovery patterns

### "Wrong account" mid-deploy

If at any point Cloudflare returns "account mismatch" or "permission denied" → STOP. Re-run pre-flight Step 3. Don't try to force-deploy.

### "Bundle too large" at dry-run

Most common cause: beta/RC dependencies bundling unreleased code. Audit `package.json` for `beta`, `rc`, `canary`, `alpha` versions. Suggest downgrade to latest stable.

Other causes: large client libraries (Material UI full bundle, full lodash import). Suggest tree-shaking or selective imports.

### Deploy succeeds but app crashes on first request

Most common cause: missing secret. Run `npx wrangler secret list` and compare with `.env.local` keys. Set any missing ones with the same batch pipe from Step 5.

Second most common cause: OAuth redirect URI not added in provider console. Check post-deploy manual steps.

### `wrangler tail` shows constant errors

Paste a sample error. Common patterns:

- `cannot find module 'X'` → Node-only lib slipped past audit. Replace or migrate to Dokploy.
- `CORS error` → Supabase ALLOWED_ORIGIN not set or wrong.
- `401 / 403` from app's own API → secret missing or wrong value.

### Workers.dev URL still Active after deploy

Re-check `wrangler.jsonc` has both `workers_dev: false` and `preview_urls: false`. Redeploy. If problem persists, fall back to manual disable in dashboard and report to user.
