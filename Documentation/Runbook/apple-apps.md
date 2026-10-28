---
id: "doc.pocketcloud.runbook.apple-apps"
title: "Apple Apps, Simulators, Storefront & Ship"
types: ["implementation-guide", "reference"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "apple", "ios", "macos", "tvos", "iphone-duo", "storefront", "testflight", "ship"]
owners: ["@steven"]
version: "1.12.3"
last_updated: "2026-10-26T20:30:00Z"
last_reviewed: "2026-10-26"
is_archived: false
---
# 2. Apple Apps, Simulators, Storefront & Ship

[← Back to Runbook Index](index.md)

Projects are generated into a single consolidated Xcode project (`Apps/PocketMind/PocketMind.xcodeproj`) by xcodegen on every build.
**Never hand-edit `.xcodeproj` or `.xcscheme`** — they are regenerated and your
edit is silently clobbered.

---

## Build

```bash
pocket build app PocketMind
pocket build apps
```

Before the app batch, `pocket build apps`/`all` run the shared-code iOS preflight.
Its compile probe cross-compiles `PocketCloudUI` for the iOS-simulator triple into
`~/.pocketcloud/build/temp/ios-platform-check/PocketCloudUI` — the reusable build
workspace, so the dependency checkouts and macro builds survive between gates.
Until 2026-09-13 the scratch sat inside the snapshot worktree, which a composite
gate deletes afterwards, so every gate paid a cold resolve and compile (165.8s
for a probe documented at 15–40s) and threw it away. Its expected outcome on this
toolchain reads:

```
✅ Shared-UI iOS compile probe: PocketCloudUI sources clean for iOS; 5 error(s) confined to swift-subprocess — macOS-only dependencies real iOS builds exclude (…s).
```

That is a pass. The probe has no iOS destination (no iOS Swift SDK is installed),
so SwiftPM keeps `.when(platforms: [.macOS])` dependencies in the graph and
swift-subprocess cannot compile for the triple; real iOS app builds resolve
against the iOS SDK and never include it. Only an error in our own sources fails
the probe, and that failure names the file.

---

## Run on the iOS Simulator

```bash
pocket dev apple build-app PocketMind --simulator
pocket dev apple build-app PocketMind --simulator --device "iPhone 16 Pro"
pocket dev apple build-app PocketMind --simulator --screenshot /tmp/mind.png --settle 14
pocket dev apple build-app PocketMind --simulator --scene memory-palace
```

`--scene` forwards `-UITestScene <name>` to the app, which is the dependable way
to land on a surface that is not the launch screen — touch injection into the
simulator is unreliable on some hosts.

`--settle` matters: without it the screenshot is the splash screen, and the
capture proves only that something launched.

### PocketBusiness screen previews

Use the registered tab id as the scene. The shared Admin shell opens the same
destination at compact and regular widths; Settings uses its registered tab.
Unknown ids show **Unknown preview scene** and log the failed route rather
than silently falling back to Dashboard.

```bash
pocket dev apple build-app PocketBusiness --simulator --device "iPhone 18 Pro" --scene leads --settle 14 --screenshot /tmp/business-leads.png
pocket dev apple storefront screenshots --app pocketbusiness --formats iphone --settle 14
```

The phone/tablet/Mac screenshot manifest covers `dashboard`, `leads`, `insights`,
`aiworkbench`, `funnel`, `databases`, and `apps`. Other registered screens,
including `settings`, `sources`, `relationships`, `products`, `archives`,
`cloudflare`, and `node`, accept the same `-UITestScene <id>` contract.
`servers` presents server management; `deployments` opens `control`;
`mesh`, `nodes`, and `nodedashboard` open `node`.
The Apple TV manifest retains its deterministic `-UITestDashboard <id>` routes.

Xcode 27 hosts simulators in **Device Hub** inside the selected Xcode bundle's
`Contents/Applications` directory. The app pipeline opens that app before
waiting for boot status, installing, launching, and capturing the simulator.
UI tests in `SceneLayoutUITests` assert `admin.scene.<id>` for each screen,
capture retained screenshots, and verify the copilot stays in native toolbar
chrome and opens a sheet. A navigation bar alone is insufficient route evidence.

---

## Simulators

```bash
pocket dev apple simulator --list
pocket dev apple simulator --boot --name "iPhone 16 Pro"
```

---

## iPhone Duo readiness

Apple's checklist: [Prepare for iPhone Duo](https://developer.apple.com/iphone-duo/prepare/).
Policy (anti-patterns + shell rules) lives in the
[Swift 6 Implementation Standard — Adaptive Layout §10](../Standards/swift-6-implementation-standard.md).

**What this tree already does**

- `PCAppShell` / `AppShellView` pick split vs tab from `horizontalSizeClass`, not
  idiom — Duo's regular-width inner display gets the sidebar the same way iPad does.
- Shared UI sizes from window / `GeometryReader` (see `DebugInfoProvider`,
  `ChatBubbleComponent`, `CameraView`); there is no layout use of `UIScreen.main`.
- No orientation-driven layout. Remaining `userInterfaceIdiom` reads are
  non-layout (StoreKit platform, sync peer labels, iPad-only top-tab inset probe).

**SDK note (this host, 2026-10-07)**

| Build SDK | Duo behaviour (Apple) |
|---|---|
| iOS 26 or earlier | Letterboxed on the inner display; outer shows content beside status/camera |
| iOS 27.0 (current Xcode 27.0 here) | Fills most of the inner display; still avoids the right-edge status bar |
| iOS 27.1+ | Full-display + vertical bars under the status bar — **preferred before a Duo-optimized ship** |

Deployment floor stays `IPHONEOS_DEPLOYMENT_TARGET: 18.0` in `configs/project_base.yml`.

**How to verify**

```bash
# Prefer the Duo simulator once Xcode 27.1+ is installed (Device Hub).
pocket dev apple simulator --list | rg -i 'duo|fold|resizable'
pocket dev apple build-app PocketMind --simulator --device "iPhone Duo" --settle 14 --screenshot /tmp/mind-duo.png

# Until Duo hardware/sim is available: resizable iPhone Mirroring on macOS 27,
# or the iOS resizable simulator in Device Hub — stretch both axes and confirm
# the shell flips compact ↔ regular without clipping.
```

When auditing a surface: fold/unfold mid-session, rotate in each pose, and check
that toolbars still work when the system presents them vertically. Custom fold-
aware layouts belong in `ArrangementView` / `reservedRegions`, not in hardcoded
frame math.

---

## Storefront Screenshots & Visual Evidence Pipeline (ADR-0048)

Capture App Store screenshots across simulator scenes without manual clicking.
Captures land in `Generated/Apple/Screenshots/<appKey>/<device>/` — **gitignored**
staging output, so a clean tree tells you nothing about whether they exist.

```bash
# Capture the full storefront screenshot set for ONE app.
# --app is required and takes the lowercase app KEY; there is no --all here
# (that flag exists on `storefront generate`, not on `screenshots`).
pocket dev apple storefront screenshots --app pocketmind --settle 14

# Apple TV uses a separate tvOS simulator build and 1920×1080 captures.
pocket dev apple storefront screenshots --app pocketbusiness --formats appletv --appearance dark --settle 20
pocket dev apple storefront push --app pocketbusiness --platform tvos --apply --replace-screenshots

# Direct scene jump, forwarded to the app as `-UITestScene <name>`.
# The name must be one the app's own tab enum resolves (PocketMind:
# daily-review · chat · knowledge · wellness · learning · story-maker)
pocket dev apple build-app PocketMind --simulator --scene knowledge --screenshot /tmp/mind_scene.png
```

Alpha stripping (32-bit RGBA → 24-bit RGB, which App Store Connect requires) is
**not** a verb you call — `AppStoreScreenshotService` applies it to every capture
on the way out. There is no `storefront strip-alpha` subcommand.

Each family's capture also leaves **`capture-record.json`** beside its shots
(ADR-0162 T06): the data the screens showed (`demo` when the capture seeded demo
content, absent otherwise), the captured app's version and build, and every
shot's SHA-256. The Marketing Graph turns a screenshot into a `media-asset` only
while its file still hashes to that record, so a hand-edited or pre-T06 shot is
reported by `pocket knowledge graph build --domain marketing` as having no
matching capture record — re-capture to record it. The upload reads only images,
so the record never reaches App Store Connect.

Screenshot manifest scenes can specify `formats` (for example `["appletv"]` or
`["iphone", "ipad", "mac"]`). Filtering happens before the screenshot budget,
so platform-specific routes do not consume another platform's slots. TV scenes
use `-UITestDashboard <id>` to pause a deterministic carousel page. Inspect the
images before pushing them. Mac captures use an isolated storage root; explicit
`POCKETCLOUD_HOME` roots must never import the operator's legacy web-fleet mirror.

---

## The Live Demo, in Every Foundation App (ADR-0102 T25/T26)

Any app that mounts `PocketCloudAccountView` (all six non-Business apps, in
Settings → Account & Sync) shows **Try the Live Demo** while signed out. It
mints an anonymous session on pocketcloud.org and fetches that app's own
corpus, keyed by the app key the sync scope already carries.

```bash
curl -s https://pocketcloud.org/api/demo/apps                      # catalog: no token needed
TOKEN=$(curl -s -X POST https://pocketcloud.org/api/auth/demo | jq -r .data.token)
curl -s "https://pocketcloud.org/api/demo/dataset?app=pocketmind" -H "Authorization: Bearer $TOKEN"
```

Corpora are one file per app under each site's `server/data/demo` directory
(for example, `web-fleet/pocket-cloud-dot-org-website/server/data/demo/pocketmind.json`).
`pocketbusiness`, `pocketmind`, and `pocketwellness` exist today. An unknown `?app=`
404s **naming what is available**, so an app whose corpus is not authored yet
says so instead of failing opaquely. Add one, deploy, and that app's demo is
live — no client release.

Fetching is not seeding: the app decodes its own collections and writes them
into its stores. PocketMind is the reference implementation
(`PocketMindDemoDataSeeder.seed(corpus:)`), and the same seeding path serves
the screenshot capture below.

> [!WARNING]
> A seeder that writes through a "log this now" API stamps every row with the
> current time, so weeks of seeded history render as N entries one minute old
> — the demo contradicting its own banner. Backdate explicitly
> (`WellnessMoodStore.log(…, at:)`) and **look at the timestamps in the UI**,
> not just the row count.

---

## Demo Modes and Launch Arguments (ADR-0102)

PocketBusiness (and the shared Pocket Admin module any app mounts) has three
data postures, and launch arguments select them deterministically — which is
what makes simulator verification reproducible instead of click-dependent:

| Launch Args | Posture | Data |
|---|---|---|
| any `-UITest*` arg | **Offline demo** | Bundled northwind corpus (`PocketCloudUI/Admin/Resources/DemoSeed/*.aml.md`) + a demo SQLite mirror, built **only** where no real mirror exists |
| `-UITestRemoteDemo` | **Live demo** | Mints `POST https://pocketcloud.org/api/auth/demo`, hydrates from `GET /api/demo/dataset` — same corpus shape over the real wire; falls back to the offline demo on any failure. Checked *before* the generic `-UITest*` short-circuit |
| `-UITestAuthTicket <url>` | **Signed in as a profile** (ADR-0127) | Redeems a single-use loopback ticket minted by `pocket auth session mint`, installs the tokens on the matching server row, then takes the ordinary connected path. Checked *first*, before every other `-UITest*` reading — a signed-in launch is a real session, not a demo. `-UITestScene <tab>` still picks the surface |
| *(none)* | Real bootstrap | Restores the persisted session / local-only choice |

```bash
xcrun simctl launch <UDID> clevercoding.com.PocketBusiness -UITestRemoteDemo -UITestDisableOnboarding
```

Proof that the live demo actually rode the wire comes from the simulator's own
log — the server's PM2 logs carry no per-request lines:

```bash
xcrun simctl spawn <UDID> log show --last 10m --predicate 'process == "PocketBusiness"' --info | grep -E "auth/demo|finished successfully"
```

---

## Signed-In Verification Without Typing a Password (ADR-0127)

Every connected surface — Dashboard, Users & Access, Control — needs a
session, and until 2026-09-13 the only way to one was a human typing a
password into the app, which an agent may not do and which the ADR-0118 argv
logger would record. Auth **profiles** fix that: a named credential set per
user/role in the encrypted store, and a **single-use loopback ticket** the
app redeems at launch.

```bash
# once, at a terminal — the password is prompted without echo (or piped, or
# read from the store with --from-env); it never appears in argv
pocket auth profile add admin --server https://pocket-admin.example.org --email admin@example.org --role admin
pocket auth profile list

# Customer Zero's owner, from the seeded control-plane env:
F=web-fleet/pocket-cloud-web-config/envs/pocketadmin/.env.main
unq() { sed -E 's/^[^=]*=//; s/^["'"'"']//; s/["'"'"']$//'; }
grep -E '^SEED_OWNER_PASSWORD=' "$F" | unq \
  | pocket auth profile add owner --server https://pocket-admin.clevercoding.com \
      --email "$(grep -E '^SEED_OWNER_EMAIL=' "$F" | unq)" --role owner

# iOS simulator: build, install, mint, launch signed in, land on a tab
pocket dev apple build-app PocketBusiness --simulator --auth-profile admin --scene users

# already installed: just mint and launch (simulator or macOS)
pocket auth session mint --profile admin --simulator PocketBusiness --scene users
pocket auth session mint --profile admin --launch PocketBusiness --scene dashboard
```

What the agent sees is `http://127.0.0.1:<port>/<nonce>` — worthless after
one fetch or 60 s — and a `✅ Ticket redeemed` line. Switching users is
relaunching with another profile; there is no in-app picker. Profiles are
tenant-scoped through `PocketCloudTenant.credentialKey`, so a client
project's profiles resolve only from that client's checkout (or `--tenant`).
Reference: `pocket auth` (`Documentation/Reference/PocketCLI/auth/README.md`).

---

## Verify Against Real Fleet Data on macOS

Run the installed macOS app in **Use Local Data** (local-only) mode — no control-plane
account involved — against a synced fleet mirror. This is also the lane that proves demo
seeding never touched real data (the demo mirror builder refuses wherever `mirror-state.json` exists).

```bash
# 1. The app syncs its OWN container when signed in to a control plane:
#    Databases → Sync site (needs an active Pocket Admin server + token).
#    The adjacent options menu offers an explicit fleet-wide sync.
# 2. Refresh the CLI mirror, then import through the app's folder picker:
pocket web db sync --site clevercoding --via-ssh   # whole-file snapshots; ~5–15 min
# Launch ~/Applications/PocketBusiness.app (installed by `pocket build app`).
# → Use Local Data → Databases → select the intended site's database
# → Import saved snapshots → choose ~/.pocketcloud/web-fleet in the folder picker.
```

Source timestamps survive import. The freshness tile uses the oldest source
timestamp, so refreshing one site cannot make every retained source look fresh.
The Insights dashboard separates human page views, captured leads, and AI
crawler hits; absent or unreadable measurements remain explicitly incomplete.

---

## The App Store Connect API Key

Everything that talks to App Store Connect — `ship`, the storefront verbs, and
the version resolution that stamps every build — needs one API key. It is
stored in two places that fail independently:

- **The manifest**, `~/.pocketcloud/apple/appstoreconnect/keys.json` (mode 600
  inside a 700 directory), holds the key id, the issuer id, a label, and which
  storage the key uses. No signing material.
- **The private key**, which lives in PocketCloud's own encrypted credential
  store. The manifest calls that storage `keychain`; it is a historical on-disk
  name and it is **not** the macOS Keychain. `Security.app` will not show it.

```bash
pocket dev apple asc add-key <KEY_ID> <ISSUER_ID> <path/to/AuthKey_KEY_ID.p8> \
  --label "<team>" --default
pocket dev apple asc bundle-ids     # cheapest live read; proves the key end to end
```

`add-key` moves the PEM into the encrypted store, so the downloaded `.p8` can be
deleted afterwards — the command says so when it succeeds.

---

## Ship — Outward-Facing Distribution

These verbs upload a binary to TestFlight and rewrite a public App Store listing.
Both are **dry-run-capable, and the dry run is the habit** — read the resolved plan
before letting either touch App Store Connect.

```bash
pocket dev apple project --regenerate --path Apps/PocketMind # after changing any included project.yml
pocket dev apple ship --scheme PocketMind-iOS --dry-run       # resolved plan only, changes nothing
pocket dev apple ship --scheme PocketMind-iOS --archive-only  # stop after archiving
pocket dev apple ship --scheme PocketMind-iOS --upload-only   # upload, do not submit to the beta group
pocket dev apple ship --scheme PocketMind-iOS                 # archive → export → TestFlight → beta group
```

`ship` infers the platform from the scheme and asks App Store Connect for *that
platform's* next version and build number — iOS and macOS are independent trains
sharing a bundle id, which is why versions are never pinned in `project.yml`.

**Every archive is audited for secrets before anything leaves the machine**
(ADR-0065 T12). Straight after the archive stage — `--archive-only` included,
since Xcode's Organizer can upload an archive on its own — `ship` scans the
archive's `Products/Applications` byte for byte. It looks for files that are
secrets by kind (`.env`, `.p8`, `.p12`, keystores, SSH keys), secret key
shapes (Stripe secret, restricted and webhook keys, PEM private keys with a
body, GitHub, AWS, AI-provider, Hugging Face, Slack, Resend), and the exact
secrets configured on this machine, which it reports by key name only.
Publishable values are expected and pass: product IDs, URLs, a Stripe
`pk_live_` key. A finding, or an archive with no apps to audit, stops the
pipeline with nothing exported or uploaded; remove the secret from the target
and rotate it. The same scan runs by hand on any build:
```bash
pocket quality scan bundle ~/Applications/PocketMind.app      # or an .xcarchive / .appex
pocket quality scan bundle build/PocketMind.xcarchive --json
```

**A cold package cache is slow, not stuck** (ADR-0144 T10). The archive resolves
packages before it compiles, into the project's own SourcePackages bucket. A
project's first archive, and the first after its bucket is cleaned, clones every
package. That includes `mlx-swift`'s `mlx` submodule: 70 MiB and 58,843 deltas,
50 s on a quiet machine and minutes at a load average of 200. The preflight keeps
running while it visibly advances: output, bytes arriving under SourcePackages, or
CPU in its own process tree. It ends in one of two ways, and the error names which:

- **stalled**: nothing changed for 180 s. CPU alone can stretch this to 540 s. The
  cause is a network or a lock, and a longer wait will not help.
- **ceiling**: still advancing at 1800 s. The machine or the link is slow. Raise the
  ceiling and rerun. Repositories that finished cloning are kept, so the next run
  starts further along.

Both errors print the SourcePackages state at the start and at the end, plus every
submodule not yet fetched. A long resolve prints a `⏳ Resolving packages …` line
every 30 s, and `--dry-run` prints both budgets.
```bash
pocket dev apple ship --scheme PocketMind-iOS --archive-only --archive-timeout 3600   # slow link: raise the ceiling
```

**A package cache that predates its pins heals itself** (ADR-0144 T11). `pocket build app`,
`apps` and `all` build every sample app from one shared cache,
`~/.pocketcloud/build/sample-apps/pocketcloud/SourcePackages`, with `-skipPackageUpdates`, so
xcodebuild never fetches. When a commit pins a version newer than the cache's last fetch, every
app build fails: `Dependencies could not be resolved because no versions of 'mlx-audio-swift'
match the requirement 0.1.6`. On 2026-10-26 the cache had last fetched on 2026-09-03, and
ADR-0137 T13 had pinned three newer versions the day before. The build now heals it, on two
facts: xcodebuild could not resolve, and a cached clone under `repositories/` lacks a revision
`Package.resolved` pins. It prints one line per clone and rebuilds once:
```
📦 'mlx-audio-swift' is pinned at 0.1.6 by …/Package.resolved, and the shared package cache's clone predates it. Fetching into …/repositories/mlx-audio-swift-4f05d7e9, then rebuilding once (ADR-0144 T11).
```
The fetch only adds tags and branches; nothing is deleted or reset, so the rebuild is
incremental. It leaves the cache alone, and the build fails as before, when another process's
command line names the cache, when the process table cannot be read, or when the build is
`--offline`. If the fetch works and the revision is still missing, the error says the pin names a
version its upstream lacks: fix `Package.resolved`, not the cache. To check the cache by hand:
```bash
SP=~/.pocketcloud/build/sample-apps/pocketcloud/SourcePackages
git -C "$SP/repositories/mlx-audio-swift-4f05d7e9" cat-file -e <pinned-revision>^{commit} && echo present
```
`--archive-timeout` also puts a wall clock on the archive build, which has none
by default. The flag is not called `--timeout` because the root `pocket` command
claims that name, so a leaf flag spelled the same would be accepted and ignored.

If development provisioning blocks a distribution archive because no physical
device is registered (for example, a first tvOS upload), use:
```bash
pocket dev apple ship --scheme PocketBusiness-tvOS --archive-only --defer-signing-to-export
```
This leaves archive signing to the distribution export; it does not disable
export signing. Export with `pocket dev apple export`, then inspect the exported
`DistributionSummary.plist` before upload.

**A ship that failed after its archive does not archive again** (ADR-0029 T26). Once the archive
passes its secret audit, `ship` writes `<archive>.ship-receipt.json` beside it, naming the version,
build and source tree it was built from: HEAD plus every uncommitted change, ignored files left
out. A rerun that plans the same version and build reuses the archive when the receipt matches, the
archive's own `Info.plist` carries that version and build, and the source still hashes the same.
Otherwise it archives, and the plan line says why, for example `archive  build: the source changed
since … was archived: 2 file(s), e.g. …`. `--fresh-archive` forces a new archive. On 2026-10-26 a
transient Apple ID login failure at export cost a tvOS ship its 14-minute archive; now a rerun of
`ship` goes straight to export.

**A missing project is generated first** (T25). `PocketMind.xcodeproj` is gitignored, so a fresh
clone or worktree has none, and `ship` used to fail in 3 s with "… does not exist". It now runs
xcodegen beside the project's `project.yml` before archiving, and the plan says so. The gitignored
inputs that spec reads (`Generated/Apple`, `Apps/PocketMind/*/Generated`, the MLX metallib) are
not generated: copy them from a working checkout first.

**An upload made by hand still arms the duplicate-build guard** (T24). `pocket dev apple appstore
upload` records the build in `~/.pocketcloud/apple/uploads/` when it is given `--platform`,
`--marketing-version` and `--build-number`, and names what is missing when it is not. Without that
record, the next `ship` can plan the number Apple has just accepted and be refused with 409 after a
full archive. Rerunning `ship` is the better way to finish; by hand it is:
```bash
pocket dev apple export --archive-path <archive> --export-path <dir> --export-options-plist scripts/export-options-tvos.plist
pocket dev apple appstore upload --bundle-id clevercoding.com.PocketBusiness --ipa-path <dir>/PocketBusiness.ipa --platform tvos --marketing-version 1.0.0 --build-number 4
```

**Ship from a clean worktree when the checkout holds someone else's work.** When the main checkout
has other sessions' uncommitted or unpushed changes in code the app links, ship from
`git worktree add --detach .claude/worktrees/<name> origin/develop`. Run `pocket build worktree` for the
generated inputs and copy `.pocketcloud/apps/<scheme>.json` in first. The upload record is machine-scoped, so the guard holds
there too (T21). A tvOS app extension with a manual `Info.plist` must list
`UIRequiredDeviceCapabilities` = `arm64`: Xcode adds it only to the app target, and Transporter
refuses the whole upload without it.

**A tvOS simulator cannot prove a build launches on a tvOS 26 Apple TV (ADR-0137 T18).** TestFlight
build 2 died in dyld on an Apple TV HD running tvOS 26.6 (`Library not loaded:
…/PDFKit.framework/PDFKit`). The tvOS 27 SDK ships PDFKit and links it strongly, and
the tvOS 26.5 simulator runtime ships PDFKit too, so the simulator launches a binary
that the device refuses. Check the archive's strong links instead. Every
`/System/Library/Frameworks` line without `weak` must exist on the oldest tvOS you
support:
```bash
otool -L build/PocketBusiness-tvOS-1.0.0-<n>.xcarchive/Products/Applications/PocketBusiness.app/PocketBusiness | grep -v weak
```
Then run it on a physical device, which `pocket dev apple run` installs and launches
with `devicectl` (before T18 it only copied the build into DSTROOT and said it had
installed it):
```bash
pocket dev apple run -p Apps/PocketMind/PocketMind.xcodeproj -s PocketBusiness-tvOS -d "platform=tvOS,id=<udid>" --bundle-id clevercoding.com.PocketBusiness
```
Pull the device's crash logs with
`xcrun devicectl device info files --device <udid> --domain-type systemCrashLogs`.

**The Apple TV icon and Top Shelf art are generated (ADR-0148 T36).** `scripts/generate-business-tv-icon.swift`
draws the three-layer image stacks (Background, Middle, Foreground; 400×240 @1x/@2x and
1280×768) and both Top Shelf images (1920×720 and 2320×720, @1x/@2x) from vectors. Change
the script rather than the PNGs: the old art was the square iOS icon stretched to 5:3.
```bash
swiftc -O -o /tmp/business-tv-icon scripts/generate-business-tv-icon.swift
/tmp/business-tv-icon --preview /tmp/business-tv-preview   # from the workspace root
```

The shared iOS export plist uses automatic signing without a bundle-specific
provisioning-profile map. This lets each sample app resolve its own App Store
profile. PocketGamer iOS build 31 verified this path on 2026-10-22 (`A14E1A95`):
the exported summary names its own bundle/profile, the intended team, production
entitlements and a cloud-managed Apple Distribution certificate. Keep a manual
profile selection in an explicitly supplied app-specific export plist.

For headless export with an existing file-based App Store Connect key, set
`APP_STORE_CONNECT_KEY_PATH`, `APP_STORE_CONNECT_KEY_ID`, and
`APP_STORE_CONNECT_ISSUER_ID` in the command environment. Export validates the
selection against the active tenant and forwards Xcode's authentication key
arguments alongside `-allowProvisioningUpdates`.

The storefront side rewrites the public listing, and is likewise dry-run by default:
```bash
pocket dev apple storefront push --app pocketmind             # DRY RUN: diff vs live listing
pocket dev apple storefront push --app pocketmind --apply     # LIVE, public
```

### Pull listing copy from ASC (pc-port seed)

For apps that already exist on App Store Connect (legacy Mathlete / First Words
ports, ADR-0151 T04), seed the AML SSOT from live metadata instead of inventing
copy from a README:

```bash
pocket dev apple asc bundle-ids                               # prove the ASC key
pocket dev apple storefront pull --preset mathlete-family --write-project
pocket dev apple storefront pull --preset firstwords-family --write-project
pocket dev apple storefront pull --bundle-id firstwords --app-key firstwords

# Then in a port repo (Apps/ is the ship discovery root):
cd Apps/Mathlete && xcodegen generate && cd ../..
pocket dev apple ship --scheme Mathlete-iOS --dry-run
```

`--write-project` seeds `Apps/<Product>/` (`project.yml`, Sources, **Assets/**
for legacy image copies, Info plists) plus sticky `.pocketcloud/apps/<scheme>.json`
at the port repo root. Ship already prefers `Apps/` when resolving schemes.
Re-run with `--overwrite` to prefer ASC over existing AML fields; Swift sources
are never clobbered.

Support subscription prices are a separate lane from the listing. `pocket dev apple payments` reads `support_catalog_plans` in the pocketcloud operational database when that snapshot is on disk (`pocket web db sync --site pocketcloud`, `--database`, or `POCKETCLOUD_SUPPORT_CATALOG_DB`). When no database is present, it reads `Documentation/Internal/Monetization/pocketcloud-support.aml.md`. `plan` does not write. `sync` writes App Store Connect only with `--apply`.

```bash
pocket dev apple payments validate
pocket dev apple payments plan --app pocketmind
```

---

## Tests

```bash
pocket dev apple test                       # 600s default; --test-timeout to override
pocket dev apple test --uitest
```

A run that executes zero tests **fails loudly** rather than reporting a vacuous
TEST SUCCEEDED — on both the plain and `--uitest` paths (ADR-0082 T11).
