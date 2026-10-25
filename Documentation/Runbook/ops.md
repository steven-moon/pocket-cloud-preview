---
id: "doc.pocketcloud.runbook.ops"
title: "Operations & Git Runbook"
types: ["implementation-guide", "reference", "source-of-truth"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "ops", "git", "git-sync", "git-hooks", "orchestrator", "disk-space"]
owners: ["@steven"]
version: "1.21.1"
last_updated: "2026-10-19T00:00:00Z"
last_reviewed: "2026-10-14"
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
Governed by ADR-0159 (`Documentation/ADRs/ADR-0159-adr-git-sync-commits-from-evidence.md`)
(ledger (`Documentation/ADRs/Tasks/0159/README.md`)).

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
# Port checkouts are opt-in (client repos; many on dead remotes)
pocket ops git sync --preview --scan-folder web-fleet --scan-folder pc-port-repos

# Leave work someone else is still writing out of the plan (repeatable; a file or a directory)
pocket ops git sync --push --exclude Packages/Kernel/Core/Sources/PocketCloudCore/Graph

# Sync only your own files and leave every other change uncommitted (repeatable; a file or a directory)
pocket ops git sync --preview --only Packages/Kernel/Core/Sources/PocketCloudCommon/Quality/ADRCommitWorkflowEngine+Scope.swift
pocket ops git sync --push --only Documentation/Runbook/ops.md --only Packages/Kernel/AIStack/Sources/PocketCloudCLI/Commands/Ops
pocket ops git sync --push --only-from my-files.txt   # the paths this session wrote, one per line ('-' = stdin)
pocket ops git status --only Documentation/Runbook/ops.md

# Fold every non-official local branch into develop, then prune (ADR-0159 D9)
pocket ops git sync --preview --consolidate
pocket ops git sync --push --consolidate
pocket ops git sync --push --consolidate --quiet-minutes 0   # commit worktree work however fresh

pocket ops git status [--json] [--only <path>…] [<path>]
```

### One session's own files (`--only`, `--only-from`)

Without a scope a sync commits every dirty file in the checkout, and in a checkout several
sessions share that includes their work in progress. `--only <path>` (repeatable) limits the
whole run to the paths named. Each one is a file, or a directory and everything under it. A
relative path is read against the current directory first, then against the repository root.
`--only-from <file>` reads more paths, one per line; `-` reads standard input, and blank lines
and `#` lines are skipped. `pocket ops git status` takes the same two options and lists the
scoped files.

- **Only the named paths are planned.** The scope applies before `--exclude` and before
  attribution, so no subject, guess or coupling comes from a file outside it. The plan prints
  `🎯 Only N path(s) (--only); M other changed files left uncommitted`. `--json` carries
  `scope`, `scopeWithoutChanges` and `filesOutsideScope`.
- **Each commit names its paths** (`git commit -- <paths>`, T13). A file a peer has staged is
  not swept in.
- **Submodules and nested repositories.** A path inside a submodule commits only that file in
  the submodule. A path inside a `web-fleet` repository commits only that file there. Naming
  the submodule or repository itself takes all of it. A repository the scope does not reach is
  not probed, committed, moved forward or pushed. With `--push`, a scoped run integrates and
  pushes the root and only the repositories it committed in.
- **Nothing is widened.** A named path with no changes is listed under the scope line
  (`⚠️ <path>: no changes here`). If none of the named paths has changes, the run fails without
  committing or pushing anything. A path outside the repository, or an `--only-from` list that
  names nothing, also fails the run.
- **`--consolidate` cannot be combined with `--only`.** Consolidation commits every worktree and
  folds every branch.
- Files that change while the run is planning are still left out (see below), within the scope.

### Work in flight (`--exclude`, file ages)

Several sessions write into the main checkout at once, and a sync commits whatever is dirty. The
plan therefore shows how old each file is (`· 3m ago`). Files changed within `--quiet-minutes`
(default 15) are marked `✍️ in flight`. Those are most likely someone else's half-finished edit:
read them before `--push`. `--exclude <path>` (repeatable; a file, or a directory and everything
under it; root-relative or absolute) keeps a path out of the plan before attribution, so no commit
subject mentions it. The plan lists what it excluded under `🚧 Excluded`. A planned file that
changes after the plan was made is being written right now. The run names it, re-plans without
it, and commits the rest; it does not commit a half-written file.

### Folding branches into develop (`--consolidate`)

`--consolidate` does what "merge every worktree into develop and prune the dead branches" used
to take by hand. It runs after the sync's own commits, in the root, every submodule and each
nested repository the run covers that has a local `develop`. `main`, `develop` and
`windows-linux` are official and are never deleted.

1. **Uncommitted work.** A worktree holding a branch commits its uncommitted files through the
   normal ADR plan. The exception is a worktree where a file changed within `--quiet-minutes`
   (default 15): someone is still writing it, so it is reported as in flight and left alone.
   Files that name no ADR stay uncommitted unless `--default-adr`/`--default-task` is given.
2. **Judgement.** `git merge-tree` decides without touching any working tree. The branch is
   either already in `develop` (an ancestor, or a cherry-pick or squash that would leave
   `develop`'s tree unchanged), or it merges cleanly, or it conflicts. A conflicting branch is
   named with its files and left alone.
3. **Merge.** Only clean branches are merged, `--no-ff`, in the worktree that has `develop`. If
   git refuses because the merge would overwrite uncommitted changes there, that is reported.
   With no worktree on `develop`, the merge commit is written without one and `develop` moves by
   a compare-and-swap `update-ref`. With `--push`, `develop` first takes `origin/develop`,
   `origin/main` and `origin/windows-linux` when they carry commits it lacks and merge cleanly.
4. **Prune.** A branch whose content is in `develop` is deleted locally. A branch a worktree has
   checked out is kept and named. Removing worktrees is the Claude app's worktree cleanup (or
   `git worktree remove`), not this verb; the next run prunes the freed branch. With `--push`,
   `develop` is pushed first. Then every non-official `origin` branch whose content
   `origin/develop` holds is deleted, including the remote copy of a branch a worktree keeps
   locally. Unmerged remote branches are listed and kept.

A kept branch is reported, not failed; a merge, push or delete that went wrong ends the run
non-zero. `--preview --consolidate` prints the per-branch judgement and moves nothing.

### What sync does on every run (target + nested dirty repos)

Steps 1, 2 and 4 write, so they run only when the sync will execute.
`--preview` and `--json` change nothing (ADR-0159 D1): a preview lists what they
*would* install or seed under `🔍 Workspace activation at <repo> would …`, and
`--json` skips them. A linked worktree is its repository: a preview of
`.claude/worktrees/<name>` plans under the main checkout's name, and its hooks
are the main checkout's `.git/hooks` (ADR-0075 T05).

1. **Hooks** — if PocketCloud shims are missing **or** behind the current
   contract version → `pocket ops git-hook install <repo>` (upgrade in place).
   A repository that runs its own hooks from its working tree (`core.hooksPath`
   → husky's `.husky`, a `.githooks` folder) gets none: install writes only into
   git's own metadata, never over files the repository versions (ADR-0075 T07).
2. **Documentation refresh** — version-scans managed files; overwrites when the
   source frontmatter `version:` is newer. Local copies already ahead of source
   are left alone. Structural scaffold (ADR-0001, Tasks/0001, READMEs) is
   create-if-missing only.
3. **Nested scan** — git repos at depth 1–2 under each scan folder (default:
   `web-fleet`; `pc-port-repos` only when passed). Each is judged first (see
   *Repository health* below): only a directory git resolves to itself, whose
   remote is ours and can still answer, is synced.
4. **Agent docs** — create missing `README.md` / `AGENTS.md` / `CLAUDE.md` from
   `Documentation/Templates/<platform>/`, or inject/refresh the
   `<!-- pocketcloud:docs-index -->` section pointing at Runbook + Standards.
5. **Submodules** — dirty gitlinks (e.g. `pocket-cloud-web-workspace`) sync
   before the parent updates the pointer.
6. **Commits** — subjects `<type>(ADR-NNNN): <summary> (TXX)` with `≤ 72` chars,
   the summary made from the files (`add FleetVaultParser and FleetDeclaredState`).
   How each file is placed is printed beside it (ADR-0159 D2), strongest first:
   - *stated* — `diff names ADR-0157 T02` (the ADR **and** task in an added line),
     `header names …`, `Tasks/0157 names ServerGraph/FleetVaultParser.swift`;
   - *shared code* — `shares code with GitNonInteractive.swift (ADR-0075 T01)`: a
     file with no evidence of its own joins the changed file whose new code it uses;
   - `guess: …` — its last commit, a path rule, an ADR cited without a task, a
     README, and last of all its folder's last commit.

   A `guess:` line is the one to check before `--push`; correct it with an
   `ADR-NNNN TXX` comment in the change, or `--default-adr`/`--default-task`.
   Commits are ordered so each builds on the ones before it: a cluster lands after
   the code it uses, and clusters that use each other's new code become one commit
   whose body names the task it also carries.
7. **Remote repairs** — a GitHub HTTPS `origin` that an unattended run cannot use
   (it needs credentials) is switched to `git@github.com:<owner>/<repo>.git` when
   that SSH form answers. Preview lists the repair; `--push` applies it.
8. **Moving forward (`--push`)** — every repository (clean nested repos, every
   submodule, then the root) fetches `origin/<branch>` first. Only behind → it
   fast-forwards. Both sides moved → it merges, if git merges cleanly. A conflict,
   or local changes in the way, is aborted and that repository is **not pushed**;
   the run ends non-zero and names it. Nothing is ever forced.

### Repository health (what sync refuses to touch)

Printed at the top of every preview and run, and recorded once per run in the
learning store (fingerprint `git_sync_repository_health`):

| Finding | Example (2026-10-12) | Sync does |
|---|---|---|
| `.git` with no `HEAD` — git walks up to the enclosing repo | `pc-port-repos/GPTAcademy/*` (5 dead clones) | skips it; before this, sync planned commits of the **workspace's** files under their names |
| Remote not ours — a third-party clone (2026-10-14) | `pc-port-repos/EmergencyApp/other/COVID-19` (`CSSEGISandData`) | skips it from the URL, before any probe: no hooks, scaffold, commit or fetch. Ours = the workspace origin's namespace, each submodule's, and `.pocketcloud/git-sync-owners.json` (ADR-0159 T11) |
| Remote host no longer resolves | every `gitlab.clevercoding.com` remote (NXDOMAIN) | skips it |
| Remote repository not found / no `origin` | | skips it |
| GitHub HTTPS needs credentials, SSH answers | `pc-port-repos/Mathlete/*`, `FirstWords/*` | repairs `origin` to SSH |
| SSH alias host `github.com-<alias>` gone from `~/.ssh/config`, `github.com` answers | `ccem-ios-old` (`github.com-steve`) | repairs `origin` to `git@github.com:` |
| Remote unreachable this run (timeout, offline) | | keeps it; a blip is not death (ADR-0117) |

When **no** remote host resolves, the machine is offline and nothing is called
dead. Probes run with the non-interactive git recipe (`GitNonInteractive`: no
credential helper, no prompts, SSH batch mode), so a dead or private remote can
never hang the sync on a dialog.

### Managed Documentation (version-refreshed)

| Kind | Paths under `Documentation/` |
|---|---|
| Standards | `Standards/swift-6-implementation-standard.md`, `Standards/unified-providers-flywheel.md` |
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
| Default | `web-fleet` (depth ≤ 2); `pc-port-repos` only when passed |
| Extend | Repeat `--scan-folder <name>` |
| Disable | Pass a non-existent name, or `nestedRepoScanFolders: []` in engine tests |

Preview labels nested repos `hooks ✅` or `hooks ⚠️ will install/upgrade`.

### How changes are clustered

The full ranking is ADR-0159 D2–D3 (`Documentation/ADRs/ADR-0159-adr-git-sync-commits-from-evidence.md#decision`);
step 6 above is the short form.

1. **Stated** — the ADR **and** task on the change's own added lines
   (`// … (ADR-NNNN TXX)`), then the file's header (front matter for markdown,
   the leading comment block for code), then a task ledger under
   `Documentation/ADRs/Tasks/<NNNN>/` that names the file.
2. **Shared code** — a file stating nothing follows the changed file whose new
   code it uses.
3. **Guesses** (`guess:`) — the file's last commit, then the path-rule matrix
   (examples: LeadVetting → ADR-0147, MLX → ADR-0010, Storefront → ADR-0048,
   Subscriptions → ADR-0065, Trivia → ADR-0149, StoryKit → ADR-0091,
   Flywheel → ADR-0050, GitSync → ADR-0159, PocketBusiness → ADR-0148), then an
   ADR the diff cites without a task, a README, and the folder's last commit.
   `--default-adr` / `--default-task` replace every guess.
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
run the Documentation version refresh. Neither writes hooks into the working tree:
when `core.hooksPath` points there (husky), install says so and leaves the
repository's own hooks alone. Prefer `install` when you only need
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
| Pipes | `pocket quality scan pipes` | `--skip-pipe-scan` | `POCKETCLOUD_SKIP_PIPE_SCAN=1` |
| CLI flags | *(gate only)* | `--skip-cli-flag-scan` | `POCKETCLOUD_SKIP_CLI_FLAG_SCAN=1` |
| Imports | *(gate only)* | `--skip-import-scan` | `POCKETCLOUD_SKIP_IMPORT_SCAN=1` |
| Orphan packages | `pocket quality scan orphans` | `--skip-orphan-scan` | `POCKETCLOUD_SKIP_ORPHAN_SCAN=1` |
| Blast radius | `pocket knowledge graph impact <mod>` | `--skip-impact-scan` | `POCKETCLOUD_SKIP_IMPACT_SCAN=1` |
| ADR decisions | *(gate only)* | `--skip-adr-decision-scan` | `POCKETCLOUD_SKIP_ADR_DECISION_SCAN=1` |
| Offer | `pocket quality offer-gate` | `--skip-offer-gate` | `POCKETCLOUD_SKIP_OFFER_GATE=1` |
| Docs | `pocket quality docs validate-all` | `--skip-docs` | `POCKETCLOUD_SKIP_DOCS=1` |

> Note: `git commit --skip-*-scan` is **not** forwarded by the installed hook.
> Use the env vars above or run `pocket ops git-hook pre-commit` directly.

### Which index the gates read (ADR-0050 T32)

The gates read the index the commit is made from: the one git hands the hook as
`GIT_INDEX_FILE`, honoured when it lives in this repository's git directory (where
git puts every index it commits from). So `git commit -- <paths>`, this workspace's
commit form, is gated on exactly those paths, whatever else is staged. Until
2026-10-14 the variable was stripped and a pathspec commit with nothing pre-staged
ran every gate over an empty list (`No Swift sources staged`). An index anywhere
else, such as a script's own `GIT_INDEX_FILE`, is refused (`commit-index` gate)
rather than silently ignored. If a gate's output says nothing was staged on a
commit that changes files, the hook is not reading your commit: stop and report it.

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

Both verbs report unless given `--apply`. A sweep's `--apply` refuses while a build
is running; a report reads sizes and removes nothing, so it runs any time. To
notice growth at 20 GB rather than at 93% full, schedule the weekly report
(ADR-0111 T05):

```bash
pocket task preset install disk-growth-weekly   # Sunday 06:00, report-only
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
