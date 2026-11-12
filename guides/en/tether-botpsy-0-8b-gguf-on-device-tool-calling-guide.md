---
title: "On-Device Tool Calling in 530MB: Tether BotPsy-0.8B GGUF Numbers and the Default Chat Template Trap"
description: "An engineering teardown of Tether AI Research's open-weight BotPsy-0.8B: Qwen3.5-0.8B fine-tuning, Q4_K_M vs Q8_0 GGUF file sizes and benchmark tradeoffs, the embedded chat template defaulting to non-thinking mode, and the 4.5-point ceiling on BrowseComp-Plus."
pubDate: 2026-11-12
author: "小可"
category: "Edge & Quantization"
tags: ["BotPsy-0.8B", "GGUF", "llama.cpp", "On-Device Agent", "Qwen3.5"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/tether-botpsy-0-8b-gguf-on-device-tool-calling-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


Tether, the stablecoin issuer, just open-sourced an on-device small model. Called `BotPsy-0.8B` and fine-tuned on `Qwen3.5-0.8B`, it targets local function calling on phones and low-spec laptops.

How small? The `Q4_K_M` GGUF is `0.53 GB` — `504.8 MiB` to be exact. Even `Q8_0` sits at `0.81 GB`.

`BFCL v4` hits `44.0` and `ToolSandbox` reaches `64.6`, putting it ahead of `Qwen3.5-2B-Think` at `2.5x` the parameter count. Those numbers look tempting. Yet if you drag the GGUF straight onto a phone right now, you will likely get broken JSON back. After reading the main model card and the `GGUFs` notes line by line, three details stand out before touching anything.

Three sources ground this breakdown: `huggingface.co/qvac/BotPsy-0.8B`, `huggingface.co/qvac/BotPsy-0.8B-GGUFs`, and the release post at `huggingface.co/blog/qvac/botpsy`. I have not benchmarked `32768`-context peak memory or thermal throttling on an Android handset, so no battery or tok/s claims appear below — no way to measure them here.

## Why 0.8B Beats 2B on Tool Benchmarks — and Where It Drops Off

`BotPsy-0.8B` builds on `Qwen3.5-0.8B`, a hybrid `Gated DeltaNet` plus gated-attention architecture, text-only here under Apache 2.0.

Against its `Qwen3.5-0.8B-Think` base in the shared harness: `BFCL v4` jumps from `23.4` to `44.0`, `ACEBench` climbs from `30.1` to `58.7`, and the `τ-bench` mean across retail, telecom, and banking rises from `23.2` to `51.5` — roughly double the base scores. That is not general intelligence scaling; fine-tuning carved out the tool-calling path specifically. On BFCL v4 hallucination rejection — refusing to invoke tools when none fit — it scores `86.1`.

Scroll down to knowledge benchmarks and the floor drops out.

`GPQA Diamond` lands at `38.1` versus `59.9` for `Qwen3.5-2B-Think`. `MMLU-Pro` sits at `48.9` versus `65.5`. Worse, `BrowseComp-Plus` peaks at `4.5`. Forget open-ended multi-hop web research. Its sweet spot is tightly bounded, short-chain tool calls with clear schemas.

## First Launch Trap: Thinking Mode Defaults to Off

This is the easiest place for first-time testers to get garbage output.

`BotPsy-0.8B` was trained and evaluated in thinking mode. Yet the `GGUFs` card explicitly notes that the embedded chat template defaults to non-thinking. Load it straight into `llama-cli` without overriding that flag and the model skips reasoning and emits raw JSON with degraded accuracy.

The official `llama.cpp` server command pins down strict sampling settings:

```bash
hf download qvac/BotPsy-0.8B-GGUFs botpsy-0.8b-q8_0.gguf --local-dir .

llama-server -m botpsy-0.8b-q8_0.gguf --jinja -c 32768 \
    --temp 0.3 --top-p 0.95 --top-k 20 --presence-penalty 1.5 \
    --chat-template-kwargs '{"enable_thinking": true}' --port 8080
```

Watch two unusual flags. First, `--chat-template-kwargs '{"enable_thinking": true}'` — omit it and you are not running the benchmarked model. Second, `--presence-penalty 1.5`. Most small-model setups rarely push presence penalty that high; leaving a client default at `0` or `1.0` invites repetition loops inside the thinking block.

Two extra caveats from the card: the GGUF exports strip out the base model's `MTP` multi-token-prediction layers, and English is recommended. How badly non-English prompts degrade tool accuracy goes unmeasured in the docs.

## Quantization Ledger: Q8_0 Halves Size, While Engine Variance Beats Quant Loss

Three GGUF files sit in the repo, and picking among them is straightforward.

Skip `botpsy-0.8b-bf16.gguf` — `1446.5 MiB`, roughly `1.52 GB`, averaging `50.4` across ten benchmarks, `0.3` points *lower* than `Q8_0`. Carrying an extra gigabyte only makes sense as an evaluation baseline. For real deployment, pick between two options: on desktop or an `8 GB` phone, grab `botpsy-0.8b-q8_0.gguf` at `774.2 MiB`, `0.001` KL divergence, and a `50.7` average — half the footprint with no accuracy loss. Only squeeze down to `botpsy-0.8b-q4_k_m-imat.gguf` at `504.8 MiB` if memory is razor-thin; importance-matrix calibration cuts KL from `0.048` to `0.036`, yet the ten-benchmark average drops to `48.9` with `GPQA Diamond` and `MMLU-Pro` losing three to four points each. Saving ~270 MiB at the cost of world knowledge is a bad trade unless RAM forces your hand.

Here is the counterintuitive part. Running the exact same unquantized BF16 weights on `vLLM` versus `llama.cpp` CUDA `b12169` shifts `τ-bench` by `4.0` points, `Claw-Eval` by `3.1` points, and `IFBench` by `2.8` points in the opposite direction. Engine and sampling variance on a `0.8B` model exceeds the drop from `BF16` to `Q8_0`. Chasing tenths of a point on a quantization table barely means anything here.

## What Fits and What Does Not

The boundary is clear. This is not a replacement for desktop `7B` or `14B` generalists.

For an offline voice trigger, calendar helper, or local file organizer — fixed tool list, clean schemas, short turns — trading `800 MB` of RAM for an Apache 2.0 local tool-calling engine pencils out.

Expecting `0.8B` to refactor a multi-file codebase, parse long non-English docs, or browse across the open web? That `4.5` on `BrowseComp-Plus` already drew the ceiling.

Before pulling the weights, verify three items:

1. Your client can pass `enable_thinking: true` into `llama-server`; default template runs are a different model in practice.
2. With `1 GB` of headroom, default to `botpsy-0.8b-q8_0.gguf` instead of gambling knowledge accuracy on `Q4_K_M`.
3. Keep context tight on edge hardware — `-c 32768` works in the example, yet full 32K KV cache plus long thinking outputs on a phone still needs short-context load testing first.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/tether-botpsy-0-8b-gguf-on-device-tool-calling-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

