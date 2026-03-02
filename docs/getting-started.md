# Getting Started with PocketCloud

PocketCloud is currently in private development. This preview repository documents what is
being built and where it is heading.

---

## How to Follow

**Star this repository** to get notified when new features are verified and documented.

**Open an issue** if you have questions, want to discuss the roadmap, or want to express
interest in early access.

---

## What Is Available Today

PocketCloud is not yet publicly distributed as a downloadable app. The repository documents
the capabilities that are verified and working in the development build.

If you are a developer interested in the project, the [FEATURES.md](../FEATURES.md) documents
every working feature with its verification command.

---

## What's Coming

See the [ROADMAP.md](../ROADMAP.md) for the near-term (Q1-Q2 2026) and long-term vision.

Key near-term milestones:
- RAG-augmented local inference (codebase-aware AI)
- Persistent MLX daemon (sub-100ms response time)
- Apple Intelligence integration

---

## System Requirements

To run PocketCloud when it becomes available:

| Requirement | Details |
|---|---|
| macOS | 14+ (Sonoma) for CLI and hub app |
| iOS | 17+ for PocketMind |
| tvOS | 17+ for EmotionalIntelligence |
| Apple Silicon | Required for local MLX inference |
| Xcode | 16+ (for building from source) |

Local inference requires an M-series Mac or Apple Silicon iPhone/iPad. Cloud providers
(OpenAI, Claude, etc.) work on any supported platform.

---

## Stay Updated

- Watch this repository for documentation updates
- Each verified feature update corresponds to a `pocket system verify` baseline improvement
- The [FEATURES.md](../FEATURES.md) is updated whenever new capabilities are verified
