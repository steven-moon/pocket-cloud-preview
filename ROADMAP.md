# PocketCloud Roadmap

This roadmap reflects what is actively in progress and where the project is heading.

**Legend:**
- ✅ **Working today** — verified in the current baseline
- 🗓 **Near-term (Q1-Q2 2026)** — actively tracked in an open ADR
- 🔭 **Long-term vision** — architectural direction, no committed timeline

For the current verified baseline, see [FEATURES.md](FEATURES.md).

---

## Recently Completed (Q1 2026)

### ✅ RAG-Augmented Local Inference (ADR-0013 — Complete)

Index your codebase, documents, and notes into a local vector store. Every AI request
automatically receives relevant context — without you having to paste code into the prompt.

- `pocket knowledge rag index` indexes your workspace
- `pocket knowledge rag query` retrieves relevant context
- AI Router automatically injects RAG context proportional to the latency budget
- Budget-proportional RAG timeout (25% of latency budget, max 30s)
- RAG metadata (`rag_status`, `rag_injected`) on every ChatResponse

---

### ✅ Persistent MLX Daemon (ADR-0013 — Complete)

The persistent daemon keeps models warm between requests for sub-100ms time-to-first-token.

- `pocket system local serve --warmup` starts daemon with model pre-warming
- `pocket system local stop` gracefully shuts down
- PID file management at `~/.pocketcloud/daemon.pid`
- Auto-start in verify workflow when daemon not running
- RAG index seeding on warmup

---

### ✅ Apple Intelligence Integration (ADR-0006 — Complete)

PocketCloud integrates with Apple's FoundationModels API for system-level AI capabilities.

- `AppleIntelligenceProvider` using `SystemLanguageModel.default` and `LanguageModelSession`
- Availability probing: ARM64 → OS version → SystemLanguageModel.availability gating
- Capability matrix with tier-based feature detection
- MCP tool `foundation_models` for live inference probes
- Small-model detection with embedded-example prompts for sub-2B parameter models

---

### ✅ Hardware-Aware Model Selection (ADR-0012 — Complete)

Intelligent model scoring that accounts for your actual hardware:

- SoC-aware bandwidth lookup (M1→55 GB/s through M4 Ultra→740 GB/s)
- Memory-fit gating with safe budget (45% of physical RAM)
- MoE active-parameter estimation from config.json
- 3-prompt oracle verification suite (JSON output, word count, exact constraint)
- Oracle feedback loop: every local inference emits quality signals
- Model reputation service: degenerate models auto-blacklisted after 3+ low-quality signals
- Benchmark auto-enqueuer: fresh benchmarks for models lacking signals

---

### ✅ App Convergence (ADR-0017 — Complete)

All 4 apps wired to the Feb–Mar 2026 infrastructure sprint:

- **PocketMind**: daemon warmup + polling, RAG chat toggle, LocalAIStatusBar
- **BrainDeck**: RAG note search (actor), debounced RAG search view, study learning signals
- **PocketCloudHub**: Live verify dashboard (--json output), AI failure diagnosis, planning panel
- **EmotionalIntelligence**: LocalAIMoodAnalyzer enforcing localOnly routing, privacy badge
- **Shared**: AppAIContext + DaemonStatusBadge + RAGStatusBadge + VerifyResultRow + PrivacyLocalBadge

---

## Near-term (Q2 2026)

### 🗓 Linux Support

Extend the Kernel layer beyond Apple platforms. The AI Router, provider integrations, and
CLI already use portable Swift — Linux support means targeting SwiftPM on Ubuntu/Fedora
and providing inference backends beyond MLX (llama.cpp, Ollama).

### 🗓 Plugin Architecture

Allow third-party developers to extend PocketCloud with custom MCP tools, inference
providers, and CLI commands without modifying the core codebase.

### 🗓 Open-Source Launch

Prepare the codebase for public contribution: licensing, contribution guidelines,
CI/CD for community PRs, documentation, and example projects.

---

## Long-term Vision

*The following items are speculative. They represent the architectural direction of the project
but have no committed timeline or guaranteed scope.*

---

### 🔭 AI-Native Project Planning

An AI assistant that understands your project's ADRs, task backlogs, and verify history —
and can reason about what to work on next, estimate complexity, and surface blocking issues
automatically.

The foundation (ADR task tracking, verify baselines, RAG indexing, planning document service)
is built. The AI-native planning layer sits on top of it.

---

### 🔭 Federated On-Device Learning

Coordinate learning across your own Apple devices without a central server. Your iPhone's
usage patterns, your Mac's inference history, and your iPad's document interactions could
improve your personal model — privately, on your own devices.

This is a long-term research direction. The edge learning primitives (`PocketCloudEdge` module)
are in early development.

---

### 🔭 Expanded MCP Ecosystem

The current MCP server covers development workflows (build, test, analyze, deploy). Future
expansion targets:
- Personal knowledge management (notes, bookmarks, research)
- Communication workflows (email drafting, meeting notes)
- Creative tools (writing assistance, brainstorming)

---

### 🔭 Multi-Device AI Coordination

Use your Mac as a local inference server for your iPhone and iPad — without any cloud
intermediary. The AI request stays on your local network, the model runs on your most
powerful device, and the result streams back to your mobile device.

---

## What Won't Change

Some things are not on the roadmap because they are fundamental constraints, not future
possibilities:

- **Local-first is not optional.** Cloud providers will always be opt-in, never required.
- **Your data stays on your device.** No telemetry, no analytics, no usage reporting.
- **Swift-only.** PocketCloud is built for Apple platforms first and will remain Swift-native.
- **Verified, not just shipped.** Every feature ships with a verify operation.
- **Open-source.** The source code is yours to inspect, modify, and run.
