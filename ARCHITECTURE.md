# PocketCloud Architecture

PocketCloud is built as a Swift-native local AI platform with three public promises:

1. **Local-first intelligence** - run on hardware you own whenever possible.
2. **Sovereign control** - you decide what data, providers, tools, and servers participate.
3. **Verified operations** - build, deploy, diagnose, and ship through reproducible CLI workflows.

---

## Stack Overview

```text
Blueprint Apps
  PocketMind · PocketLearning · PocketWellness · PocketBusiness
  PocketHub · PocketGamer · PocketShowcase
        |
        v
Toolkit Layer
  PocketCloudUI · PocketCloudAIUI · PocketCloudAdminUI
  PocketCloudInfrastructureUI · PocketCloudInfrastructureKit
  PocketCloudStarterKit · app-specific UI/tooling packages
        |
        v
Kernel Layer
  AIStack
    AIRouter · providers · MLX · llama.cpp · RAG · model registry
    AIAgent · task queue · CLI command surfaces

  Core
    MCP · logger · privacy · FileKit · Platform · Runtime
    document system · sync · unified provider protocols
        |
        v
Operations + Inference
  MLX · llama.cpp · LM Studio · Ollama · cloud providers
  App Store Connect · web fleet · local admin · verification
```

---

## Apps Layer

The app layer is now aligned with the public **Blueprint Apps** story:

| App | Role |
|---|---|
| PocketMind | Private AI assistant and knowledge interface |
| PocketLearning | Study, flashcards, quizzes, spaced repetition |
| PocketWellness | Mood, journaling, mindfulness, local reflection |
| PocketBusiness | Native web fleet and business control center |
| PocketHub | Developer AI hub for code, prompts, telemetry, and providers |
| PocketGamer | On-device AI games: voice adventures, procedural dungeons, board games, build-your-own engine |
| PocketShowcase | UI component and design-system gallery |
| PocketDesktop | Tauri shell — the Windows and Linux surface |

The mid-2026 workspace includes project manifests and Xcode projects for the seven
Apple apps, plus the Tauri manifest for PocketDesktop. The standalone installer was
retired — machine setup now lives inside every app.

Each Apple app also embeds a hybrid web surface built from the same
`@pocketcloud/web-lib` custom elements that PocketDesktop and the web fleet mount.
One component therefore renders on iPhone, iPad, Mac, Windows, and Linux — and is
loaded from inside the `.app` over `file://` on Apple platforms, which is a
materially stricter environment than a browser or a Tauri webview (ADR-0066).

---

## AI Routing

PocketCloud's AI routing is local-first:

```text
Request
  |
  +-- local MLX model fits the task and hardware? -> MLX
  |
  +-- local server available? -> LM Studio or Ollama
  |
  +-- cross-platform llama.cpp model available? -> llama.cpp
  |
  +-- explicit cloud provider configured? -> provider API
  |
  +-- otherwise -> explain unavailable path
```

The router is backed by model discovery, model reputation, hardware fit checks, benchmark signals, and local RAG context where available.

---

## Unified Provider Layer

Recent source history renamed the broad provider foundation from `UnifiedProvider` to `PCUnifiedProvider` to avoid collisions with generic names in app and package code.

The architectural intent remains the same: Context, AI, Data, Network, and telemetry providers use consistent protocols so apps can swap implementations without rewriting product logic.

Public docs may still use "Unified Intelligence" as the product concept. Engineering docs should use `PCUnifiedProvider` for the concrete code-level foundation.

---

## Web Fleet Architecture

The current system includes a complete web platform:

```text
pocket web CLI
  deploy · sync · rollback · backup · jobs · logs · analytics
        |
        v
web fleet config + SSH/client orchestration
        |
        v
self-hosted sites
  @pocketcloud/web-lib
  clean URLs · AI chat · contact funnel · admin
  presence · backups · bot traffic · SEO Pilot
        |
        v
PocketBusiness
  native iPhone/Mac control center
```

The web fleet is designed to replace a pile of hosted dashboards with one owned control plane.

---

## App Store Automation

ADR-0048 introduced a source-of-truth metadata and storefront automation flow:

```text
App metadata AML
  |
  +-- generated website app pages
  +-- TestFlight/public beta CTAs
  +-- screenshot capture and evaluation
  +-- App Store Connect metadata/screenshot push
```

This is why the official site and preview repo should use the same app names and product taxonomy.

---

## Verification Architecture

The verification system is an operational subsystem, not just a test command:

- `pocket system verify --exhaustive`
- integrated exhaustive profiles
- run budgets for long operations
- timeout and SwiftPM contention handling
- RAG and MLX daemon bootstrap/self-healing
- build failure classification
- verify run history and observability UI

Historical baseline: the early preview reported `109/112` operations passing in March 2026. Current public docs should treat that as historical, not the latest score.

---

## Privacy Boundaries

| Boundary | Default |
|---|---|
| Prompts and documents | stay local |
| RAG indexes | local filesystem |
| MLX/llama models | local model store |
| Logs and verify artifacts | local log roots |
| MCP tools | local process communication |
| Cloud providers | explicit opt-in |
| Web analytics | owned/self-hosted fleet data |

PocketCloud's architecture is hybrid, but control stays with the user.
