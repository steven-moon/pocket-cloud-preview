# PocketCloud

**Your AI. Your Device. Your Data.**

PocketCloud is an open-source, local-first AI stack for Apple platforms — built to prove that the future of AI doesn't have to live in a datacenter.

It runs powerful language models entirely on your hardware using Apple MLX, integrates with cloud providers when you choose, and gives you a complete development toolkit — CLI, MCP server, cross-platform apps, and a privacy-first architecture — all without sending a single byte of your data anywhere you haven't explicitly chosen.

---

## Why PocketCloud Exists

The AI revolution has a concentration problem.

Worldwide AI spending hit **$1.5 trillion in 2025** and is projected to reach **$2.5 trillion in 2026**. Hyperscalers are spending **$600+ billion per year** building datacenters. Meta alone plans **$115-135 billion in 2026 capex** — including a 4.5 million sq ft facility in Eagle Mountain, Utah. US datacenter power consumption is projected to **nearly triple by 2030**, reaching 134 GW. Datacenters consume **449 million gallons of water per day** in the US alone, and communities are pushing back — **$98 billion in datacenter projects** have been blocked or delayed by 188+ opposition groups.

Meanwhile, there are **7.5 billion smartphones** and **2+ billion PCs** in the world. Modern Apple Silicon chips deliver 38 TOPS on the Neural Engine and up to 800 GB/s memory bandwidth. The average connected device sits idle 60% of the time. We are sitting on an ocean of compute.

**PocketCloud exists because you shouldn't need a $200/month API subscription to build with AI.** A $599 MacBook Neo can run 7B-parameter models at 28-35 tokens/second. A Mac Mini M4 Pro can run 32B models. The hardware is here. The models are open. What's been missing is the software stack to tie it all together.

This is that stack.

> Read the full story: **[STORY.md](STORY.md)** — How one developer and a team of AI agents built an AI operating system in 101 days.

---

## What's Working Today

| Feature | Status | Details |
|---|---|---|
| Local MLX inference | ✅ | 33+ models, <2s cold start on Apple Silicon |
| Multi-provider AI | ✅ | OpenAI, Claude, Gemini, XAI, OpenRouter, LM Studio, Ollama, LLama.cpp |
| `pocket` CLI | ✅ | 5-domain, 50+ commands with scheduling and recurring jobs |
| MCP server | ✅ | 40+ tools, verified 97%+ pass rate |
| Cross-platform apps | ✅ | iOS 17+, macOS 14+, tvOS 17+, visionOS 1+ |
| Privacy-first architecture | ✅ | On-device processing, SecureEnclave secrets, zero telemetry |
| Self-healing build infra | ✅ | Automatic lock-fix, stale-cache recovery, offline-first builds |
| RAG-augmented inference | ✅ | Codebase-aware context injection into local AI requests |
| Persistent MLX daemon | ✅ | Warm model serving with sub-100ms TTFT |
| Apple Intelligence integration | ✅ | FoundationModels API, on-device system-level AI |
| Hardware-aware model selection | ✅ | SoC bandwidth scoring, memory-fit gating, oracle feedback loops |
| AI-native project planning | 🗓 | Near-term roadmap |
| Federated learning network | 🔭 | Long-term vision |

**Legend:** ✅ Working today · 🗓 Near-term roadmap · 🔭 Long-term vision

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
│  AIRouter         │  PocketCloudPrivacy           │
│  ModelSelector    │  PocketCloudDocumentSystem    │
│  pocket CLI       │  PocketCloudBlockchain        │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│                  Inference Layer                 │
│                                                   │
│  Local: Apple MLX · Apple Intelligence            │
│         LM Studio · Ollama · LLama.cpp           │
│  Remote: OpenAI · Claude · Gemini · XAI           │
│          OpenRouter                               │
└─────────────────────────────────────────────────┘
```

See [ARCHITECTURE.md](ARCHITECTURE.md) for a detailed breakdown.

---

## The `pocket` CLI

```
pocket system      # Verification, health, local AI, workspace
pocket dev         # Providers, Apple build tools, configuration
pocket quality     # Code analysis, review, testing, documentation
pocket ops         # Build, CI/CD, monitoring, bootstrapping
pocket knowledge   # RAG indexing, document search, model registry
pocket task        # Task management, scheduling, lifecycle
```

Every command supports three execution modes:
- **Immediate**: `pocket quality test contract`
- **Scheduled**: `pocket quality test --at "tomorrow 9am"`
- **Recurring**: `pocket quality test --daily`

---

## Development Velocity

PocketCloud was built in **101 days** by a solo developer working with AI agents:

| Metric | Value |
|---|---|
| Total commits | 2,140 |
| Days with commits | 100 of 101 (99%) |
| Average commits/day | 21.2 |
| Peak day | 127 commits (Jan 12, 2026) |
| Swift files | 7,556 |
| Lines of Swift | 474,294 |
| Test files | 2,180 |
| AI co-authored commits | 246 (Claude Sonnet 4.5 + 4.6) |
| Architecture decisions | 18 ADRs |
| Active streak | 66 consecutive days (ongoing) |

> See **[docs/development-velocity.md](docs/development-velocity.md)** for the full breakdown.

---

## Platform Support

| Platform | Minimum Version |
|---|---|
| iOS | 17+ |
| macOS | 14+ (Sonoma) |
| tvOS | 17+ |
| visionOS | 1+ |
| watchOS | 10+ |

All platforms share a common Swift 6.2 package core with strict concurrency. Local MLX inference runs on iOS and macOS via Apple Silicon.

---

## Follow for Updates

This repository is the public preview of PocketCloud. Star it to follow progress.

For questions, issues, or to express interest: open a GitHub issue.

**Author**: [Steven Moon](https://www.linkedin.com/in/stevenmoon/) · [Clever Coding](https://clevercoding.com/about) · [@stevenmoon](https://x.com/stevenmoon)

---

## Verified Baseline

Every ✅ feature above has been verified:

```
pocket system verify --exhaustive --local-first
# Result: 109/112 operations (97.3%) — 2026-03-01
```

See [FEATURES.md](FEATURES.md) for individual verification details.
