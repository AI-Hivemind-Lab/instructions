# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This is an internal developer onboarding documentation project. It produces instructional guides for Anduin engineers covering three topics:

1. **Self-hosted Supabase** — how to spin up Supabase on internal infrastructure, following the setup defined in `https://github.com/anla-124/selfhosted-supabase.git`
2. **LiteLLM API keys** — how to obtain an API key from the company's internal LiteLLM deployment
3. **Dokploy deployment** — how to deploy an app internally via Dokploy, following the guide at `https://www.notion.so/anduin/Draft-Dokploy-User-Guide-2fa3f653b1df80248273e5de25ae1d6a`

## Content Conventions

- Guides are written for internal Anduin engineers, not the general public. Assume company SSO, VPN access, and internal tooling are available.
- Each guide should be self-contained: a reader with no prior context for that tool should be able to complete the task end-to-end.
- Source of truth for each guide lives in the upstream repos/Notion pages listed above. When content conflicts, defer to those sources.
- Prefer concrete commands and screenshots over abstract descriptions.
- Use Markdown. One guide = one `.md` file.

## Behavioral Guidelines

These apply when generating or editing any content in this repo:

**Think before writing.** State assumptions explicitly. If the upstream source is ambiguous, surface the ambiguity rather than guessing.

**Simplicity first.** Minimum steps that accomplish the goal. No optional detours, no "you might also want to..." tangents unless they prevent a common failure mode.

**Surgical edits.** When updating an existing guide, touch only what changed in the upstream source. Don't reformat or restructure sections that aren't broken.

**Goal-driven execution.** For multi-step guide work, define what "done" looks like before starting:
```
1. [Section] → verify: reader can complete X without additional context
2. [Section] → verify: commands are tested/accurate
```
