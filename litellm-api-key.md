# LiteLLM API Key Guide

## What is this?

**What is LiteLLM?**
LiteLLM is a proxy server that gives you a single API endpoint for multiple AI providers — OpenAI, Anthropic, Google Gemini, and others. Instead of managing separate keys and SDKs for each provider, you call one URL with one key and specify which model you want.

**Why does Anduin run its own LiteLLM instead of calling OpenAI/Anthropic directly?**
Cost control, usage visibility, and model flexibility. The company pays for access centrally, can see aggregate usage, set rate limits, and swap in new models without touching your code. You write `model="anthropic.claude-sonnet-4-6"` and the proxy handles the routing.

**What is an API key and why do I need one?**
The API key authenticates your requests to the proxy. Without it, the server rejects your call. Think of it as a password that tells the proxy "this request comes from an authorized Anduin engineer."

**Why a shared team key instead of individual keys?**
We use a shared key so access can be granted or revoked centrally. When a key needs rotating, one change covers everyone. You don't need to sign up for any provider account yourself.

**Which models can I use?**
See Section 2 for the full list. For most coding tasks, start with `anthropic.claude-sonnet-4-6`. For lightweight tasks, `anthropic.claude-haiku-4-5-20251001-v1:0` is faster. For the most capable reasoning, use `anthropic.claude-opus-4-6-v1`.

---

## Prerequisites

- Access to the Anduin 1Password vault (ask your manager if you don't have it)
- A terminal (macOS Terminal, iTerm2, or any shell)
- `curl` and `jq` installed (`brew install jq` if missing)
- Optional: an app project where you'll use the key

---

## Section 1 — Get the API key from 1Password

1. Open 1Password and sign in with your Anduin SSO credentials.

2. Search for **LiteLLM** in the search bar.

   [SCREENSHOT: 1Password search results showing "LiteLLM API Key" item]

3. Open the **LiteLLM API Key** item.

4. Click the copy icon next to the **API Key** field (the value starts with `sk-`).

   [SCREENSHOT: 1Password item detail view with the API Key field highlighted]

5. Keep the key somewhere you can paste it in the next section — don't close 1Password yet.

> **Keep this key private.** Don't commit it to git, paste it in Slack, or share it publicly. If you suspect it was exposed, notify the infra team immediately.

✅ **You're done when:** You have a string starting with `sk-` ready to paste.

## Section 2 — Available models

Use the exact model string shown below as your `model` parameter. The proxy routes it to the right provider automatically.

### Anthropic (via AWS Bedrock) — recommended for most tasks

| Model | Best for |
|---|---|
| `anthropic.claude-sonnet-4-6` | Default choice — strong reasoning, good speed |
| `anthropic.claude-opus-4-6-v1` | Hardest tasks, complex multi-step reasoning |
| `anthropic.claude-haiku-4-5-20251001-v1:0` | Fast, lightweight tasks |
| `anthropic.claude-sonnet-4-5-20250929-v1:0` | Previous Sonnet generation |
| `anthropic.claude-opus-4-5-20251101-v1:0` | Previous Opus generation |
| `us.anthropic.claude-sonnet-4-20250514-v1:0` | Claude Sonnet 4 (US cross-region) |
| `us.anthropic.claude-3-7-sonnet-20250219-v1:0` | Claude 3.7 Sonnet (US cross-region) |
| `anthropic.claude-3-7-sonnet-20250219-v1:0` | Claude 3.7 Sonnet |

Regional variants (`eu.*`, `apac.*`, `au.*`, `global.*`, `jp.*`) are available for latency-sensitive workloads — ask the infra team if you need them.

### Google Gemini

| Model | Notes |
|---|---|
| `gemini/gemini-2.5-pro` | Most capable Gemini |
| `gemini/gemini-2.5-flash` | Fast, general purpose |
| `gemini/gemini-2.5-flash-lite` | Fastest Gemini option |
| `gemini/gemini-2.0-flash` | Previous generation, stable |
| `gemini-2.5-pro-preview-05-06` | Preview alias |

### OpenAI

| Model | Notes |
|---|---|
| `openai/gpt-4o` | GPT-4o |
| `openai/gpt-4o-mini` | Lightweight GPT-4o |
| `openai/gpt-4.1` | GPT-4.1 |
| `openai/gpt-4.1-mini` | GPT-4.1 mini |
| `openai/gpt-4.1-nano` | GPT-4.1 nano |
| `openai/o3` | O3 reasoning model |
| `openai/o3-mini` | O3 mini |
| `openai/o4-mini` | O4 mini |
| `openai/gpt-image-1` | Image generation |
| `openai/dall-e-3` | DALL-E 3 image generation |
| `openai/tts-1` | Text-to-speech |
| `openai/whisper-1` | Speech-to-text |

### Other

| Model | Notes |
|---|---|
| `hosted_vllm/gpt-oss-120b` | Internal hosted open-source model |
| `openai/Kimi-K2.5` | Moonshot AI Kimi K2.5 |
| `bedrock/deepseek.v3.2` | DeepSeek V3.2 via Bedrock |
| `bedrock/moonshotai.kimi-k2.5` | Kimi K2.5 via Bedrock |

Many additional Bedrock models exist (Meta Llama, Mistral, Cohere, Amazon Nova). To see the full live list:

```bash
curl -s https://litellm.anduin.center/model/info \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  | jq '[.data[].model_name]'
```

✅ **You're done when:** You've picked the model you'll use for your project.

## Section 3 — Add the key to your `.env`

Open (or create) your project's `.env` file and add these three lines:

```env
LITELLM_BASE_URL=https://litellm.anduin.center/v1
LITELLM_API_KEY=sk-your-key-here
LITELLM_DEFAULT_MODEL=anthropic.claude-sonnet-4-6
```

Replace `sk-your-key-here` with the key you copied from 1Password.

> **Next.js projects:** Put the key in `.env.local` instead of `.env` — it takes precedence and is gitignored by default in Next.js projects.

Check that your `.env` (or `.env.local`) is gitignored:

```bash
grep -E "^\.env" .gitignore
```

If it's not listed, add it:

```bash
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
git add .gitignore
git commit -m "chore: gitignore .env files"
```

✅ **You're done when:** Running `grep LITELLM .env` (or `.env.local`) shows all three variables.

## Section 4 — Verify it works

Load your key into the shell and make a test call:

```bash
export LITELLM_API_KEY=sk-your-key-here

curl -s https://litellm.anduin.center/chat/completions \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "anthropic.claude-sonnet-4-6",
    "messages": [{"role": "user", "content": "Say hello in one word."}]
  }' | jq '.choices[0].message.content'
```

Expected output (exact word varies):

```
"Hello!"
```

**Using the OpenAI Python SDK:**

```python
from openai import OpenAI
import os

client = OpenAI(
    base_url=os.environ["LITELLM_BASE_URL"],
    api_key=os.environ["LITELLM_API_KEY"],
)

response = client.chat.completions.create(
    model=os.environ.get("LITELLM_DEFAULT_MODEL", "anthropic.claude-sonnet-4-6"),
    messages=[{"role": "user", "content": "Say hello in one word."}],
)
print(response.choices[0].message.content)
```

**Using the OpenAI TypeScript/Node.js SDK:**

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: process.env.LITELLM_BASE_URL,
  apiKey: process.env.LITELLM_API_KEY,
});

const response = await client.chat.completions.create({
  model: process.env.LITELLM_DEFAULT_MODEL ?? "anthropic.claude-sonnet-4-6",
  messages: [{ role: "user", content: "Say hello in one word." }],
});
console.log(response.choices[0].message.content);
```

✅ **You're done when:** The call returns a non-error JSON response with a message in `choices[0].message.content`.

## Troubleshooting

**`401 Unauthorized` or `Authentication Error`**
Your API key is wrong or missing. Check:
- The key starts with `sk-`
- No extra spaces or newlines in the value
- The `Authorization` header is exactly `Bearer <key>` (capital B, one space)

**`404 Not Found` on the model**
The model string doesn't match anything on this proxy. Copy it exactly from Section 2 — including slashes and version suffixes. Run the model list command at the bottom of Section 2 to see what's currently available.

**`Connection refused` or `Could not resolve host`**
You're not on the Anduin network. Connect to VPN and retry.

**The call hangs and never returns**
Large models (Opus) can take 30–60 seconds for the first token on a cold start. Wait up to 2 minutes. If it still hangs, check VPN, then ask in `#vibecoding` on Slack.

**`RateLimitError`**
The proxy is enforcing a usage limit. Wait 60 seconds and retry. If you're hitting limits repeatedly, ask the infra team to review your quota.
