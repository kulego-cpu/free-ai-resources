---
title: "Fitting a 27B Coding Model into 5.9GB: Testing Ternary Bonsai 2.5 and Four llama.cpp Pitfalls"
description: "Hands-on check of PrismML Ternary Bonsai 2.5 27B at 1.58-bit quantization, auditing its 5.93GB footprint, SWE-bench 75.6 score, RTX 4090 decode throughput, and four real deployment traps in llama.cpp."
pubDate: 2026-11-25T13:35:00.000Z
author: 小可
category: 极客实操
tags:
  - Ternary Bonsai
  - Ternary Quantization
  - llama.cpp
  - VRAM Optimization
  - Qwen3.8
  - Local Coding
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/ternary-bonsai-2-5-27b-5gb-vram-agentic-coding-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


Can an 8GB graphics card actually run a 27B parameter coding agent?

I was skeptical at first. Compressing a 53.8GB FP16 model down to 5.93GB leaves roughly 1.76 bits per weight. Surely that turns multi-step reasoning into gibberish?

On November 24, PrismML released `Ternary Bonsai 2.5 27B`, quickly hitting nearly 400 points on Hacker News.

I checked the whitepaper and pulled the repository instructions from `prism-ml/Ternary-Bonsai-2.5-27B-GGUF`.

Verdict: it works. Just do not pull it directly with an outdated `llama.cpp` binary, or you will run straight into silent failures.

---

## Why Did Ternary Quantization Hold Up This Time?

Previous ultra-low-bit builds fell apart the moment multi-turn tool calling started.

The earlier Bonsai 2 27B scored zero on DeepSWE coding tasks and only 51.2 on AppWorld multi-step workflows. Whenever a tool call returned an error, the model looped endlessly or fabricated a completion message.

Bonsai 2.5 still builds on `Qwen3.8-27B`: a 24.35B language backbone, 0.47B vision tower, and 2.54B embeddings, mixing roughly 75% linear attention with 25% full attention.

The storage layout remains unchanged: g128 group-wise FP16-scaled 1.58-bit ternary weights restricted to `-1`, `0`, and `+1`. Blockwise orthogonal Hadamard rotations are folded into the weights offline, so inference only rotates activations on the fly without eating extra memory bandwidth.

How much accuracy actually survived? Here are the benchmarks run under unified `xhigh` thinking mode:

| Benchmark | Footprint | Qwen3.8 FP16 | GSQ-RCO IQ2_XS | Bonsai 2 27B | Bonsai 2.5 27B |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Model Size | GB | 53.8 | 8.4 | 5.93 | 5.93 |
| Bits per Weight | bpw | 16.0 | 2.50 | 1.76 | 1.76 |
| SWE-bench Verified | Coding | 80.6 | 74.4 | 60.8 | 75.6 |
| DeepSWE v1.1 | Coding | 30.1 | 17.7 | 0.0 | 27.0 |
| Terminal-Bench 2.1 | Terminal | 70.8 | 53.4 | 52.8 | 60.1 |
| AppWorld | Tooling | 93.2 | 46.5 | 51.2 | 90.5 |
| τ³-Bench | Multi-turn | 72.8 | 67.5 | 69.1 | 73.9 |

Look closely at `GSQ-RCO IQ2_XS`. Despite being 40% larger, it collapses to 46.5 on AppWorld. Meanwhile Bonsai 2.5 averages 77.2 across five tool-calling suites, sitting just 0.5 points behind FP16.

Does that mean it beats full precision across the board?

No. On Terminal-Bench 2.1 it drops 10.7 points below FP16, and trails by 3.6 points on BFCL v4. Several Hacker News commenters also recalled doom-loop frustration on earlier builds and questioned potential benchmark tuning.

Trading one-ninth the memory footprint for roughly 90% of FP16 coding capability is a deal I will gladly take for daily local autocomplete.

---

## How Fast Is It? Stop Confusing MTP with Base Speeds

Plenty of reposts throw around 204 tok/s without mentioning how that number was measured.

The repository ships two separate sets of GGUF files: base weights without a draft head, and `-mtp.gguf` variants carrying a multi-token prediction draft layer.

Without MTP speculative decoding, pure `PTQ1_0` and `PQ2_0` throughput looks like this:

- RTX 4090: `PTQ1_0` decodes at 112 tok/s; prompt prefill hits 3,452 tok/s
- RTX 5090: `PTQ1_0` decodes at 163 tok/s. Prefill hits 4,198 tok/s
- Apple M5 Pro: Metal tested on `PQ2_0` only, decoding at 30.2 tok/s

Only when switching to `-mtp.gguf` alongside the official `presets/llama.cpp/mtp-cuda.json` preset does an RTX 4090 reach 204 tok/s. The vision tower `mmproj-Q8_0.gguf` ships separately, saving 630MB of VRAM during text-only coding runs.

---

## Skip the Hype and Watch Out for Four Traps

Once you actually try booting it locally, four practical traps matter far more than benchmark tables.

### Downloaded 5.99GB PTQ1_0 and Stock llama.cpp Rejected It?

The repo hosts three distinct packings tied to different runtimes:

- `PTQ1_0`: 5.99 GB dense ternary packing at 1.76 bpw; requires the `PrismML-Eng/llama.cpp` fork
- `PQ2_0`: 7.25 GB 2-bit-slot ternary packing aimed at Apple Metal; also requires the fork
- `Q2_0`: 7.67 GB standard `llama.cpp` format

Too lazy to compile the fork and want to run `Q2_0` on stock upstream binaries?

Sure, provided you check the version cutoff: **build `b11556` or newer** is mandatory for the Hadamard activation runtime.

Worse still, older stock `llama.cpp` builds load `Q2_0` **without a single warning** and emit pure garbage. Run `llama-cli --version` before blaming the weights.

### Four Default Slots Blew Past 10GB VRAM

With a 6GB model file, why did a 10GB RTX 3080 nearly crash desktop rendering on startup?

Parallel slots. By default `llama-server` spawns four parallel slots sharing a unified KV cache. A user on Hugging Face reported total GPU memory jumping to 9.5 GB until `-np 1` locked it to one request slot, dropping usage down to 7.8 GB.

### Forgetting --min-p 0 Breaks Sampling

Upstream `llama.cpp` defaults to `min_p=0.05`. Bonsai 2.5 relies on Qwen3.8 thinking parameters that require disabling `min_p` truncation. Omit that flag and long coding chains quickly degrade into repetitive loops.

### Passing -n 256 Cuts Output Mid-Thought

This is a heavy thinking model defaulting to `xhigh` reasoning effort, and `low` effort is unsupported. Forcing `low` still triggers near-`xhigh` thinking chains.

Passing `-n 256` on the CLI exhausts the token budget inside `<think>` tags before a single line of code appears.

---

## Copy-Paste Local Server Launch Script

Once your PrismML fork or `b11556+` stock `llama.cpp` binary is ready, launch with these flags:

```bash
# Option A: Build PrismML fork for the smallest 5.99GB PTQ1_0 packing
git clone https://github.com/PrismML-Eng/llama.cpp && cd llama.cpp
cmake -B build -DGGML_CUDA=ON && cmake --build build -j

hf download prism-ml/Ternary-Bonsai-2.5-27B-GGUF Ternary-Bonsai-2.5-27B-PTQ1_0.gguf --local-dir .

./build/bin/llama-server -m Ternary-Bonsai-2.5-27B-PTQ1_0.gguf \
  -ngl 99 -fa on -np 1 -c 32768 -n 16384 \
  --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0 \
  --host 127.0.0.1 --port 8080
```

On a 16GB Mac, switch to `Ternary-Bonsai-2.5-27B-PQ2_0.gguf` and drop `-DGGML_CUDA=ON` to enable Metal acceleration. Prefer zero compilation? Pull the 7.67GB `Q2_0` file and feed it to stock `b11556+` `llama-server`.

---

## Should It Stay in Your Daily Toolchain?

Developers stuck on 8GB GPUs or 16GB laptops previously stood zero chance of running Qwen3.8-27B. Trading 6GB of disk space for 75.6 on SWE-bench gives offline coding a genuinely viable local fallback.

The trade-off remains clear: 1.76 bpw does not replace full precision, and losing 10 points on Terminal-Bench hurts. Whether such ultra-low-bit compression holds together across dozens of messy multi-file refactors is something I am still stress-testing on my own private repositories.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/ternary-bonsai-2-5-27b-5gb-vram-agentic-coding-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

