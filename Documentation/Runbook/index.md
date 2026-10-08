---
id: "doc.pocketcloud.runbook.index"
title: "PocketCloud Build, Run, and Verify Runbook"
types: ["implementation-guide", "reference", "source-of-truth"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "build", "verification", "android", "apple", "hybrid", "deploy", "disk-space", "git-sync"]
owners: ["@steven"]
version: "1.12.0"
last_updated: "2026-10-07T00:00:00Z"
last_reviewed: "2026-10-07"
is_archived: false
---
# PocketCloud Build, Run, and Verify Runbook

The commands that actually build, run, and publish PocketCloud, in the form they
were last run successfully. **Every command is issued from the workspace root
unless stated otherwise** — several resolve config relative to the working
directory and fail confusingly from a subdirectory.

A green build is build evidence, not behavior evidence. Behavior is proven by
running the surface and capturing what it did — which is why every section here
ends in something that *runs*, not something that compiles. This is the Swift 6.3
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

---

## Freshness Table

Freshness is tracked per section, because the whole document is rarely exercised at
once. State what you re-ran when you edit here:

| Section | Last Actually Run |
|---|---|
| ops.md (git sync / hooks / scaffold) | 2026-10-07 — path-targeted sync, nested `web-fleet` scan, stale-hook upgrade, Standards+full Runbook version refresh from CLI bundle; WorkspaceScaffold tests green |
| web-fleet.md | 2026-08-18 — full 9-site redeploy, verify + presence green |
| apple-apps.md | 2026-10-07 — iPhone Duo prepare audit (size-class shell OK; Xcode 27.0 / SDK 27.0 — Duo sim needs 27.1+) |
| verification-and-gates.md / build-tiers.md | 2026-09-08 — bronze ×4, gold attempts, ADR-0117 lease proved live |
| knowledge-console-recovery.md (disk) | 2026-08-24 — recovering the 97 GB snapshot incident |
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

### Storage Root Resolution

PocketCloud probes each candidate data root — create, write, read back, enumerate
— and uses the first that answers, rather than the first that merely exists. On a
machine where the App Group container is TCC-denied, `stat` succeeds while every
real operation fails, so an unprobed ladder would select a root nothing can use
and every store beneath it would fail silently at exit code 0.

Override with `POCKETCLOUD_HOME` (read live, and deliberately not probed — an
operator naming a directory is authoritative). `MLX_HOME_OVERRIDE` still works
for test-fixture isolation.
