# oMLX Hardware Optimization Report

**Date:** 2026-07-02 (supersedes 2026-06-30 report, which covered the previous model pair)
**Status:** ✅ Implemented — changes applied, pending server restart

---

## 1. Hardware Profile

| Component | Specification | Implication |
|-----------|---------------|-------------|
| **CPU/GPU** | Apple M1 Max (40-core GPU) | Excellent MLX throughput |
| **Unified Memory** | 64 GB | Model weights + KV cache + context must fit here |
| **Storage** | 1 TB SSD via Thunderbolt 4 (permanently attached) | Fast reads (~2-3 GB/s), holds model weights and KV cache |
| **OS** | macOS | Native MLX support |

### Memory Budget Per Model (current models, measured on disk)

| Model | Weights on disk | Headroom vs ~56 GB ceiling (90% of 64 GB) |
|-------|-----------------|--------------------------------------------|
| **Qwopus3.6-27B-Coder-8bit** (dense, 8-bit, VLM) | ~27 GB | ~29 GB for KV cache + context |
| **Qwen3.6-35B-A3B-6bit** (MoE, 6-bit, VLM) | ~27 GB | ~29 GB for KV cache + context |

Uncompressed KV at the configured context windows fits comfortably: roughly 12–16 GB at 65K for the dense 27B, and well under 10 GB at 32K for the MoE.

> ⚠️ **Both models resident at once is tight**: 27 + 27 GB weights ≈ 54 GB before any KV cache. The 300 s idle timeout normally unloads the inactive model; avoid workloads that keep both hot simultaneously at large contexts.

---

## 2. Key Finding: TurboQuant vs MTP Mutual Exclusivity

**Constraint (carried over from the 2026-06-30 investigation):** `turboquant_kv_enabled` and `mtp_enabled` are mutually exclusive — only one can be `true` at a time.

| Feature | What It Does | Benefit |
|---------|--------------|---------|
| **TurboQuant** | 4-bit KV cache compression | Longer contexts in RAM |
| **MTP** (Multi-Token Prediction) | Predicts multiple tokens per step | **2-3× generation speedup** |

**Decision: MTP.** With 64 GB and ~27 GB per model, RAM is not the constraint at the configured context windows. Both current models ship MTP heads (`mtp_num_hidden_layers: 1` in `text_config`), verified in their `config.json`.

---

## 3. Second Finding: `reasoning_effort` Was a Silent No-op

The configs shipped `chat_template_kwargs: {"reasoning_effort": "medium"}` for both models. Neither model's `chat_template.jinja` reads `reasoning_effort` — the only kwarg the Qwen-family templates honor is `enable_thinking`. The kwarg was removed everywhere (model settings, profiles, global templates). Thinking on/off via `enable_thinking: true` is the only reasoning control these models support; there is no graded effort level.

---

## 4. Changes Applied 2026-07-02

Applied consistently to `base/model_settings.json`, the active profile in `base/model_profiles.json`, and `base/global_templates.json`.

### 4.1 Qwopus3.6-27B-Coder-8bit (coding, temp 0.1, 65K context)

| Parameter | Before | After | Rationale |
|-----------|--------|-------|-----------|
| `chat_template_kwargs` | `{"reasoning_effort": "medium"}` | **`{}`** | No-op — template only reads `enable_thinking` |
| `mtp_enabled` | `false` | **`true`** | 2-3× generation speedup (model has MTP head) |
| `vlm_mtp_enabled` | `false` | **`true`** | Model has a vision config (see §6 caveat) |
| `turboquant_kv_enabled` | `true` | **`false`** | Mutually exclusive with MTP |
| `dflash_enabled` | `false` | **`true`** | Draft speculative decoding, stacks with MTP |
| `max_tokens` | `4096` | **`16384`** | 4096 truncates thinking + response |
| `max_tool_result_tokens` | `4096` | **`16384`** | Was truncating agentic tool results hard |
| `specprefill_enabled` | `false` | `false` (unchanged) | Dropping prompt tokens is riskiest for code accuracy |

### 4.2 Qwen3.6-35B-A3B-6bit (general, temp 0.7, 32K context)

| Parameter | Before | After | Rationale |
|-----------|--------|-------|-----------|
| `chat_template_kwargs` | `{"reasoning_effort": "medium"}` | **`{}`** | Same no-op removal |
| `mtp_enabled` | `false` | **`true`** | Same speedup (MoE also has MTP head) |
| `vlm_mtp_enabled` | `false` | **`true`** | Model has a vision config (see §6 caveat) |
| `turboquant_kv_enabled` | `true` | **`false`** | Mutually exclusive with MTP |
| `dflash_enabled` | `false` | **`true`** | Stacks with MTP |
| `specprefill_keep_pct` | `0.2` | **`0.35`** | More RAM headroom → keep more prefill tokens |
| `max_tokens` | `4096` | **`8192`** | Thinking + response headroom at 32K context |
| `max_tool_result_tokens` | `0` | **`16384`** | Explicit cap instead of unlimited into a 32K window |

Unchanged intentionally: per-profile sampling (temperature, top_p, repetition_penalty), `enable_thinking: true`, thinking budget off, `turboquant_kv_bits`/`turboquant_skip_last` kept for completeness (no-op while MTP is on), D-Flash cache limits (8 GB RAM / SSD cache off).

### 4.3 Server-level (`base/settings.json`)

| Parameter | Before | After | Rationale |
|-----------|--------|-------|-----------|
| `hot_cache_max_size` | `"8GB"` | **`"12GB"`** | RAM-backed hot KV cache; plenty of headroom |

Everything else verified accurate for the current install: `ssd_cache_dir` under `models/cache`, 92 GB SSD cache, `chunked_prefill: true`, `max_concurrent_requests: 4`, `burst_decode_mode: "aggressive"`, memory guard `aggressive`, global `max_context_window: 131072` (per-model settings override it downward).

---

## 5. Post-Restart Monitoring Checklist

- [ ] Server starts without errors; both models load
- [ ] MTP active — generation visibly faster
- [ ] D-Flash active — draft model loading appears in logs
- [ ] No VLM MTP errors/warnings in logs (see §6)
- [ ] No OOM during extended sessions (< ~56 GB in Activity Monitor)
- [ ] Cache hit rate stays high (`base/stats.json`: `total_cached_tokens / total_prompt_tokens` > 85%)

---

## 6. Open Items / Known Uncertainties

| Item | Status |
|------|--------|
| `vlm_mtp_enabled: true` | The 2026-06-30 report contradicted itself (rationale said true, final JSON said false) and was never validated post-restart. If the server complains about VLM MTP, flip only this flag back to `false`. |
| No default model | Both models have `is_default: false`. Set one to `true` if unnamed API requests should resolve. |
| Claude Code model mapping | `claude_code.opus_model` / `sonnet_model` / `haiku_model` are `null` in `settings.json`. |
| `max_context_window` growth | Coder could go beyond 65K if needed — re-check KV memory first since TurboQuant compression is now off. |

---

## 7. Rollback

This directory is its own git repository; config changes are tracked there.

```bash
cd /Volumes/SSDSCX_DATA/workspace/applications/omlx

git checkout HEAD -- base/model_settings.json base/model_profiles.json \
                     base/global_templates.json base/settings.json

# then restart the omlx server
```
