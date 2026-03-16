# Getting Started with PocketCloud

> **Preview Repository** — PocketCloud is preparing for open-source release. This repository
> contains documentation and the public roadmap. The full source code will be published soon.

---

## System Requirements

| Requirement | Minimum |
|---|---|
| macOS | 14+ (Sonoma) |
| iOS | 17+ |
| tvOS | 17+ |
| visionOS | 1+ |
| Hardware | Apple Silicon (M1 or later) for local MLX inference |
| Xcode | 16+ |
| Swift | 6.2 |

**For local AI inference:** Any Apple Silicon Mac. A Mac Mini M4 ($599) or MacBook Neo ($599) is sufficient to run 7B-parameter models at 28-35 tokens/second.

**For cloud providers (optional):** API keys from any supported provider (OpenAI, Anthropic, Google, XAI, OpenRouter).

---

## What You Can Do Today

Once PocketCloud is installed, the `pocket` CLI gives you:

```bash
# Check system health
pocket system health

# List available local AI models
pocket system local list

# Run local AI verification
pocket system local verify

# Index your codebase for RAG
pocket knowledge rag index

# Query your codebase
pocket knowledge rag query "how does authentication work"

# Full system verification (112 operations)
pocket system verify --exhaustive --local-first

# Start the persistent MLX daemon
pocket system local serve --warmup

# Schedule recurring tasks
pocket quality test --daily
pocket system health --every15min
```

---

## Learn More

- **[STORY.md](../STORY.md)** — Why PocketCloud exists and how it was built
- **[FEATURES.md](../FEATURES.md)** — Complete feature list with verification commands
- **[ARCHITECTURE.md](../ARCHITECTURE.md)** — System architecture deep dive
- **[ROADMAP.md](../ROADMAP.md)** — What's next
- **[docs/why-local-ai.md](why-local-ai.md)** — The case for local-first AI
- **[docs/local-ai.md](local-ai.md)** — How local inference works in PocketCloud
- **[docs/privacy-model.md](privacy-model.md)** — Privacy guarantees and data handling
- **[docs/development-velocity.md](development-velocity.md)** — Development statistics

---

## Stay Updated

Star this repository to follow progress. For questions or to express interest, open a GitHub issue.

**Author**: [Steven Moon](https://www.linkedin.com/in/stevenmoon/) — 26+ years of software development, 18+ years on Apple platforms
