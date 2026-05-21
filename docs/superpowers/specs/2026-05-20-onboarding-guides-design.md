# Onboarding Guides Design Spec

**Date:** 2026-05-20
**Status:** Approved

## Overview

Three standalone developer onboarding guides for Anduin engineers. Each guide is a self-contained `.md` file. Audience is wide — assume no prior experience with Docker, Supabase, or deployment platforms. Lots of hand-holding.

---

## Shared Structure (all three guides)

Every guide follows this template:

```
# [Tool] Guide

## What is this? (Q&A)
4–6 Q&A pairs, plain English, explains the "why" before the "how"

## Prerequisites
What you need before starting (software, access, accounts)

## Section N — [Task name]
Steps with commands + screenshot placeholders where UI interaction is required
✅ You're done when: [specific verifiable outcome]

## Troubleshooting
Top 3–5 common failure points with fixes
```

**Formatting rules:**
- Option B structure: named task sections, each ending with a ✅ completion marker
- Screenshots with callouts where UI steps are involved — placeholder noted as `[SCREENSHOT: description]` until captured from a live instance
- Code blocks for every command — no inline commands
- One guide = one `.md` file

---

## Guide 1 — `supabase-selfhosted.md`

**Source of truth:** `https://github.com/anla-124/selfhosted-supabase.git`

**Q&A topics:**
- What is Supabase?
- What is self-hosting?
- Supabase has a free tier — why self-host?
- What does self-hosted mean practically for me as a developer?
- Who manages the server?

**Prerequisites:** Git, Node.js (for key generation), terminal access

**Sections:**
1. Clone the repo and configure your `.env`
2. Generate secrets
3. Generate ANON_KEY and SERVICE_ROLE_KEY
4. Set your URLs
5. Deploy via Dokploy
6. Run database setup in Studio
7. Connect your app

**✅ Final end state:**
```bash
curl https://supabase.yourdomain.com/auth/v1/health
# returns: {"version":"...","name":"GoTrue",...}
```

---

## Guide 2 — `litellm-api-key.md`

**Source of truth:** Internal — key stored in 1Password, models confirmed live at `https://litellm.anduin.center/`

**Q&A topics:**
- What is LiteLLM?
- Why does Anduin run its own LiteLLM instead of calling OpenAI/Anthropic directly?
- What is an API key and why do I need one?
- Why a shared team key instead of individual keys?
- Which models can I use?

**Prerequisites:** Access to Anduin 1Password

**Sections:**
1. Get the API key from 1Password
2. Available models (hardcoded list grouped by provider: Anthropic via Bedrock, Gemini, OpenAI, Other)
3. Add the key to your `.env`
4. Verify it works

**Available models to list:**

*Anthropic (via Bedrock)*
- anthropic.claude-sonnet-4-6
- anthropic.claude-opus-4-6-v1
- anthropic.claude-sonnet-4-5-20250929-v1:0
- anthropic.claude-haiku-4-5-20251001-v1:0
- anthropic.claude-opus-4-5-20251101-v1:0
- us.anthropic.claude-sonnet-4-20250514-v1:0
- us.anthropic.claude-3-7-sonnet-20250219-v1:0
- anthropic.claude-3-7-sonnet-20250219-v1:0
- Plus regional variants (eu.*, apac.*, au.*, global.*, jp.*)

*Gemini*
- gemini-2.5-pro, gemini-2.5-flash, gemini-2.5-flash-lite
- gemini-3-pro-preview, gemini-3.1-flash-*, gemini-3.1-pro-*
- gemini-2.0-flash, gemini-2.0-flash-lite
- imagen-4.0, veo-3.1, lyria-3

*OpenAI*
- gpt-4o, gpt-4o-mini, gpt-4.1, gpt-4.1-mini, gpt-4.1-nano
- gpt-5, gpt-5-mini, gpt-5-pro, gpt-5-nano
- o3, o3-mini, o3-pro, o4-mini
- gpt-image-1, dall-e-3, tts-1, whisper-1

*Other*
- hosted_vllm/gpt-oss-120b (internal hosted model)
- openai/Kimi-K2.5
- bedrock/deepseek.v3.2, bedrock/moonshotai.kimi-k2.5
- Meta Llama, Mistral, Cohere, Amazon Nova, and other Bedrock models

**✅ Final end state:** A test API call to `https://litellm.anduin.center/` returns a valid model response

---

## Guide 3 — `dokploy-deployment.md`

**Source of truth:** `https://www.notion.so/anduin/Draft-Dokploy-User-Guide-2fa3f653b1df80248273e5de25ae1d6a`

**Q&A topics:**
- What is Dokploy?
- Why Dokploy and not Vercel, Render, or Railway?
- Who manages the Dokploy server?
- What is a Compose service?
- Do I need Docker on my laptop?

**Prerequisites:** App code in a Git repo (GitHub/GitLab), Dockerfile or Nixpacks-compatible project structure, environment variables ready

**Sections:** *(detail to be confirmed after reading Notion source)*
1. Prepare your app (Dockerfile + `.env.example`)
2. Create a service in Dokploy
3. Set environment variables
4. Add a domain
5. Deploy and watch logs
6. Connect to internal Supabase (if app uses self-hosted Supabase)

**✅ Final end state:** App is live at the assigned domain and returning responses

---

## Out of Scope

- Setting up the Dokploy server itself (managed by infra team)
- Setting up LiteLLM (already deployed at `litellm.anduin.center`)
- Database schema design or app-specific SQL
