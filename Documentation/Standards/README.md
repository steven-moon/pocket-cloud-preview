---
id: "doc.pocket-cloud-preview.standards.readme"
title: "pocket-cloud-preview Standards Pointer"
types: ["standard", "index", "source-of-truth"]
tags: ["workspace", "pocketcloud"]
owners: []
version: "1.0.0"
last_reviewed: "2026-08-04"
---

# Standards — canonical, referenced, not copied

The PocketCloud standards are **singular documents** in the primary
workspace (`pocket-cloud-workspace/Documentation/Standards/`). They are not
copied into secondary repos, because eleven copies drift eleven ways.

Query them from anywhere:

```bash
pocket knowledge rag query --namespace docs.standards "<question>"
pocket knowledge rag query --namespace docs.adr "<question>"
```

Key ones for a documentation repo:
- **Unified Providers Flywheel** — telemetry/learning signals; no siloed logging.
- **Documentation naming standard** — kebab-case, no temporal names, frontmatter.


A standard that applies only to this repo may live here; anything broader
belongs upstream.