---
name: clep-instrument
description: Instrument a web app with data-clep attributes so Clep can record it. Use when the user asks to instrument, add data-clep, prepare a feature for clipping, or wire clip states and actions.
version: 0.1.0
---

# Clep instrument — mark features for clipping

No npm SDK to install (we are not publishing one yet). You add plain HTML
attributes. The platform's agent finds them with Playwright.

## The pattern

```html
<button data-clep="ai-research" data-clep-action="primary">Start Research</button>
<div data-clep="ai-research" data-clep-state="result">…</div>
```

React:

```tsx
<div data-clep="demo-convert" data-clep-state={phase}>
  <button data-clep-action="primary" onClick={run}>Convert</button>
  <input data-clep-action="input" placeholder="Search…" />
</div>
```

Rules:

1. `data-clep="feature-name"` (kebab-case) scopes the feature. All nodes sharing
   a name merge into one feature (union bbox).
2. `data-clep-action="primary"` marks the main button. `action:<name>` marks
   secondary buttons (e.g. `data-clep-action="export"`).
3. `data-clep-state="<state>"` marks states the camera waits for
   (`empty`, `generating`, `completed`, `exported`, …). Put it on the root AND
   any inner node that changes — the agent watches the whole subtree.
4. Keep one visible text input per feature when the flow needs typing
   (skip `type="file"` / `type="hidden"` — the agent ignores those for typing).
5. Don't restyle to accommodate Clep. Don't add wrappers that change layout.
   Attributes only.

## Workflow

1. Find the flow the user names (e.g. "research flow", "checkout", "dashboard
   convert"). Read the component files; locate the input + primary button +
   result states.
2. Add the three attributes above with minimal diffs. Reuse the existing
   state variable for `data-clep-state` (e.g. `phase`).
3. Verify locally with the bundled CLI (env: `CLEP_API_URL`, `CLEP_API_KEY`):

```bash
${CLAUDE_PLUGIN_ROOT}/bin/clep scan http://127.0.0.1:3000/
```

The feature name you added must appear. If it doesn't, the selector
`[data-clep="<name>"]` isn't rendering (conditional render, wrong route, auth
wall) — fix that before clipping.
4. Report: feature name, file:line of each attribute, states wired, and the
   scan result. Then offer `/clep:clip <name>` (see the `clep-clip` skill).

## Multi-step flows

If the flow chains clicks across states (type → click → wait → click export),
note the chain for the clip step. The clip plan format:

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
