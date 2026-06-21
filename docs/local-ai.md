# Local-First AI

PocketCloud's central design principle is that AI should run on your device, not in a
data center. This is not a fallback mode — it is the default.

---

## Why Local-First?

**Privacy:** When your AI runs locally, your prompts, your code, and your documents never
leave your machine. There is no API server receiving your context. There is no company
logging your queries.

**Latency:** A local model responding from RAM is faster than a round-trip to a remote API.
Once a model is loaded into memory, generation is bounded only by your hardware.

**Reliability:** Local inference works without an internet connection. No API outages, no
rate limits, no service disruptions.

**Cost:** Local inference has no per-token cost. Once you have the model, inference is free.

---

## How It Works

PocketCloud starts with Apple's [MLX framework](https://github.com/ml-explore/mlx), an array
framework designed specifically for Apple Silicon's unified memory architecture. The current
workspace also includes fully integrated llama.cpp support for cross-platform Linux, Unix, and Windows capabilities.

```
Your prompt
    │
    ▼
AIRouter
    │
    ├── Local MLX path
    │   Selects compatible quantized models in ~/.pocketcloud/models/
    │
    ├── Local server path
    │   LM Studio or Ollama when available
    │
    ├── Cross-platform path
    │   llama.cpp provider for GGUF-compatible models
    │
    └── Explicit cloud path
        Uses a configured provider only when you choose one
```

---

## Supported Models

PocketCloud supports 33+ quantized models in the MLX format, including:

| Family | Examples |
|---|---|
| Mistral | Mistral-7B-Instruct, Mixtral-8x7B |
| Llama | Llama-3.2-1B, Llama-3.1-8B |
| Qwen | Qwen2.5-7B, Qwen2.5-14B |
| Phi | Phi-3.5-mini |
| Gemma | Gemma-2-2B, Gemma-2-9B |

All models run in 4-bit or 8-bit quantization to fit in Apple Silicon unified memory.

Models are discovered from `~/.pocketcloud/models/` and managed via:
```bash
pocket system local list      # List available models
pocket knowledge model list   # Full model registry
```

---

## The AI Router

PocketCloud's `AIRouter` handles provider selection with a local-first priority:

```
Request arrives
    │
    ├── Local MLX available and configured?
    │   Yes → Use local inference
    │
    ├── LM Studio running on localhost:1234?
    │   Yes → Use LM Studio
    │
    ├── Ollama running on localhost:11434?
    │   Yes → Use Ollama
    │
    ├── llama.cpp model available?
    │   Yes → Use llama.cpp
    │
    └── Cloud provider configured?
        Yes → Use configured cloud provider (OpenAI, Claude, etc.)
```

Cloud providers are never used unless explicitly configured. The router never "upgrades"
to a cloud provider without your knowledge.

---

## Persistent Daemon

PocketCloud includes a persistent local AI daemon path so models can stay warm between
requests. Recent verification work also auto-heals daemon startup when exhaustive checks
need local inference.

---

## RAG-Augmented Context

RAG (Retrieval-Augmented Generation) indexes your codebase, documents, and notes
into a local vector store. The AI router can inject relevant context from your indexed
knowledge into requests, making local inference codebase-aware without forcing you to
manually paste source files into prompts.

The indexing pipeline is already working:
```bash
pocket knowledge rag index     # Index workspace
pocket knowledge rag query "..." # Query the index
```

The June 2026 source history includes RAG bootstrap and self-healing work in verification.
