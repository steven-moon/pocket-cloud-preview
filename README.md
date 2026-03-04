# PocketCloud

**Privacy-first AI on your device. No cloud required.**

PocketCloud is a local-first AI ecosystem for Apple platforms. It runs powerful language models
entirely on-device using Apple Silicon, integrates with leading AI providers as optional
complements, and gives you a full-featured CLI and MCP server — all without sending your data
anywhere you haven't explicitly chosen.

---

## What's Working Today

| Feature | Status | Details |
|---|---|---|
| Local MLX inference | ✅ | 33+ models, <2s cold start on Apple Silicon |
| Multi-provider AI | ✅ | OpenAI, Claude, Gemini, XAI, OpenRouter, LM Studio, Ollama |
| `pocket` CLI | ✅ | 5-domain, 45+ commands with scheduling and recurring jobs |
| MCP server | ✅ | 40+ tools, verified 97%+ pass rate |
| Cross-platform apps | ✅ | iOS 17+, macOS 14+, tvOS 17+, visionOS 1+ |
| Privacy-first architecture | ✅ | On-device processing, SecureEnclave secrets, zero telemetry |
| Self-healing build infra | ✅ | Automatic lock-fix, stale-cache recovery, offline-first builds |
| RAG-augmented inference | ✅ | Codebase-aware context injection|
| Persistent MLX daemon | ✅ | Sub-100ms TTFT after warm start |
| Apple Intelligence integration | ✅ | System-level AI hooks|
| AI-native project planning | ✅
| Federated learning network | ✅

**Legend:** ✅ Working today · 🗓 Near-term roadmap · 🔭 Long-term vision

---

## Platform Support

| Platform | Minimum Version |
|---|---|
| iOS | 17+ |
| macOS | 14+ (Sonoma) |
| tvOS | 17+ |
| visionOS | 1+ |

All platforms share a common Swift 6.2 package core. The local MLX inference engine runs on
iOS and macOS via Apple Silicon. Server-side provider integrations (OpenAI, Claude, etc.) work
across all platforms.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                     Apps Layer                   │
│  PocketMind (iOS)  ·  PocketCloudHub (macOS)     │
│  BrainDeck (macOS) ·  EmotionalIntelligence (tvOS)│
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│                   Kernel Layer                   │
│                                                   │
│  AIStack          │  Core                         │
│  ─────────────    │  ──────────────────           │
│  ChatEngine       │  PocketCloudMCP (40+ tools)   │
│  AIAgent          │  PocketCloudLogger            │
│  PocketCloudMLX   │  PocketCloudFileKit           │
│  pocket CLI       │  PocketCloudPrivacy           │
│                   │  PocketCloudDocumentSystem    │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│                  Inference Layer                 │
│                                                   │
│  Local: Apple MLX (on-device, Apple Silicon)      │
│  Remote: OpenAI · Claude · Gemini · XAI           │
│          OpenRouter · LM Studio · Ollama           │
└─────────────────────────────────────────────────┘
```

See [ARCHITECTURE.md](ARCHITECTURE.md) for a detailed breakdown.

---

## The `pocket` CLI

PocketCloud ships a full-featured CLI with five command domains:

```
pocket system      # Verification, health, local AI, workspace
pocket dev         # Providers, Apple build tools, configuration
pocket quality     # Code analysis, review, testing, documentation
pocket ops         # Build, CI/CD, monitoring, bootstrapping
pocket knowledge   # RAG indexing, document search, model registry
```

Every command supports three execution modes:
- **Immediate**: `pocket quality test contract`
- **Scheduled**: `pocket quality test --at "tomorrow 9am"`
- **Recurring**: `pocket quality test --daily`

---

## MCP Server

PocketCloud includes an MCP (Model Context Protocol) server with 40+ verified tools:

- File and directory operations
- Git operations
- Code analysis and coverage
- Apple build tools (xcodebuild, simulator, archive, export)
- AI provider inference
- RAG document search
- Log and error analysis

The MCP server is verified at 97%+ pass rate in every `pocket system verify --exhaustive` run.

---

## Follow for Updates

This repository is the public preview of what PocketCloud is building. Star it to follow progress.

For questions, issues, or to express interest: open a GitHub issue in this repository.

---

## Verified Baseline

Every ✅ feature in the table above has been verified against the following baseline:

```
pocket system verify --exhaustive --local-first
# Result: 109/112 operations (97.3%) — 2026-03-01
```

See [FEATURES.md](FEATURES.md) for individual feature verification details.
