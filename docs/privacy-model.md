# Privacy Model

PocketCloud is designed so that your data stays on your device by default.
This is not a marketing claim — it is an architectural constraint enforced throughout the stack.

---

## What Stays on Your Device

Everything, unless you explicitly configure a cloud provider.

| Data | Location | Leaves device? |
|---|---|---|
| Chat history and sessions | `~/.pocketcloud/` or app sandbox | No |
| MLX model weights | `~/.pocketcloud/models/` | No |
| RAG knowledge index | `~/.pocketcloud/knowledge/` | No |
| Task schedules and history | `~/.pocketcloud/tasks/` | No |
| Workspace verification logs | `~/.pocketcloud/verify/` | No |
| API keys for cloud providers | macOS Keychain | No (key never transmitted) |

---

## What Leaves Your Device

Only the content of requests you explicitly send to a cloud provider you have configured.

If you have not configured any cloud provider, nothing leaves your device. The local MLX
inference engine, the `pocket` CLI, the MCP server, and all verification tools operate
entirely locally.

If you configure a cloud provider (OpenAI, Claude, Gemini, etc.):
- The prompt text you send is transmitted to that provider
- PocketCloud adds no metadata to requests beyond what the provider's API requires
- Your API key is used for authentication only and is never stored on external servers

---

## API Key Security

PocketCloud uses macOS Keychain for API key storage. Keys are:

- Loaded at runtime from Keychain into process memory
- Never written to log files
- Never transmitted as part of request payloads or metadata
- Never stored in plain text on disk (unless you explicitly use a `.env` file, which is
  `.gitignore`-protected and clearly documented as development-only)

For applications that run on iOS or other platforms, keys are stored in the platform
Keychain with appropriate access controls.

---

## Zero External Telemetry

PocketCloud contains no external telemetry:

- No External Analytics SDK (no Firebase, Amplitude, Mixpanel, etc.)
- No External Crash reporting service (no Sentry, Crashlytics, etc.)
- No External Usage telemetry (no "what features are used" reporting)
- No A/B testing infrastructure
- No "Phone home on startup" behavior
- All telemetry and logging stays on device

This is verifiable. The `pocket system verify --exhaustive` suite runs 112 operations and
includes workspace integrity checks. The source code contains no third-party analytics imports.

---

## Local Inference Privacy

When using local MLX inference:

- Your prompt is processed entirely on-chip (CPU, GPU, Neural Engine)
- Token generation happens in Apple Silicon unified memory
- No inference request is transmitted anywhere
- The model weights you download are stored locally and used locally

The only network activity associated with local inference is the one-time download of
model weights from Hugging Face (or a configured mirror). Once downloaded, models work
offline indefinitely.

---

## MCP Server Privacy

The PocketCloud MCP server runs as a local process communicating via stdio or a local socket.
It does not have a network-accessible endpoint by default.

When an AI assistant (Claude, etc.) uses PocketCloud's MCP tools:
- The AI sends tool calls over the local stdio/socket connection
- PocketCloud executes the tool locally
- Results are returned to the AI over the same local connection
- No tool input or output is transmitted to external services by PocketCloud

Note: The content of tool inputs and outputs *is* transmitted to the AI provider you are using
(Claude, etc.) as part of the AI's context — that is the nature of MCP. PocketCloud does not
add any additional transmission beyond the MCP protocol itself.

---

## Trust Model Summary

| Action | Who controls it | Where data goes |
|---|---|---|
| Local MLX inference | You (on-device) | Nowhere |
| RAG indexing and query | You (on-device) | Nowhere |
| `pocket` CLI commands | You (on-device) | Nowhere |
| MCP tool execution | You (on-device) | Nowhere |
| Cloud provider request | You (explicit opt-in) | To your configured provider only |
| API key storage | macOS Keychain | Nowhere |
| Logs and history | Local filesystem | Nowhere |
