# PocketCloud Architecture

PocketCloud is built on three principles:
1. **Local-first** — AI runs on your device, not in a data center
2. **Privacy by design** — your data never leaves unless you explicitly choose a cloud provider
3. **Verified, not assumed** — every layer is continuously verified against a 112-operation test suite

---

## Three-Layer Stack

```
┌─────────────────────────────────────────────────────────────┐
│                        Apps Layer                            │
│                                                               │
│   PocketMind           BrainDeck        EmotionalIntelligence │
│   iOS 17+              macOS 14+        tvOS 17+              │
│   AI chat assistant    Study notes      Mood + insights       │
│                                                               │
│   PocketCloudHub                                              │
│   macOS 14+                                                   │
│   AI agent serving + developer tools                          │
└───────────────────────────┬─────────────────────────────────┘
                            │  Swift Package imports
┌───────────────────────────▼─────────────────────────────────┐
│                       Kernel Layer                           │
│                                                               │
│  ┌─────────────────────┐  ┌──────────────────────────────┐  │
│  │      AIStack        │  │            Core              │  │
│  │                     │  │                              │  │
│  │  ChatEngine         │  │  PocketCloudMCP              │  │
│  │  (sessions, context)│  │  (40+ MCP tools)             │  │
│  │                     │  │                              │  │
│  │  AIAgent            │  │  PocketCloudLogger           │  │
│  │  (7+ providers)     │  │  PocketCloudFileKit          │  │
│  │                     │  │  PocketCloudPrivacy          │  │
│  │  PocketCloudMLX     │  │  PocketCloudChatCore         │  │
│  │  (local inference)  │  │  PocketCloudDocumentSystem   │  │
│  │                     │  │                              │  │
│  │  pocket CLI         │  │  PocketCloudCommon           │  │
│  │  (5 domains)        │  │                              │  │
│  └─────────────────────┘  └──────────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                     Inference Layer                          │
│                                                               │
│  On-device (local-first):                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Apple MLX Framework (binary XCFramework)            │    │
│  │  33+ quantized models · Apple Silicon only           │    │
│  │  No network · No API key · Unified memory            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
│  Cloud providers (optional, explicit opt-in):                │
│  OpenAI · Claude · Gemini · XAI · OpenRouter                 │
│  LM Studio (local server) · Ollama (local server)            │
└─────────────────────────────────────────────────────────────┘
```

---

## Platform Matrix

| Component | iOS 17+ | macOS 14+ | tvOS 17+ | visionOS 1+ | watchOS 10+ |
|---|---|---|---|---|---|
| Core packages | ✅ | ✅ | ✅ | ✅ | ✅ |
| Local MLX inference | ✅ | ✅ | — | — | — |
| MCP server | — | ✅ | — | — | — |
| `pocket` CLI | — | ✅ | — | — | — |
| Chat UI | ✅ | ✅ | ✅ | ✅ | — |
| Cloud providers | ✅ | ✅ | ✅ | ✅ | — |

All packages are compiled with Swift 6.2 strict concurrency enabled.

---

## Privacy Model

### What stays on your device

- All chat history and session data (`~/.pocketcloud/` or app sandbox)
- All downloaded MLX models (`~/.pocketcloud/models/`)
- All RAG vector indexes (`~/.pocketcloud/knowledge/`)
- All task schedules and history (`~/.pocketcloud/tasks/`)
- All API keys (macOS Keychain or `.env` file, never transmitted)

### What leaves your device (only with explicit configuration)

- Prompt content sent to a cloud provider you have configured and authenticated
- Nothing else

### SecureEnclave and Keychain

API keys for cloud providers are stored in macOS Keychain. PocketCloud uses the
`PocketCloudPrivacy` module with SecureEnclave-backed storage where the platform supports it.
Keys are loaded at runtime and never written to log files or transmitted in metadata.

### Zero telemetry

PocketCloud contains no analytics SDK, no crash reporting service, and no usage telemetry.
There is no "phone home" on startup, no anonymous usage statistics, and no A/B testing
infrastructure. The codebase is verifiable: `pocket system verify --exhaustive` runs 112
operations and none of them involve external reporting endpoints.

---

## The MCP Server

The Model Context Protocol (MCP) server exposes PocketCloud's capabilities to AI assistants
like Claude. When configured as an MCP server in your IDE or Claude Desktop, it gives the
AI direct access to your local filesystem, build tools, and knowledge base — without any
cloud intermediary.

```
Claude Desktop / IDE
        │
        │  MCP (local socket / stdio)
        │
┌───────▼───────────────────────────────────┐
│          PocketCloud MCP Server            │
│                                            │
│  file_operations   · git                  │
│  code_analysis     · apple_build          │
│  model_registry    · orchestrated_inference│
│  rag_query         · log_analysis         │
│  documentation     · ...                  │
└────────────────────────────────────────────┘
        │
        │  Local calls only
        │
   Your filesystem, models, and build tools
```

---

## Build Architecture

PocketCloud uses a monorepo with centralized build artifacts:

```
workspace root/
├── Package.swift              # Workspace manifest
├── Packages/
│   ├── Kernel/                # Core systems (AIStack, Core, AML, DevTools)
│   └── Toolkit/               # Higher-level utilities (UI, Build, Infrastructure)
├── Apps/                      # Platform apps
├── Frameworks/                # Binary MLX XCFrameworks
└── .build/                    # Centralized Swift build artifacts
```

All builds run from the workspace root to centralize artifacts. The `pocket ops build` command
wraps `swift build` with additional steps for MLX framework setup and CLI installation.

---

## Verification Architecture

Every layer is continuously verified via `pocket system verify --exhaustive`:

```
pocket system verify --exhaustive --local-first

Endpoints verified (24):
  filesystem · logs · workspace · dev · ai · providers
  docs · analyze · mcp-server · git · rag · apple
  apple-simulator · apple-project · apple-archive
  apple-ecosystem-deploy · apple-cleanup
  ai-agents · project-management · project-onboarding

Operations verified (112):
  97.3% pass rate as of 2026-03-01
```

The verify command runs in under 12 minutes and is a required gate before every release.
