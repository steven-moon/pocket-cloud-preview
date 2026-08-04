# PocketCloud Preview

**Private AI that runs on your hardware.**

This is the public narrative repository for PocketCloud: a local-first, privacy-first AI ecosystem built in Swift. It mirrors the positioning from [pocketcloud.org](https://pocketcloud.org) while adding the engineering context GitHub readers need before the primary source repository opens.

PocketCloud is a cross-platform engine supporting Apple Silicon, Linux, Unix, and Windows through Swift 6.3, MLX, llama.cpp, MCP, and the `pocket` CLI.

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

Reviewed from the main workspace git history through **August 3, 2026**:

| Area | Current state |
|---|---|
| Main workspace commits | 4,205 since December 5, 2025 |
| Last 30 days | 892 commits across apps, AIStack, GameStack, Core, web fleet, distribution, hybrid shells, knowledge mesh, and verification — July 2026 was the busiest month in the project's history (923 commits, ahead of January's 860) |
| Swift code | 4,502 tracked Swift files, roughly 872K lines |
| ADRs | 81 architecture decision records (through ADR-0084) |
| App projects | 8 — seven Apple apps (PocketMind, PocketLearning, PocketWellness, PocketBusiness, PocketHub, PocketGamer, PocketShowcase) plus **PocketDesktop**, the Tauri shell that is the Windows and Linux surface. The **Android packaging milestone shipped 2026-08-02**: `pocket build android` produces both APK and AAB, and all eight apps build, install, and launch on Android emulators as remote-core clients of a paired node (ADR-0082); store distribution is the remaining step |
| Build workflow | Tiered composites — `pocket build bronze` (fast iteration), `silver` (pre-merge), `gold` (release: clean → all → test) — with strict-by-default tests where a vacuous run fails loudly |
| Verification culture | `pocket system verify --exhaustive` remains the core confidence gate, joined by deterministic scanners (`pocket quality scan paths\|flywheel\|aml\|tests`) in the pre-commit hook and the build — including a scanner that hunts **tests no target compiles**, after an audit proved green gates can hide suites that never run |
| Recent focus | Cross-platform trust (a fork had silently disabled Metal GPU inference — caught and replaced with verified artifacts), the Android/Windows lanes, a distributed knowledge mesh with encrypted snapshot sync (ADR-0080), fleet server lifecycle tooling (ADR-0084), and the first commercial implementation offer built on the platform (ADR-0081) |

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
| Web fleet | Working | Swift-native deploy/sync/rollback/backup, admin control plane, fleet presence monitoring, config audits, server lifecycle tiers (lease/reap/spend ceilings), local mirror, SEO Pilot |
| Membership | Web loop verified in sandbox | account-based membership with a single supporter tier, Stripe checkout → webhook → entitlement wired through the shared web library; the Apple side — server entitlement unification, a PocketMind pilot, then the seven-app rollout and App Review submission — is still in progress |
| Knowledge mesh | Tier 1 working | local entity/knowledge processing with encrypted-by-default snapshot bundles and LAN sync groundwork (ADR-0080) |
| Observability | Working | local logs, verify run history, settings observability hubs, unified web-to-hub telemetry |
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
| PocketDesktop | The Tauri shell that carries PocketCloud to **Windows and Linux** |

Every Apple app also ships a hybrid web surface built from the same
`@pocketcloud/web-lib` components that PocketDesktop and the web fleet mount, so a
screen written once renders on iPhone, iPad, Mac, Windows, and Linux. The **Android
packaging milestone shipped** (ADR-0082, 2026-08-02): `pocket build android`
produces both APK and AAB, and all eight apps build, install, and launch on Android
emulators, running at launch as clients of models on a paired PocketCloud node
rather than on-device. Store distribution is the remaining step.

These map to the public site’s **Blueprint Apps** section and the App Store metadata automation in the main repo.

---

## Architecture Snapshot

```text
Blueprint Apps
  PocketMind · PocketLearning · PocketWellness · PocketBusiness
  PocketHub · PocketGamer · PocketShowcase · PocketDesktop (Win/Linux)
  Android remote-core client (packaging shipped, ADR-0082)
        |
        v
Toolkit Layer
  PocketCloudUI · AI UI · Infrastructure UI · Admin UI · StarterKit
  Subscriptions + Subscriptions UI
        |
        v
Kernel Layer
  AIStack · Core · GameStack · AML · AMLFormat · Flywheel · DevTools · HardwareDiagnostics
  (Core modules include MCP · FileKit · Privacy · Logger)
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
pocket build       # tiered build workflow: bronze (iterate) · silver (pre-merge) · gold (release)
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
