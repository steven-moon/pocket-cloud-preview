---
id: "doc.internal.architecture.standards.unified_providers_flywheel"
title: "Unified Providers Flywheel Standard"
types: ["standard", "source-of-truth"]
tags: ["architecture", "standards", "observability", "unified-providers", "telemetry", "learning"]
owners: ["@steven"]
version: "26.9.3"
status: "active"
last_reviewed: "2026-09-03"
---

# Unified Providers Flywheel Standard

This document is the canonical source of truth for integrating telemetry, observability, and learning signals across the PocketCloud ecosystem. It replaces feature-local logging and analytics with a unified intelligence loop that powers the PocketBusiness cockpit, AI evaluations, and system diagnostics.

If another ADR, task, or local implementation note conflicts with this document, this standard is authoritative unless the conflict is intentional and updated in the same change set.

Canonical standards index:
[`Documentation/Standards/README.md`](README.md).

## The Intelligence Hub

The PocketCloud Ecosystem operates on a unified flywheel (as outlined in ADR-0050). Data produced by individual apps, tools, and the web fleet feeds into a central intelligence hub. This data powers RAG inference, analytics, and self-healing systems.

To participate in this flywheel, all code must use the established Unified Providers rather than siloed logging paths.

> [!NOTE]
> **ADR-0073 §5 landed (2026-08-02): the web fleet now has a path to the hub.**
> The TypeScript entry point is web-lib's `UnifiedTelemetry`
> (`pocket-cloud-web-lib/src/server/utils/telemetry.ts`, import
> `@pocketcloud/web-lib/server/telemetry`) — same `component`/level/metadata
> vocabulary as Swift's `UnifiedLogEmitter`. Emits write `server_log` funnel
> events fire-and-forget into the bound warehouse (bound at boot in the
> vanilla-site template); the mirror pipe lifts warn/error rows into
> `UnifiedRAG` + `UnifiedLearningSignal` via
> `WebFleetMirrorImporter.ingestHighValueEventsIntoRAG` — telemetry can never
> slow or fail the caller (the git-hook flywheel discipline). The legacy
> `AppLogger` (`utils/logger.ts`, a `console.log` wrapper) is deleted; its only
> consumers were the APNs push modules, now on `UnifiedTelemetry`.
>
> **Baseline Policy #4 consequence, updated:** the funnel warehouse is no longer
> an *exception* — it is the sanctioned **transport leg** of the web→hub path
> (warehouse → NDJSON mirror → importer → hub), the same store the analytics
> contract already rides. The 2026-07-17 accuracy note recording that "the web
> fleet has no path to the hub" is retired; new web server code routes through
> `UnifiedTelemetry`, never `console.log` (`FlywheelScanner.Rule.consoleLog`
> enforces this at commit).

## Proper Unified Type Usage

When writing or refactoring code in the PocketCloud workspace, use the following core provider classes:

### 1. Diagnostics & Observability
- **`UnifiedLogEmitter`**: Use for all application and system logging. Do not use raw `print()`, `os_log`, or local `Logger` wrappers.
  - *Example*: `let telemetry = UnifiedLogEmitter(component: "AdminMonitoring")`
- **`SystemObservability`**: Use for high-level, operator-visible system events that describe the health of the host or cluster.

### 2. Telemetry & Metrics
- **`TelemetryManager`**: Use for structured, semantic event telemetry that tracks user actions, system transitions, and feature usage.
- **`MetricsCollector`**: Use for aggregate, time-series data (e.g., CPU, memory, request rates) where continuous metrics are appropriate.

### 3. Knowledge & Execution Graph (ADR-0113)
- **`PocketGraphStore`**: The typed node/edge substrate that persists knowledge, ingestion, retrieval, and parallel execution data (ADR-0113). This is the concrete implementation of the intelligence hub described above.
- **Graph Producers**: Typed producers that emit structured edges into the graph:
  - `DecisionTraceGraphProducer` (`PocketCloudAIAgent/Core/Observability/`) — AI decision traces, routing choices, and inference outcomes.
  - `TelemetryGraphProducer` (`PocketCloudCore/Graph/`) — system telemetry edges from unified log events.
  - `SwiftPMGraphProducer` (`PocketCloudCore/Graph/`) — dependency and target edges from the Swift package graph.
- Prefer graph producers over feature-local persistence for any signal that should be traversable, queryable, or used by the progressive resolution pyramid (ADR-0115).

### 4. Unified Learning & Signals
- **`UnifiedLearningCoordinator`**: The central dispatcher for recording semantic learning signals, failures, and inferences that agents can use for self-improvement and self-healing.
- **`UnifiedLearningSignal`**: The core data structure for a learning event.
  - *Example*:
    ```swift
    await UnifiedLearningCoordinator.shared.recordSignal(
        UnifiedLearningSignal(
            source: .logger,
            severity: .error,
            message: "Failed to sync process",
            metadata: ["sync_phase": "fetch"]
        )
    )
    ```
- **App-Specific Adapters**: Use specialized adapters to map app-specific events into standard Unified Learning signals. All conform to `UnifiedLearningAdapter` (`PocketCloudCommon/UnifiedLearningAdapter.swift`); extend an existing one rather than adding a parallel path. The fleet, verified against code 2026-09-03:
  - `SharedUnifiedLearningAdapter` (`PocketCloudCommon/UnifiedLearningAdapter.swift`)
  - `PocketMindUnifiedLearningAdapter` (`Packages/Toolkit/PocketCloudSecondBrainUI/`)
  - `PocketLearningUnifiedAdapter` (`Packages/Toolkit/PocketCloudLearningUI/`)
  - `PocketWellnessUnifiedLearningAdapter` (`Packages/Toolkit/PocketCloudWellnessUI/`)
  - `PocketBusinessUnifiedLearningAdapter` (`Packages/Toolkit/BusinessInsightsPack/`)
  - `GameUnifiedLearningAdapter` (`Packages/Kernel/GameStack/Sources/PocketCloudGameEngine/Logic/`)
  - `DeveloperHubUnifiedLearningAdapter`, `HubUnifiedLearningAdapter` (`Packages/Kernel/AIStack/Sources/PocketCloudServer/Services/`)

  > **Path correction (2026-09-03):** ADR-0098's single-project consolidation moved most
  > app-specific adapters from `Apps/` into their respective `Packages/Toolkit/` targets.
  > The Hub adapters moved from `Apps/PocketHub/` into `PocketCloudServer/Services/`.
  > Earlier path references in this standard pointed to the pre-consolidation locations.

## Baseline Policy

1. Treat integration with PocketCloud's unified providers as part of the standard implementation process, not as optional polish.
2. When adopting older code or moving an existing class into an active runtime path, audit it for missing unified-provider integration.
3. New or modernized integration paths must emit enough context to debug, test, iterate, and evaluate the system later: include operation name, subsystem, actor/source, input shape, outcome, duration or cost when relevant, and failure reason when available.
4. Do not introduce a parallel logging, telemetry, learning, or metrics store unless an ADR/task explicitly explains why the unified providers cannot model the signal.
5. **Proactive Flywheel Engine Integration**: When working in or analyzing *any* source file, both AI agents and human developers must proactively inspect the code for missing telemetry, incorrect log levels, or outdated type usage. You must proactively add the necessary unified telemetry, diagnostics, and learning signals as you go, which in turn surfaces hidden architectural issues, warnings, and system weaknesses.
6. **Immediate Resolution of Surfaced Issues**: Once these weaknesses, warnings, compiler alerts, or telemetry errors are surfaced, you must proactively resolve them as you go. Do not bypass, skip, or postpone addressing warnings, performance regressions, or type errors simply because they are "pre-existing" in the file or outside the immediate scope of the current task. Resolving these anomalies in-stride is a core requirement of the flywheel loop.
7. **Adaptive UI Regressions Are Flywheel Signals**: When a sample app, toolkit package, or operator view exposes a narrow-width layout failure, fix the underlying adaptive pattern in-stride and apply the same audit to sibling views. iPhone portrait, iPad split view, and narrow macOS windows are normal verification surfaces under the Swift 6.3 implementation standard, not follow-up polish.

## Long-Running Operation Heartbeats

Long-running runtime operations must use a reusable heartbeat monitor from the foundation layer instead of ad hoc timers inside leaf commands.

- Emit a heartbeat when a command or tool operation has produced no meaningful output for 30 minutes.
- Repeat the heartbeat every 30 minutes while the operation remains quiet and healthy.
- Observe both unified logs and tool notifications so the monitor can distinguish real progress from true silence.
- Treat a heartbeat as a progress signal and a flywheel gap signal, not as a substitute for failure handling or completion reporting.
- Let leaf commands customize the wording and surface-specific phrasing, but keep the cadence, cancellation behavior, and learning-signal behavior consistent.
- Prefer policy-driven customization over ad hoc timers or command-local timers so the foundation monitor can be extended through the CLI stack without fragmenting the signal model.

## Evaluation, Benchmarking & Trust

These signals form the PocketCloud flywheel: they help the current codebase become easier to debug and test, and they create the necessary context that future local AI and coding agents will use for evaluation, recommendations, and self-improvement loops.

- **Trust Taxonomy (ADR-0116)**: Every analytics edge carries a trust level:
  - `high` — server-side, corroborated by serve receipts or HMAC-stamped events.
  - `corroborated` — client-claimed but JS-verified (human presence confirmed).
  - `floor` — client-claimed only, uncorroborated.
  New analytics integrations must stamp their trust level; never ingest client-claimed data at `high` trust without server-side verification.
- **GameSession Traces & GameIntents**: Use `UnifiedDataProvider` to persist session interactions so the Benchmark Provider can systematically evaluate local AI decisions.
- **AI Feedback**: AI decisions (like routing, NPC generation, or semantic reviews) should emit traces (preferably via `DecisionTraceGraphProducer`) that can be reviewed in the PocketBusiness operator cockpit.
- **Progressive Resolution (ADR-0115)**: The rollup pyramid provides bounded projection over large signal volumes. Use rollup-aware queries when displaying aggregate metrics rather than scanning raw edges.

## ADR and Task Integration Rule

Every active ADR and task document that adds or modifies runtime behavior must consider its impact on the Unified Providers Flywheel. Do not close an ADR/task without ensuring the feature emits proper diagnostic and learning signals.
