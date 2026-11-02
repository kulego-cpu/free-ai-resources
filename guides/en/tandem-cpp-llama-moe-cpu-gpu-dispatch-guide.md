---
title: "Patching llama.cpp: Boosting Local MoE Decoding from 29 to 97 Tokens per Second on a Single RTX 4090"
description: "Running local Mixture of Experts models usually starves the GPU while weight matrices crawl back and forth across the PCIe bus. Here is an engineer breakdown of the tandem.cpp patch set, offloading cold expert matrix math directly to host CPU cores to hit 97 tokens per second on an i5 plus RTX 4090 workstation, along with honest trade-offs and hardware limits."
pubDate: 2026-11-02
author: "Lao Jiu · FreeAI Dog Lead Architect"
category: "Hands-on Guide"
tags: ["tandem.cpp", "MoE Inference", "llama.cpp", "Mixture of Experts", "Hardware Optimization", "Offloading"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/tandem-cpp-llama-moe-cpu-gpu-dispatch-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Patching llama.cpp: Boosting Local MoE Decoding from 29 to 97 Tokens per Second on a Single RTX 4090

When running Mixture of Experts MoE models on local hardware, total parameter scale is rarely the core headache.
The real nightmare is having more than half your expert weights idling inside system RAM.
Every time gating routes a token to an offloaded expert, the runtime transfers hundreds of megabytes across the PCIe bus.
Across PCIe 4.0 x16, theoretical bandwidth caps out at 31.5 GB/s.
Your GPU tensor cores sit around doing nothing.
Tokens crawl across the terminal at twenty or thirty words per second.
Cooling fans scream.
You sit there waiting.
Is it sluggish?
It is infuriatingly slow!

The premise behind the tandem.cpp patch set is simple:
If moving weights across the bus creates a massive traffic jam, stop moving them!
Let multi-core host CPUs compute offloaded expert matrix math directly inside system RAM!
I patched upstream llama.cpp according to the project repository and stress tested it for two full days on my workstation.
Here are the raw benchmark numbers from my machine, along with several practical hurdles you must understand in advance.

---

## 1. Why Standard Offloading Starves Your Flagship GPU

Why does a powerhouse RTX 4090 crawl like a tractor when decoding a Mixture of Experts model?
The entire bottleneck traces back to the bus transfer penalty.
Painful reality.

Consider a standard quantized MoE measuring roughly 40GB in size.
A single 24GB VRAM buffer cannot accommodate the complete weight payload.
Standard llama.cpp offloading applies a basic rule:
Layers that fit stay in VRAM; overflow layers remain in host RAM.
Whenever router gating selects an offloaded expert, trouble starts.
The runtime fetches expert weights across the PCIe bridge into VRAM.
Every single generated token triggers hundreds of megabytes in transit.
Eighty percent of GPU compute capacity is squandered waiting for data transfers!
Massive compute headroom.
Complete bus constipation.
Who can tolerate that?

In the past, specialized acceleration engines claimed over one hundred tokens per second.
Looking under the hood revealed dirty shortcuts:
Some quietly dropped tail experts to inflate synthetic benchmarks;
Others locked users into proprietary file formats, locking out community GGUF releases;
Some even forced greedy decoding to fake throughput.
Sacrificing model accuracy for synthetic speed records is never acceptable in serious engineering!

---

## 2. Deconstructing tandem.cpp: True Heterogeneous CPU and GPU Synergy

The repository README outlines two basic principles:
Never ship offloaded expert weights across the bus;
Let host CPU and GPU cores compute concurrently.
Does it actually work?
It works remarkably well.

The operational pipeline relies on two coordinated mechanisms:

### 1. Hot Expert VRAM Pinning
The most frequently summoned core experts remain locked inside 24GB VRAM.
They run at full speed leveraging terabytes per second of graphics memory bandwidth.

### 2. In-Place RAM Math and Host-Side Aggregation
When gating routes a token to a cold expert residing in system RAM, the runtime never transfers weights across the PCIe bus.
Instead, it dispatches the matrix math directly to host CPU cores.
The GPU handles hot experts.
The CPU crunches cold experts in-place.
Once both compute jobs finish, the host aggregates and sums activation tensors directly in system memory.
Bus transfer overhead drops to absolute zero!

The project maintainer explicitly stated: the patch does not skip experts, nor does it distort router weights.
In practical terms, output quality aligns closely with stock llama.cpp.

However, I must debunk an unfounded myth:
Anyone claiming mathematical bit-level equivalence is completely mistaken!
I ran two hundred sampled sequences using identical random seeds.
The initial one hundred tokens matched perfectly.
Beyond that point, minor bit-level variations occasionally surfaced.
Why?
Because floating-point accumulation order between CPU SIMD and GPU CUDA cores is never identical!
Hardware floating-point addition does not follow strict associativity.
Whenever parallel tile accumulation sequences change, final truncation bits inevitably diverge.
Yet this variance remains well within industrial tolerance: benchmark score deviation across GSM8K was below 0.3 percent.

---

## 3. Workstation Benchmarks: Five-Run Medians and Hardware Costs

Do these claims hold up under real scrutiny?
Skip synthetic marketing claims and inspect measurements taken from physical hardware.

### Benchmark Testbed Specifications
- CPU: Intel Core i5-12400F six cores running at 4.4GHz all-core boost
- GPU: NVIDIA GeForce RTX 4090 24GB VRAM with power limit at 450W
- Host Memory: 64GB DDR4-3200 dual channel with 51.2 GB/s theoretical ceiling
- Operating System: Ubuntu 24.04 LTS with Linux kernel 6.8.0-45-generic
- Benchmark Model: Qwen2-57B-A14B-Instruct standard GGUF IQ3_S quantization

### Five-Run Median Benchmark Comparison Table

Test parameters were kept uniform: n_predict was 512, batch size was 2048, and temperature matched recommended defaults.

| Model Variant and Scenario | Vanilla llama.cpp Offloading | tandem.cpp Heterogeneous Patch | Observed Speedup | Operational Overhead Notes |
| :--- | :--- | :--- | :--- | :--- |
| Qwen2-57B-A14B Greedy Decoding | 29.8 tokens per second | 97.3 tokens per second | 3.26x | GPU at 210W and all six CPU cores at 96% |
| Qwen2-57B-A14B Random Sampling | 29.4 tokens per second | 96.1 tokens per second | 3.27x | Added sampling latency was under 2ms |
| Mixtral-8x7B-Instruct Q4_K_M | 33.5 tokens per second | 68.2 tokens per second | 2.03x | More active experts increased CPU workload |
| 64k Prompt Deep Prefill Phase | Baseline latency | Roughly 1.4x faster | 1.42x | Prefill bottleneck is attention not offloading |

Jumping from 29.8 to 97.3 tokens per second represents a clean three-fold leap.
To be candid, this exceeded my original expectations.
I initially doubted a budget six-core i5 could push past 60 tokens per second.
Monitoring `htop` and `perf stat` in the background revealed what was really happening:
All six physical CPU cores were saturated at 96 percent!
Host memory bus channels were operating above 75 percent bandwidth saturation!
This patch does not pull free compute out of thin air.
It trades multi-core CPU cycles and system memory bandwidth to bypass an otherwise fatal PCIe bottleneck!

---

## 4. Which Machines Should Avoid This? Unresolved Engineering Hurdles

Before rushing to install this patch, keep the following practical limitations in mind:

First, never expect a laptop or 16GB mini-PC to replicate these results!
If 40GB of weights cannot fit into physical host RAM, the operating system relies on mmap virtual memory paging.
The moment disk thrashing kicks in, generation throughput plummets to single digits alongside total system lockups.
If you lack sufficient physical memory, do not attempt this!

Second, VRAM exhaustion can still trigger OOM panics.
You must manually tune the `--moe-cache-mib` parameter to match your card, reserving at least 3GB of VRAM headroom for growing context buffers.

Third, a six-core CPU struggles when handling larger models.
If you intend to run larger MoE architectures, step up to at least an eight-core CPU paired with fast DDR5 or quad-channel memory.

Fourth, long-context prompt prefill sees modest gains of roughly 1.4x.
Do not assume your initial time to first token will triple just because token decoding did.

---

## 5. Deployment Guide: Fetching Code and Compilation Steps

The patch set is hosted on GitHub under github.com/ewhacc/tandem.cpp.
It touches only three core source files.
Fetch the code, apply the patch, and build with standard tooling:

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp && git checkout b3920
git clone --depth 1 https://github.com/ewhacc/tandem.cpp /tmp/tandem
git apply /tmp/tandem/patches/*.patch
cmake -B build -DGGML_CUDA=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j8
./build/bin/llama-cli -m ./models/qwen2-57b-a14b.gguf --n-gpu-layers 33 --moe-cache-mib 6144 -p "Write a high-performance scraper in Python" -n 512
```

Clone the repository and compile it on your hardware.
If you encounter segmentation faults on your setup, drop a note in the issues section so we can debug together!

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/tandem-cpp-llama-moe-cpu-gpu-dispatch-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

