# clep-plugin — marketplace for the Clep Claude Code plugin

Video layer for software products. Claude instruments your app with `data-clep`
attributes, the hosted backend drives the live feature and edits the capture
into a 1080p60 product clip.

## Install

```
/plugin marketplace add techwarq/clep-plugin
/plugin install clep@clep-marketplace
```

## Setup

```bash
export CLEP_API_URL=https://your-backend.example.com  # default http://127.0.0.1:8787
export CLEP_API_KEY=clep_live_...                     # dashboard → API Keys
```

## Use

One command:

```
/clep:clep make a clip of the signup flow at http://localhost:3000, cinematic style
```

Claude checks if the feature is already marked up, instruments it (`data-clep`
attributes) if not, then renders and reports back where the MP4 landed.

See [`plugins/clep/README.md`](plugins/clep/README.md) for the full CLI + backend docs.
