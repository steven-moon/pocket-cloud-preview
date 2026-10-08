---
id: "doc.pocketcloud.runbook.android-hybrid"
title: "Android Apps & Hybrid (Tauri/Vite) Bundles"
types: ["implementation-guide", "reference"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "android", "hybrid", "tauri", "vite", "google-play"]
owners: ["@steven"]
version: "1.11.0"
last_updated: "2026-10-04T00:00:00Z"
last_reviewed: "2026-10-04"
is_archived: false
---
# 3. Android Apps & Hybrid (Tauri/Vite) Bundles

[← Back to Runbook Index](index.md)

Android renders the same web bundle the desktop shells render. There is **no
sidecar** — the Android shell plugin cannot spawn, and the OS forbids exec from
the app data directory (ADR-0082 D3, still in force). Inference is either a
paired PocketCloud node over HTTP (ADR-0082's remote-core lane) or, per
**ADR-0103**, the kernel and llama.cpp **linked in-process** — loaded by the
dynamic linker, which none of D3's exec arguments touch.

---

## Cross-Compiling Swift for Android from a Windows Host (ADR-0103 T01)

Verified 2026-08-15 on this workstation — **no WSL, no Docker, no Mac**. The
Swift 6.3.2 Windows installer already ships the Android platform at
`%LOCALAPPDATA%\Programs\Swift\Platforms\6.3.2\Android.platform`. Note that
`swift sdk list` reports nothing installed — that lists artifactbundles, which
this route does not use. Drive the bundled platform directly:

The full working recipe against **NDK r27d** (`ndk;27.3.13750724`, clang 18),
which is what Foundation requires (`PRE` = the NDK's
`toolchains/llvm/prebuilt/windows-x86_64`):

```bash
swift build --triple aarch64-unknown-linux-android28 \
  --sdk "$LOCALAPPDATA/Programs/Swift/Platforms/6.3.2/Android.platform/Developer/SDKs/Android.sdk" \
  -Xswiftc -sysroot -Xswiftc "$PRE/sysroot" \
  -Xcc "--sysroot=$PRE/sysroot" \
  -Xcc "-I$LOCALAPPDATA/Programs/Swift/Platforms/6.3.2/Android.platform/Developer/SDKs/Android.sdk/usr/include" \
  -Xlinker -L -Xlinker "$PRE/lib/clang/18/lib/linux" \
  -Xlinker -L -Xlinker "$PRE/lib/clang/18/lib/linux/aarch64"
```

Every flag is load-bearing, and each omission fails with a message that does not
name the cause (all hit and verified 2026-08-15):

| Missing Flag | Symptom |
|---|---|
| `-L …/lib/linux` | `ld.lld: unable to find library` (clang builtins live there) |
| `-L …/lib/linux/aarch64` | `ld.lld: unable to find library -l:libunwind.a` |
| `-Xcc --sysroot` | `LibcOverlayShims.h: 'semaphore.h' file not found` — the **clang importer** needs its own sysroot; `-Xswiftc -sysroot` does not reach it |
| `-Xcc -I<sdk>/usr/include` | `cannot load underlying module for 'Dispatch'` — the `-Xcc --sysroot` override hides the Swift SDK's bundled Block/dispatch/os headers |

More gotchas, all verified:
- **NDK r27+ required for Foundation.** On r25c, `import Foundation` fails at
  `sys/stat.h: 'timespec' must be imported from module '_linux_time'` — the old
  sysroot headers predate the Swift SDK's module map. Stdlib-only code builds
  fine on r25c.
- **sdkmanager needs `JAVA_HOME`** — the JDK on this box is at
  `%LOCALAPPDATA%\Java\jdk-17.0.20+8`.
- **Stale module caches poison retries.** After switching NDKs, `rm -rf .build`
  or the previous NDK's precompiled modules produce misleading errors.
- **Only `aarch64` and `x86_64`** are shipped in the Swift Android SDK; the
  four-ABI jniLibs layout narrows to two for anything carrying Swift.
- The Swift **runtime `.so`s to ship in jniLibs** (libswiftCore, libFoundation,
  libdispatch, …) are all in
  `Android.sdk/usr/lib/swift/android/{aarch64,x86_64}/`.

The verb surface deliberately mirrors the Apple lane rather than inventing a
second vocabulary:

| Apple | Android |
|---|---|
| `pocket build apps` | `pocket build android` / `pocket build android apps` |
| `pocket build app <Name>` | `pocket build android app <Name>` |
| — | `pocket build android check` · `pocket build android list` |
| `pocket dev apple simulator --list` | `pocket dev android devices` |
| `pocket dev apple simulator --boot` | `pocket dev android emulator --boot` |
| `pocket dev apple build-app X --simulator` | `pocket dev android build-app X --emulator` |
| — | `pocket dev android logs <App>` |

`pocket dev android` is **not** macOS-gated, unlike `pocket dev apple`: the
Android toolchain runs on Linux and Windows too.

---

## Toolchain

The SDK and NDK are **discovered**, not demanded — the same way the Apple lane
finds Xcode without anyone exporting a path to it. `ANDROID_HOME`, `NDK_HOME`,
and their aliases remain available as overrides.

```bash
pocket build android check        # every prerequisite, where it was found, and the fix
```

Last verified on: NDK **27.3.13750724**, rustup targets `aarch64-linux-android`,
`armv7-linux-androideabi`, `i686-linux-android`, `x86_64-linux-android`, JDK
17.0.17, Gradle 8.14.3. Pin the NDK deliberately — NDK releases have broken Rust
builds before.

---

## Build

```bash
pocket build android                          # every sample app, debug APK
pocket build android app PocketMind
pocket build android apps --release           # release AAB
pocket build android list                     # apps with an Android target, and whether built
```

Android is **not** in `pocket build all` and not in `SampleAppRegistry`:
`AppBuildSpec` hardcodes a macOS scheme and the installer resolves an Xcode
project, so registering it would silently enroll a Gradle target in the Apple
pipeline.

`src-tauri/gen/android` is generated and gitignored; the verb re-runs
`tauri android init` when it is missing, so a fresh clone needs no extra step.

---

## Run on the Emulator

```bash
pocket dev android devices                    # AVDs + attached devices, with boot state
pocket dev android emulator --boot            # first AVD
pocket dev android emulator --boot --name pocketcloud-test

pocket dev android build-app PocketMind --emulator
pocket dev android build-app PocketMind --emulator --device pocketcloud-test
pocket dev android build-app PocketMind --emulator --screenshot /tmp/mind.png --settle 14
pocket dev android build-app --all --emulator --screenshot-dir /tmp/android-shots
```

The device is resolved and booted **before** the build, for two reasons: a boot
failure should surface before the expensive work, and knowing the target lets
the build compile only that device's ABI. A universal debug APK is ~690 MB and
pays for three cross-compiles the device cannot execute; one ABI is ~185 MB.
This is the same narrowing the Apple lane does with `ONLY_ACTIVE_ARCH=YES`.

Point a run at a paired node:

```bash
pocket dev android build-app PocketMind --emulator --core-origin http://192.168.1.10:8089
```

---

## Diagnose a Launch That Renders Nothing

A blank screen with a completely green build is the characteristic hybrid
failure, and it is invisible everywhere except the webview console:

```bash
pocket dev android logs PocketMind
```

This is how `Uncaught ReferenceError: exports is not defined` was found — a bare
`import "@pocketcloud/web-lib"` pulling the package's CommonJS barrel, whose
`require`/`exports` survive into the IIFE bundle and abort the rest of the
module at load. Import web-lib by **deep path** only.

Skip the build when iterating on the device:
```bash
pocket dev android build-app PocketMind --emulator --skip-build
```

---

## The Google Play Store Lane (ADR-0106)

The store lane is **read-write** — D4's edit-session ladder governs writes, it
does not forbid them (ADR-0106, amended 2026-08-22; nothing may brand the CLI
"read-only"). Every write rides one Play edit with three rungs, and the default
is always the safest:

| Rung | Flag | What Google does |
|---|---|---|
| rehearse | *(none)* | validates the edit server-side, then it is **abandoned** — a real answer, nothing kept |
| apply | `--apply` | commits, `changesNotSentForReview=true`; refused outright on a never-published app |
| submit | `--submit-for-review` | commits AND lets it go to review — the only externally visible rung |

```bash
# Mirror a track into the tenant's local snapshot (per-track, tenant-scoped):
pocket dev stores sync --google-play --play-track internal

# Upload a bundle / compose bundle+listing+images in one edit:
pocket dev android publish --app PocketMind --track internal
pocket dev android release --app PocketMind --submit-for-review

# Promote the release already SERVING on one track to another — no re-upload.
pocket dev android promote --package com.wanderlore.player --from internal --to beta
pocket dev android promote --package com.wanderlore.player \
    --from internal --to beta --as-draft --submit-for-review
```

Support prices use that same pocketcloud database and do not open a Play edit. When `support_catalog_plans` is absent, the command reads the support AML.

```bash
pocket dev android payments validate
pocket dev android payments plan --app pocketmind
```

Three findings that will save the next operator a day, all verified against Google:
1. **A tenant must declare its packages or the mirror refuses.** `stores sync`
   reads the tracked app-portfolio registry
   (`Packages/Kernel/Core/Sources/PocketCloudCommon/Resources/app-portfolio.json`).
2. **"Published on internal testing" is not "reviewed."** Internal never
   triggers review. A draft app — one that has never completed first
   publication — takes only **draft** releases on public tracks (`promote --as-draft`).
3. **Provenance prints before anything acts** (ADR-0090): every write verb
   states the service account and package first.

---

## 4. Hybrid (Tauri/Vite) Bundles

Eight shells consume `web-lib`: the seven sample apps under `HybridApps/` and
the desktop shell at `Apps/PocketDesktop`.

```bash
pocket build bundles              # web-lib first, then every shell — order matters
pocket quality scan hybrid        # gates the BUILT dist/, not the source
pocket quality scan hybrid --json
```

`scan hybrid` is what catches a bundle that will render blank inside a `.app` or
a webview: absolute asset paths, ES-module output where an IIFE is required,
remote subresources. The shared fix lives in
`HybridApps/shared/vite-hybrid-config.ts` — use `hybridViteConfig()` rather than
re-deriving it per app.

The Rust half is shared too: `HybridApps/shared/pocket-tauri-shell` owns the
`pocket://` relay, core-origin resolution, and the desktop tray for all eight
shells.

```bash
cargo test --manifest-path HybridApps/shared/pocket-tauri-shell/Cargo.toml
```
