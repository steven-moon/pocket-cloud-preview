# PocketCloud Roadmap

This roadmap reflects what is actively in progress and where the project is heading.

**Legend:**
- ✅ **Working today** — verified in the current baseline
- 🗓 **Near-term (Q1-Q2 2026)** — actively tracked in an open ADR
- 🔭 **Long-term vision** — architectural direction, no committed timeline

For the current verified baseline, see [FEATURES.md](FEATURES.md).

---

## Near-term (Q1-Q2 2026)

### 🗓 RAG-Augmented Local Inference (ADR-0013)

Index your codebase, documents, and notes into a local vector store. Every AI request
automatically receives relevant context — without you having to paste code into the prompt.

**What this means in practice:**
- Ask `pocket` about your codebase and get answers grounded in your actual files
- Local inference becomes context-aware, not just prompt-aware
- The RAG index lives entirely on your device

**Progress:** Core indexing pipeline is working (`pocket knowledge rag index`). Context
injection into the AI routing layer is in active development.

---

### 🗓 Persistent MLX Daemon (ADR-0013)

The first request to a local MLX model requires loading the model into memory (~1-2s on Apple
Silicon). A persistent daemon keeps the model warm between requests, targeting sub-100ms
time-to-first-token for all subsequent requests.

**What this means in practice:**
- Responsive local AI without cold-start latency
- The daemon manages memory automatically, freeing resources when idle
- `pocket system local serve` will expose the daemon lifecycle

**Progress:** Model loading and inference are working. The persistent daemon is in design.

---

### 🗓 Apple Intelligence Integration (ADR-0006)

Integrate PocketCloud with Apple Intelligence — the system-level AI layer introduced in
iOS 18 / macOS 15. This means PocketCloud can participate in Writing Tools, App Intents,
and Siri Shortcuts natively.

**What this means in practice:**
- Use PocketCloud AI from any Apple app via the system Writing Tools menu
- Expose `pocket` commands as Siri Shortcuts
- App Intents for the most common AI workflows

**Progress:** Architecture defined. Implementation targeting Q2 2026.

---

## Long-term Vision

*The following items are speculative. They represent the architectural direction of the project
but have no committed timeline or guaranteed scope.*

---

### 🔭 AI-Native Project Planning

An AI assistant that understands your project's ADRs, task backlogs, and verify history —
and can reason about what to work on next, estimate complexity, and surface blocking issues
automatically.

The foundation (ADR task tracking, verify baselines, RAG indexing) is being built now. The
AI-native planning layer sits on top of it.

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
- **Swift-only.** PocketCloud is built for Apple platforms and will remain so.
- **Verified, not just shipped.** Every feature ships with a verify operation.
