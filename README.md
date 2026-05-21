# Instructions

Onboarding guides for Hivemind learners. Each guide takes one task end-to-end — from "I have no idea what this is" to "it's running."

## Available guides

| # | Guide | What you'll do | Status |
|---|---|---|---|
| 1 | [LiteLLM API key](litellm-api-key.md) | Get a shared LiteLLM key from 1Password and make your first call against `litellm.anduin.center` | ✅ Available |
| 2 | [Dokploy deployment](dokploy-deployment.md) | Deploy an app to a Dokploy instance — Dockerfile to live URL | ✅ Available |
| 3 | [Self-hosted Supabase ↗](https://github.com/AI-Hivemind-Lab/selfhosted-supabase) | Spin up a Supabase stack on Dokploy | ✅ Available (separate repo) |
| 4 | [Self-hosted Qdrant ↗](https://github.com/AI-Hivemind-Lab/selfhosted-qdrant) | Spin up a Qdrant vector database on Dokploy | ✅ Available (separate repo) |

## Who this is for

Hivemind learners, especially those new to vibecoding or deployment. Each guide assumes you have company SSO / VPN access and can reach the 1Password vault.

## Guide structure

Every guide follows the same shape:

1. **What is this?** — Q&A explaining the tool and why we use it
2. **Prerequisites** — what you need before starting
3. **Numbered steps** — concrete actions, each ending with a ✅ marker
4. **Troubleshooting** — the top failure modes and fixes

## Contributing

Edit the relevant `.md` file and open a PR. For guides where Notion is the source of truth (currently Dokploy), update Notion first, then sync changes here.
