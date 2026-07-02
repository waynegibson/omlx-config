# Model Guide — Ranked Lineup for M1 Max / 64 GB

**Date:** 2026-07-02 · Ranked per category. "Speed feel" is relative decode speed on this hardware (memory-bandwidth-bound; MoE ≫ dense).

## Coding

| Rank | Model | Size | Speed feel | When to use it | Notes |
|------|-------|------|------------|----------------|-------|
| 1 | **Qwopus3.6-35B-A3B-Coder-6bit** | 29.1 GB | ★★★★★ (MoE, ~3B active) | **Daily driver.** Claude Code sessions, refactors, everyday coding. Start here for everything. | MTP head ✓ — use MTP only, TurboQuant off |
| 2 | **Qwopus3.6-27B-Coder-8bit** *(installed)* | 27 GB | ★★☆☆☆ (dense) | **Hard problems only.** When #1 gets stuck or produces sloppy logic — tricky algorithms, subtle bugs, architecture decisions. Quality ceiling of the lineup. | MTP head ✓ — already configured |
| 3 | **Qwen-AgentWorld-35B-A3B-oQ4** | 20.4 GB | ★★★★★ (MoE) | **Tool-heavy agent runs.** Long autonomous sessions where tool-call reliability matters more than code elegance (file ops, shell work, multi-step automation). | MTP head ✓ · text-only |
| 4 | **gemma-4-12b-coder-fable5-composer2.5-8bit** | 12.7 GB | ★★★★☆ (small dense) | **Quick hits.** One-file edits, boilerplate, explaining code, or when you want RAM free for other apps. | **No MTP** — configure with TurboQuant instead |

## General

| Rank | Model | Size | Speed feel | When to use it | Notes |
|------|-------|------|------------|----------------|-------|
| 1 | **Qwen3.6-35B-A3B-6bit** *(installed)* | 27 GB | ★★★★★ (MoE) | **Default for everything non-code.** Writing, Q&A, summarizing, brainstorming, image understanding. | MTP head ✓ — already configured |
| 2 | **gemma-4-31B-it-OptiQ-4bit** | 23.5 GB | ★★☆☆☆ (dense) | **Second opinion & careful writing.** Different model family = different blind spots. Use when #1's answer feels off, or for polished long-form writing. Strong vision. | **No MTP** — configure with TurboQuant instead |
| 3 | **gemma-4-26B-A4B-it-OptiQ-4bit** | 18.8 GB | ★★★★☆ (MoE) | Optional. Faster Gemma if #2 feels too slow. Skippable — overlaps heavily with #1. | No MTP |

## Rules of thumb (while you're learning)

1. **MoE beats dense for speed** on Apple Silicon: only active parameters are read per token. Dense 27-31B models will always feel slow here regardless of quantization.
2. **One model hot at a time.** Two ~27 GB models resident together ≈ 54 GB — right at the memory ceiling. The 300 s idle timeout unloads the inactive one; don't pin multiple large models.
3. **One speculative path per model** — omlx rejects the whole settings entry otherwise (check `server.log` for "Failed to load settings"). Qwen3.6/Qwopus family: `mtp_enabled: true` only (no D-Flash, no TurboQuant, no vlm_mtp). Gemma family (no MTP head): `turboquant_kv_enabled: true` only.
4. **Start at rank 1, escalate on failure.** Speed compounds over a session; quality only matters when the fast model actually fails.
5. Total lineup ≈ 131 GB on disk if you download everything — fine on the 1 TB SSD.

## Download commands

```bash
cd /Volumes/SSDSCX_DATA/workspace/applications/omlx/models

hf download mlx-community/Qwopus3.6-35B-A3B-Coder-6bit          --local-dir mlx-community/Qwopus3.6-35B-A3B-Coder-6bit
hf download mlx-community/Qwen-AgentWorld-35B-A3B-oQ4           --local-dir mlx-community/Qwen-AgentWorld-35B-A3B-oQ4
hf download mlx-community/gemma-4-12b-coder-fable5-composer2.5-8bit --local-dir mlx-community/gemma-4-12b-coder-fable5-composer2.5-8bit
hf download mlx-community/gemma-4-31B-it-OptiQ-4bit             --local-dir mlx-community/gemma-4-31B-it-OptiQ-4bit
```
