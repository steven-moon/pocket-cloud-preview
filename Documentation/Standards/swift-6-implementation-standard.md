---
id: "doc.internal.architecture.standards.swift_6_implementation"
title: "Swift 6 Implementation Standard"
types: ["standard", "source-of-truth"]
tags: ["architecture", "standards", "swift", "swift-6", "swift-6-4", "concurrency", "ios", "macos", "swiftui", "adaptive-layout", "accessibility", "voiceover", "theming", "dynamic-type", "api-design", "observability", "unified-providers"]
owners: ["@steven"]
version: "26.10.16"
status: "active"
last_reviewed: "2026-10-16"
---

# Swift 6 Implementation Standard

This document is the canonical source of truth for Swift 6 implementation expectations across PocketCloud ADRs, task plans, and code changes.

It is named for the **language mode**, not a toolchain release. The current toolchain is recorded once, in *Toolchain Baseline* below, so a minor-version advance edits this file and the manifests — not the hundreds of ADRs and tasks that link here.

If another ADR, task, or local implementation note conflicts with this document, this standard is authoritative unless the conflict is intentional and updated in the same change set.

Canonical standards index:
[`Documentation/Standards/README.md`](README.md).

## Toolchain Baseline

| Surface | Current (since 2026-10-15) |
|---|---|
| Language mode | Swift 6: complete data-race checking for every target |
| Toolchain | Swift 6.4, Xcode 27.0 (`swiftlang-6.4.0.34.1`) |
| Package manifests | `// swift-tools-version: 6.4` in every tracked `Package.swift` |
| Xcode projects | `SWIFT_VERSION: "6.4"` in `configs/project_base.yml` and every app `project.yml`. Xcode reduces it to the language mode and passes `-swift-version 6`; `swiftc` itself accepts only `4`, `4.2`, `5` and `6`. Don't "correct" it in either direction. |
| SwiftPM build system | Swift Build, which is the SwiftPM 6.4 default. `--build-system native` is deprecated. |
| Linux cross-compile | swiftly `6.4.0` with the `swift-6.4.0-RELEASE_static-linux-0.1.0` SDK, and no `SDKROOT` override. Use swiftly 1.2.0 or later: 1.1.x builds the wrong download URL for 6.4.x. Until a released 6.4.x carries SwiftPM #10556, a graph with Apple-only conditional dependencies also needs `--build-system native` (Toolchain §7). |
| Windows / Android hosts | A swift.org toolchain of the same version. A 6.3 toolchain cannot parse a `swift-tools-version: 6.4` manifest at all. |

> **Toolchain history:** Swift 6.2 (adopted 2026-03-21) → 6.3 on 2026-07-25
> (Xcode 26.6, Swift 6.3.3; tools 6.3 unblocked `mlx-swift` `0.31.6`'s
> experimental C generation) → 6.4 on 2026-10-15 (Xcode 27.0). The 6.4 advance
> also renamed this file from `swift-6-3-implementation-standard.md`.
> Governance remains ADR-0019 (`Documentation/ADRs/ADR-0019-swift-6-2-compliance-and-toolchain-governance.md`),
> whose filename records the original 6.2 adoption.

## Primary References

Use primary Swift and Apple sources when making language, concurrency, package, debugger, or framework decisions:

1. [Swift 6.4 Released](https://www.swift.org/blog/swift-6.4-released/) and the [proposals implemented in 6.4](https://www.swift.org/swift-evolution/#?version=6.4)
2. [Swift 6 Concurrency Migration Guide: Enabling Complete Data-Race Safety](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/)
3. [Value and Reference Types in Swift](https://www.swift.org/documentation/articles/value-and-reference-types.html)
4. [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
5. [Swift Standard Library](https://www.swift.org/documentation/standard-library/)
6. [Swift Core Libraries](https://www.swift.org/documentation/core-libraries/)
7. [Swift Package Manager documentation](https://docs.swift.org/swiftpm/documentation/packagemanagerdocs/)
8. [Swift Package Manager repository](https://github.com/swiftlang/swift-package-manager)
9. [Swift REPL and Debugger](https://www.swift.org/documentation/lldb/)
10. [Swift Source Code](https://www.swift.org/documentation/source-code/)
11. [Swift Continuous Integration](https://www.swift.org/documentation/continuous-integration/)
12. [Swift language repository](https://github.com/swiftlang/swift)
13. [swiftlang GitHub organization](https://github.com/swiftlang)
14. [Apple Swift Standard Library documentation](https://developer.apple.com/documentation/swift/swift-standard-library)
15. [Swift Subprocess](https://github.com/swiftlang/swift-subprocess)

## Baseline Policy

1. New and materially changed Swift code targets the Swift 6 language mode on the toolchain in *Toolchain Baseline*, and may use any language feature that toolchain ships.
2. Do not add backward-compatibility shims for unreleased app behavior. PocketCloud apps have not shipped; prefer the clean Swift 6 design over preserving stale internal behavior.
3. Every runtime change must work on iOS and macOS at minimum unless the task explicitly scopes itself to a platform-specific surface.
4. For SwiftUI surfaces, "works on iOS and macOS" includes iPhone portrait widths, iPad split-view widths, and narrow macOS windows unless the task explicitly declares a desktop-only/operator-cockpit surface.
5. Prefer the newest stable Swift, Foundation, SwiftPM, Swift Testing, Observation, and Apple platform APIs available to the target platform.
6. Use `./scripts/swiftw` for local Swift build/test commands in this workspace.
7. Validate completed work with `pocket system verify --exhaustive` unless a narrower task explicitly documents why that gate is impractical.
8. Minimize third-party code in this order: delete an unnecessary dependency;
   use the Swift standard library/Foundation; use the official upstream stable
   release with a reproducible pin; maintain a fork only as a temporary,
   documented last resort. Vendoring a package is not dependency removal.

## Concurrency and Data-Race Safety

1. Treat complete data-race safety as the default expectation.
2. Prefer structured concurrency (`async`/`await`, task groups, async sequences) over callback pyramids, unstructured background queues, or ad-hoc locks.
3. Use actors for shared mutable state that must outlive one call.
4. Keep UI-facing state and view models isolated to `@MainActor`.
5. Use `@concurrent` only when code intentionally leaves the caller's actor to run concurrently.
6. Avoid `Task.detached`; when unavoidable, document the isolation boundary, cancellation behavior, priority, and Sendable inputs.
7. Mark cross-actor values, closures, and protocol requirements `Sendable` / `@Sendable` where they cross concurrency domains.
8. Treat `@unchecked Sendable`, `nonisolated(unsafe)`, and global mutable state as last-resort escape hatches requiring a comment, tests, and an ADR/task note.
9. Do not block actor executors or the main actor with synchronous I/O, long CPU work, or process waits. Move that work behind async APIs or an explicit concurrent boundary.
10. Propagate cancellation and timeouts through async workflows; do not ignore `Task.isCancelled`.
    An `async` XCTest method that calls into a SwiftUI view type, even a static helper on it, is marked `@MainActor`. A synchronous test already runs on the main thread, but an async one does not, and Swift's runtime isolation check then traps the whole test process (signal 5) instead of failing one assertion. `PlayHistoryScopeTests` hit this on 2026-10-15 calling `PlayHistoryPanel.gameName`.
    **A timeout must not wait for the work it times out.** Racing work against a `Task.sleep` in a task group bounds nothing: a group awaits every child before it returns, even after one has thrown, and a child blocked in a synchronous call never sees cancellation. The command pre-run's five-second budget lasted 32 minutes that way (2026-10-15). For optional work (context, hints, enrichment), use `BoundedWait.value(within:)` (PocketCloudCommon/Concurrency), which returns at the budget and leaves the work behind.
11. The Swift 6 language mode is in force by default: any `swift-tools-version: 6.x` manifest selects Swift 6 language mode (complete data-race checking) for every target — no explicit `swiftLanguageMode` is needed. Do **not** set `swiftLanguageMode` to an older version to silence isolation diagnostics; fix the isolation. Use `SwiftSetting.enableUpcomingFeature`/`.enableExperimentalFeature` per target only to adopt forthcoming behavior early. See the [Swift 6 Concurrency Migration Guide](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/).
12. **Cleanup that must finish runs in the scope that owns it.** In an `async` context a `defer` body may `await` ([SE-0493](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)), and the deferred work runs to completion before the scope exits. Release a lease, close a handle, or `finish()` a continuation there. Do not do it in an unstructured `Task { }` fired from `defer`, which outlives the scope and races the next acquirer.
13. **A cancelled task still has to clean up.** When teardown awaits work that checks cancellation, wrap it in `withTaskCancellationShield` ([SE-0504](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md)). Otherwise the cancellation that triggered the teardown also aborts it. A shield hides cancellation from the code inside it, child tasks included, until the block returns. It does not un-cancel the task. Shield only teardown and commit-or-rollback steps, never a whole operation: a shielded loop is a task nobody can stop.

## Value and Reference Semantics

1. Prefer `struct`, `enum`, and immutable value semantics for domain models, configuration, reports, events, and DTOs.
2. Use `class` only for identity, inheritance, Objective-C/Apple framework integration, or explicit shared mutable state.
3. Mark classes `final` unless subclassing is required and documented.
4. Use `actor` rather than `class + lock` for Swift-owned mutable shared state.
5. Keep mutation local and explicit. Avoid hidden shared state because it breaks local reasoning.
6. Prefer standard collection value types (`Array`, `Dictionary`, `Set`, `String`) and copy-on-write semantics before custom reference containers.
7. Ownership machinery is for cases that need it. That covers non-copyable types (`~Copyable`, `borrowing`/`consuming`) and the Swift 6.4 additions: `UniqueBox` (SE-0517), `UniqueArray` (SE-0527), `Ref`/`MutableRef` (SE-0519), borrowing iteration with `Iterable` (SE-0516), and borrow/mutate accessors (SE-0507). Use them for a resource whose uniqueness is the point, such as a file descriptor, a GPU buffer or a lease token, or for a hot path where a profile shows copies. Ordinary models, DTOs and view state stay copyable values. Non-copyable types can now conform to `Equatable`, `Comparable` and `Hashable`, so that is no longer a reason to make one copyable.

## API Design

1. Follow the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/): optimize clarity at the call site over brevity. Concretely:
   - Methods read as grammatical phrases at the call site; factory methods begin with `make` (`makeIterator()`). Side-effect-free members are noun phrases; mutating members are imperative verbs, with the non-mutating pair as the `-ed`/`-ing` participle or `form`-prefixed form.
   - Booleans/Boolean members read as assertions about the receiver (`isEmpty`, `contains(_:)`).
   - Protocols describing *what something is* are nouns (`Collection`); protocols describing a *capability* end in `-able`/`-ible`/`-ing` (`Equatable`, `ProgressReporting`).
   - A first argument forming a prepositional phrase takes a label starting at the preposition (`removeBoxes(havingLength:)`); otherwise fold the preceding words into the base name. All other arguments take labels. Omit the first label only for value-preserving conversions (`Int64(someUInt32)`).
2. Name variables, parameters, methods, and associated types by role, not implementation type.
3. Omit needless words, but keep labels needed to disambiguate behavior.
4. Public and package-visible declarations need concise documentation comments when their behavior is not obvious from the declaration.
5. Prefer protocol boundaries only when they express a real substitutable contract; avoid abstraction for its own sake.
6. Keep file size under the workspace 500-line rule and split extensions as `{ParentType}+{FunctionalGrouping}.swift`.
7. Disambiguate a name with a module selector, `ModuleName::Name` ([SE-0491](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0491-module-selectors.md)), when `ModuleName.Name` cannot do it. That happens when a type shares its module's name, or when two imported modules add the same nested name to one type. Don't use a file-local `typealias` or a rename for this. Plain `PocketCloudLogger.Logger` qualification still works where it is unambiguous.
8. Write optional opaque and existential types without parentheses: `some View?` and `any Error?`, not `(some View)?` ([SE-0521](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0521-improved-optional-opaque-and-any.md)).

## Libraries and Frameworks

1. Prefer the [Swift Standard Library](https://www.swift.org/documentation/standard-library/) and [Swift Core Libraries](https://www.swift.org/documentation/core-libraries/) before custom infrastructure — reach first for the current types: `Duration`/`Clock` (not ad-hoc `TimeInterval` math), `Regex` (not `NSRegularExpression`), `Synchronization`'s `Mutex`/`Atomic` (not `NSLock`/`os_unfair_lock`), `Span`/`InlineArray` for bounded contiguous data, and `FoundationEssentials` on non-Apple platforms. For raw bytes, use the safe `RawSpan` loading API ([SE-0525](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0525-rawspan-safe-loading-api.md)) and span-based `withTemporaryAllocation` (SE-0524). Use pointer arithmetic on `UnsafeRawPointer`/`withUnsafeBytes` only at a C boundary.
2. Prefer modern Foundation APIs and typed data models over stringly typed dictionaries.
3. Prefer Swift Testing for new test surfaces when the package already uses it or when adopting it does not create duplicate test stacks. Keep XCTest where it is already the local convention. Since Swift 6.4, `#expect` works inside an XCTest method and `XCTAssert` inside a Swift Testing test (ST-0021). An XCTest target can therefore adopt Swift Testing one test at a time, with no second test stack.
4. Use the Swift Testing affordances when they improve evidence quality: attachments (on Apple platforms an attachment can be any `Transferable`, ST-0023), exit tests, and `CustomTestReflectable` (ST-0022) for readable failure output. To chase a flaky test, repeat just that test case with `swift test --filter … --maximum-repetitions N --repeat-until fail` (ST-0024, Swift Testing suites only) rather than looping a whole suite.
5. Prefer `Observation` and `Observations` for state-change streams in SwiftUI-facing code where available. A consumer outside SwiftUI that needs every change, not just the next one, uses advanced observation tracking (SE-0506) rather than polling or a Combine bridge.
6. Prefer typed `NotificationCenter` messages over raw names and dictionary payloads where platform availability allows.
7. Use SwiftPM manifest APIs for package-level settings; avoid shell glue or generated package files.
8. For process execution in Swift code, prefer Swift-native async process APIs available in the target toolchain. [Swift Subprocess](https://github.com/swiftlang/swift-subprocess) reached **1.0** alongside Swift 6.4 and is the cross-platform choice; otherwise isolate legacy `Process` usage behind a thin async adapter. The workspace still pins `swift-subprocess` `0.4.0` in AIStack. Moving that pin to 1.0 is its own task, not something to do in passing inside a feature change.
9. **Drain a child's pipes before you reap it.** A pipe holds about 64 KB, and a child that writes more blocks until someone reads. `waitUntilExit()` followed by `readDataToEndOfFile()` (or a read inside `terminationHandler`) therefore hangs both processes for ever, at 0% CPU, the first time the output outgrows the pipe. Read with `PipeDrain.readToEnd(stdout:stderr:)` or `PipeDrain.readToEnd(_:)`, then wait. For git, use `DefaultGitRunner`. Never drain two pipes one after the other, and never drain on a GCD queue (`PipeDrain`'s header says why). `pocket quality scan pipes` gates it at the commit (ADR-0049 T08).

## Unified Provider Integration

1. Treat integration with PocketCloud's unified providers as part of the standard
   implementation process, not as optional polish.
2. When adopting older code or moving an existing class into an active runtime path, audit
   it for missing unified-provider integration before closing the task.
3. Prefer existing providers over feature-local systems:
   `UnifiedLogger`/`Logger` for diagnostics, `TelemetryManager` for structured events,
   `MetricsCollector`/`MetricsEvent` for aggregate observability,
   `UnifiedLearningAdapter`/`UnifiedLearningCoordinator` for learning signals,
   `SystemObservability` for operator-visible system events, and
   `PocketGraphStore` with typed graph producers (`DecisionTraceGraphProducer`,
   `FlywheelGraphProducer`, `VerifyGraphProducer`, `SwiftPMGraphProducer`, `ADRGraphProducer`,
   `CodeGraphProducer`, `UnifiedGraphProducer`; the full list is in the
   [Unified Providers Flywheel Standard](unified-providers-flywheel.md#3-knowledge--execution-graph-adr-0113)) for persisted knowledge and
   execution edges (ADR-0113 (`Documentation/ADRs/ADR-0113-pocket-graph-unified-knowledge-and-execution-graph.md`), ADR-0154 (`Documentation/ADRs/ADR-0154-specialized-adr-graph-ontology-and-visual-navigator.md`)).
4. Algorithm and decision-making code should also use the appropriate unified algorithm
   provider or registry when one exists, such as `UnifiedAlgorithmProvider`,
   `UnifiedSearchProvider`, or `AlgorithmRegistry`.
5. New or modernized integration paths must emit enough context to debug, test, iterate,
   and evaluate the system later: operation name, subsystem, actor/source, input shape,
   outcome, duration or cost when relevant, and failure reason when available.
6. Do not introduce a parallel logging, telemetry, learning, or metrics store unless the
   ADR/task explains why the unified providers cannot model the signal.
7. Verification should include at least one assertion, fixture, or manual closure note for
   the relevant provider integration when the changed code affects runtime behavior.
8. These signals are part of the PocketCloud flywheel: they help the current codebase
   become easier to debug and test, and they create context that future local AI and coding
   agents can use for evaluation, recommendations, and self-improvement loops.

## Platform Policy

1. Shared runtime and UI infrastructure must compile for both iOS and macOS unless explicitly scoped otherwise.
2. Platform-specific code belongs behind compile-time availability checks and small adapters.
3. macOS-only implementation details must not leak into iOS-shared protocols or domain models. The `MacOSOnlyAPIScanner` runs as a build-blocking preflight gate (via `pocket quality platform-check` and the silver/gold build pipeline) scanning every kernel source that iOS apps link. It flags `Process`, `NSWorkspace`, `homeDirectoryForCurrentUser`, and other macOS-only APIs when they appear outside a `#if os(macOS)` / `#if canImport(AppKit)` guard. To exempt a genuinely safe usage on a specific line, add the marker `pocketcloud:allow-macos-api`.
4. tvOS, watchOS, visionOS, Linux, and Windows support can be additive, but must not compromise iOS/macOS correctness. tvOS 10-foot living room interfaces (ADR-0154/ADR-0155 graph cockpits and ADR-0160 party game host screens) must support Siri Remote directional navigation and focus management via `@FocusState` and `.focusSection()`. A control that removes itself (Hide, Delete) puts focus back explicitly. Otherwise the focus engine chooses the next item, and a neighbouring copy of the same control means the next Select repeats the action. A control that sits outside the column of the focused one is out of reach. Up from a centred button finds nothing in a right-aligned column, so wrap the list in `.focusSection()`. Pin remote paths with an `XCUIRemote` UI test, because tvOS has no tap and a focus gap is invisible in a screenshot. `PocketGamerUITests-tvOS` found the party stage's Hide controls out of reach this way (ADR-0160 T19). tvOS targets must be kept strictly decoupled from AIStack / MLX dependencies (e.g., separating `PocketCloudPartyUI` and `PocketCloudPartyLAN` from heavy server/agent modules so tvOS targets like `PocketGamer-tvOS` compile clean without linking AIStack).
5. Use platform availability annotations rather than runtime apology paths when an API is unavailable.
6. **Non-Apple platforms (Linux, Windows) build via compile-time capability, not runtime branching.** Gate Apple-only APIs behind `#if canImport(<Framework>)` (e.g. `#if canImport(Network)` in `PocketCloudAIServer/HTTPServer.swift`, which ships a Linux/Windows stub in the `#else`) rather than assuming the framework exists. Apple-only surfaces to guard: `Network`, `AppKit`, `Cocoa`, `NSWorkspace`, `SecTask`, and the MLX stack.
7. **MLX is Apple-only.** Condition MLX product/target dependencies on `.iOS/.macOS/.visionOS` in `Package.swift`, and route non-Apple platforms through `PocketCloudMLXStub`. GGUF inference uses the official `llama.cpp` XCFramework in process on iOS/macOS. The accepted Linux/Windows design is a checksum-verified official `llama-server` out of process; those platforms remain stubbed until ADR-0069's host acceptance criteria pass (ADR-0042 (`Documentation/ADRs/ADR-0042-llamacpp-cross-platform-ai-provider-integration.md`), ADR-0069 (`Documentation/ADRs/ADR-0069-windows-dynamic-llamacpp-engine-upgrade.md`)). Linux/Windows never use MLX.
8. **Filesystem code must be cross-platform and sandbox-safe.** Use `URL(fileURLWithPath:)` and `FileManager` path APIs — never hard-coded `/` separators or POSIX assumptions — and respect Windows `MAX_PATH` limits (Windows uses `\`; Developer Mode is required for SwiftPM symlink handling). Use `FileManager.default.pocketCloudHomeDirectory` / `pocketCloudDataDirectory` for `~/.pocketcloud` paths — never raw `homeDirectoryForCurrentUser`, which is macOS-only. In App Sandbox contexts (macOS and iOS sample apps), never construct raw filesystem paths to workspace or repository roots; resolve granted directories via `UserSelectedRootAccess` / security-scoped bookmarks in `PocketCloudFileKit`.
9. **Console output redaction is mandatory.** All runtime console output passes through the `SensitiveValueRegistrar` redaction layer (ADR-0112 (`Documentation/ADRs/ADR-0112-console-output-contract.md`)). Secrets, home-directory paths, and registered sensitive values are masked at every output seam. Do not bypass `Logger.console` with raw `print` or `FileHandle.standardOutput`.
10. **Honesty over aspiration.** A platform is "supported" only when its build is actually verified. Do not mark Linux/Windows green until proven; cross-compilation via the Static Linux SDK (see *Toolchain*, below) is the verification path.

## SwiftUI Adaptive Layout Policy

1. Design SwiftUI surfaces from the narrow container up. Treat 320-430 pt iPhone portrait widths, **iPhone Duo** outer/inner poses, iPad split view, and narrow macOS windows as normal operating modes, not edge cases.
2. Prefer container-width decisions over device assumptions. Use `ViewThatFits`, adaptive `LazyVGrid`, custom `Layout`, local `GeometryReader` / `onGeometryChange` measurement, and `safeAreaInset` where appropriate. `horizontalSizeClass` is useful, but it is not sufficient because macOS windows and iPad split views can be narrow without behaving like a phone.
3. Avoid permanent fixed-width sidebars, fixed inspector panes, or desktop-only `HStack` split panes in shared app views. If a list/detail surface must run narrow, provide a `NavigationStack` drill-in flow, a collapsing `NavigationSplitView`, a sheet, a popover, or an explicit inspector toggle.
4. Avoid single-line toolbar, filter, and pagination `HStack`s that depend on spacers plus fixed-width controls. Compact modes should move secondary actions into `Menu`, use icon-only buttons with accessible labels, stack controls vertically, or place primary actions in a bottom `safeAreaInset`.
5. Raw data tables may offer horizontal scrolling, but that cannot be the only phone experience for important workflows. Provide essential-column rows, card/detail presentations, a sticky context column, or a row-detail drill-in when users need to compare or act on records.
6. Metric strips, card groups, and action panels should use adaptive grids or `ViewThatFits` instead of assuming three or four cards fit in one row.
7. Long identifiers, URLs, paths, status badges, and button labels must wrap, truncate, or move to secondary rows predictably. Do not allow trailing badges/actions to crowd out primary text in narrow widths.
8. Floating assistants, inspectors, and overlays must use container-relative `maxWidth`, safe-area-aware placement, and keyboard-safe behavior. On iPhone or very narrow windows, prefer a sheet or full-width panel over a hard-coded 320-400 pt overlay.
9. New shared UI components should expose narrow-mode behavior through reusable helpers or local layout functions instead of scattering one-off `#if os(iOS)` checks.
10. **iPhone Duo (foldable) readiness** — follow Apple's [Prepare for iPhone Duo](https://developer.apple.com/iphone-duo/prepare/) checklist:
    - **Never size from `UIScreen.main`.** Use the view/window/`GeometryReader` bounds; re-read on every layout pass (a fold mid-session changes allocated size). Prefer `@Environment(\.displayScale)` over screen scale.
    - **Never drive layout from orientation or idiom.** Ban `UIDevice.current.orientation`, `statusBarOrientation`, `interfaceOrientation`, and `userInterfaceIdiom == .phone/.pad` for layout. Compare container width vs height, or use size classes. Idiom remains OK only for non-layout facts (StoreKit platform, sync peer labels).
    - **Regular width is not "iPad-only".** Duo's inner display is regular width; shells such as `PCAppShell` must choose `NavigationSplitView` vs tab bar from `horizontalSizeClass` (or measured width), so the open pose is a natural extension of the iPad layout — not a stretched phone layout.
    - **No hardcoded device sizes.** Drop `bounds.height == 844` (and peer) lists; they will not match Duo.
    - **Full-bleed media.** Prefer fit / focal-point logic over unconditional `.scaleAspectFill` when the aspect ratio widens.
    - **System bars go vertical** on some Duo poses (outer closed; some inner columns). Prefer `NavigationStack` / `NavigationSplitView` + `.toolbar { }` over hand-rolled `UIToolbar`/`UITabBar`. Give toolbar items both title and icon; use semantic placements and overflow priority.
    - **Fold / camera reserved regions.** Custom drawings that straddle the hinge should consult `GeometryProxy.reservedRegions` (or UIKit's equivalent) and prefer `ArrangementView` for primary/secondary pose layouts when available.
    - **Ship with the newest iOS SDK you can.** Apps built with the iOS 27.1 SDK (or later) get the full-display Duo chrome; older SDKs letterbox or leave camera/status gutters. Target SDK ≠ deployment target (`IPHONEOS_DEPLOYMENT_TARGET` stays the floor).
11. **Lifecycle modifiers need a view that exists.** `Group` is not a container: it hands each modifier to its children. A `.task` or `.onAppear` on a `Group` whose content is empty until the task fills it never runs, and nothing reports that. A view that loads its own content attaches those modifiers to a stack or another view that is always present. `PartyNearbyRoomsView` never browsed for rooms for this reason (2026-10-14).

## SwiftUI Accessibility Policy

Accessibility is part of the standard implementation process, not optional polish. Every new or materially changed SwiftUI surface must be usable with VoiceOver, Dynamic Type, and adequate contrast on iOS and macOS. Build it in as the view is written — retrofitting a whole app later is far more expensive.

### Labels, traits, and structure

1. Every actionable control must expose a human-readable VoiceOver label. Icon-only buttons (`Button { … } label: { Image(systemName:) }`) require an explicit `.accessibilityLabel(...)`; without one VoiceOver reads the SF Symbol name ("rectangle dot stack") or an emoji literally ("high voltage"). Give reused/testable controls a stable `.accessibilityIdentifier(...)`.
2. Hide decorative imagery with `.accessibilityHidden(true)`: SF Symbols that sit beside a text label, mood/status emoji, colour dots, and progress rings whose value is stated elsewhere. VoiceOver should convey each meaning once.
3. Convey state through traits and values, never colour alone: `.accessibilityAddTraits(.isSelected)` for selected chips/tabs, `.accessibilityValue(...)` for sliders/steppers/progress ("45 percent"), `.accessibilityAddTraits(.isHeader)` for section titles.
4. Collapse dense rows and cards into one navigable element with `.accessibilityElement(children: .combine)` — or `.ignore` plus an explicit `.accessibilityLabel(...)` — so a VoiceOver user swipes once per row, not once per subview. Note this only affects the accessibility tree; the visual contrast/Dynamic-Type audits still inspect the rendered text underneath.
5. Custom-drawn surfaces (`Canvas`, sprite/tile boards, charts, any single-gesture drawing) are invisible to VoiceOver. Provide a parallel accessible representation: mark the visual layer `.accessibilityHidden(true)` and overlay a **non-hit-testing** grid of `.accessibilityElement()` cells, each with a label and `.accessibilityAction` (VoiceOver activation fires the action independently of touch hit-testing, so sighted gesture play is preserved). Keep the label-generating describer a pure, unit-testable function on the model, not in the view.

### Contrast, Dynamic Type, and hit region

6. Text must meet WCAG AA contrast: 4.5:1 for normal text, 3:1 for large text (≥18pt, or ≥14pt bold). Do not put `.secondary`/muted grey or a data-driven tint (`.orange`/`.green`/pastel) on small text over a light surface — it fails. Use the theme's measured text slots (`theme.textPrimary`/`textSecondary`, or the text-safe `theme.success`/`warning`/`error`/`info` for a status word), or move the colour onto a non-text element (a leading dot in the matching `…Fill`). `.primary` is not the fix: it is a fixed system grey the theme cannot reach (ADR-0143 (`Documentation/ADRs/ADR-0143-one-source-of-colour-theme-enforced.md`) D3). White on the system accent (`#007AFF`/`#0A84FF`) is ~3.6:1 — acceptable only as large **and** bold text, never as a small button/segment label; a label on any fill takes that fill's measured `on…` partner.
7. `.ultraThinMaterial` and other materials apply vibrancy that renders text semi-transparent and drops it below the contrast bar; use an opaque fill when a label sits on a floating banner/capsule.
8. Support Dynamic Type: use text styles (`.caption`, `.subheadline`, `.headline`, `.body`) on readable prose, not `.font(.system(size:))`. At accessibility sizes, equal-width segmented rows must scroll horizontally or reflow to a wrapping grid rather than clip. A horizontal `ScrollView` of chips leaves the last item clipped at the screen edge — prefer a `LazyVGrid` when every option should stay readable. Text styles are also what the in-app Text Size setting (`StyleManager.textScale`, ADR-0143 D8) can move; a fixed point size ignores both it and the system setting. The rules, the exceptions and the current count are in the Theme Colour and Type Standard (`Documentation/Standards/theme-colour-and-type-standard.md#text-size-d8`).
9. Interactive elements need a ≥44×44pt hit region. Give short text buttons `.padding` + `.contentShape(Rectangle())`; do not ship 18-pt-tall tappable text or a 4-pt progress bar as an interaction target.
10. **One source of colour — the theme, enforced (ADR-0143 (`Documentation/ADRs/ADR-0143-one-source-of-colour-theme-enforced.md`)).** In a SwiftUI view, raw system colours (`.primary`, `.secondary`, `.white`, `.red`, ...) or inline `Color(hex:)` passed to colour-bearing modifiers (`foregroundColor`, `foregroundStyle`, `fill`, `tint`, `background`, `stroke`, `border`, `shadow`, …) are brand-blind: High Contrast, the appearance picker and a client's brand cannot reach them, and nothing measures them. Views read colours from `@Environment(\.pcTheme)` by **role**: copy is `theme.textPrimary`/`textSecondary`/`textTertiary`, a status word is `theme.success`/`warning`/`error`/`info`, the same status as a mark is the matching `…Fill`, a link is `theme.accentText`, and a label on a fill is that fill's `on…` partner. `pocket quality scan colours` gates it at the commit as a diff-scoped ratchet over the existing backlog. Colour carried as **data**, such as a sketch stroke's ink index or a chart series, resolves against the surface when it is drawn. Store an index, give each hue a deep and a bright shade, and let `PCColorValue.mostReadable(of:)` pick for the surface. A test then holds every shade to 3:1 on `PCPalette.allPalettes`. A fixed white ink made a guest's sketch vanish on the light party stage (ADR-0160 T05, `PartyDrawingCanvasView.inks`). The full contract — the role-to-token table across `PCTheme`, `ThemeTokens`, `PCPalette` and the CSS `--pc-*` properties, exemptions, High Contrast, Text Size and brands — is the Theme Colour and Type Standard (`Documentation/Standards/theme-colour-and-type-standard.md`); the palettes themselves are in Appearance Colour System (`Documentation/Standards/appearance-colour-system.md`).

### Regression coverage — `performAccessibilityAudit`

11. Primary app surfaces ship an XCUITest that runs Apple's `performAccessibilityAudit()`, reached via `-UITestScene` deep-links (raw touch injection is unreliable on this host). The audit class is annotated `@MainActor`: `XCUIApplication` is main-actor-isolated under the Swift 6 language mode, and a capturing issue-handler closure otherwise fails the `Sendable` check.
12. Enforce the audit by element **locatability and category**, not a blanket `.all`. Real, developer-actionable defects always resolve to a concrete element with readable text:
    - **FAIL** on any locatable `contrast` (hard "Contrast failed", <3:1), `hitRegion`, `sufficientElementDescription`, `trait`, or `elementDetection` issue. Drive these to zero.
    - **Record-but-tolerate** (with a written rationale in the test) the findings Apple's audit reports as un-actionable on a themed, emoji-rich design system: nil-element findings (no locatable view); decorative-glyph labels (no letters — emoji, dashes); the `textClipped`/`dynamicType` catch-22 (a scalable emoji is flagged "may clip", a fixed one "won't scale", and capping the range triggers both — verified empirically); and `contrast` "…unless font size is larger" (3:1–4.5:1, which passes the WCAG large-text bar and is the inherent iOS system-accent tension present in Apple's own `.borderedProminent`/tinted controls).
13. Wire the UI-test bundle as a `UITestBundle` target in the app `project.yml` (scheme `test:` block + `TEST_TARGET_NAME` + `$(POCKETCLOUD_BUNDLE_<APP>).uitests`), then regenerate with `xcodegen`. Run it through the pocket CLI — `pocket dev apple test --uitest --clean --derived-data-path <cache>` — never raw `xcodebuild`. `--clean` prepends a `clean` action so the test bundle recompiles from scratch; incremental UI-test builds otherwise silently reuse a stale test runner.

## Toolchain, Packages, and Debugging

1. Every tracked manifest declares the tools-version in *Toolchain Baseline*. A toolchain advance moves all of them, the app `project.yml` `SWIFT_VERSION`, and this standard in one change. A single manifest is never left behind.
2. Prefer manifest-level warning and concurrency settings over ad-hoc command-line flags. A target-wide warning policy goes in the manifest: `treatWarning(_:as:)` / `treatAllWarnings(as:)`. A single declaration gets `@diagnose(<Group>, as: error | warning | ignored, reason: "…")` ([SE-0522](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0522-source-warning-control.md)). An `ignored` must carry its `reason:`. Never silence a group for a whole module to quiet one site, and never use `unsafeFlags` for warning control.
3. SwiftPM dependency decisions must be reproducible and reflected in ADR/task closure evidence when they affect build behavior.
4. Use a matched Swift compiler and LLDB toolchain for debugging; mismatched LLDB can inspect Swift types unpredictably.
5. Treat Xcode, SwiftPM, and toolchain version drift as diagnosable workspace state, not a mystery build failure.
6. **Linux cross-compilation uses the Static Linux SDK, not the Xcode toolchain.** The Xcode-bundled toolchain **cannot** drive the Swift Static Linux SDK. Install an open-source **swift.org toolchain that matches the local Swift version** via `swiftly` (home-dir, `--no-modify-profile`, no sudo), then install the **Static Linux SDK of the exact same version** (`swift sdk install …_static-linux-*.artifactbundle.tar.gz --checksum …`; the SDK and toolchain versions must match). Cross-compile with that toolchain: `~/.swiftly/bin/swift build --swift-sdk x86_64-swift-linux-musl …` (web-fleet servers are x86_64; `aarch64-swift-linux-musl` is also available). The SDK is musl-static — when a C dependency cannot be satisfied statically, fall back to the glibc Docker image (`Dockerfile`, `swift:6.4-jammy`). Governance in ADR-0019 (`Documentation/ADRs/ADR-0019-swift-6-2-compliance-and-toolchain-governance.md`); build procedure in ADR-0049 (`Documentation/ADRs/ADR-0049-cross-platform-build-scripts-and-shell-abstraction.md`).
   A toolchain advance is not finished until this lane moves too. The swiftly toolchain and SDK must reach the manifests' tools-version, because an older toolchain refuses the manifest before it compiles anything. The same holds for the Windows installer and the Android SDK. Swift 6.4's Android SDK is built against NDK r30 LTS, and SwiftPM's Swift Build drives it without the old post-install script.
7. **SwiftPM builds with Swift Build** (the 6.4 default). Do not pin `--build-system native` (deprecated) to get past a failure. The two build systems lay out scratch directories and report diagnostics differently, so a workaround on one hides the real failure on the other. Report the failure instead.
   One exception has a named upstream bug and an exit condition. In Swift 6.4.0 and 6.4.1, Swift Build also compiles the dependencies of a conditionally excluded dependency ([SwiftPM #10538](https://github.com/swiftlang/swift-package-manager/issues/10538)). Cross-compiling AIStack for musl therefore builds the Apple-only `swift-huggingface` and MLX packages for Linux, and `swift-huggingface` stops with an `#error` on musl. The native build system honours the conditions: on 2026-10-15 it cross-compiled the whole `PocketCloudCLI` to a static x86-64 binary without compiling either package. That lane passes `--build-system native` until swiftly offers a 6.4.x containing the fix ([#10556](https://github.com/swiftlang/swift-package-manager/pull/10556), on `release/6.4.x`). Then drop the flag and rerun the build to confirm.
8. **Dependency evidence can be generated.** `swift package generate-sbom`, or `swift build --sbom-spec cyclonedx|spdx`, emits a Software Bill of Materials (SE-0509). Prefer it to a hand-maintained list when an audit needs the resolved third-party graph.

### Dependency, Fork, and Artifact Policy

1. Before adding or updating a package, audit product-level imports. Remove
   unused package declarations and unused products instead of carrying them for
   convenience or relying on accidental transitive imports.
2. Prefer official upstream releases. Rapidly changing AI packages and binary
   runtimes use exact versions/checksums; never track a moving branch for a
   release lane.
3. A private fork requires a written upstream gap, a minimal patch, a plan or
   issue for upstreaming/removal, and an exact temporary pin. Remove the fork as
   soon as an official stable release or artifact satisfies the contract.
4. Do not call vendoring "dependency removal." Vendoring transfers update,
   security, license, and rebase responsibility into PocketCloud. It is allowed
   only when the net source/runtime surface is demonstrably smaller or an
   upstream distribution cannot satisfy a required offline/platform contract.
5. Third-party binary and executable artifacts must be versioned, SHA-256
   verified before use, and covered by license/notices. A platform artifact
   built by PocketCloud must record its exact upstream commit, compiler,
   configuration, and generated checksum; do not attribute that checksum to
   upstream.
6. Historical vendored forks and submodules are not targets for workspace-wide
   modernization. Delete them when superseded; until then, patch only
   build-critical or upstream-worthy behavior and keep changes minimal.

### Audited AI Dependency Lane (2026-07-25)

The audited exact lane is below. The 2026-10-15 tools-6.4 advance changed none of
these pins. All of them compiled on the 6.4 toolchain that day, through both the
Xcode path (`pocket build bronze`) and SwiftPM's Swift Build (the AIStack test build).

- `mlx-swift` `0.31.6` — newest audited stable release; requires Swift tools
  6.3+ and experimental C generation (the earlier `0.31.4` compatibility pin
  existed only for the tools-6.2 era);
- official `mlx-swift-lm` `3.31.4`;
- official `mlx-audio-swift` `0.1.3`;
- `swift-huggingface` at revision `5c60886` — `0.9.0` plus the upstream tvOS fix
  for unavailable OAuth presentation APIs
  ([PR 58](https://github.com/huggingface/swift-huggingface/pull/58)), held as
  an exact revision until it ships in a release;
- `swift-transformers` `1.3.3`, limited to imported products;
- transitive `swift-jinja` `2.4.2`; and
- official `llama.cpp` `b10107`: the active XCFramework on iOS/macOS and the
  selected, not-yet-integrated `llama-server` artifacts on Linux/Windows.

`llama.cpp` has no formal stable/LTS channel, so a reviewed numbered release is
treated as a compatibility pin. Gemma 4 26B-A4B remains quarantined until a
stable MLX LM release includes upstream PR 364. Details and integrity hashes are
recorded in
`Packages/Kernel/AIStack/DEPENDENCY-AUDIT.md`.

## Verification Expectations

1. Every implementation task must include a Swift 6 concurrency/platform verification note.
2. Tests should cover actor isolation, Sendable boundaries, cancellation, and platform availability where those are relevant to the change.
3. For iOS/macOS shared code, test at least one macOS path locally and include an iOS compile/smoke strategy in task closure evidence.
4. For SwiftUI work, closure evidence must include a compact/narrow-width check strategy. Use an iPhone simulator/screenshot when practical, and include at least one narrow macOS or iPad split-view consideration when the view can appear in a resizable shell.
5. New or materially changed SwiftUI surfaces state their accessibility posture: icon-only controls are labelled, decorative imagery is hidden, state is exposed via traits/values, and readable text meets contrast. Where an app carries an accessibility-audit UI test, it must be green (locatable `contrast`/`hitRegion`/`sufficientElementDescription`/`trait`/`elementDetection` at zero) before closure. The contrast audit samples screens that are still settling, so a `contrast` finding counts when it reproduces on a re-run of the same build (ADR-0143 (`Documentation/ADRs/ADR-0143-one-source-of-colour-theme-enforced.md`) T05). Colour work also needs a visual pass in two styles of opposite character and with High Contrast on (Theme Colour and Type Standard (`Documentation/Standards/theme-colour-and-type-standard.md#verification-for-a-ui-change`)).
6. Closure evidence should state whether the code uses value semantics, actor isolation, `@MainActor`, explicit concurrent execution, or adaptive SwiftUI layout behavior.
7. Do not close an ADR/task that introduces `@unchecked Sendable`, `nonisolated(unsafe)`, a new global mutable singleton, or a shared UI surface that only works at desktop widths without documenting why safer alternatives were rejected.
8. Vector stroke geometry, sensor streams, and interactive game inputs must define deterministic boundary conditions: decimation algorithms (e.g., Ramer-Douglas-Peucker) must enforce payload bounds (< 8 KB per sketch), motion/pitch classifiers must employ hysteresis neutral zones and cooldown windows to prevent re-entrant event bounce, and custom SwiftUI `Canvas` renderers draw anything a person must read — a label, an axis, an ink — in theme or data-colour tokens resolved against `@Environment(\.pcTheme)`. Literals are acceptable only for a drawing's own content (a card face, a sprite). The raw-colour gate exempts drawing files wholesale, so this rule is held by review and by per-palette tests, not by the scanner (Theme Colour and Type Standard (`Documentation/Standards/theme-colour-and-type-standard.md#exemptions-d3`)).

## ADR and Task Integration Rule

Every active ADR and task document must include a Swift 6 alignment reference to this standard. The reference can be a compact breadcrumb; do not copy this whole standard into each file.
