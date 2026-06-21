# PocketCloud Roadmap

This roadmap reflects the June 13, 2026 state review from the main workspace and the official website.

---

## Recently Completed or Substantially Implemented

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

### Observability and Settings

- verify run history and UI
- observability hub settings tabs
- MCP tool execution logging
- shared log roots
- local diagnostic views

### Cross-Platform AI Hardening

- llama.cpp provider integration
- Swift 6.2 concurrency hardening
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

- llama.cpp provider hardening
- FoundationNetworking and platform stubs
- SwiftNIO server path
- package and build compatibility cleanup

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
