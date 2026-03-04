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

PocketCloud uses Apple's [MLX framework](https://github.com/ml-explore/mlx) — an array
framework designed specifically for Apple Silicon's unified memory architecture.

```
Your prompt
    │
    ▼
PocketCloudMLX
    │
    ├── Model selection (ModelRegistry)
    │   Finds a compatible quantized model in ~/.pocketcloud/models/
    │
    ├── Model loading (MLXEngine)
    │   Loads weights into Apple Silicon unified memory
    │   Typical cold start: under 2 seconds on M-series
    │
    └── Token generation
        Runs entirely on-chip (CPU + GPU + Neural Engine)
        No network connection required
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
    └── Cloud provider configured?
        Yes → Use configured cloud provider (OpenAI, Claude, etc.)
```

Cloud providers are never used unless explicitly configured. The router never "upgrades"
to a cloud provider without your knowledge.

---

---

## RAG-Augmented Context

RAG (Retrieval-Augmented Generation) indexes your codebase, documents, and notes
into a local vector store. The AI router automatically injects relevant context
from your indexed knowledge into every request — making local inference codebase-aware
without you having to manually paste code into prompts.

The indexing pipeline is already working:
```bash
pocket knowledge rag index     # Index workspace
pocket knowledge rag query "..." # Query the index
```

Context injection into the AI router is in active development.
