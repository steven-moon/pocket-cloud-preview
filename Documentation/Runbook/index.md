---
id: "doc.pocketcloud.runbook.index"
title: "PocketCloud Build, Run, and Verify Runbook"
types: ["implementation-guide", "reference", "source-of-truth"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "build", "verification", "android", "apple", "hybrid", "deploy", "disk-space", "git-sync"]
owners: ["@steven"]
version: "1.15.3"
last_updated: "2026-10-23T01:15:00-06:00"
last_reviewed: "2026-10-23"
is_archived: false
---
# PocketCloud Build, Run, and Verify Runbook

The commands that actually build, run, and publish PocketCloud, in the form they
were last run successfully. **Every command is issued from the workspace root
unless stated otherwise** — several resolve config relative to the working
directory and fail confusingly from a subdirectory.

A green build is build evidence, not behavior evidence. Behavior is proven by
running the surface and capturing what it did — which is why every section here
ends in something that *runs*, not something that compiles. This is the Swift 6
standard's Platform Policy #9 applied to a runbook: a platform is "supported"
only when its build is actually verified.

---

## Modular Topic Index

To ensure maintainability and strict adherence to the `<500 lines` standard, this
runbook is partitioned into focused operational topic guides:

1. [**Build Tiers, Leases & Requirement Gates**](build-tiers.md)
   - Bronze, Silver, Gold compilation composites and cancellation
   - Graph tiers (planned ADR-0113 T19)
   - Reusable build lanes, conflict-level leases, and visible queue (ADR-0117)
   - Requirement-gated local-first execution (ADR-0126)

2. [**Apple Apps, Simulators, Storefront & Ship**](apple-apps.md)
   - Consolidated Xcode project generation (`project.yml`)
   - Inter-app URL schemes (`CFBundleURLTypes`) & fleet queries (`LSApplicationQueriesSchemes`)
   - iOS / tvOS Simulator execution and scene jumping
   - iPhone Duo readiness (size-class shell, SDK 27.1 note, test matrix)
   - Storefront visual evidence pipeline & screenshots (ADR-0048)
   - Live demo modes and launch arguments (ADR-0102)
   - Signed-in verification with loopback auth tickets (ADR-0127)
   - App Store Connect API keys and distribution `ship`

3. [**Android Apps & Hybrid (Tauri/Vite) Bundles**](android-and-hybrid.md)
   - Cross-compiling Swift for Android from Windows hosts (ADR-0103)
   - Toolchain prerequisites and AVD emulator execution
   - Google Play store lane and release promotion (ADR-0106)
   - Hybrid web-lib bundles and Tauri desktop/mobile shells

4. [**Verification, Quality Gates & Automated ADR Sync**](verification-and-gates.md)
   - Local warmup, RAG index refresh, and exhaustive system verify
   - Reading test logs and diagnosing test-target breaks
   - Pre-commit, pre-push, and commit-msg quality gates
   - Automated ADR Staging, Validation & Push (`pocket ops git sync`)

5. [**Operations & Git Runbook**](ops.md)
   - Complete `pocket ops` domain: Git sync, git hooks, orchestrator, and disk management
   - Folding every branch into develop and pruning what is merged (`git sync --consolidate`), and
     keeping a peer's half-written work out of a sync (`--exclude`, file ages, `✍️ in flight`)
   - Syncing only your own files in a shared checkout (`git sync --only <path>`, `--only-from`)
   - Commit-msg traceability rules, character length constraints, and bypass flags
   - Shared build-root leases, task list queues, and background task presets
   - Submodule synchronization and SSH environment troubleshooting

6. [**Web Fleet Deployment & Server Operations**](web-fleet.md)
   - 9-site web fleet deployment, verify, and rollback
   - Environment bundles, encryption, and per-machine config
   - Vultr fleet server tiers, snapshots, and clone divergence (ADR-0084)
   - Stage OS upgrades and transactional mail via Resend (ADR-0128)

7. [**Knowledge Systems, Console Contracts & Recovery**](knowledge-console-recovery.md)
   - RAG status, namespace routing, and Gmail Second Brain sync
   - Console contracts (ADR-0112): heartbeats, silence rules, and redaction
   - Free disk space management, runtimes prune, and archive sweeps
   - Comprehensive table of known failure classes and recovery fixes

8. **Testing, Simulator Verification & Visual Evidence** (`Documentation/Runbook/testing.md`)
   - Ladder of operations: unit/package test before app compilation
   - Targeted suite testing (`pocket quality test`) and full package runs (`pocket build test`)
   - The MCP server's wire contract and tools: package suites, then a stdio probe of the installed
     server for both protocol eras (ADR-0164)
   - Simulator app execution with direct surface jumping via `-UITestScene`
   - Automated screenshot capture with settle budgets (`--settle`, `--screenshot`)
   - Single source of colour theme auditing (ADR-0143) and WCAG AA contrast scanning
   - Party remote rejection/retry JavaScript tests and browser accessibility checks (ADR-0160 T16)
   - Authenticated loopback ticket verification (ADR-0127) and local-only fleet data testing

---

## Freshness Table

App media creation has a dedicated capture, Story Maker and delivery guide (`Documentation/Runbook/app-media-creation.md`)
and ADR-0165 (`Documentation/ADRs/ADR-0165-app-capture-story-and-marketing-delivery.md`).
The post-pause integration review (`Documentation/ADRs/Tasks/0165/reconciliation-review.md`)
records parallel command additions, strict ingestion reconciliation and
delivery gates.
The T08 artifact-evidence slice (`Documentation/ADRs/Tasks/0165/T08.md`) replaces guessed
file properties with shared probes; its task records runtime verification and
the remaining custody/receipt boundary.
The T10 draft screenshot slice (`Documentation/ADRs/Tasks/0165/T10.md`) verifies copied and
reused bytes and resolved path containment before saving project documents.
Conflicting destinations require explicit resolution; ingestion preserves them.

T11's real-media rehearsal (`Documentation/ADRs/Tasks/0165/T11.md`) records capture, brand
composition, narration, explicit edits, render and delivery evidence, including
operator interventions still required before unattended marketing automation.

The completion audit (`Documentation/ADRs/Tasks/0165/completion-audit.md`) reviews
workflow/staging evidence and records source blockers before
adoption. T12 gameplay capture (`Documentation/ADRs/Tasks/0165/T12.md`) continues verification.
T13 composition (`Documentation/ADRs/Tasks/0165/T13.md`) has passing StoryKit tests and a portable render.
T14 preparation (`Documentation/ADRs/Tasks/0165/T14.md`) repairs missing included-app version inputs found during the app build.

The T09 custody and receipt slice (`Documentation/ADRs/Tasks/0165/T09.md`) rehashes render-used
catalog media and separates input reuse from output freshness. Its task records
the receipt schema change, runtime evidence and remaining render-origin limits.
It reuses storefront screenshot records for social stills, carousels, thumbnails
and films alongside simulator video. Workflow verbs (`pocket story ingest`,
`capture`, `compose`, `variants`, `delivery`, `workflow`) are operational with
durable checkpoints and policy gates.

Freshness is tracked per section, because the whole document is rarely exercised at
once. State what you re-ran when you edit here:

| Section | Last Actually Run |
|---|---|
| testing.md (MCP timeout regression) | 2026-10-24 — 72-hour Git and timeout audit (`Documentation/ADRs/Tasks/0164/mcp-timeout-audit.md`). Explicit concurrent dispatch, bounded/cancellable markdown listing and isolated RAG probes pass 25 focused tests. CLI `1.0.26.10.24.4` installed (`77728C8A`). Exhaustive local-first verification (`E27C7EF4`): 116 operation rows pass, 13 profile skips, two environment warnings, zero failures; release gate passes. All 28 failures from the October 23 report now pass. |
| testing.md (production trivia pack) | 2026-10-23 — owner-approved `trivia-core@2026.9.30.2` bundles 15,483 questions in fourteen shards. Package verification passed 84 tests (`BACB5322`); four Release UI tests passed (`845C39E5`). Normal simulator computer use verified all four modes with no launch flags. TestFlight 1.0.0 (31) is VALID, external beta review APPROVED, assigned to `initial-beta-testers` (`329AD5C1` upload). Physical gameplay remains blocked by runner signing (`0xe8008018`). Exhaustive workspace verification failed on 28 operations (`C79B3FB9`); see testing.md for scope and evidence. |
| app-media-creation.md / testing.md (app capture & Story Maker ingestion) | 2026-10-23 — ADR-0165 T01–T07 verified: `StorefrontCaptureVerificationTests` (6/6), `StorefrontCaptureIngestorTests` (5/5), `AppCapturePlannerTests` (4/4), `SimulatorVideoRecorderTests` (3/3), `StoryCaptureRunnerTests` (2/2), `StoryComposerTests` (4/4), `StoryComposeCommandTests` (6/6), `TargetDeliveryProfileTests` (5/5), `StoryDeliveryCommandTests` (7/7), `StoryWorkflowCheckpointTests` (4/4), `StoryWorkflowOrchestratorTests` (4/4), `StoryWorkflowCommandTests` (6/6), `DeliveryStagingPackageTests` (5/5), and `StoryDeliveryStageCommandTests` (4/4). Shared capture records, video recorders, branded story composer, delivery profiles, and `StoryWorkflowOrchestrator` deployed with durable checkpoints (`26.10.21`), resume support, exact-hash destination review, stale review refusal, publication reconciliation, profile-scoped cache key invalidation (T07), and CLI commands `pocket story ingest`, `capture`, `compose`, `variants plan|render`, `delivery validate|stage`, and `workflow run`. Platform check passed (session `F0FD1C6A`). |
| testing.md (leads & vendor classification) | 2026-10-21 — `pocket build app PocketBusiness` compiled and installed cleanly for macOS and iOS (session `A931AF67`). `CommunicationDraftingTests` (26/26 passing), `LeadFirstReplyTests` (12/12 passing), `AdminLeadPipelineTests` (15/15 passing), `LeadOperatorVerdictLabelsTests` (4/4 passing), and `BusinessHighlightsTests` (8/8 passing) verified. Section #1 communication-first layout verified with Second Brain / Archive relationship context banner, automated vendor dismissal dispatch (`KnownVendorDomainStore` memory, `.vendor` verdict, and `.archived` transition), `effectiveKind` query pipeline reconciliation (auto-excluding vendors from Prospects), toolbar/footer bulk screened archive, and `pocket quality platform-check` passed with 0 warnings/errors. |
| testing.md (app unit tests) | 2026-10-19 — `pocket dev apple test --project Apps/PocketMind/PocketMind.xcodeproj --scheme PocketBusiness-macOS --destination platform=macOS --only-testing PocketBusinessTests-macOS/ConversionFunnelLogicTests` executed 21 tests and passed. The `--package-path Apps/PocketMind/PocketBusiness` form the section listed before finds no package. |
| testing.md (party pack CLI) | 2026-10-16 — `PartyPackCommandTests` 3/3 (`7BA7C25D`), CLI installed through `pocket build cli` (`9172C39D`), and installed-command checks passed: four valid example items exit 0; malformed/missing files exit 1. Host import and family toggle remain pending. |
| testing.md (party pack foundation) | 2026-10-16 — `pocket quality test 'Party\|QuipClashTests' --package-path Packages/Kernel/GameStack` passed 115 tests (94 engine/ambience, 21 LAN), including four new `PartyPackTests` for atomic validation, explicit maturity, family/mode selection and private-content-free provider evidence. Session `718A9868`. CLI/import and host pack selection remain pending. |
| testing.md (game play-throughs / stale-build crashes) | 2026-10-16 — `BoardPlaythroughUITests` 6/6 on iPhone 17 Pro (Tic-Tac-Toe, Connect Four, Reversi, Nim and Chess to a result; Checkers 120 taps without a stall); `MathAnswerTypesUITests` answers a choice, an order, pairs and a number line; card flows 12/12 with the Go Fish play-through and audit. Diagnostic step 5 added: a SwiftUI-internal launch crash after a view edit is a stale incremental build, so touch the composing files before debugging. Math endings: `MathFinishUITests` 2/2 (Daily and Practice to their finish) and `QuickFireUITests` 1/1, both on the shared result card. |
| testing.md (trivia table games) | 2026-10-20 — Topic Wheel and Question Ladder (ADR-0149 T19), which no app had imported, open from the Trivia page and `-UITestScene trivia-wheel` / `trivia-ladder` over the synthetic `TriviaTableDemo` fixture (`-UITestTriviaTableFixture`, `-UITestFastPace`). On the iPhone 17 simulator (iOS 27.0) `TriviaTableFlowUITests` 2/2 (the wheel to a winner with AI turns; the ladder past the safe rung to a miss that keeps 1,000) and the trivia-table accessibility audit with contrast pass. Package suites: engine 32/32, TriviaUI 23/23, GameUI 20/20. `name=iPhone 17 Pro` finds no device when the newest runtime has none: name the OS or pass `id=<UDID>`. |
| testing.md (card computer-use / replay follow-up) | 2026-10-15 — all ten tables interacted with on iPhone 18 Pro / iOS 27.0; exact completion limits in testing.md. Blackjack replay defect fixed and rechecked through a second hand; runtime tests 7/7 and focused replay UI test 1/1. Remote submission JavaScript tests 4/4. |
| testing.md (suites / simulators / visual evidence) | 2026-10-19: Topic Wheel & Question Ladder on the trivia table verified with `TriviaTableHostTests` (4/4 passing, session `18B75511`); `NumberTileLineTests` @MainActor isolation fix verified in GameUI (session `6198C5B3`); party pack family mode dealing and toggle verified with `PartyPackTests` (6/6 passing); the UI test that opens every PocketBusiness tab on iPhone (a tab bringing its own NavigationStack drops the push), PocketHub's graph tabs on one explorer and `-UITestSegment <domain>`, Specialized Marketing Graph (ADR-0162) and Company Brain Graph (ADR-0163) verified; `pocket knowledge graph publish --domain brain` writes exclusively to host data directory (`~/.pocketcloud/graph/brain.snapshot.json`), `MarketingGatesTests` 14/14 passed, graph shape audits clean across all 8 registered domains (`workspace`, `adr`, `code`, `git`, `server`, `builds`, `marketing`, `brain`); graph surfaces as nodes (rebuilt on this device or published on it, and which one answered; primary-node and peer sync marked not built), the Relationships Map screenshots for PocketBusiness and PocketMind with the synthetic `.example` sample, signing a node in through `pocket auth`, and the brain root-mirror defect fixed. 2026-10-14: Specialized Graphs (ADR-0157 Server Graph, ADR-0161 Ontology, Alignment, Retraction Guard, Manager, CLI tree, and PocketCloudUI Graph Viewer Kit; GraphLayerBudgetTests pinned), Federation Provenance, Resend Webhook Audit, and Pocket Party Games (ADR-0160 Bluff Party, Quip Clash duels & pairing, PocketCloudPartyUI tvOS decoupling, LAN party listener, drawing decimation, motion classifier) unit suites and CLI commands verified green; simulator UITestScene targets expanded; party UI tests re-run on iPhone (`PartyFlowUITests`) and on the Apple TV by Siri Remote (`PartyRemoteUITests`, which found the Hide controls out of reach), plus a LAN Doodle round with a phone drawing in Safari |
| ops.md (git sync / hooks / scaffold) | 2026-10-15 — ADR-0159 finished (T06–T14). The sync engine suites (`pocket quality test ADRCommitWorkflowEngine --package-path Packages/Kernel/Core`, 43 XCTest + 27 Swift Testing) are green. A CLI built from the branch ran `--preview --consolidate` and `--exclude` against the main checkout with every ref unchanged, marked a peer's minutes-old edits `✍️ in flight`, and named a live `index.lock` instead of failing on it. 2026-10-13: 49 ADR clusters pushed through pre-commit gates |
| web-fleet.md | 2026-08-18 — full 9-site redeploy, verify + presence green |
| apple-apps.md | 2026-10-21 — PocketBusiness iOS, macOS, and tvOS apps uploaded to App Store Connect / TestFlight via Apple runbook (`pocket dev apple ship` / `export` / `appstore`). iOS: v1.0.0 build 28 uploaded (session `BFBE2A28`, clean bundle secret audit). macOS: v1.0.0 build 16 exported and uploaded (clean bundle secret audit, headless ASC key export). tvOS: v1.0.0 build 3 uploaded (session `8CF62A51`, clean bundle secret audit, signing deferred to export per runbook). `KnownVendorDomainStore` decoupled from tvOS platform exclusion. |
| verification-and-gates.md / build-tiers.md | 2026-09-08 — bronze ×4, gold attempts, ADR-0117 lease proved live |
| knowledge-console-recovery.md (disk) | 2026-10-14 — Gmail thread repair (--repair-thread) and resilient lead source reconciliation |
| android-and-hybrid.md | 2026-08-15 (Windows cross-compile), otherwise 2026-08-03 |

---

## 0. Cold Start

`pocket` does not exist on a fresh clone, so the deprecated bootstrap script is
the only option for the first command. After it, install the gatekeeper git hooks and use `pocket build …`.

```bash
./scripts/build/build.swift cli    # installs ~/.local/bin/pocket + per-machine MCP configs
git submodule update --init --recursive pocket-cloud-web-workspace
pocket ops git-hook install         # hooks + Documentation scaffold/version refresh
pocket knowledge document generate-code-docs . --recursive   # AML shadow index

# Activate / refresh another repo (client, fleet leaf) from this machine:
# pocket ops git-hook install /path/to/repo
# pocket ops git sync --preview /path/to/repo

# Only if you will touch the web fleet. `web-fleet/` is GITIGNORED — it carries
# deploy secrets — so a clone does not have it and nothing in the build says so.
# Without this, every `pocket web …` command in web-fleet.md fails on missing config.
git clone git@github.com:steven-moon/pocket-cloud-web-config.git web-fleet/pocket-cloud-web-config
# …then unpack the env bundle before deploying anything.
```

`git-hook install` / `git sync` seed missing ADRs/Standards/Runbook and
**version-refresh** the managed Standards + full Runbook from the primary tree
or the CLI resource bundle (`WorkspaceScaffold/` + `Runbook/` next to `pocket`).

The shadow-index command is not optional if you want an agent to have context.
`.index-aml/` is **generated and not tracked**, so a clone has none. It covers
Swift *and* Markdown, takes ~37s for the whole repository, and is idempotent —
unchanged source produces no write, so re-running is free and leaves no diff.
It also repairs `.gitignore` when a repository is missing the entries for
generated PocketCloud state. Related verbs:

```bash
pocket knowledge document prune-code-docs        # entries whose source is gone (dry run; --apply)
pocket system workspace shadow-index --path <f>  # what the index knows, and whether it is stale

# Unified Graph Layer (ADR-0161)
pocket knowledge graph build --domain <domain>   # ingest domain graph (workspace, git, server, builds, marketing)
pocket knowledge graph ontology --domain <domain> # inspect ontology rules & signatures
pocket knowledge graph audit --all               # validate graph shape & catch drift
pocket knowledge graph publish --domain <domain> # export canonical SHA-256 snapshot
pocket knowledge graph join --node <node-id>     # execute cross-domain identity join
pocket knowledge graph export --format mermaid --domain <domain> [--focus <node-id>] [--diagram mindmap]
                                                 # one node's bounded neighbourhood as Mermaid text (MCP: graph export_mermaid)
pocket knowledge graph churn                     # identify change hotspots & risk scores
pocket knowledge graph authors                   # report code ownership & bus factor
```

`git-hook install` is also the **activation** that registers the repo in the
machine-wide workspace registry at `~/.pocketcloud/workspace_registry.json`
(override with `POCKETCLOUD_WORKSPACE_REGISTRY`; ADR-0085). A load merges every
location the registry has ever lived in.

Migration is **additive** — the other copies are kept, which is what makes them
redundancy rather than debris. That is only safe because deletions are recorded
as tombstones carrying a monotonic revision; without them a merge is a union
with no way to express absence, so every delete would be re-imported from any
copy that still held it.

An entry is present after a merge iff its revision is **higher** than the highest
tombstone for its key, so re-registering something you deleted is deliberate and
wins. Wall-clock time is never the comparator; clocks disagree between machines
and jump backwards.

A repository is identified by a set of **fingerprints** (git root commits, git
remotes, declared tenant) rather than by its path, so deleting and re-cloning it
elsewhere rebinds the existing registration instead of orphaning it:

```bash
pocket workspace identity --relations
```

Agents and CI want a non-interactive shell. Set the doctor escape, but **not**
the credential-store one — skipping the credential store sabotages the test
suite rather than speeding it up:

```bash
export POCKETCLOUD_SKIP_DOCTOR=1
```

### Gameplay and Composition Implementation Status

For ongoing ADR-0165 gameplay/composition work, see
T12 (`Documentation/ADRs/Tasks/0165/T12.md`) and T13 (`Documentation/ADRs/Tasks/0165/T13.md`).
The gameplay/composition CLI rebuilt as `1.0.26.10.23.3`; live gameplay remains
pending. Build/test snapshots allow
continued source editing while gates run; shared leases still serialize compiler
resources. Preserve the gate's snapshot and caches. Verification and publication
remain distinct milestones.

### Storage Root Resolution

PocketCloud probes each candidate data root — create, write, read back, enumerate
— and uses the first that answers, rather than the first that merely exists. On a
machine where the App Group container is TCC-denied, `stat` succeeds while every
real operation fails, so an unprobed ladder would select a root nothing can use
and every store beneath it would fail silently at exit code 0.

Override with `POCKETCLOUD_HOME` (read live, and deliberately not probed — an
operator naming a directory is authoritative). `MLX_HOME_OVERRIDE` still works
for test-fixture isolation.
