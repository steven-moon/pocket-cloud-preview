# PocketCloud Features

All features listed here are verified working as of 2026-03-01.
Verification baseline: `pocket system verify --exhaustive --local-first` — 109/112 (97.3%)

---

## Local MLX Inference

**What it does:** Runs large language models entirely on-device using Apple's MLX framework.
No internet connection required. No data leaves your machine.

**Verify:**
```bash
pocket system local list
# Lists all downloaded MLX models

pocket system local verify
# Runs a local inference round-trip and verifies the response contains an expected token
```

**Evidence:**
- 33+ models supported including Mistral, Llama, Qwen, and Phi variants
- Cold start typically under 2 seconds on Apple Silicon M-series
- Quantized models (4-bit) fit comfortably in unified memory alongside the OS
- Supports chat templates for instruction-tuned models

---

## Multi-Provider AI Integration

**What it does:** Routes AI requests to any of 7+ providers with a unified interface.
Local MLX inference is the priority; cloud providers are optional complements.

**Verify:**
```bash
pocket dev providers list
# Shows all configured providers and their status
```

**Supported providers:**
| Provider | Type | Notes |
|---|---|---|
| MLX (local) | On-device | Apple Silicon required |
| OpenAI | Cloud | Requires API key |
| Claude (Anthropic) | Cloud | Requires API key |
| Gemini (Google) | Cloud | Requires API key |
| XAI (Grok) | Cloud | Requires API key |
| OpenRouter | Cloud aggregator | Single key, many models |
| LM Studio | Local server | localhost:1234 |
| Ollama | Local server | localhost:11434 |

---

## `pocket` CLI

**What it does:** Full-featured CLI with five command domains, 45+ commands, universal
scheduling support, and task management.

**Verify:**
```bash
pocket system health
# Reports system health across all subsystems

pocket system verify --exhaustive --local-first
# Runs 112 verification operations across 24 endpoints
```

**Five command domains:**
```
pocket system      — verify, health, local AI, workspace, hub management
pocket dev         — providers, Apple build tools, configuration, REPL
pocket quality     — analyze, review, test, docs audit/validate/generate
pocket ops         — build, CI/CD, monitor, bootstrap, planner
pocket knowledge   — RAG index/query, document search, model registry
```

**Scheduling:** Every command supports immediate, one-time scheduled, and recurring execution:
```bash
pocket quality test contract           # Immediate
pocket quality test --at "9am"         # One-time scheduled
pocket quality test --daily            # Recurring
pocket task list                       # View all scheduled/recurring tasks
```

---

## MCP Server (40+ Tools)

**What it does:** Model Context Protocol server exposing PocketCloud capabilities as tools
for Claude and other MCP-compatible clients.

**Verify:**
```bash
pocket system verify --exhaustive --local-first
# Verifies all MCP endpoints and tools
# Baseline: 109/112 (97.3%) pass rate
```

**Tool categories:**
| Category | Tools | Status |
|---|---|---|
| File system | file_operations, directory_operations, search | ✅ |
| Git | git (status, diff, log, branch, commit) | ✅ |
| Code analysis | code_analysis, coverage_analysis | ✅ |
| Documentation | documentation, document_workspace | ✅ |
| MLX models | model_registry, model_compatibility | ✅ |
| Apple build | apple_build, apple_simulator, apple_project | ✅ |
| AI inference | orchestrated_inference, lmstudio, cloud_provider | ✅ |
| Logging | log_analysis, error_analysis | ✅ |
| RAG | rag_query, knowledge_context | ✅ |

---

## Cross-Platform Apps

**What it does:** Native Swift apps for four Apple platforms sharing a common Kernel package.

**Verify:**
```bash
pocket system verify --operation apple.test
# Runs Xcode unit tests for PocketMind (iOS)

pocket system verify --operation apple.archive
# Produces a signed IPA archive for PocketMind
```

**Apps:**
| App | Platform | Description |
|---|---|---|
| PocketMind | iOS 17+ | AI chat assistant |
| PocketCloudHub | macOS 14+ | AI agent serving + developer tools |
| BrainDeck | macOS 14+ | Study notes + learning workflows |
| EmotionalIntelligence | tvOS 17+ | Mood tracking + emotional insights |

---

## Privacy-First Architecture

**What it does:** Ensures your data stays on your device by design.

**Verify:**
```bash
pocket system verify --operation workspace.integrity
# Verifies no unexpected external calls in workspace configuration
```

**Privacy guarantees:**
- **On-device inference:** MLX runs entirely locally. Tokens never leave your device.
- **SecureEnclave secrets:** API keys stored in macOS Keychain, never in plaintext files.
- **Zero telemetry:** No analytics, no usage reporting, no crash reporting to external servers.
- **Local-first storage:** All chat history, RAG indexes, and model files stay in `~/.pocketcloud/`.
- **Explicit provider choice:** Cloud providers are opt-in and require explicit configuration.

---

## Self-Healing Build Infrastructure

**What it does:** Automatically detects and recovers from common build failures without
manual intervention.

**Verify:**
```bash
pocket system verify --operation apple.swift_build
# Builds PocketCloudCLI via SwiftPM; auto-recovers stale vendor state on retry
```

**Self-healing capabilities:**
- **Lock contention:** Detects competing `swift build` processes and cleans lock files before retrying
- **Stale vendor mirrors:** Automatically clears stale `workspace-state.json` and `checkouts/`
  when vendor mirrors are out of sync; retries with clean resolution
- **Derived data detection:** Identifies and reuses valid Xcode derived data to speed up builds
- **Offline mode:** Falls back to cached SwiftPM checkouts when network is unavailable
