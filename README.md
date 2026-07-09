# PocketCloud Preview

**Private AI that runs on your hardware.**

This is the public narrative repository for PocketCloud: a local-first, privacy-first AI ecosystem built in Swift. It mirrors the positioning from [pocketcloud.org](https://pocketcloud.org) while adding the engineering context GitHub readers need before the primary source repository opens.

PocketCloud is a cross-platform engine supporting Apple Silicon, Linux, Unix, and Windows through Swift 6.2, MLX, llama.cpp, MCP, and the `pocket` CLI.

---

## What This Repo Is

This repository is not the full source tree yet. It is the open-source launch companion for the main PocketCloud workspace:

- a consistent public explanation of the product, mission, and architecture
- engineering evidence from the private workspace as it approaches open source
- a changelog-style view of what is real today versus what is still roadmap
- a place for issues, questions, and early community feedback

The official marketing site presents the product experience. This repository explains how the system is being built and verified.

---

## Official Positioning

PocketCloud is a reimagining of AI as a personal, distributed, sovereign computing ecosystem:

- **Your data, your control:** AI runs on devices you own.
- **Local by default:** MLX, llama.cpp, LM Studio, and Ollama come before cloud.
- **Cloud when you choose:** OpenAI, Anthropic, Gemini, XAI, OpenRouter, and other providers are opt-in.
- **Open ecosystem:** blueprint apps show how private AI experiences can be built without starting from scratch.
- **Swift-native operations:** the `pocket` CLI handles build, verification, App Store, web fleet, and automation workflows.

The line from the website is the short version:

> AI for the People, by the People, on the People's Devices.

---

## State of the Code

Reviewed from the main workspace git history through **July 9, 2026**:

| Area | Current state |
|---|---|
| Main workspace commits | 3,439 since December 5, 2025 |
| Last 30 days | 739 commits across apps, AIStack, GameStack, Core, web fleet, distribution, accessibility, observability, and verification |
| Swift code | 4,068 tracked Swift files, roughly 800K lines |
| ADRs | 58 architecture decision records |
| App projects | PocketMind, PocketLearning, PocketWellness, PocketBusiness, PocketHub, PocketGamer, PocketShowcase (7 apps) |
| Verification culture | `pocket system verify --exhaustive` remains the core confidence gate |
| Recent focus | PocketGamer game engine (voice adventures, procedural dungeons, on-device board games), ecosystem-wide accessibility audits, SwiftUI adaptive layout, App Feature Catalog SSOT (ADR-0061), Skill Taxonomy curriculum (ADR-0062), in-app machine setup (installer retired) |

The older "101 days" story is still true as an early milestone. It is no longer the whole story.

---

## What Is Working

| Capability | Status | Evidence |
|---|---|---|
| Local AI routing | Working | MLX-first routing, LM Studio/Ollama local fallback, cloud opt-in |
| MLX inference | Working | model registry, benchmarking, daemon self-healing, iOS Simulator hardening |
| llama.cpp integration | Working/integration hardening | ADR-0042, fork pinning, cross-platform provider work |
| `pocket` CLI | Working | system, dev, quality, ops, knowledge, task, web, build, App Store, ASC surfaces |
| MCP server | Working | local tool execution, dry-run hardening, workspace and agent instruction tools |
| RAG and knowledge context | Working | local index/query, verification bootstrap/self-healing |
| Blueprint Apps | Public beta path | TestFlight-oriented app pages and screenshot automation |
| Web fleet | Working | Swift-native deploy/sync/rollback/backup, admin control plane, local mirror, SEO Pilot |
| Observability | Working | local logs, verify run history, settings observability hubs |
| App Store automation | Working | ADR-0048 metadata SSOT, screenshot capture/eval/push, ASC read commands |

See [FEATURES.md](FEATURES.md) for a fuller engineering breakdown.

---

## Blueprint Apps

PocketCloud is more than one app. The current ecosystem is organized around blueprint apps that demonstrate private AI patterns:

| App | Purpose |
|---|---|
| PocketMind | Private AI chat, notes, local knowledge, daily review |
| PocketLearning | Notes, flashcards, quizzes, spaced repetition |
| PocketWellness | Mood, journaling, reflection, privacy-first wellness |
| PocketBusiness | Native control center for servers, deployments, leads, and BI |
| PocketHub | AI hub for code, telemetry, prompts, providers, and docs |
| PocketGamer | On-device AI games: voice adventures, procedural dungeons, board games, and a build-your-own engine |
| PocketShowcase | Living gallery of the PocketCloud UI kit |

These map to the public site’s **Blueprint Apps** section and the App Store metadata automation in the main repo.

---

## Architecture Snapshot

```text
Blueprint Apps
  PocketMind · PocketLearning · PocketWellness · PocketBusiness
  PocketHub · PocketGamer · PocketShowcase
        |
        v
Toolkit Layer
  PocketCloudUI · AI UI · Infrastructure UI · Admin UI · StarterKit
        |
        v
Kernel Layer
  AIStack · Core · MCP · FileKit · Privacy · Logger · Platform · Runtime
        |
        v
Inference + Operations
  MLX · llama.cpp · LM Studio · Ollama · cloud providers · web fleet
```

Read [ARCHITECTURE.md](ARCHITECTURE.md) for the deeper version.

---

## The `pocket` CLI

The main workspace has converged on `pocket` as the operating surface:

```text
pocket system      # health, verify, local AI, logs, workspace
pocket build       # canonical gold-standard build workflow
pocket dev         # providers, Apple tooling, App Store, ASC
pocket quality     # platform checks, analysis, tests, docs
pocket ops         # CI, monitoring, bootstrapping, web fleet ops
pocket knowledge   # RAG, model registry, document search
pocket task        # scheduling and lifecycle
pocket web         # deploy, sync, rollback, backup, analytics, jobs
```

Repository automation in the source workspace should use `./scripts/swiftw` or `pocket`, not raw `swift`.

---

## Learn More

- [STORY.md](STORY.md) - the original build narrative, updated through the July state
- [FEATURES.md](FEATURES.md) - current capability map
- [ARCHITECTURE.md](ARCHITECTURE.md) - system architecture
- [ROADMAP.md](ROADMAP.md) - completed, active, and future work
- [docs/local-ai.md](docs/local-ai.md) - local inference details
- [docs/privacy-model.md](docs/privacy-model.md) - privacy model
- [docs/development-velocity.md](docs/development-velocity.md) - development stats
- [docs/why-local-ai.md](docs/why-local-ai.md) - the case for local-first AI

---

## Follow Along

Star this repository to follow the open-source rollout.

Questions, corrections, and early community feedback are welcome through GitHub issues.

**Author:** [Steven Moon](https://www.linkedin.com/in/stevenmoon/) · [Clever Coding](https://clevercoding.com/about) · [@stevenmoon](https://x.com/stevenmoon)
