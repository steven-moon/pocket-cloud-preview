# pocket-cloud-preview — Claude Guide

# Role: other

This repository is neither a PocketCloud native client nor a `pc-port-repos`
port. Hooks + Documentation scaffold still apply so the machine-wide `pocket`
CLI can sync, gate, and index it. Keep product-specific process in this repo's
ADRs; ecosystem standards stay referenced (or version-refreshed copies under
`Documentation/Standards/`).


<!-- pocketcloud:docs-index:start -->
## PocketCloud docs (authoritative)

- **[Documentation/Runbook/index.md](Documentation/Runbook/index.md)** — build, verify, git sync, hooks
- **[Documentation/Standards/](Documentation/Standards/)** — day-one copies of Swift 6.3 + Unified Providers Flywheel; query the rest via `pocket knowledge rag query --namespace docs.standards`
- **[Documentation/ADRs/](Documentation/ADRs/)** — decisions *about this repo*

```bash
pocket ops git sync --preview          # from this repo root (or pass an absolute path)
pocket ops git-hook status
pocket knowledge rag query --namespace docs.standards "<question>"
```

Role: **other** · Platform template: **other-non-code**
<!-- pocketcloud:docs-index:end -->
