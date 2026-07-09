# PocketCloud Features

This file summarizes the public feature story as of the July 9, 2026 git review of the main workspace.

The source repo is still private, so this document favors verifiable engineering claims over marketing breadth. Where the website says what PocketCloud is, this file says what has been implemented or actively hardened.

---

## Local and Hybrid AI

PocketCloud prioritizes local inference and treats cloud providers as explicit fallbacks.

| Feature | State |
|---|---|
| MLX local inference | Implemented and actively hardened |
| llama.cpp provider | Implemented for cross-platform expansion |
| LM Studio and Ollama routing | Supported as local server options |
| Cloud providers | Opt-in provider layer for frontier models |
| Hardware-aware model selection | Implemented with model fit and reputation signals |
| Local model benchmarking | Exposed through the `pocket` CLI |
| RAG context | Implemented through local indexing, query, and verification bootstrap |
| Persistent daemon | Implemented with verify self-healing paths |

Recent git evidence includes llama.cpp ADR work, MLX daemon and cache fixes, model reputation cleanup, RAG self-healing, and iOS Simulator MLX hardening.

---

## The `pocket` CLI

`pocket` is the operational interface for the ecosystem.

```text
pocket system      # health, verify, local AI, logs, workspace
pocket build       # canonical build workflow
pocket dev         # providers, Apple tooling, App Store, ASC
pocket quality     # platform checks, analysis, tests, docs
pocket ops         # CI, monitoring, bootstrapping, web fleet ops
pocket knowledge   # RAG, model registry, document search
pocket task        # scheduling and lifecycle
pocket web         # deploy, sync, rollback, backup, analytics, jobs
```

Recent changes added or hardened:

- top-level `pocket build`
- `pocket quality platform-check`
- web fleet deploy/sync/rollback/backup/jobs/logs/analytics commands
- App Store storefront generation, screenshot capture, evaluation, and push commands
- App Store Connect read commands
- verify run history and observability surfaces

---

## Blueprint Apps

The public site now frames the app suite as **Blueprint Apps**. The source workspace mirrors that direction.

| App | Current role |
|---|---|
| PocketMind | Private AI chat, notes, local knowledge, daily review |
| PocketLearning | AI study notes, flashcards, quizzes, spaced repetition, skill-graph curriculum (ADR-0062) |
| PocketWellness | Mood tracking, journaling, mindfulness, local reflection |
| PocketBusiness | Native control center for web fleet, servers, deployments, leads, and BI |
| PocketHub | Code, telemetry, prompts, providers, and developer diagnostics |
| PocketGamer | On-device AI games: voice adventures, procedural dungeons, board games, build-your-own engine |
| PocketShowcase | UI kit gallery and provider/prompt playground |

The mid-2026 history includes the rename from `PocketCloudHub` to `PocketHub` and `PocketGameEngine` to `PocketGamer`, public TestFlight framing, screenshot automation, refreshed app metadata, an App Feature Catalog SSOT (ADR-0061), and the retirement of the standalone installer (machine setup now lives inside every app). All seven apps received an accessibility audit pass — VoiceOver labels, audit suites, and regression tests — plus a SwiftUI adaptive layout policy for iPhone/iPad/Mac.

---

## Web Fleet

PocketCloud now includes a self-hostable web fleet, not just native apps.

Implemented or recently hardened:

- `@pocketcloud/web-lib` shared site runtime
- Swift-native `pocket web` deployment flow
- zero-downtime hot swaps and rollback support
- Cloudflare tunnel sync
- local admin control plane
- multi-user JWT auth with fail-closed APIs
- lead capture and contact funnel
- real-time presence and operator messaging
- APNs-backed push notifications for fleet events
- bot and AI-agent traffic intelligence
- `/robots.txt` and `/llms.txt` AI-crawl readiness
- data lifecycle services for rollup, archive, prune, eviction, and vacuum
- local mirror store, job runner, and SEO Pilot services

This is the largest difference from the original March preview docs.

---

## App Store and Source Rollout Automation

The current workspace has a full App Store metadata and screenshot pipeline:

- ADR-0048 App Store storefront automation and SSOT metadata
- generated site pages from app metadata
- iPhone and iPad screenshot capture
- screenshot evaluation and feature cataloging
- App Store Connect metadata/screenshot push tooling
- App Store Connect read commands
- public TestFlight call-to-action framing

The preview repo should therefore describe PocketCloud as preparing for source rollout, while the apps already have a public beta path.

---

## Observability

PocketCloud's observability layer is local-first and increasingly visible inside the apps.

Recent work includes:

- verify run history model and UI
- observability hub integration in settings
- MCP tool execution logging
- shared log roots
- process terminal and terminal manager hardening
- local log viewers
- build failure classification and remediation signals

The public site calls this "Unified Intelligence & Observability." In code, recent work renamed the generic provider foundation to `PCUnifiedProvider` to avoid collisions.

---

## Privacy and Control

PocketCloud's privacy posture remains consistent:

- local inference first
- no cloud provider unless configured
- local RAG indexes
- local logs and verify artifacts
- Keychain or local secret storage for credentials
- no product analytics dependency required for core AI workflows
- MCP server as a local process, not a hosted control point

Web fleet analytics are owned and self-hostable rather than outsourced to third-party SaaS.

---

## Verification

The main workspace continues to use `pocket system verify --exhaustive` as the core confidence gate. Recent history shows work on:

- integrated exhaustive profiles
- run budgets
- timeout hardening
- SwiftPM contention serialization
- RAG and MLX daemon self-healing
- platform checks before app builds
- build failure classification
- verify run pruning and history UI

The older March baseline of `109/112` operations is preserved in the story as historical evidence, but current docs should not present it as the latest verified run.
