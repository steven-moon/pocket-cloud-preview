---
id: "doc.pocketcloud.runbook.build-tiers"
title: "Build Tiers, Leases & Requirement Gates"
types: ["implementation-guide", "reference"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "build", "leases", "bronze", "silver", "gold"]
owners: ["@steven"]
version: "1.11.0"
last_updated: "2026-10-04T00:00:00Z"
last_reviewed: "2026-10-04"
is_archived: false
---
# 1. Build Tiers, Leases & Requirement Gates

[← Back to Runbook Index](index.md)

Three composites trade speed for coverage. Reach for `gold` at a release, not in
the iteration loop.

```bash
pocket build bronze     # incremental CLI/kernel compile (~2 min warm; the old "~24s" predates the CLI's growth). "Did I break the build?"
pocket build silver     # incremental all -> test. The pre-merge gate.
pocket build gold       # clean -> all -> test. Releases, tags, suspected stale-cache drift. (Note: Gold does NOT rebuild MLX from scratch if the compiler and MLX source fingerprint match, as the built MLX framework dependencies survive clean.)
pocket build cancel     # cooperatively stop the active bronze/silver/gold gate
```

Individual verbs, when a composite is more than you need:

```bash
pocket build clean
pocket build all
pocket build cli                  # rebuilds/reinstalls `pocket` itself
pocket build app PocketMind       # one Apple app (macOS only)
pocket build apps                 # every Apple app
pocket build cancel --lane apps   # cooperatively stop the sample-app lane
pocket build bundles              # web-lib + all 8 Tauri/hybrid web bundles
pocket build test                 # distributed test suite (all packages)
pocket build test --package-path Packages/Toolkit/PocketCloudUI # test a single package
```

For deterministic source-analysis work, use the dedicated leaf test lane:

```bash
./scripts/swiftw test --package-path Packages/Kernel/Algorithms
```

This executes the algorithm and CLI-operation suites without resolving or
compiling MLX. Do not substitute a filtered `swift test` against
`Packages/Kernel/Core`: SwiftPM still builds that package's complete test
graph before applying the execution filter.

> **A gate names its neighbours before it starts (ADR-0117 D8).** `gold` and `silver`
> print every other `pocket build` / swift-build / swift-test / xcodebuild / xctest
> process on the machine, excluding their own ancestry. Nothing is refused or killed —
> the lease serialises the *build root*, not the *CPU*, and a gate that runs beside
> heavy neighbours slows into false verdicts. Gold #4 (2026-09-08) died at a 600s
> silence verdict because three package-test compiles were running beside it.

> ⚠️ A root `swift build` does **not** compile `PocketCloudInfrastructureKit` or
> `PocketCloudBuildOrchestration` — a syntax error there still prints "Build
> complete". Validate those with `pocket build cli` or
> `./scripts/swiftw test --package-path Packages/Toolkit/PocketCloudInfrastructureKit`.

---

## Graph Tiers (Planned — ADR-0113 T19)

The Pocket Graph goes stale on **every commit** by construction (HEAD's
committer date newer than the store), and today nothing refreshes it but a
human. What exists now, and what it costs — measured 2026-09-07:

```bash
pocket knowledge graph status                 # 0.8s. Says "stale — a commit landed after the graph was built" on most days.
pocket knowledge graph build --if-stale       # 28–30s on a quiet disk; 127s beside a running build. NOT "~5s".
pocket task preset install workspace-graph-nightly   # the 01:45 nightly exists but is NOT installed by default — check `pocket task list --recurring`
```

The plan gives graph refresh the same three tiers as the build, each doing what
fits its budget, each wired to the build tier of the same name and to the commit
path, all behind one verb (`pocket knowledge graph refresh --tier …`), one lease,
and a recorded built-at commit:

| Tier | Question it answers | Work | Budget | Runs on |
|---|---|---|---|---|
| 🥉 bronze | did this change move the graph? | delta of files dirty *or* changed since the built-at commit; a docs-only commit is a re-stamp, not a rescan | ≤ 1s wall per git event, measured from outside — the shim and CLI bootstrap are most of it; the refresh itself runs in a fire-and-forget child | `post-commit` / `post-merge` / `post-checkout` / `post-rewrite`; `pocket build bronze` |
| 🥈 silver | does the graph describe this tree? | the full `graph build --if-stale`, every producer, as a child process under the build watchdog | ≤ 180s, ~30s typical, non-fatal | `pocket build silver` **before `all`** (the app batch inside `all` reads the graph); `bronze --test` before `test` (bronze has no `all`); the nightly |
| 🥇 gold | is the graph provably a clean rebuild? | shadow index → cold rebuild at a sibling path, verified against `quality scan imports`, then swapped into the live store in one transaction → export | ≤ 10 min; a verification mismatch fails gold | `pocket build gold` (after `clean`, before `all`); on demand |

Until it ships, the honest sequence is by hand: `pocket knowledge graph build
--if-stale` **before** a scoped `pocket build silver --scope-to-changes`, or the
scope silently widens to everything. Design, measurements and the adversarial
review live in `Documentation/ADRs/Tasks/0113/README.md` under T19.

---

## Reusable Build Lanes & Conflict-Level Leases (ADR-0117)

Bronze, silver, gold, `pocket quality test`, and `pocket system verify`'s Apple
lane share the gate workspace and its lease
(`~/.pocketcloud/locks/build-root.lock`). Standalone `pocket build app`,
`pocket build apps`, and `pocket dev apple build-app` use one separate reusable
sample-app workspace and the `sample-apps-build-root` lease. A gate and a
standalone sample-app build may therefore run concurrently without duplicating
DerivedData per invocation. Two gates serialize with each other, and two
standalone sample-app builds serialize with each other.

The two broad lanes take short named leases only while they mutate a truly
shared artifact: `mlx-artifacts`, `apple-project-generation`, and the destination
for the specific app being installed. This keeps MLX publication, xcodegen, and
same-app installation safe without holding one global lock for hours.

A second builder on the same lane waits
(`POCKETCLOUD_LEASE_WAIT_SECONDS`, default 3600; `0` fails fast) with a heartbeat
naming the holder, and appears in `pocket task list` / `pocket ops orchestrator
status` as a waiter. Transient network failures during resolution are retried by
the pipeline with backoff (`🌐` lines). Gold and silver now create a run-owned
Git index/tree/detached-worktree snapshot (ADR-0126 T02), and every phase uses
the same receipt-backed source path. Gold and silver acquire one build-root
lease for the complete composite, so clean, all, test, smoke, and evidence run
under one critical section; inner pipeline entries re-enter the process lease
instead of opening gaps for another builder. The explicit infrastructure
override used by command tests bypasses the machine-wide lease and snapshot so
tests stay independent of a live build.

Contention reports immediately, then every 30 seconds. Each heartbeat includes
a rotating PocketCloud slogan, holder command/PID/lease age, wait budget, other
waiters, current phase, activity age, and the reported log path. The fail-fast
hint appears on the initial notice. `POCKETCLOUD_NO_FLAVOR=1` removes slogans
while retaining the facts. Lease age starts when the kernel lock is acquired;
pipeline elapsed time is separate. Activity age measures observed work/output:
a timer never refreshes it. Cache copies report their boundaries and explicitly
say that file-level progress is unavailable. Phase estimates, when supplied,
use that phase's clock and do not carry into the next phase.

**What the snapshot prints.** The console carries the milestones only —
`📸 Preparing Git source snapshot`, `📸 Materialized N ignored build input(s)`,
`📸 Source snapshot ready: …`, and any failure — while the per-command trace
(`Running git …` / `Finished git …: exit 0, 0.07s, stdout 41 bytes` / submodule
clone chatter) goes to `attempt-N/source-snapshot.log` and the progress feed a
waiting build reads (2026-09-13; before that, ~30 trace lines per gate reached
the console and said nothing a milestone does not). Materializing the ignored
build inputs is now sub-second: every pathspec is a literal directory, with the
per-app entries expanded from the `Apps/` listing. It was 30s per gate because a
glob (`Apps/*/…`) or an exclude (`**/.build/**`) makes git walk everything under
the common prefix — the 4.7 GB Rust target under `Apps/PocketDesktop` — before
matching, and the 464 extra entries that walk produced were all nested-repository
directories the code discarded anyway.

For a gate contention smoke check, start `pocket build silver` in one terminal and
`POCKETCLOUD_LEASE_WAIT_SECONDS=65 pocket build bronze` in another. Expect an
immediate notice and two rotating heartbeats, followed by a bounded refusal if
silver still holds the lease. Repeat with `POCKETCLOUD_NO_FLAVOR=1` to check the
operational-only output. A successful silver build is not required for this
check; the first process must remain the sole holder throughout the wait.

```bash
pocket task list                       # first line: holder and queue
pocket ops orchestrator status         # same line, beside the host's state
POCKETCLOUD_LEASE_WAIT_SECONDS=0 pocket build bronze   # fail fast instead of waiting
pocket build cancel                    # request gate cancellation; owner stops its own child tree
pocket build cancel --lane apps        # request standalone sample-app cancellation
pocket build cancel --all              # request both independently
cat ~/.pocketcloud/locks/build-root-owner.json         # who holds it, since when
ls ~/.pocketcloud/locks/build-root.waiting             # one file per waiter
```

---

## Requirement-Gated Local-First Execution (ADR-0126)

The run-level network state is not a build result. Each step must declare its
own tool, platform, local-artifact, credential, source, and exact endpoint
requirements immediately before it runs, and again before a retry or resume.
The default policy checks local mirrors/caches first; it probes an exact
endpoint only when that step actually needs a missing online input. `--offline`
is strict and performs no DNS, socket, HTTP, Git fetch, SwiftPM automatic
resolution, ASC, npm download, or Docker pull.

The implementation and evidence contract are defined in
ADR-0126. Until
T01–T08 land, current runs still expose only the older attempted/passed/failed
step model, so a platform or network omission in an existing phase must not be
read as a first-class `skipped` result.

Three things a live run teaches that the design does not:

- **The sidecar can outlive its holder.** `build-root-owner.json` names a process; the
  kernel releases the `flock` when that process dies, but the JSON stays. Observed
  2026-09-08 20:01: the file named `pocket build all` (pid 63965) which no longer
  existed. That is handled — the next acquirer reports `🔓 Build-root lease recovered
  from a process that never released it` — but **do not read the sidecar as proof
  anything is running.** `pocket task list` checks; `cat` does not.
- **A dead holder can leave a live orphan.** The same incident left an `xcodebuild`
  reparented to `launchd`, 25 minutes old at 0% CPU, still holding the CLI
  DerivedData. No lease covers it, because its owner is gone. **Nothing reaps it** —
  on either tree. This used to say the private DerivedData was reaped "by design",
  and ADR-0117 T11 deleted that: the premise was that a per-app DerivedData path
  could only ever be ours, and `pocket dev apple build-app` takes no lease and
  resolves to the same path, so the reap could destroy a healthy peer. Both trees now
  name the holder and refuse to delete; the build fails with `database is locked`
  and says who to wait for.
- **A refusal is not a bug.** `🔒 the shared build root is leased by …` means the
  machine is doing what it should. Wait, or use `POCKETCLOUD_LEASE_WAIT_SECONDS=0`
  and come back. Do not `pkill`, and do not delete `~/.pocketcloud/build` — four code
  paths that "helped" that way were deleted on 2026-09-08 (ADR-0117 T01).
- **The lease does not stop the scheduler from eating the machine.** A silver on
  2026-09-09 lost five packages to watchdog timeouts, four reading `child tree was
  still burning CPU` — not stuck, starved: load average **37.56 on ten cores**. The
  two thieves were `pocket ops orchestrator start` (22.8% of a core with *nothing
  due* — ADR-0117 T17) and an orphaned 25-hour `knowledge graph rollup`. Neither is
  in the heavy family, so neither ever probes the lease. **Before trusting a gate's
  verdict, read `uptime`**: a load average well above the core count makes every
  timeout unreadable, and a starved build looks exactly like a broken one.
