---
name: clep-clip
description: Scan data-clep features and render MP4 product clips via the hosted Clep backend. Use when the user asks to scan features, make a clip, render a video, or check clip jobs.
version: 0.1.0
---

# Clep clip — features to MP4

The backend (`pipeline_clep`: Playwright agent + polish + ffmpeg) is hosted by
us. You never run it locally — you call it through the bundled CLI, which
talks REST with `CLEP_API_URL` + `CLEP_API_KEY` (the user takes the key from
the platform dashboard → API Keys).

## 0. Preconditions

- App is instrumented (`data-clep` attributes — see `clep-instrument` skill).
- App URL is reachable from the backend. `http://localhost` / `127.0.0.1`
  means the *backend's* localhost, not the user's — for local dev the backend
  must run on the same machine, otherwise use a public URL (preview deploy,
  ngrok, etc.).
- If `CLEP_API_KEY` is missing, stop and tell the user where to get it
  (dashboard → API Keys → `export CLEP_API_KEY=clep_live_…`). Don't invent keys.

## 1. Scan (list features)

```bash
${CLAUDE_PLUGIN_ROOT}/bin/clep scan https://your-app.com/dashboard
```

Output is the Feature Registry: name, title, has_input/has_button,
button_label, states. If the expected feature is missing, go back to
instrumentation — don't guess names.

## 2. Make Clip

```bash
${CLAUDE_PLUGIN_ROOT}/bin/clep clip \
  --url https://your-app.com/dashboard \
  --name ai-research \
  --query "AI browser agents" \
  --style cinematic --fps 60 --quality 1080p --wait
```

Flags: `--query` (text the agent types; default comes from SDK/placeholder),
`--style {minimal,saas,cinematic,apple}`, `--fps {30,60}`,
`--quality {720p,1080p}`, `--steps-file plan.json` (multi-step chain),
`--wait` (poll until done/error, then print the MP4 URL).

Omit `--steps-file` for the auto arc (type → click → wait-for-change). Pass it
for chained flows (see `clep-instrument` for the plan format).

## 3. Jobs

```bash
${CLAUDE_PLUGIN_ROOT}/bin/clep jobs            # newest first
${CLAUDE_PLUGIN_ROOT}/bin/clep job job-000123 --wait --out ./ai-research.mp4
```

Job states: `queued → recording → editing → done | error`. On `done`, `out`
is the MP4 path/URL — hand it to the user with style/fps/quality. On `error`,
surface the backend message verbatim and suggest the fix (usually: feature not
found at URL, state `wait` timed out, or URL unreachable from backend).

## Output contract

- 2–5s 16:9 MP4, 1080p60 default, gradient backdrop + floating window +
  auto-zoom + custom cursor + click ripple.
- `trace.json` (cursor, clicks, bbox per step) is kept server-side per job.
- Re-renders are one command — never re-record by hand.
