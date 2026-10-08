---
id: "doc.pocketcloud.runbook.verification-and-gates"
title: "Verification, Quality Gates & Automated ADR Sync"
types: ["implementation-guide", "reference"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "verification", "git-hooks", "pre-commit", "commit-msg", "git-sync"]
owners: ["@steven"]
version: "1.12.0"
last_updated: "2026-10-07T00:00:00Z"
last_reviewed: "2026-10-07"
is_archived: false
---
# 4. Verification, Quality Gates & Automated ADR Sync

[← Back to Runbook Index](index.md)

## 5. Verification

```bash
pocket system local warmup                     # optional: removes MLX cold-start from verify
pocket knowledge rag index --if-stale
pocket system local verify
pocket system verify --exhaustive --local-first
pocket system verify --operation apple.test
pocket system verify --endpoints apple
```

Artifacts from exhaustive Apple verification land in `/tmp/pocketcloud-verify`,
not the workspace root.

The `heavySerial` Apple lane (`apple.test` and friends) runs xcodebuild in the shared
build root, so since ADR-0117 T07 it takes the build-root lease and waits for it like a
build does, printing the same `⏳ Waiting for the shared build root` heartbeat. If the
lease never frees inside the wait, that endpoint is reported **blocked** — `success:
false`, with a note saying the endpoint did not run — never skipped, because a skip
reads as green.

### Reading Test Evidence

Every `pocket build test` run persists the full `swift test` transcript
(ADR-0082 T14):

```bash
ls ~/.pocketcloud/build/logs/<date>/attempt-*/test-suite.log
grep "recorded an issue" ~/.pocketcloud/build/logs/<date>/attempt-1/test-suite.log
```

Since ADR-0082 T16 (2026-08-24) that evidence survives a disk-space emergency:
the low-disk prune that runs before every spawned SwiftPM command used to list
`build/logs` as a candidate and deleted the in-flight run's transcripts the
moment the volume filled mid-silver — the failure destroyed its own diagnosis.
If the logs directory is empty after an older run, that prune is why.

A package line reading `counts unavailable — exited non-zero with no bundle
summary` means the process died before printing the summary — it is NOT proof
that nothing ran. An abort far under the package's usual runtime (e.g.
Kernel/AIStack in ~90s against its usual ~50 min) is almost always a
test-target **compile break**: reproduce with
`./scripts/swiftw build --build-tests --package-path <pkg>` and read the
`error:` lines. Nothing in the commit-gate matrix compiles test targets, so a
Sources-side rename can pass every hook and die at the next silver.

**The unit-test lane never reaches Hugging Face.** `MLXBaseTestCase` sets
`POCKET_MLX_LOCAL_ONLY=1` and `HF_HUB_OFFLINE=1` for exactly that. A `Redownloading:
NN%` line in `test-suite.log` is therefore a defect to report, never lane noise;
the real repair-and-redownload behaviour is covered with a mock repository in
Core's `ModelDownloadCoordinatorTests`.

Grep gotchas, all real: swift-testing on macOS renders **SF-Symbol private-use
glyphs**, not ASCII — searching `✘` finds nothing; `"recorded an issue"` is the
reliable failure marker with name + file:line + assertion. Most bare `failed`
hits are CoreData XPC noise. `--filter "Suite Display Name"` runs **zero tests
and exits 0** — filter by type name, and treat any 0-test run as failure. And
`cmd | tail` swallows the exit code — redirect to a file and echo `$?`.

---

## 6. Git Hooks and Gates

The hooks are Swift shims that call `pocket ops git-hook <verb>` — which means
**every hook runs by hand**, identically to how git runs it. Nothing below
requires an actual commit or push to test.

```bash
pocket ops git-hook pre-commit                # diff-scoped gates (~1s typical; no build)
pocket ops git-hook pre-push                  # quick pre-push checks (<30s)
POCKETCLOUD_FULL_PREPUSH=1 git push           # deep pre-push; same as: pocket ops git-hook pre-push --full
pocket ops git-hook commit-msg <file>         # judge a commit message in a file
pocket ops git-hook status                    # contract version per managed shim
pocket ops git-hook install [path]            # install / upgrade shims + Documentation refresh
```

Hook shims are **versioned contracts** — `status` reports drift. Prefer
`pocket ops git-hook install <repo>` or `pocket ops git sync <repo>` to upgrade
stale shims (sync does this automatically for nested/submodule repos). Full
bypass tables and activation details: [ops.md](ops.md).

**`commit-msg` requires the work to be traceable**, and this is the single most
likely thing to stop a first commit. Beyond rejecting WIP/checkpoint subjects,
every message must name an ADR **and** a task, and the gate proves both exist on
disk:

| Failure | Meaning |
|---|---|
| no `ADR-NNNN` in the message | untraceable — name the ADR |
| `ADR-NNNN` with no file under `Documentation/ADRs/` | typo'd number |
| task id not found in that ADR's task files | the work was never decomposed — write `Documentation/ADRs/Tasks/<NNNN>/` **first** |

The third is the strict one, and the point of the whole gate: the task id must
appear in the task files, not merely the directory. Escapes, in preference order:
an `ADR-Exempt: <why>` trailer (the reason lands in history), `POCKETCLOUD_SKIP_ADR_REF=1`
for scripted or bulk work, `--no-verify` for everything. Merges, reverts and `fixup!`/`squash!`
are exempt automatically.

Gate outcomes feed the learning flywheel (pre-commit only today): a repeat
offence is answered with the fix that resolved it last time, not a restatement
of the rule. Bypasses are recorded too — a rule that is routinely skipped is
either a bad rule or a real gap.

### Every Gate is Individually Runnable

Every scanner in the hook is **diff-scoped** — it judges only what the commit
adds, so pre-existing debt never blocks a commit. Each also exists as a
standalone verb with uniform flags: bare = staged (matches the gate), `--all` =
whole tree, explicit file args = judge exactly those files.

| Pre-commit Gate | Standalone Verb | Bypass Flag |
|---|---|---|
| 🛡️ paths | `pocket quality scan paths` | `--skip-path-scan` |
| 🔑 secrets | `pocket quality scan secrets` | `--skip-secret-scan` |
| 🛡️ security & privacy | `pocket ops git-hook security` | `--skip-security-scan` |
| 🎡 flywheel | `pocket quality scan flywheel` | `--skip-flywheel-scan` |
| 🧬 conflict markers | `git diff --cached --check` | *(none)* |
| 📐 aml | `pocket quality scan aml` | `--skip-aml-scan` |
| 🧩 platform | `pocket quality platform-check` | `--skip-platform-scan` |
| 📏 standards ratchet | `pocket quality scan standards` | `--skip-standards-scan` |
| 🕳️ test-targets | `pocket quality scan tests` | `--skip-test-target-scan` |
| 🏷️ customer-names | `pocket quality scan customer-names` | `--skip-customer-name-scan` |
| ™️ brands | `pocket quality scan brands` | `--skip-brand-scan` |
| 🗣️ client-terminology | `pocket quality scan client-terminology` | `--skip-client-terminology-scan` |
| 🌀 streams | `pocket quality scan streams` | `--skip-stream-scan` |
| 🚩 cli-flags | *(gate only)* | `--skip-cli-flag-scan` |
| 📦 imports | *(gate only)* | `--skip-import-scan` |
| 💥 blast-radius | `pocket knowledge graph impact <module>` | `--skip-impact-scan` |
| 📜 adr-decision | *(gate only)* | `--skip-adr-decision-scan` |
| 📦 offer | `pocket quality offer-gate` | `--skip-offer-gate` |
| 📚 docs | `pocket quality docs validate-all <dir> --ci` | `--skip-docs` |
| 🪝 registration | `pocket ops git-hook status` / `install` | — |

Related gates that genuinely live **outside** the hook:
```bash
pocket quality scan hybrid          # built dist/ bundles that won't render in a .app
pocket quality project-yml   # xcodegen drift + test-target integrity
pocket quality offer-gate    # proves Android at build depth
```

---

## Automated ADR Staging, Validation, and Push (`pocket ops git sync`)

**SSOT:** [ops.md §1](ops.md) — path targeting, nested `web-fleet` scan, stale-hook
upgrade, and Standards/Runbook version refresh. Short form:

```bash
pocket ops git sync --preview              # CWD
pocket ops git sync --preview /path/to/repo
pocket ops git sync --preview --json
pocket ops git sync --push
pocket ops git status [--json] [<path>]
```

Clustering (tags → task ledgers → subsystem matrix → nested repos → submodules →
parent) and the `≤ 72` subject rule are documented in ops.md. Unmapped files warn
via flywheel `adr_mapping_gap`; pass `--default-adr` / `--default-task` when needed.
6. **Self-Healing Telemetry:** Any unmapped file without an explicit ADR emits a `UnifiedLogEmitter` warning (`workflow_failure_fingerprint_id: "adr_mapping_gap"`) and feeds into the flywheel.
