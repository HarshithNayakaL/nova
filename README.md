# Nova — Model Router

A chat app that picks the model for you. Every turn is scored for difficulty and
routed to the cheapest model that can answer it well.

## Lanes

Three models, GPT OSS and Qwen only:

| Lane | Model | Handles | USD / 1M tokens |
| --- | --- | --- | --- |
| **L1 Swift** | `gpt-oss-20b` | Lookups, rewrites, translation, short answers | 0.10 in · 0.50 out |
| **L2 Broad** | `qwen3.6-27b` | Drafting, explanation, long context, everyday work | 0.32 in · 3.20 out |
| **L3 Deep** | `gpt-oss-120b` | Multi-step reasoning, math, debugging, code review | 0.15 in · 0.75 out |

Prices are approximate and move with the upstream provider.

`llama-3.3-70b-versatile` was decommissioned on GroqCloud on 2026-08-16 and is
not in the roster. Neither are the Mistral, Zephyr and Phi entries the app used
to list.

## How routing works

Each turn goes through `/api/route` before it is answered:

1. **Signal scoring** — deterministic and instant. Four axes, each 0–100:
   `reasoning`, `code`, `breadth`, `context`. Keyword evidence uses diminishing
   returns, so one strong hit already counts and the fifth adds little.
2. **Arbiter** — `gpt-oss-20b` reads the turn plus the scores and returns a lane
   with a one-line rationale, in JSON, capped at 120 tokens. It can override the
   signal reading.

L2 is the default. A turn has to earn its way *down* to L1 by being
demonstrably trivial, or *up* to L3 by being demonstrably hard.

If the arbiter is slow (4s budget), unreachable, or returns something
unparseable, the signal score stands. If `/api/route` itself fails, the browser
falls back to its own copy of the scorer. Routing never blocks an answer.

Two rules sit on top of the model's judgement:

- A turn that pushes back on the previous answer ("that's wrong", "go deeper")
  never routes below L2.
- **Escalate to L3** on any answer re-runs the turn on `gpt-oss-120b`.

Every answer is stamped with the lane, the complexity reading, the rationale,
and what it cost. Pin a lane with the `AUTO · L1 · L2 · L3` switch to bypass
routing entirely.

## Features

Threads with search, pinning, rename, and Markdown / JSON export · streaming
with stop · retry and escalate · edit and resend · automatic thread naming ·
system prompt with presets · temperature and response limit · command palette
(`Ctrl K`) · light and dark · keyboard shortcuts · full mobile layout.

Threads are stored in `localStorage` only — nothing is persisted server-side.

## Runtime and security

The serverless functions are pinned to the Node.js 24 LTS line:

```json
{ "engines": { "node": "24.x" } }
```

`engines.node` overrides the Vercel project setting, and Vercel resolves `24.x`
to the newest patch it supports in that major.

The project setting already read `24.x` before this pin, and there is no record
of it ever being changed, so the pin did not move the runtime off a bad version.
What it does is make the version a reviewable fact in the repo rather than a
dashboard setting nobody reads, so a stray change or a default shift cannot move
it silently. `"type": "module"` is the part with an immediate effect: the build
log for the previous deploy warned `Node.js functions are compiled from ESM to
CommonJS` and transpiled all three files; they now run as native ESM.

The pin matters because of the Node.js security releases of 21 October 2026
(`22.23.4`, `24.22.1`, `26.12.1`), which fixed four High-severity issues. Two of
them reach the only outbound call this app makes — `callUpstream()` in
`api/_provider.js`, which POSTs to Groq or Hugging Face over TLS with the API key
as a bearer token:

- **CVE-2026-111497** — out-of-bounds write in the Brotli decoder, exploitable
  for remote code execution. `fetch` does not advertise `br`, but undici decodes
  a response body according to the *response's* `Content-Encoding`, so a
  `Content-Encoding: br` reply is decoded whether or not we asked for it. No
  request header avoids this; only a patched runtime does.
- **CVE-2026-105481** — a keep-alive socket accepting a response written before
  its request was sent. The upstream connection is keep-alive by default.

The two TLS hostname bypasses, **CVE-2026-106069** and **CVE-2026-105483**, do
not apply. Both need a connection to a host written as a non-canonical IP
literal — a leading-zero octet like `0127.0.0.1`, read as octal by WHATWG URL
rules but as decimal by some OS resolvers, or an IP written with fullwidth
digits. `GROQ_URL` and `HF_URL` are module constants naming `api.groq.com` and
`router.huggingface.co`; `resolveUpstream()` returns one of the two and
`callUpstream()` uses nothing else, so no request body can steer the upstream
host. There is no path here that connects to an IP literal at all.

Also not applicable: the app has no npm dependencies, so there is no
supply-chain surface, and nothing in `api/` calls `dns.resolve`,
`crypto.randomUUID`, `crypto.randomInt`, the zip reader, or the permission
model, which is where the remaining CVEs land.

Vercel deployed WAF rules on 21 October 2026 that block inbound exploitation of
the Brotli issue for every project. They do not help here: Vercel's own advisory
states the WAF "does not cover responses your functions fetch from other
servers", which is precisely the path `callUpstream()` takes. Patching the
runtime is the only mitigation that reaches it.

Vercel manages patch versions for the `22.x` and `24.x` lines. Their advisory
states that **as of 28 October 2026, 11am PT, all new deployments use a patched
Node.js**, and that the rollout across existing managed runtimes is still
completing. A deployment built before that cut-off therefore keeps running the
Node it was built with: being pinned to `24.x` is what makes a redeploy land on
the patch, it is not a patch by itself. Any deployment older than the cut-off
needs rebuilding, not just reconfiguring.

`_provider.js` logs `[nova] node <version>` once per cold start; read it in the
project's Logs tab to see the exact patch a running function is on.

## Deployment

Set **one** of these environment variables in your Vercel project. Groq wins
when both are present.

| Variable | Upstream |
| --- | --- |
| `GROQ_API_KEY` | `api.groq.com` |
| `HF_TOKEN` | `router.huggingface.co` |

Then import the repo at [vercel.com](https://vercel.com) and deploy. The key
never reaches the browser; `/api/chat` and `/api/route` proxy every call.

## Local development

```
npx vercel dev
```

Needs the same environment variable in a `.env` file. There is no build step. `package.json`
carries no dependencies; it is there to declare the module type and pin the
Node version.

## Stack

Vanilla HTML/CSS/JS, no framework · Vercel serverless functions · marked +
highlight.js from CDN (the app degrades to a built-in renderer if either fails
to load).
