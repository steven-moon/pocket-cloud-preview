---
id: "doc.internal.architecture.standards.unified_providers_flywheel"
title: "Unified Providers Flywheel Standard"
types: ["standard", "source-of-truth"]
tags: ["architecture", "standards", "observability", "unified-providers", "telemetry", "learning", "git-hooks", "branding"]
owners: ["@steven"]
version: "26.10.24"
status: "active"
last_reviewed: "2026-10-24"
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
- **`PocketAppLauncher` Telemetry**: Deep-link queries and ecosystem inter-app navigation events record structured logs and telemetry (`Logger(label: "PocketAppLauncher")` / `UnifiedLogEmitter(component: "PocketAppLauncher")`). Each launch captures target `PocketAppKey`, URL scheme resolution, installation status, and fallback dispatch to web-fleet endpoints (`https://pocketcloud.org/<appKey>`), ensuring deep-link anomalies and broken scheme queries surface directly to the flywheel.

### 3. Knowledge & Execution Graph (ADR-0113)
- **`PocketGraphStore`**: The typed node/edge substrate that persists knowledge, ingestion, retrieval, and parallel execution data (ADR-0113 (`Documentation/ADRs/ADR-0113-pocket-graph-unified-knowledge-and-execution-graph.md`)). This is the concrete implementation of the intelligence hub described above.
- **Graph Producers**: Typed producers that emit structured edges into the graph:
  - `DecisionTraceGraphProducer` (`PocketCloudAIAgent/Core/Observability/`) — AI decision traces, routing choices, and inference outcomes.
  - `FlywheelGraphProducer` (`PocketCloudCore/Graph/`) — the flywheel's learned remediations as `failure-fingerprint --fixed-by--> remediation` edges, marked `.inferred` with the learned confidence (ADR-0113 (`Documentation/ADRs/ADR-0113-pocket-graph-unified-knowledge-and-execution-graph.md`) T06).
  - `VerifyGraphProducer` (`PocketCloudCore/Graph/`) — the `pocket system verify` ledger as one `verify-operation` node per operation, aggregated across runs (ADR-0113 T06).
  - `SwiftPMGraphProducer` (`PocketCloudCore/Graph/`) — dependency and target edges from the Swift package graph.
  - `BrandGraphProducer` (`PocketCloudCore/Branding/`) — each brand kit projected into the tenant's `marketing` store (`pocket knowledge graph build --domain marketing`): the brand, a content-addressed `brand-revision` (`<brand>.<digest12>`, `supersedes` its predecessor), pillars, audiences, accounts and the products it `promotes`. Identity and lineage only — never kit strategy (`contentMix`, `research`), brand documents or secrets; an unchanged kit writes nothing; a product node is minted only when absent, because its facts belong to the product producer (ADR-0124 (`Documentation/ADRs/ADR-0124-unified-brand-manager.md`) D6, ADR-0162 (`Documentation/ADRs/ADR-0162-specialized-marketing-graph-brand-voice-trends-and-ideas.md`) T05).
  - `MarketingProductProducer` (`PocketCloudCore/Graph/Marketing/`) — every product the workspace's records describe, in the same tenant `marketing` store: `product` (owns the body T05 only mints), catalog `product-feature`s with their claim label, `product-release`s read from TestFlight ledgers (a cell counts only as `1.0.0 (27), … VALID`; a ledger never yields `app-store`), `media-asset`s only for screenshots whose file still hashes to its storefront `capture-record.json`, and `built-by` to the `repo` (origin remote's name) and to the last `commit` touching a feature's `sourcePath` — named, never minted. `MarketingProductReader` is Linux-safe and decodes only the storefront's identity fields, never its review contact. `MarketingGateFacts.loadProducts(from:)` reads the extracted claims back for the D8 gates (ADR-0162 (`Documentation/ADRs/ADR-0162-specialized-marketing-graph-brand-voice-trends-and-ideas.md`) T06).
  - `ADRGraphProducer` (`PocketCloudCore/Graph/ADR/`) — architectural decisions, standards governance, task ledgers, and cognitive contradictions/gaps (ADR-0154 (`Documentation/ADRs/ADR-0154-specialized-adr-graph-ontology-and-visual-navigator.md`)).
  - `CodeGraphProducer` (`PocketCloudCore/Graph/Code/`) — multi-language symbols, call sites, imports, dead code, and architectural boundaries (ADR-0155 (`Documentation/ADRs/ADR-0155-specialized-code-graph-ontology-and-visual-navigator.md`)).
  - `ServerGraphProducer` (`PocketCloudCore/Graph/Server/`) — fleet topologies, host observations, and drift history (ADR-0157 (`Documentation/ADRs/ADR-0157-specialized-server-graph-ontology-and-history-navigator.md`)).
  - `BuildGraphProducer` (`PocketCloudCore/Graph/Build/`) — build attempt evidence, compilation lineages, and failure analysis (ADR-0158 (`Documentation/ADRs/ADR-0158-specialized-pocket-build-graph-ontology-and-history-navigator.md`)).
  - `UnifiedGraphProducer` (`PocketCloudCore/Graph/`) — connects architectural "why" (ADRs) to symbol-level "what" (Code Graph) (ADR-0155 (`Documentation/ADRs/ADR-0155-specialized-code-graph-ontology-and-visual-navigator.md`)).
  - `PlaceGraphProducer` (`PocketCloudCore/Graph/Place/`) — places, for every app: a `place` (keyed `<source>:<id>`, e.g. `osm:node/123`), the `locality` it stands in (`ames.ia`) and the public `evidence` it was read from, with its IANA zone read offline from the coordinate (`GeoTimeZoneLocator`) and its town cleaned from address tags (`GeoLocality`). Deterministic, Linux-safe, no model or network. Splice `PlaceOntology.nodeKinds` / `.edges` / `.producerExtractor` into a domain's ontology to hold places there; `brain` registers them first (ADR-0161 (`Documentation/ADRs/ADR-0161-unified-graph-layer-ontology-registry-and-viewer-kit.md`) T23). Re-projection declares `GraphEdgeSupersession`s, so a place's town, evidence and operator stay current rather than accumulating. A place is `operated-by` the `company` its own website's registrable domain names, the key the company brain uses (T28; `SharedWebHosts` refuses social pages, site builders and a state's `ak.us`). The company is passed as `referencing:`, so it is inserted when absent and never overwrites the brain's own company body. Place facts (coordinate, zone, town, website) pass the brain field policy on `place` and `locality` only, and a phone never enters the brain. First adopter: the Fieldnotes flywheel (ADR-0146 (`Documentation/ADRs/ADR-0146-local-first-web-search-on-composed-swift-sources.md`) T46); its ledger records the producer version, so a producer change re-projects every place once (T48), and it publishes the graph's answer to "who runs this place" back to the tenant (`partner_research_place_operator`, T50), so a tenant's app reads the graph rather than re-deriving it.
  - `SocialGraphProducer` (`PocketCloudSocialKit/Sources/PocketCloudSocialGraph/`) — marketing campaigns, publication receipts, channel performance, and brand voice adherence (ADR-0162 (`Documentation/ADRs/ADR-0162-specialized-marketing-graph-brand-voice-trends-and-ideas.md`)).
  - `CommunicationGraphProducer` (`PocketCloudSecondBrain/Entities/`) — company brain graph, parties (`person`, `company`), interactions (`email-thread`, `concerns`, `works-at`, `co-participated`, `in-corpus`), archive history, and lead context with tenant-scoped privacy field policies; paired with cognitive insight engines `IdentitySuggester` and `LeadContextResolver` (ADR-0163 (`Documentation/ADRs/ADR-0163-specialized-company-brain-graph-entities-engagements-and-knowledge.md`)).
- Prefer graph producers over feature-local persistence for any signal that should be traversable, queryable, or used by the progressive resolution pyramid (ADR-0115 (`Documentation/ADRs/ADR-0115-progressive-resolution-and-bounded-projection.md`)).
- **One concept, one kind, one owner** (ADR-0161 (`Documentation/ADRs/ADR-0161-unified-graph-layer-ontology-registry-and-viewer-kit.md`) D4). A producer reuses the owning domain's kind and key for any concept another domain already models — `module`, `package`, `imports`, `declares`, `code-file`, `commit`, `person`, `repo`, `failure-fingerprint`, `produced` — and mints a kind only for a concept nobody owns. A producer that points at a node another producer owns writes it through `PocketGraphStore.write(referencing:)` (inserted when absent, never refreshed), never as an upsert. A vocabulary spliced into a domain also answers to that domain's field policy, and since ADR-0161 T29 the domain-scoped shape audit enforces it (`field-policy` violations), so `knowledge graph audit --domain brain` and every producer that validates before writing catch a forbidden field. Insight engines write `.inferred` under a registered extractor name and retract only their own batches. The Code Graph's `code-module` / `code-package` / `imports-module` / `declares-symbol` are the counter-example: four second spellings that silently failed to join the graph the gates read.
- **Graph Shape Audit & Snapshot Flywheel Signals** (ADR-0161 (`Documentation/ADRs/ADR-0161-unified-graph-layer-ontology-registry-and-viewer-kit.md`) D12). Graph shape audits (`GraphShapeAudit`), envelope publications, and cross-domain joins emit structured telemetry via `UnifiedLogEmitter(component: "GraphShapeAudit")` and `UnifiedLearningCoordinator` with the stable fingerprint `graph_shape_audit`. Graph invalidations, retraction failures, or contract violations are recorded as actionable learning signals that spin the system flywheel.

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
  - **Await the write; never hand it to a detached `Task {}`.** `recordSignal` persists before it enriches, so an awaited call is on disk when it returns. A detached task is not: it dies with the process, and the CLI, the git hooks and every build verb are short-lived processes. `UnifiedLearningRemedyOutcomeSink` wrote that way, and on 2026-10-15 both learning stores held zero `build.remedy` records after a week of heals (ADR-0038 T11). Pass `enrich: false` for an identifier-only event (a fingerprint, an outcome) when the caller is latency-sensitive, because enrichment can cold-load an embedding model. Prove durability by reading the store back through a **second** coordinator over the same `storageURL`. Reading the first coordinator only proves memory.
  - **Two things can still remove a written signal, and both are bounded.** Another process's save no longer can: every save is a read-merge-write under a lock beside the store (ADR-0085 (`Documentation/ADRs/ADR-0085-canonical-storage-resolution.md`) T04). The 2,000-signal cap still can, but only within the signal's own class: emitter (`component`, else `feature`) × routine or notable (T05). Over the cap, the largest class loses its oldest, so a rare signal outlives any flood. Set `component` or `feature` on what you record; a signal with neither shares one class per source with every other anonymous signal.
- **App-Specific Adapters**: Use specialized adapters to map app-specific events into standard Unified Learning signals. All conform to `UnifiedLearningAdapter` (source (`Packages/Kernel/Core/Sources/PocketCloudCommon/UnifiedLearningAdapter.swift`)); extend an existing one rather than adding a parallel path. The fleet, verified against code 2026-09-03:
  - `SharedUnifiedLearningAdapter` (source (`Packages/Kernel/Core/Sources/PocketCloudCommon/UnifiedLearningAdapter.swift`))
  - `PocketMindUnifiedLearningAdapter` (`Packages/Toolkit/PocketCloudSecondBrainUI/`)
  - `PocketLearningUnifiedLearningAdapter` (`Packages/Toolkit/PocketCloudLearningUI/`, in `PocketLearningUnifiedAdapter.swift`)
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
7. **Adaptive UI Regressions Are Flywheel Signals**: When a sample app, toolkit package, or operator view exposes a narrow-width layout failure, fix the underlying adaptive pattern in-stride and apply the same audit to sibling views. iPhone portrait, iPad split view, and narrow macOS windows are normal verification surfaces under the Swift 6 implementation standard, not follow-up polish.
8. **A signal names what happened, never what was said.** A signal that follows a person's input (a vote, an answer, a drawing, a name) records the kind of action, its outcome and its context. It never records the content, the choice or the author. A fact derived from content that would leak while it still matters, such as which choice was true while a vote is open, is counted afterwards in a trace. Keep one function that turns a payload into its kind, and a test that no payload gets through it. Example: `party.lan.intent` (ADR-0160 (`Documentation/ADRs/ADR-0160-pocket-party-games.md`) T15) carries `action=castVote` and the room, from `PartyRoutes.kind(of:)`.

9. **A wire is not a log.** A protocol channel — the MCP server's stdout, a sidecar's stdout — carries its protocol and nothing else. Diagnostics go to the unified providers (and stderr through `Logger`), never into the channel as unsolicited messages. Progress reaches a peer only when the request asked for it: an MCP tool's notifications become `notifications/progress` for a request whose `_meta` carried a `progressToken`, and are dropped otherwise. The MCP server used to push every CLI event as `notifications/message` without declaring logging (ADR-0164 (`Documentation/ADRs/ADR-0164-mcp-server-current-protocol-and-honest-tool-surface.md`) D6).

10. **Customer Zero Lead Vetting & Vendor Flywheel.**
    - For Customer Zero (Clever Coding), the dominant volume of inbound contact submissions consists of vendor pitches, offshore dev staffing, SEO solicitations, and cold appointment setting.
    - These solicitations must be filtered at intake via `VENDOR_PHRASES` in `lead-provenance.ts`, guarded against positive AI scoring in `PocketBusinessAIProvider`, and given an operator-facing verdict of `.vendor` ("Vendor Pitch").
    - **Flywheel Calibration:** When an operator marks a lead as a "Vendor Pitch" via `AdminLeadVerdict.vendor`, the verdict maps directly to `LeadOperatorVerdictLabels` as a positive screening/junk signal (`.junk`). This calibrates local RAG vetting weights, zeroes out lead scores (`leadScore = 0`), strips false hot-lead temperatures, and excludes vendors from genuine conversion funnels and high-priority reply alerts.

## Long-Running Operation Heartbeats

Long-running runtime operations must use a reusable heartbeat monitor from the foundation layer instead of ad hoc timers inside leaf commands.

- Emit a heartbeat when a command or tool operation has produced no meaningful output for 30 minutes.
- Repeat the heartbeat every 30 minutes while the operation remains quiet and healthy.
- Observe both unified logs and tool notifications so the monitor can distinguish real progress from true silence.
- Treat a heartbeat as a progress signal and a flywheel gap signal, not as a substitute for failure handling or completion reporting.
- Let leaf commands customize the wording and surface-specific phrasing, but keep the cadence, cancellation behavior, and learning-signal behavior consistent.
- Prefer policy-driven customization over ad hoc timers or command-local timers so the foundation monitor can be extended through the CLI stack without fragmenting the signal model.

## Evaluation, Benchmarking & Trust

Verification probes that write synthetic knowledge use an explicitly owned store
and namespace through the production provider APIs. They must not seed the live
tenant corpus or depend on its size to finish. Keep production readiness checks
(such as canonical-store status) separate and label probe evidence as verification.
Record failures and outcomes through the normal logger/learning path; never weaken
explicit-target guards or replace real index/retrieval with a fabricated success.

These signals form the PocketCloud flywheel: they help the current codebase become easier to debug and test, and they create the necessary context that future local AI and coding agents will use for evaluation, recommendations, and self-improvement loops.

- **Trust Taxonomy (ADR-0116 (`Documentation/ADRs/ADR-0116-measurement-2-0-trustworthy-edges-and-durable-dimensions.md`))**: Every analytics edge carries a trust level:
  - `high` — server-side, corroborated by serve receipts or HMAC-stamped events.
  - `corroborated` — client-claimed but JS-verified (human presence confirmed).
  - `floor` — client-claimed only, uncorroborated.
  New analytics integrations must stamp their trust level; never ingest client-claimed data at `high` trust without server-side verification.
- **GameSession & PartySession Traces**: Persist sessions as `GameSessionTrace` records through `GameSessionTracer` / `GameTraceStore` (`<pocketCloudDataDirectory>/gamestack/traces`), which `pocket system evaluate traces` and the play ledger read back. A family that must not record content adds a typed, content-free evidence slot to `GameSessionStep` instead of free text: `TriviaAnswerEvidence` (ADR-0149) and `PartyRoundEvidence` (counts, deception rate, engagement — no prompt, lie, drawing or name; ADR-0160 (`Documentation/ADRs/ADR-0160-pocket-party-games.md`) T15). *(Corrected 2026-10-14: this line named a `UnifiedDataProvider`, a type that does not exist.)*
- **Specialized Graph Ingestion & Topology Audits (ADR-0157 (`Documentation/ADRs/ADR-0157-specialized-server-graph-ontology-and-history-navigator.md`), ADR-0161 (`Documentation/ADRs/ADR-0161-unified-graph-layer-ontology-registry-and-viewer-kit.md`))**: `ServerGraphProducer` captures fleet topologies, live edge probes, and deployment lineages into `PocketGraphStore` with bi-temporal validity (`validFrom`/`invalidFrom`). These graph snapshots connect directly to `UnifiedGraphProducer` and are rendered in the operator cockpit through the shared graph viewer kit (`PocketCloudUI/GraphViewer/`, ADR-0161 (`Documentation/ADRs/ADR-0161-unified-graph-layer-ontology-registry-and-viewer-kit.md`)) to feed real-time infrastructure telemetry into `UnifiedLearning`. *(Corrected 2026-10-16: this line named a `FleetGraphView`, a type that does not exist.)*
- **Vector Decimation & Motion Gestures (ADR-0160 (`Documentation/ADRs/ADR-0160-pocket-party-games.md`))**: The < 8 KB per-sketch budget is enforced by `PartyStrokeSimplifier` and pinned by `PartyDrawingTests`; each scored round records the decimated vertex count it received (`PartyRoundEvidence.drawingPoints`). `PartyStrokeSimplifier` and `PartyMotionClassifier` are pure and emit nothing themselves: the raw-versus-decimated ratio and tilt latency are **not** recorded yet. The T04/T05 remotes now deliver drawings and tilts to the host through `PartyRoutes.intent`, which already logs one content-free `party.lan.intent` signal per accepted guest action. Record the two measurements at that seam, not inside the algorithms.
- **AI Feedback**: AI decisions (like routing, NPC generation, or semantic reviews) should emit traces (preferably via `DecisionTraceGraphProducer`) that can be reviewed in the PocketBusiness operator cockpit.
- **Resend Delivery & Webhook Audit Signals (ADR-0128 (`Documentation/ADRs/ADR-0128-resend-transactional-email-integration.md`), ADR-0148 (`Documentation/ADRs/ADR-0148-pocketbusiness-where-customer-zero-comes-together.md`) T13)**: Inbound webhook payloads from Resend pass through `ResendWebhookJournal` with Svix HMAC signature verification, strict timestamp replay-window protection, and idempotent deduplication by `message_id`. Delivery outcomes (delivered, bounced, complained) map to audit records in `ResendSendAuditStore`. Deliverability failures emit unified learning signals under fingerprint `resend_bounced_<domain>` and operator warnings under `resend.delivery.read`, closing the loop between sent communications and real-world inbox delivery.
- **Graph Federation Provenance Retention (ADR-0136 (`Documentation/ADRs/ADR-0136-pocket-graph-self-evolving-ontology.md`), ADR-0161 (`Documentation/ADRs/ADR-0161-unified-graph-layer-ontology-registry-and-viewer-kit.md`) T05)**: Federation loaders (`ADRFederationLoader`, `CodeFederationLoader`) retain original edge provenance—including source references, extractor identity, extractor version, derivation mode (`.inferred` vs `.extracted`), temporal validity (`validFrom`/`invalidFrom`), and confidence—rather than restamping imported edges. This enables deterministic graph shape audits and trustworthy retraction loops across federated subgraphs.
- **Commit-Gate Outcomes (Local Git Hook Standard (`Documentation/Standards/local-githook-standard.md`))**: Every pre-commit gate records its outcome as a `UnifiedLearningSignal`. A violation is `.error`, carries the scanner's guidance as `metadata["remediation"]`, and uses the stable fingerprint `pre-commit-<gate>-violation`. A bypass is recorded as one. `UnifiedLearningCoordinator.learnRemediationIfPossible` keeps the **first** recommendation per fingerprint, so a gate whose guidance is later corrected goes on replaying the old text in the 🧠 recall. The raw-colour gate is in that state on 2026-10-16 (Theme Colour and Type Standard (`Documentation/Standards/theme-colour-and-type-standard.md#flywheel-integration`)). Correcting a gate's guidance therefore needs a way to supersede the stored recommendation, not just a text edit.
- **Brand Resolution Signals (ADR-0083 (`Documentation/ADRs/ADR-0083-brand-and-showcase-contract.md`), ADR-0143 (`Documentation/ADRs/ADR-0143-one-source-of-colour-theme-enforced.md`) D5)**: A client brand that declines to apply is otherwise invisible: the operator sees our colours, and we see nothing. `BrandKitResolution.recordRejections` (PocketCloudCore) writes one durable signal per refused layer to `UnifiedLearningCoordinator`, under the fingerprint `brand_override_rejected`, carrying the app, the layer, the reason and the fallback source. Syntax refusals and contrast refusals take the same path: app surfaces resolve through `BrandKitResolver.resolveAudited`, so a contrast failure is refused at resolution and its reason names the pairing, the ratio and the bar. Call it when an operator loads or saves a brand, never from a view body. *(Corrected 2026-10-18: `BrandResolver.recordRejection`, named here before, has no callers and writes to `SharedUnifiedLearningAdapter`, an in-process ring that is never persisted.)*
- **Progressive Resolution (ADR-0115 (`Documentation/ADRs/ADR-0115-progressive-resolution-and-bounded-projection.md`))**: The rollup pyramid provides bounded projection over large signal volumes. Use rollup-aware queries when displaying aggregate metrics rather than scanning raw edges.

## ADR and Task Integration Rule

### Capture-to-creative evidence (ADR-0165)

ADR-0165 (`Documentation/ADRs/ADR-0165-app-capture-story-and-marketing-delivery.md`) proposes
one trace from existing screenshot/video capture evidence through story and brand
revisions to exact target artifacts, destination review and measured feedback.
Reuse typed marketing/social producers; record stage failures, hash mismatches,
missing audio, invalidated review, cache reuse and receipts through shared
providers. Store pixels/audio in local artifact custody, not graph/RAG payloads.
Unverified capture data mode remains unknown. Variant metrics retain sample size
and trust; a capture/render success is not publication or conversion evidence.

Every active ADR and task document that adds or modifies runtime behavior must consider its impact on the Unified Providers Flywheel. Do not close an ADR/task without ensuring the feature emits proper diagnostic and learning signals.
