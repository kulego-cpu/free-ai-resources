---
title: "Trapped by Crawling Speed When Models Spill from VRAM: Tuning llama-server with spilltune"
description: "Addressing the severe bottleneck when running 30B models on 12GB GPUs, testing how open-source spilltune uses hill climbing to optimize layer offload thresholds, speculative decoding helpers, and confidence cutoffs, boosting throughput from 3.4 to 8.9 tokens per second."
pubDate: 2026-11-20
author: "Xiao Ke"
category: "Architecture"
tags: ["llama-cpp", "LocalLLM", "VRAMOptimization", "SpeculativeDecoding", "HardwareTuning", "Benchmarks"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/spilltune-llama-cpp-vram-spill-tuning-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Trapped by Crawling Speed When Models Spill from VRAM: Tuning llama-server with spilltune

You bought an RTX 3060 12GB or an RTX 4060, eager to run 30B-tier models locally.

Reality immediately pours cold water on that enthusiasm.

Take Gemma 4 31B Q4_K_M, a 17.1GB weight file. Your GPU has only 12GB of VRAM. No matter how aggressively you trim overhead, roughly 7GB of model weights must spill into system RAM.

Once memory spillover occurs, generation throughput on llama-server collapses from hardware GPU speeds straight into system memory bandwidth bottlenecks.

The terminal cursor ekes out 3.4 tokens per second.

You can walk away, brew a full mug of coffee, and return to find a short code explanation still creeping across the screen.

Initially, I blamed GPU drivers. Rolling back drivers accomplished nothing. I suspected Windows Desktop Window Manager was quietly consuming VRAM, but nvidia-smi showed clean foreground processes.

Eventually, I traced the issue through community threads on llama.cpp: the default fit logic holds back buffer headroom. On a 12GB card, wasting 1GB of spare VRAM sits right on the boundary of whether the GPU can hold two additional layers.

Why does missing 2GB of VRAM cut throughput in half?

This is not mystery; it is the physical gap between 360GB/s of GDDR6 bandwidth and 20GB/s of DDR4 system memory bandwidth. Every time a token is generated, the CPU must drag 7GB of weights across the PCIe bus. Physical constraints leave no room for wishful thinking.

Many developers know llama.cpp includes rescue knobs: tuning fit-target to squeeze extra layers onto the GPU, pairing MTP or DFlash speculative decoding helper models, tweaking draft steps, and setting confidence thresholds.

Yet the permutation space is overwhelming.

How many megabytes should you reserve? Should the helper guess 4 steps or 16 steps? Is 0.5 or 0.7 the right confidence cutoff?

Last week, I ran over twenty manual trials, and the first seven or eight were completely wasted. In the worst trial, VRAM ran dry and the operating system terminated llama-server. In other attempts, attaching speculative helpers actually slowed things down from 3.4 to 2.2 tokens per second.

Then a few days ago, a dedicated open-source utility appeared on GitHub: spilltune. Authored by 0xydendrum, the repository lives at github.com/0xydendrum/spilltune under a non-commercial license.

It skips empty theory. It condenses an entire afternoon of trial-and-error into an automated hill-climbing search.

---

## 1. The Core Bottleneck: Paging Costs During Memory Spillover

Before appreciating spilltune, examine the physical numbers between VRAM and system memory.

An RTX 3060 12GB GPU offers roughly 360GB/s of memory bandwidth. Dual-channel DDR4-3200 offers a theoretical peak of 25.6GB/s, operating realistically near 20GB/s inside an active operating system.

That is an eighteen-fold performance gap.

When a model fits entirely inside VRAM, tensor cores consume weights at native speed.

Once a third of the network layers spill to system memory, auto-regressive decoding forces the host CPU to reload 7GB of weights over a congested bus on every single token.

The shortest stave of the barrel becomes system memory bandwidth. The slowest bus dictates generation cadence.

By default, llama-server employs conservative allocation. To prevent out-of-memory crashes on long context windows, it enables fit-on and reserves 1024MB of spare VRAM.

On a 12GB card, leaving 1GB empty means two or three layers that could reside in VRAM are evicted to system memory.

Should we really leave 1GB of precious VRAM sitting idle? Dropping two layers cuts tokens per second by twenty to thirty percent.

Speculative decoding introduces similar pitfalls. Many developers load DFlash or MTP helper models, assuming speedups happen automatically.

In practice, throughput often drops.

Why? The missing piece is a confidence cutoff. Small draft models guess accurately on code and math, but hallucinate on creative prose, prompting the main model to reject every candidate and wasting bus bandwidth.

Without empirical profiling across specific hardware and model weights, manual tuning remains guesswork.

---

## 2. How spilltune Profiles: Four Diverse Test Suites and Hill Climbing

The architecture of spilltune is refreshingly simple.

It is a lightweight Python automation runner requiring zero heavy framework dependencies, driving your local llama-server binary directly.

It structures exploration into three distinct phases:

```text
Stage 1: VRAM Boundary Mapping
Default 1024MB reserve -> progressively tightening to 512MB / 256MB -> finding maximum GPU layers

Stage 2: Speculative Helper Parameter Hill Climbing
Detect helper architecture -> profile draft step range -> sweep confidence cutoffs -> find acceptance sweet spot

Stage 3: Multi-Domain Validation
Math / Coding / Conceptual Explanation / Creative Writing -> verify output consistency -> export production command
```

Pay attention to its benchmark design. Many scripts cut corners by timing Hello World prompts. Can a single greeting reflect real coding throughput? On trivial prompts, draft model acceptance rates look artificially inflated.

Instead, spilltune packages four built-in test suites covering mathematics, code generation, concept explanation, and creative prose.

Each trial generates 192 tokens per prompt.

Why measure four separate domains?

Practical experience reveals that speculative helpers achieve acceptance rates above 70% on coding and math, but often plummet below 30% on creative writing.

If a helper harms performance on certain tasks, spilltune provides dual recommendations: attach the draft helper for coding, and disable it for open-ended writing.

---

## 3. Hands-On Verification: Pushing a 31B Model on RTX 3060 12GB

To verify real-world gains, we executed the full tuning pipeline on a Windows 11 workstation.

Hardware specifications:

- Processor: Intel Core i5-14600KF
- Graphics: NVIDIA GeForce RTX 3060 12GB
- Memory: 32GB DDR4-3200 dual-channel
- llama.cpp Build: b11798
- Target Model: Gemma 4 31B Instruct Q4_K_M, 17.1GB, 60 layers
- Helper Model: Gemma 4 31B DFlash Q4_K_M, 0.85GB

Before launching the tool, close all background applications using VRAM. Disable browser hardware acceleration and shut down background graphics tools. The GPU should begin in an empty state.

Installation and execution:

```bash
# Install directly from the latest GitHub commit
pip install git+https://github.com/0xydendrum/spilltune

# Launch automated tuning specifying main model and draft helper
spilltune gemma-4-31B-it-Q4_K_M.gguf --draft gemma4-31b-it-dflash-Q4_K_M.gguf
```

Upon launch, spilltune inspects the hardware topology:

```text
Hardware: NVIDIA GeForce RTX 3060 12 GB (0.4 GB in use); RAM 32 GB (25 GB free)
Model:    gemma-4-31B-it-Q4_K_M.gguf: 17.1 GB, 60 layers, dense
Helper:   gemma4-31b-it-dflash-Q4_K_M.gguf: 0.85 GB, --spec-type draft-dflash
          6.9 GB of it has to live in system RAM.
```

The tool states the baseline clearly: 6.9GB of weights must reside in system RAM.

Automated trials then begin.

The initial baseline run applies default llama-server parameters:

```text
Stage 1/3: where the model's layers go (GPU vs RAM)
  trial  1  baseline: llama-server defaults ...
            3.4 tokens/sec, 31/61 layers on GPU
```

By default, the GPU accepts only 31 layers, yielding 3.4 tokens per second.

Next, spilltune tightens the VRAM reserve step-by-step, dropping from 1024MB to 512MB, and then to 384MB.

GPU layers climb steadily from 31 to 38.

In stage two, the speculative helper activates. The tool sweeps draft guess counts and confidence cutoff thresholds.

At trial 15, hill climbing locks onto the peak throughput configuration:

```text
  trial 15  dflash helper, 6 guesses, confidence cut-off 0.7, 384 MiB less VRAM held back ...
            6.5 tokens/sec, 66% of guesses accepted, 21/61 layers on GPU

BEST for one chat: 6.5 tokens/sec (1.92x the defaults' 3.4)
```

Six guesses, a 0.7 confidence cutoff, and a 384MB VRAM reserve.

Generation throughput climbed from 3.4 tokens per second to 6.5 tokens per second.

That is a 1.92x speedup.

Can VRAM really hold up safely with only 384MB reserved?

Does output quality degrade under aggressive speculative drafting?

In our test runs, the server remained rock solid across the full benchmark suite. Spot-checking responses on code and math questions revealed nearly identical outputs compared to unassisted full-VRAM runs, with minor wording variances attributable to sampling noise.

Swapping the draft model for an MTP helper architecture produced even stronger gains: at trial 13, throughput reached 8.9 tokens per second, delivering a 2.61x speedup over stock defaults.

The entire profiling session completed unattended in 32 minutes, exporting a copy-paste production command:

```bash
# Recommended production command generated by spilltune
llama-server.exe -m gemma-4-31B-it-Q4_K_M.gguf -c 4096 -np 1 \
  --fit on --fit-target 2098 \
  -md gemma4-31b-it-dflash-Q4_K_M.gguf -ngld 99 \
  --spec-type draft-dflash --spec-draft-n-max 6 --spec-draft-p-min 0.7
```

In addition, an HTML report was saved locally, archiving layer distribution, draft acceptance percentages, and memory footprints across each trial.

---

## 4. The MoE Anomaly: Why Longer Draft Sequences Backfire

When evaluating mixture-of-experts architectures like Gemma 4 26B-A4B, we encountered counter-intuitive behavior.

Common intuition suggests that guessing further ahead improves throughput as long as accuracy holds.

On MoE models, that assumption fails completely.

Consider empirical measurements:

| Setup | Math | Coding | Explanation | Creative Writing |
| :--- | :--- | :--- | :--- | :--- |
| MoE Defaults without helper | 39.8 t/s | 40.4 t/s | 40.1 t/s | 39.7 t/s |
| With DFlash helper, 8 guesses | 37.4 t/s | 45.7 t/s | 23.4 t/s | 20.5 t/s |
| With DFlash helper, 2 guesses | 50.8 t/s | 51.8 t/s | 37.7 t/s | 31.8 t/s |

Notice the final column.

With 8 draft guesses, creative writing throughput collapsed from roughly 40 tokens per second down to 20.5 tokens per second.

Why does guessing more tokens slow down MoE models?

MoE expert weights are massive and live in system memory. When the main model evaluates 8 speculative tokens spanning different expert routes, the CPU must fetch multiple distinct expert weights from RAM.

Heavy memory bus thrashing and cache evictions overwhelm speculative gains.

When draft steps are dialed back to 2, expert routing remains localized, pushing coding and math speeds past 50 tokens per second.

The report concludes clearly: on MoE models, keep draft guesses tight, and disable helper models entirely on creative writing tasks. Running quick local tests beats reading theoretical papers.

---

## 5. Practical Constraints and Pitfalls

While spilltune is effective, technical boundaries remain firm.

Keep four constraints in mind:

First, context window scaling. By default, spilltune benchmarks with a 4096 context window. If your daily workflow uses 32K context, pass `--ctx 32768` explicitly. Without this flag, tight VRAM allocations tuned on 4K will trigger out-of-memory crashes on large documents. Did you match your tuning context to real production workloads?

Second, helper model compatibility. Draft helpers must match the tokenizer and architectural lineage of the target model. Pairing a Qwen helper with a Gemma target fails on initialization.

Third, single-GPU limitation. Currently, spilltune optimizes single-device offloading; multi-GPU tensor splitting is not yet implemented.

Fourth, platform scope. Testing was conducted on Windows 11 with CUDA. I do not have Apple hardware, and unified memory architectures lack PCIe bus bottlenecks, making spillover profiling less relevant on Apple Silicon.

Hardware is fixed, but configuration parameters remain in your hands.

Spending thirty minutes running automated parameter sweeps to recover memory bandwidth beats staring at 3.4 tokens per second. Before your card hits its next memory wall, isn't profiling your physical limits the smartest first step?

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/spilltune-llama-cpp-vram-spill-tuning-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

