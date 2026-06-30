# oMLX — Local LLM Inference Server

oMLX-based local inference server running Qwen 3.5/3.6 models on Apple Silicon. Replaces cloud APIs (Claude, GPT) with self-hosted models via MLX quantization.

## Models

| Model                                     | Size              | Role                | Profile              |
| ----------------------------------------- | ----------------- | ------------------- | -------------------- |
| **Qwen3.6-35B-A3B-oQ6-mtp**               | ~21 GB (6 shards) | Default / Opus-tier | `coding` (temp 0.35) |
| **Qwen3.5-27B-Claude-4.6-Opus-Distilled** | ~14 GB (3 shards) | Pi / Sonnet-tier    | `general` (temp 0.5) |

## Quick Start

```bash
# Set credentials (required)
export OMLX_API_KEY=<your_api_key>
export OMLX_SECRET_KEY=<your_secret_key>

# Start server (auto-starts on launch per config)
omlx serve
```

Server runs on `127.0.0.1:8055` by default.

## Architecture

- **Quantization**: oQ mixed-precision (6-bit MoE) + 4-bit dense
- **KV Cache**: SSD-backed (92 GB max), TurboQuant 4-bit KV compression
- **Speculative Prefill**: Enabled, 20% keep ratio
- **Context Window**: 65,536 tokens (configurable)

## Directory Structure

```
base/          — Server config, model profiles, usage stats
cache/         — SSD-backed KV cache blocks (~4 GB current)
models/        — Model weights (gitignored, ~42 GB total)
```

## Environment Variables

| Variable          | Default               | Description                       |
| ----------------- | --------------------- | --------------------------------- |
| `OMLX_API_KEY`    | `spacecomx`           | API key for client authentication |
| `OMLX_SECRET_KEY` | _(see settings.json)_ | HMAC secret for token signing     |

## Model Profiles

Each model supports 4 profiles with different sampling parameters:

| Profile    | Temperature | Use Case                      |
| ---------- | ----------- | ----------------------------- |
| `general`  | 0.5–0.55    | Balanced daily use            |
| `coding`   | 0.35        | Deterministic code generation |
| `agentic`  | 0.45        | Tool use, agent workflows     |
| `creative` | 0.8–0.85    | Writing, brainstorming        |

## Integrations

Configured for: Claude Code, Codex, OpenCode, OpenClaw, Copilot, Pi.

## Stats

Usage telemetry tracked in `base/stats.json` — total tokens, cached tokens, request counts, and per-model breakdowns.
