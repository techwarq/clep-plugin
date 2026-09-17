---
name: clep
description: Turn a plain-English request ("make a clip of the signup flow", "record the AI research demo") into a rendered product video. Instruments data-clep attributes if the feature isn't marked up yet, then scans and renders via the hosted Clep backend. Use whenever the user asks for a demo clip, product video, or to record/clip/capture a feature — this is the only Clep command needed, don't ask the user to run a separate instrument step first.
version: 0.2.5
---

# Clep — one command, feature request to MP4

The user describes a flow in plain English. You figure out the rest: check
if it's already markup'd, instrument it if not, render it, report back where
the video landed. Never make the user invoke a separate step manually.

## Talk like a product, not a systems log

The user asked for a video, not a diagnosis. Never narrate internals in
chat: API URLs, config file paths, whether the backend is "hosted" vs
local, which CLI tools you checked for or ran `which`/`npm ls` against, or
raw error internals. If you catch yourself about to type "the config
points to X" or "let me check for Y", stop and say what's happening and
what you need from them instead — nothing else.

- Bad: "The API key points api_url at http://127.0.0.1:8787 (nothing's
  listening there) instead of the hosted... make sure you don't pass a
  custom --url this time."
- Good: fix it silently, or if you can't, ask once, plainly, for what you
  need.

This applies throughout every step below.

## 0. Read the ask

Extract from the user's request:
- **App URL** — if not given, ask, or infer from a dev server already running
  in this repo (e.g. `localhost:3000`).
- **Feature** — what flow/element (kebab-case a short name, e.g. "signup
  flow" → `signup`).
- **What to type/click** (optional) — becomes `--query` or a steps plan.
- **Look** (optional) — style/fps/quality; default `saas` / 60fps / 1080p.

Backend: `${CLAUDE_PLUGIN_ROOT}/bin/clep` talks to the hosted Clep backend by
default — no URL to configure, ever, unless the user is self-hosting (then
`CLEP_API_URL`/`clep configure --url` overrides it). The **App URL** you're
extracting in this step is a different thing entirely: it's the app *being
clipped* (e.g. their `localhost:3000` dev server), not the Clep backend.

**First run**: `${CLAUDE_PLUGIN_ROOT}/bin/clep` also reads
`~/.clep/config.json` (falls back to it when `CLEP_API_KEY` isn't set). On a
fresh install there's no key yet, so the *first* call (usually `scan`) 401s:
`error: unauthorized: no valid API key...`. You need real credentials, full
stop.

**Do not** offer to start a local backend, run `platform/server.py`, or
present alternatives — Clep is a hosted product; nobody using this plugin is
expected to run their own backend.

**Never ask for the key in chat, and never run `clep configure` yourself
with a key typed into a tool call** — either way the secret ends up sitting
in the conversation transcript. `clep configure` run with no flags is an
interactive prompt (a bordered box, paste-and-enter) built for exactly this,
but it only works as a live terminal prompt when a human is typing into it
directly — not when you invoke it as a tool call. So stop and tell the user,
in one short message:

> I need a Clep API key before I can render anything. Run this in your
> terminal (not here in chat):
>
> `${CLAUDE_PLUGIN_ROOT}/bin/clep configure`
>
> It'll prompt you for the key — grab one from your dashboard's API Keys
> page. Let me know once it's done and I'll pick up where I left off.

Then stop and wait — don't retry until they confirm. When they do, re-run
the call that originally failed; `clep configure` already persisted the
config to `~/.clep/config.json`, so nothing else needs redoing.

## 1. Check instrumentation — scan first, always

```bash
${CLAUDE_PLUGIN_ROOT}/bin/clep scan <url>
```

If the feature name (or something clearly matching what the user described)
is already in the output, skip straight to step 3 (render).

**If `<url>` is a local address** (`localhost`/`127.0.0.1`/a LAN IP): the
hosted backend runs elsewhere and can't reach it, so this will fail with an
unreachable/timeout error. That's expected, not a bug — it needs a public
URL. Never start a tunnel yourself (`ngrok`, `cloudflared`, etc.) — exposing
someone's local dev server to the internet is their call, not something to
do on their behalf. Just ask once, plainly:

> The app you want clipped is only reachable on your machine, and Clep's
> renderer needs a public URL. Start a tunnel (`ngrok http <port>`,
> `cloudflared tunnel --url <url>`, or similar) and give me the URL it
> prints — I'll take it from there.

Don't list tools you'd consider, commands you ran to check for them, or
describe "nothing's listening" — just state the ask.

## 2. Instrument, only if missing

No npm SDK — plain HTML attributes, found by the platform's Playwright agent.

```html
<button data-clep="ai-research" data-clep-action="primary">Start Research</button>
<div data-clep="ai-research" data-clep-state="result">…</div>
```

```tsx
<div data-clep="demo-convert" data-clep-state={phase}>
  <button data-clep-action="primary" onClick={run}>Convert</button>
  <input data-clep-action="input" placeholder="Search…" />
</div>
```

Rules:

1. `data-clep="feature-name"` (kebab-case) scopes the feature. All nodes
   sharing a name merge into one feature (union bbox).
2. `data-clep-action="primary"` marks the main button. `action:<name>` marks
   secondary buttons (e.g. `data-clep-action="export"`).
3. `data-clep-state="<state>"` marks states the camera waits for (`empty`,
   `generating`, `completed`, `exported`, …). Put it on the root AND any inner
   node that changes — the agent watches the whole subtree.
4. Keep one visible text input per feature when the flow needs typing (skip
   `type="file"` / `type="hidden"`).
5. Don't restyle to accommodate Clep. Attributes only, minimal diff.

Workflow: read the component files, locate input + primary button + result
states, reuse the existing state variable for `data-clep-state`. Then
re-run `clep scan <url>` to confirm the name now appears — if it doesn't, the
selector isn't rendering (conditional render, wrong route, auth wall) and
that has to be fixed before rendering.

**Multi-step flows** (type → click → wait → click export): build a steps plan
instead of relying on the auto arc.

```json
[
  { "action": "type", "target": "input", "value": "AI browser agents" },
  { "action": "click", "target": "primary" },
  { "action": "wait", "for": "state:completed", "timeout": 6 },
  { "action": "click", "target": "action:export" },
  { "action": "wait", "for": "state:exported", "timeout": 4 }
]
```

Targets resolve inside `[data-clep=name]`: `input`, `primary`,
`action:<name>`, `text:<label>`, `file` (upload only), or any CSS selector.

## 3. Render

```bash
${CLAUDE_PLUGIN_ROOT}/bin/clep clip \
  --url <url> --name <feature-name> \
  --query "<text to type, if any>" \
  --style saas --fps 60 --quality 1080p \
  --wait
```

Omit `--query`/`--steps-file` for the auto arc (type → click → wait-for-change).
Use `--steps-file plan.json` for the chained flow built in step 2. Add
`--out <path>` only if the user asked for a local file — otherwise the video
already lands in the platform UI and that's enough.

`--wait` polls until `done`/`error` and prints the MP4 URL. Job states:
`queued → recording → editing → done | error`. On `error`, surface the
backend message verbatim (usually: feature not found at URL, a `wait` state
timed out, or the URL is unreachable from the backend) and suggest the fix.

## 4. Report back

Tell the user, in one short message:
- The video is live in their Clep dashboard's Usage page — no action needed.
- If `--out` was used, the local file path too.
- Style/fps/quality used, so they know what to ask for differently next time.

Re-renders are one command — never re-record by hand. Output is a 2–5s 16:9
MP4, gradient backdrop + floating window + auto-zoom + custom cursor + click
ripple by default.
