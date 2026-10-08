---
id: "doc.pocketcloud.runbook.ops"
title: "Operations & Git Runbook"
types: ["implementation-guide", "reference", "source-of-truth"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "ops", "git", "git-sync", "git-hooks", "orchestrator", "disk-space"]
owners: ["@steven"]
version: "1.17.0"
last_updated: "2026-10-07T00:00:00Z"
last_reviewed: "2026-10-07"
is_archived: false
---
# Operations & Git Runbook

[← Back to Runbook Index](index.md)

SSOT for `pocket ops` git sync, gatekeeper hooks, build leases, and disk ops.
Other Runbook topics link here for sync/hook detail rather than duplicating it.

---

## 1. ADR Git Sync (`pocket ops git sync`)

Clusters dirty files by ADR/task, stages, commits, and optionally pushes —
deterministically, no LLM. Defaults to the **current working directory**.

```bash
# Preview (CWD)
pocket ops git sync --preview

# Another repo — relative or absolute (positional; --workspace is an alias)
pocket ops git sync --preview ../clients/wanderlore/wanderlore
pocket ops git sync --preview /absolute/path/to/repo
pocket ops git sync --preview --workspace ~/GitRepo/clients/wanderlore/wanderlore

pocket ops git sync --preview --json
pocket ops git sync --push
pocket ops git sync --push --default-adr ADR-0065 --default-task T10
pocket ops git sync --push --no-self-heal

# Extra nested scan roots (repeatable). Default when omitted: web-fleet
pocket ops git sync --preview --scan-folder web-fleet --scan-folder other-fleet

pocket ops git status [--json] [<path>]
```

### What sync does on every run (target + nested dirty repos)

1. **Hooks** — if PocketCloud shims are missing **or** behind the current
   contract version → `pocket ops git-hook install <repo>` (upgrade in place).
2. **Documentation refresh** — version-scans managed files; overwrites when the
   source frontmatter `version:` is newer. Local copies already ahead of source
   are left alone. Structural scaffold (ADR-0001, Tasks/0001, READMEs) is
   create-if-missing only.
3. **Nested scan** — git repos at depth 1–2 under each scan folder (defaults:
   `web-fleet`, `pc-port-repos`) that are dirty.
4. **Agent docs** — create missing `README.md` / `AGENTS.md` / `CLAUDE.md` from
   `Documentation/Templates/<platform>/`, or inject/refresh the
   `<!-- pocketcloud:docs-index -->` section pointing at Runbook + Standards.
5. **Submodules** — dirty gitlinks (e.g. `pocket-cloud-web-workspace`) sync
   before the parent updates the pointer.
6. **Commits** — subjects `<type>(ADR-NNNN): <summary> (TXX)` with `≤ 72` chars.

### Managed Documentation (version-refreshed)

| Kind | Paths under `Documentation/` |
|---|---|
| Standards | `Standards/swift-6-3-implementation-standard.md`, `Standards/unified-providers-flywheel.md` |
| Runbook (full primary set) | `Runbook/index.md`, `ops.md`, `build-tiers.md`, `verification-and-gates.md`, `apple-apps.md`, `android-and-hybrid.md`, `web-fleet.md`, `knowledge-console-recovery.md` |

**Source order:** live primary workspace (when resolvable and not the destination)
→ **CLI bundle** next to `~/.local/bin/pocket`
(`WorkspaceScaffold/`, `Runbook/`, `Templates/`) → Common module fallback.

### Repo roles & templates

| Role | Detection | Template overlay |
|---|---|---|
| `pocketcloud-native` | `.pocketcloud/tenant.json` or path under `clients/` | `Templates/roles/pocketcloud-native/` |
| `pocketcloud-port` | path under `pc-port-repos/` | `Templates/roles/pocketcloud-port/` (assets → PocketLearning/PocketGamer rebuild) |
| `other` | everything else | `Templates/roles/other/` |

Platform folders: `ios`, `android`, `web`, `mixed`, `other-code`, `other-non-code`
under `Documentation/Templates/`.

Initial port SKUs (gitignored parent `pc-port-repos/`): Mathlete `mathlete-{ios,jr-ios,jr-mac,mac}`,
FirstWords `first-words-{ios,mac}`.

| Nested scan | Behaviour |
|---|---|
| Default | `web-fleet` + `pc-port-repos` (depth ≤ 2) |
| Extend | Repeat `--scan-folder <name>` |
| Disable | Pass a non-existent name, or `nestedRepoScanFolders: []` in engine tests |

Preview labels nested repos `hooks ✅` or `hooks ⚠️ will install/upgrade`.

### How changes are clustered

1. In-diff tags (`// ADR-XXXX (TXX)`) and AML front-matter (`adr: "ADR-XXXX"`).
2. Task ledgers under `Documentation/ADRs/Tasks/<NNNN>/`.
3. Subsystem matrix (examples): LeadVetting → ADR-0147, MLX → ADR-0010,
   Storefront → ADR-0048, Subscriptions → ADR-0065, Trivia → ADR-0149,
   StoryKit → ADR-0091, Flywheel → ADR-0050, GitSync → ADR-0075,
   PocketBusiness → ADR-0148.
4. Nested repos → submodules → parent.
5. Unmapped files emit flywheel `adr_mapping_gap` (use `--default-adr` /
   `--default-task` when previewing a thin client).

---

## 2. Git Hooks (`pocket ops git-hook`)

Swift shims call `pocket ops git-hook <verb>`. Same code path by hand or via Git.

```bash
pocket ops git-hook pre-commit                 # diff-scoped gates (~1s typical; no build)
pocket ops git-hook pre-push                   # fast (<30s)
POCKETCLOUD_FULL_PREPUSH=1 git push            # deep pre-push
pocket ops git-hook commit-msg <file>
pocket ops git-hook status                      # contract version per managed shim
pocket ops git-hook install                     # this workspace
pocket ops git-hook install /path/to/other/repo # activate / upgrade another repo
```

`install` and `git sync` both upgrade **stale** shims (not only missing ones) and
run the Documentation version refresh. Prefer `install` when you only need
activation (also registers the workspace and starts background RAG indexing).

### Commit-msg traceability

| Rule | Requirement | On failure |
|---|---|---|
| ADR | `ADR-NNNN` matching `Documentation/ADRs/` | reject |
| Task | `(TXX)` / `(PXX)` in `Documentation/ADRs/Tasks/<NNNN>/` | reject |
| Length | subject `≤ 72` characters | warn |

Escapes: `ADR-Exempt: <why>` trailer → `POCKETCLOUD_SKIP_ADR_REF=1` →
`git commit --no-verify` (last resort).

### Pre-commit bypasses (diff-scoped gates)

| Gate | Standalone | CLI bypass | Env |
|---|---|---|---|
| Paths | `pocket quality scan paths` | `--skip-path-scan` | `POCKETCLOUD_SKIP_PATH_SCAN=1` |
| Secrets | `pocket quality scan secrets` | `--skip-secret-scan` | `POCKETCLOUD_SKIP_SECRET_SCAN=1` |
| Security | `pocket ops git-hook security` | `--skip-security-scan` | `POCKETCLOUD_SKIP_SECURITY_SCAN=1` |
| Flywheel | `pocket quality scan flywheel` | `--skip-flywheel-scan` | `POCKETCLOUD_SKIP_FLYWHEEL_SCAN=1` |
| AML | `pocket quality scan aml` | `--skip-aml-scan` | `POCKETCLOUD_SKIP_AML_SCAN=1` |
| Platform | `pocket quality platform-check` | `--skip-platform-scan` | `POCKETCLOUD_SKIP_PLATFORM_SCAN=1` |
| Standards | `pocket quality scan standards` | `--skip-standards-scan` | `POCKETCLOUD_SKIP_STANDARDS_SCAN=1` |
| Test targets | `pocket quality scan tests` | `--skip-test-target-scan` | `POCKETCLOUD_SKIP_TEST_TARGET_SCAN=1` |
| Customer names | `pocket quality scan customer-names` | `--skip-customer-name-scan` | `POCKETCLOUD_SKIP_CUSTOMER_NAME_SCAN=1` |
| Brands | `pocket quality scan brands` | `--skip-brand-scan` | `POCKETCLOUD_SKIP_BRAND_SCAN=1` |
| Terminology | `pocket quality scan client-terminology` | `--skip-client-terminology-scan` | `POCKETCLOUD_SKIP_CLIENT_TERMINOLOGY_SCAN=1` |
| Streams | `pocket quality scan streams` | `--skip-stream-scan` | `POCKETCLOUD_SKIP_STREAM_SCAN=1` |
| CLI flags | *(gate only)* | `--skip-cli-flag-scan` | `POCKETCLOUD_SKIP_CLI_FLAG_SCAN=1` |
| Imports | *(gate only)* | `--skip-import-scan` | `POCKETCLOUD_SKIP_IMPORT_SCAN=1` |
| Blast radius | `pocket knowledge graph impact <mod>` | `--skip-impact-scan` | `POCKETCLOUD_SKIP_IMPACT_SCAN=1` |
| ADR decisions | *(gate only)* | `--skip-adr-decision-scan` | `POCKETCLOUD_SKIP_ADR_DECISION_SCAN=1` |
| Offer | `pocket quality offer-gate` | `--skip-offer-gate` | `POCKETCLOUD_SKIP_OFFER_GATE=1` |
| Docs | `pocket quality docs validate-all` | `--skip-docs` | `POCKETCLOUD_SKIP_DOCS=1` |

> Note: `git commit --skip-*-scan` is **not** forwarded by the installed hook.
> Use the env vars above or run `pocket ops git-hook pre-commit` directly.

Headless/agents: `export POCKETCLOUD_SKIP_DOCTOR=1` (and usually
`POCKETCLOUD_SKIP_CREDENTIAL_STORE=1` only when decrypt would block).

---

## 3. Orchestration & leases (ADR-0117)

```bash
pocket task list
pocket ops orchestrator status
POCKETCLOUD_LEASE_WAIT_SECONDS=0 pocket build bronze
pocket build cancel
pocket build cancel --lane apps
```

```bash
pocket task preset install workspace-graph-nightly
pocket task preset install web-fleet-presence-audit
pocket task preset install ephemeral-lease-reaper
pocket task list --recurring
```

---

## 4. Disk (`pocket ops disk`)

```bash
pocket ops disk runtimes
pocket ops disk runtimes --superseded
pocket ops disk runtimes --keep iOS --apply
pocket ops disk runtimes --apply

pocket ops disk sweep reference-repos
pocket ops disk sweep reference-repos --apply
```

---

## 5. Troubleshooting

### SSH / `cannot run : No such file or directory`
Empty `core.sshcommand=""`. Prefer `pocket ops git sync --push` (forces
`core.sshCommand=ssh`) or `export GIT_SSH_COMMAND=ssh`.

### Detached web submodule
```bash
git -C pocket-cloud-web-workspace checkout -B develop origin/develop
```

### Stale build-root lease
```bash
cat ~/.pocketcloud/locks/build-root-owner.json
# Next pocket invocation reclaims if the owner PID is dead.
```
Never `rm -rf ~/.pocketcloud/build` or `pkill` peer builds (ADR-0117).
