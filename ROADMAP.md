# PocketCloud Roadmap

This roadmap reflects the August 3, 2026 state review from the main workspace and the official website.

---

## Recently Completed or Substantially Implemented

### Cross-Platform Trust Pass and the Android Lane (ADR-0082)

- A task-by-task audit of the hybrid stack found and fixed real trust defects: a
  vendored fork had **silently disabled Metal GPU inference** while every gate
  stayed green (replaced with verified official artifacts), a test verb reported
  success over failed runs, and desktop "Online" indicators are now facts the
  core states rather than timers
- One shared Tauri shell now serves all eight apps; the **Android packaging
  milestone shipped 2026-08-02** — `pocket build android` produces both APK and
  AAB, and all eight apps build, install, and launch on Android emulators as
  remote-core clients of a paired node. On-device Android inference is
  explicitly out of launch scope; store distribution is the remaining step
- Windows parity work made the quality gates, test runner, and build bundles run
  on Windows itself

### Distributed Knowledge Mesh — Tier 1 (ADR-0080)

- Local knowledge processing with addressable derived snapshots, a single
  portability filter deciding what may leave a device, and
  **encrypted-by-default snapshot bundles**
- LAN sync groundwork with honest JSON contracts for the knowledge verbs

### Fleet Server Lifecycle and Presence (ADR-0084)

- Typed cloud-provider API client and a `pocket web server` verb family: tier
  inventory, snapshot, safe prod-clone with first-boot lockdown, lease/connect/
  reap with rolling spend ceilings
- `pocket web presence` and scheduled config audits — built after real incidents
  where most fleet sites were silently unmonitored and a missing port config
  caused a long-running crash loop
- The fleet now carries its own SSH identity instead of borrowing the
  workstation's

### Commercial Implementation Offer (ADR-0081)

- PocketCloud's first commercial offer — a fixed-price custom implementation
  delivered by Clever Coding as founding official implementation provider — is
  live, with prices served as operator-editable data rather than code constants,
  and a delivery gate that proves every advertised platform surface actually
  builds

### Test Integrity Hardening

- New scanners catch **test files no target compiles** and scan verbs that pass
  vacuously over zero files; several never-running suites were found and adopted
  or deleted — the standing lesson is that a green gate must be proven capable of
  failing

### PocketGamer Game Engine

- Voice-native branching adventures with a live AI Dungeon Master
- Procedurally generated dungeon crawls, endless Explore worlds, and an "invent a game" mode
- On-device classic board games (chess, checkers, and more) with device-to-device multiplayer
- Swift-native engine with a build-and-remix scene editor

### Cross-Platform Surface and Verification You Can See

- One hybrid web surface shared by all seven Apple apps, PocketDesktop, and the web
  fleet — repaired end to end so a screen written once renders on iPhone, iPad, Mac,
  Windows, and Linux
- `pocket dev apple build-app --simulator --scene <name>` builds an app, installs it
  on a simulator, opens a named screen, and captures it — "it compiles" is not the
  same claim as "it renders"
- Deterministic pre-commit and build gates for hardcoded paths, telemetry
  violations, AML contracts, platform hygiene, and hybrid bundle integrity
- Customer Zero inventory: 152 GitHub repositories and 50 App Store apps captured as
  durable, regenerable snapshots

### Ecosystem Accessibility and Adaptive Layout

- VoiceOver labels, per-app accessibility audit suites, and regression tests across all seven apps
- SwiftUI adaptive layout policy so every app reflows from iPhone to iPad to Mac
- App Feature Catalog SSOT (ADR-0061) and a Skill Taxonomy curriculum in PocketLearning (ADR-0062)
- Standalone installer retired — machine setup now lives inside every app

### Blueprint Apps and Public Beta Framing

- Renamed public app taxonomy to **Blueprint Apps**
- Renamed `PocketCloudHub` to `PocketHub`
- Renamed `PocketGameEngine` to `PocketGamer`
- Added refreshed app icons, splash screens, and screenshot sets
- Added public TestFlight CTAs across the site

### App Store Storefront Automation

- ADR-0048 metadata source of truth
- generated app pages and website injection
- screenshot capture for iPhone/iPad
- screenshot evaluation and feature cataloging
- App Store Connect push/read tooling

### Web Fleet

- Swift-native `pocket web` commands
- deploy, sync, rollback, backup, jobs, logs, analytics
- secure multi-user local admin control plane
- lead capture/contact module
- APNs push infrastructure
- live presence and operator messaging
- bot/AI traffic intelligence
- `/robots.txt` and `/llms.txt` shared routes
- local mirror store, job runner, and SEO Pilot

### Membership and Subscriptions (ADR-0065)

- a single supporter tier ($9.99) with a two-week free trial as the model
- StoreKit runtime for the Apple apps (`PocketCloudSubscriptions`)
- shared support/paywall UI with an A/B paywall-name experiment (`PocketCloudSubscriptionsUI`)
- canonical web membership bridge in `web-lib` (MembershipController + Stripe
  service + subscriber accounts), driven by a Swift-generated payment projection —
  the full web loop (checkout → webhook → entitlement) is sandbox-verified
- the Apple side is still in progress: server entitlement unification, a
  PocketMind pilot, then the seven-app rollout and App Review submission

### Observability and Settings

- verify run history and UI
- observability hub settings tabs
- MCP tool execution logging
- shared log roots
- local diagnostic views

### Cross-Platform AI Hardening

- llama.cpp provider integration
- Swift 6.3 concurrency hardening
- Linux/Windows cross-platform support on AIStack and Core
- platform checks before app builds
- MLX simulator and daemon fixes

---

## Active Open-Source Launch Work

- keep the preview repo synchronized with the official site messaging
- finish GitHub community files and contribution path
- separate public claims from private implementation details
- keep app names, TestFlight framing, and source rollout copy consistent
- publish accurate verification evidence without overclaiming stale baselines
- continue removing generated, local, and machine-specific artifacts from public-facing repos

---

## Near-Term Engineering Focus

### Source Release Readiness

Prepare the main workspace for public source access:

- license and contribution model
- secrets and local path audit
- issue templates and support boundaries
- public build instructions
- reproducible `pocket build` and `pocket system verify` path

### Cross-Platform Expansion

Apple Silicon remains the lead platform, and recent work has successfully extended AIStack/Core to Linux and Windows:

- llama.cpp provider hardening (now on verified, Metal-enabled artifacts)
- FoundationNetworking and platform stubs
- SwiftNIO server path
- package and build compatibility cleanup
- take the Android remote-core lane (ADR-0082, packaging shipped) through store
  distribution, and verify in-process inference on the Windows/Linux desktop
  shells on real non-Apple hosts

### Web Fleet Productization

Turn the web fleet into a clearly documented public capability:

- admin setup docs
- data lifecycle and backup model
- lead/contact funnel docs
- bot/AI traffic intelligence docs
- PocketBusiness control-plane story

---

## Long-Term Direction

- multi-device local AI coordination
- federated on-device learning
- broader provider/plugin ecosystem
- richer AI-native project planning
- community-built blueprint apps

---

## Non-Negotiables

- Local-first remains the default.
- Cloud remains opt-in.
- Privacy claims must map to architecture.
- Public docs must distinguish shipped, beta, and roadmap work.
- Swift-native automation remains preferred over shell-script sprawl.
