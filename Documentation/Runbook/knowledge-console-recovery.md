---
id: "doc.pocketcloud.runbook.knowledge-console-recovery"
title: "Knowledge Systems, Console Contracts & Recovery"
types: ["implementation-guide", "reference"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "knowledge", "rag", "secondbrain", "console", "disk-space", "recovery"]
owners: ["@steven"]
version: "1.11.0"
last_updated: "2026-10-04T00:00:00Z"
last_reviewed: "2026-10-04"
is_archived: false
---
# 6. Knowledge Systems, Console Contracts & Recovery

[← Back to Runbook Index](index.md)

## 8. Knowledge

```bash
pocket knowledge rag status                  # namespaces that exist, counts, staleness — ALWAYS first
pocket knowledge rag index                   # auto-routes by what the repo IS (below)
pocket knowledge rag query "<question>"      # searches `codebase`, the default namespace
pocket knowledge rag query --namespace docs.adr "<question>"
```

**Check `rag status` before blaming a namespace.** A query against an empty or
non-existent namespace returns nothing and looks exactly like a broken index —
and on a fresh clone the index does not exist at all (`0 document(s)`), because
it is generated, not tracked.

Routing, with no `--namespace` (ADR-0076 §4):

| Repo | `Documentation/Standards` | `Documentation/ADRs` | Everything Else |
|---|---|---|---|
| this workspace (primary) | `docs.standards` | `docs.adr` | `codebase` |
| any other repo | — | — | `workspace.<repo-name>` |

Two consequences the earlier examples got wrong:
**`workspace.main` can never exist in this workspace** — `workspace.<name>` is
produced only for non-primary repos, and this repo is not named "main". And a
routed namespace does not exist until an index run has actually reached those
directories.

The code graph (ADR-0113) is a separate, also generated, also stale-by-default
artefact: `pocket knowledge graph status | build --if-stale | impact <Module> | export`.

---

## Gmail, the Go-Forward Lane

Before changing credentials, inspect what the next process can actually read:

```bash
pocket knowledge secondbrain sync-gmail --account <you@domain> --status
```

This is metadata-only: it neither contacts Google nor prints tokens.

```bash
# 1. Store the OAuth client. No browser, no consent, safe to re-run.
pocket knowledge secondbrain sync-gmail --account <you@domain> \
  --client-secrets ~/Downloads/client_secret_*.json

# 2. Grant access. Opens a browser; you consent.
pocket knowledge secondbrain sync-gmail --account <you@domain> --authorize

# 3. Thereafter, incremental. Resumes from its own checkpoint.
pocket knowledge secondbrain sync-gmail --account <you@domain>
```

Consent binds an ephemeral loopback port, which only a **Desktop app** client may use.

---

## 8.5 What the Console Tells You (ADR-0112)

**Long builds report in.** A compile streams to its log file, not the console,
unless `--verbose` is set — so a quiet terminal is normal and says nothing about
health. Every 60s a quiet command prints one line:

```
💭 Private by construction, not by policy. · xcodebuild PocketCloudCLI still running (3m, quiet 0s) · log: ~/.pocketcloud/build/logs/…
```

`quiet` is the number that matters: it is how long the **child** has been
silent, not how long *you* have. `quiet 0s` with a still terminal means the
compiler is working normally.

```
… still running (14m, quiet 11m, working — CPU 3s ago) · log: …
```

`working` means the pipe is quiet **and the child tree's CPU is still rising** — slow,
not stuck — and in that state the 600s inactivity verdict is *deferred*, up to three
budgets (30 minutes). `SUSPECT` means quiet with no CPU, which is reaped on schedule.

**Output is redacted at the last hop.** Known credential values, published token
shapes, and home directory paths are masked (`~`).

**Scrubbing existing logs on disk:**
```bash
pocket system logs scrub --plan-only   # what would change, nothing written
pocket system logs scrub               # Logger root + ~/.pocketcloud/logs + learning store
pocket system logs scrub --root ~/some/other/logs --json
```

**A rival build is named:**
```
👀 Another xcodebuild holds this DerivedData: pid 61213, running 00:23.
```
If your build later fails with `database is locked`, that process is why: wait for it.

> [!WARNING]
> **Ctrl-C does not stop the compiler.** After interrupting a build, clean up orphans:
> ```bash
> pgrep -x xcodebuild
> pgrep -x xcodebuild | while read p; do kill -TERM -$(ps -o pgid= -p $p | tr -d ' '); done
> ```

---

## 9. Free Disk Space

Check before heavy runs, not during:

```bash
df -h /System/Volumes/Data
du -sh ~/.pocketcloud/* | sort -rh | head
du -sh ~/.pocketcloud/build/pocketcloud/swiftpm/* | sort -rh | head
```

> ⚠️ **Never delete any of these while a build or test session is running.**

### The Two Biggest Wins (ADR-0111)

```bash
# Simulator runtimes prune
pocket ops disk runtimes                      # report
pocket ops disk runtimes --superseded         # only redundant builds
pocket ops disk runtimes --keep iOS --apply   # prune non-iOS
pocket ops disk runtimes --apply              # newest-per-platform

# Regenerable build output sweep
pocket ops disk sweep reference-repos         # report
pocket ops disk sweep reference-repos --apply # delete
```

### Shared Rust Build Output

`.cargo/config.toml` at the workspace root points every Cargo package at one
shared `<workspace>/target`:
```bash
du -sh target                   # shared dir; safe to delete, fully regenerated
```

### Reclaim Ladder

```bash
# 1. Build scratch + temp
pocket build clean
rm -rf ~/.pocketcloud/build/pocketcloud ~/.pocketcloud/build/temp

# 2. Thirdparty cache snapshot
rm -rf ~/.pocketcloud/vendor/thirdparty-cache/latest

# 3. Xcode DerivedData
rm -rf ~/Library/Developer/Xcode/DerivedData

# 4. Local AI models
pocket system local cleanup                          # plan
pocket system local cleanup --execute --budget 20GB  # execute
```

---

## Recovering From Known Failure Classes

| Symptom | Cause | Fix |
|---|---|---|
| `".swiftpm" couldn't be copied … already exists` at CLI Installation | Stale workspace-local SwiftPM state in a copied checkout | `rm -rf ~/.pocketcloud/build/pocketcloud ~/.pocketcloud/build/temp`, then rebuild |
| `🔒 the shared build root is leased by …` | Another gate uses the gate workspace — the machine working as designed (ADR-0117) | Wait, use `POCKETCLOUD_LEASE_WAIT_SECONDS=0` to fail fast, or run `pocket build cancel` when that gate should stop. Never `pkill` or delete its workspace |
| `🔒 resource 'sample-apps-build-root' is leased by …` | Another standalone sample-app build uses the reusable app workspace | Wait, fail fast with `POCKETCLOUD_LEASE_WAIT_SECONDS=0`, or run `pocket build cancel --lane apps` |
| A build fails with `database is locked` on the SHARED app DerivedData | A peer session's build holds it; nothing is killed on the shared tree by design | `pocket task list` names the lease holder; wait for it. If no `pocket` process exists, it is an orphan — see §8.5 |
| Silver dies mid-test-phase with "out of space"; `~/.pocketcloud/build/logs` is empty afterwards | Free space fell under the 2 GB floor; pre-T16 the emergency prune also deleted the run's own logs | Free disk **before** the run (§9); the prune spares logs since T16 |
| A package reports `counts unavailable — exited non-zero with no bundle summary` far under its usual runtime | Test-target compile break (the gates never compile test targets) | `./scripts/swiftw build --build-tests --package-path <pkg>`; read the `error:` lines (§5) |
| A gate reports a line count ≈ **2× `wc -l`** (e.g. the ratchet calls a 415-line file "832 lines") | CRLF content split by `CharacterSet.newlines`, which splits it twice | Count with `String.portableLines`. Check the file: `tr -cd '\r' < f \| wc -c`; normalize with `perl -pi -e 's/\r\n/\n/g; s/\r/\n/g'` |
| `App Store Connect private key not found: Keychain item missing for keyID …` | The manifest exists but the PEM was never written to the encrypted credential store, or was lost with it. The manifest alone looks complete | `pocket dev apple asc add-key <KEY_ID> <ISSUER_ID> <path.p8> --default`, then prove it with `pocket dev apple asc bundle-ids` |
| Builds stamp `1.0.0` build `1` with no error | Same cause as the row above. Version resolution falls back rather than blocking an offline build, so a dead ASC key is invisible here | Same fix. Verify with `pocket dev apple asc versions --app <key>` |
| `No stored Google authorization for <account>` | The OAuth client is stored but consent has not been granted | `pocket knowledge secondbrain sync-gmail --account <you> --authorize` |
| Google consent fails with `redirect_uri_mismatch` | The OAuth client is type `web`; loopback consent needs `Desktop app` | Create a Desktop app client and re-import it with `--client-secrets` |
| `Couldn't update submodules` / exit 74 | Usually DNS, not a stale cache | **The pipeline now waits it out itself** (ADR-0117 T05): `🌐 …` lines, `getaddrinfo` probes every 15s, four attempts over 30/90/180s. If it still fails, the network was down for five minutes — `git ls-remote <url>` to confirm, and never wipe a cache for this class |
| Android build fails in the bundler on a missing `bin/pocket-aarch64-linux-android` | The app has no `tauri.android.conf.json` stripping `externalBin` | Add it; there is no Android sidecar and cannot be (ADR-0082 D3) |
| Android app installs but renders blank | A JS error at bundle load aborts the module | `pocket dev android logs <App>` — check for the CommonJS barrel import |
| `pocket build android check` reports the SDK missing on a machine with Android Studio | The SDK is somewhere non-standard | `export ANDROID_HOME=…`; the check prints where it looked |
| `Model container for id(...) has been released` mid-inference | Model container invalidated or evicted (e.g. network outage or concurrent model switch) | `InferenceEngine.generate` now auto-reloads and retries up to 3 times from local cache (`MLXService+Execution.swift`); verify local weights in `~/.cache/huggingface/hub` |
| `Completion telemetry did not reach the provider` | Isolated log provider missed completion telemetry | Pass `provider` to both initial successful persist and blocked persist in `TriviaPreparationTests` |
| `error: cannot run : No such file or directory` on git push | An empty `core.sshcommand=""` in runner environment confuses git | Use `pocket ops git sync --push` (sanitizes `core.sshCommand=ssh`) or pass `GIT_SSH_COMMAND="ssh"` |
