---
id: "ADR-0001-adopt-pocketcloud-workspace-standards"
title: "ADR-0001: Adopt PocketCloud Workspace Standards"
types: ["adr", "source-of-truth"]
tags: ["workspace", "pocketcloud"]
owners: []
version: "1.0.0"
status: "accepted"
last_updated: "2026-08-04T00:00:00Z"
is_archived: false
last_reviewed: "2026-08-04"
---

# ADR-0001: Adopt PocketCloud Workspace Standards

## Status
Accepted (2026-08-04). Seeded by `pocket ops git-hook install` — this
document records the adoption it is part of.

## Context
This repo (`pocket-cloud-preview`, a documentation repo) joined the
PocketCloud ecosystem: git hooks call the shared `pocket` CLI, the repo's
files index into its own knowledge namespace (`workspace.pocket-cloud-preview`),
and commit outcomes feed the shared learning flywheel.

## Decision
- Documentation follows the standard PocketCloud layout (this directory).
- Decisions about this repo are recorded as ADRs here, starting with this one.
- The canonical cross-repo standards are **referenced, not copied** — see
  [Standards/README.md](../Standards/README.md).
- Gates arrive incrementally ("brick by brick"): the hooks observe and teach
  before they ever block, and only gates that genuinely apply to a
  documentation repo will be enabled.

## Consequences
- Humans and AI agents get the same entry points here as in every other
  PocketCloud repo.
- This ADR log is the place future decisions land — the next one is
  ADR-0002.