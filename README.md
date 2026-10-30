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

The serverless functions are pinned to the Node.js 22 LTS line:

```json
{ "engines": { "node": "22.x" } }
```

**22.x, not 24.x, and that is deliberate.** `engines.node` overrides the Vercel
project setting, and Vercel resolves the range to the newest patch in its own
image for that major — which is not necessarily the newest patch Node has
released. Measured on 30 October 2026, in `iad1`, from the cold-start log below:

| `engines.node` | Vercel served | Node's patched release | Patched? |
| --- | --- | --- | --- |
| `24.x` | `v24.21.0` | `24.22.1` | no |
| `22.x` | `v22.23.4` | `22.23.4` | **yes** |

Vercel's advisory says all deployments created after 28 October 2026, 11am PT
use a patched Node. For the `24.x` image that was not true of a build made on
30 October. `22.x` is the line that actually carries the October 21 fixes, so
that is the line this project pins until the 24.x image catches up.

`"type": "module"` matters separately: the build log for the deploy before the
pin warned `Node.js functions are compiled from ESM to CommonJS` and transpiled
all three files. They now run as native ESM.

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

A deployment keeps the runtime it was built with, so changing the pin only takes
effect on a rebuild — reconfiguring alone does nothing.

Do not take the pin on trust. `_provider.js` logs `[nova] node <version>` once
per cold start; read it in the project's Logs tab, or via the runtime logs API,
to see the exact patch a running function is on. That log is the only thing that
caught the `24.x` image being a patch behind — the build log does not print the
version, the deployment file tree is not exposed, and the project setting only
echoes the range that was asked for. Re-read it after any runtime change.

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
