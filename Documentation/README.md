---
id: "doc.pocket-cloud-preview.readme"
title: "pocket-cloud-preview Documentation"
types: ["overview"]
tags: ["workspace", "pocketcloud"]
owners: []
version: "1.0.0"
last_reviewed: "2026-08-04"
---

# pocket-cloud-preview — Documentation

This documentation repo is a PocketCloud workspace. Its documentation
follows the standard layout every PocketCloud repo shares:

- **[ADRs/](ADRs/README.md)** — architecture decisions *about this repo*.
  Significant choices get an ADR here, however short.
- **[Standards/](Standards/README.md)** — pointer to the canonical,
  ecosystem-wide standards (they live in the primary workspace and are not
  copied here).

## Working in this repo

```bash
pocket knowledge rag query --namespace workspace.pocket-cloud-preview "<question>"   # this repo's knowledge
pocket knowledge rag query --namespace docs.standards "<question>"   # the ecosystem rules
pocket knowledge rag index                                           # refresh this repo's index
```