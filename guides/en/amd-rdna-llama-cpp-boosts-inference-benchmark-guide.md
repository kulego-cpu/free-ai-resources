---
title: "Testing RDNA3 Patch Branch on 7900 XTX: Qwen3.8-27B Q6_K_L Decode Speeds from 35t/s to 55t/s"
description: "Generic inference frameworks often compromise per-hardware efficiency for cross-platform compatibility. Testing on a single Radeon RX 7900 XTX with 16k context, the llama-cpp-rdna-boosts branch optimizes assembly kernels for RDNA3 Wave32, raising token decoding from 35t/s to 55t/s. Here is an honest review of the version chain, topology bus penalty, regression results, and version pinning trade-offs."
pubDate: 2026-11-27T14:30:00.000Z
author: "Xiao Ke · FreeAI Dog Core Dev Group"
category: "Hardware & Edge Compute"
tags:
  - "AMD ROCm"
  - "RDNA3"
  - "llama.cpp"
  - "Local Inference"
  - "GPU Tuning"
  - "Kernel Optimization"
heroImage: "/images/blog/free-ai-hero-card.webp"
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/amd-rdna-llama-cpp-boosts-inference-benchmark-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


The graphics card cooling fan suddenly screeched.

Rpm surged straight to twenty eight hundred.

The terminal token generation rate was stubbornly stalled at 35 tokens per second.

Is that slow? Not completely unusable.

Yet staring at this Radeon RX 7900 XTX with 24GB VRAM and 96 compute units, I kept wondering: why did vanilla llama.cpp run like this?

Most stream processors sat idle. Memory controller read bandwidth barely touched thirty percent of theoretical limits.

First, let us establish the exact test boundaries: single card, single model, 16k context window, host running ROCm 6.2.4. Numbers presented here reflect solely this specific rig and cannot be casually extrapolated to all hardware.

I switched to the `llama-cpp-rdna-boosts` branch by stew675.

Same model slice: Swift-1.5-Qwen3.8-27B-GGUF.

Same quantization precision: Q6_K_L.

Token decode generation instantly climbed to 55 tokens per second.

Prefill prompt processing showed an even clearer leap: rising from 712 to 1318 tokens per second.

Did replacing kernels silently degrade precision? Did output text quality collapse?

Staring at the logs, I decided to pull apart every single detail.

---

## The Price of Generalist Abstraction and the Rise of Cheat Engines

Local inference runtimes have arrived at an awkward fork in the road.

Flagship projects like llama.cpp serve as broad multi-hardware model zoos.

They must juggle CPU, Apple Metal, NVIDIA CUDA, AMD ROCm, Intel SYCL, Vulkan, and more.

The codebase naturally fills with layers of generalized abstraction.

Merging deeply specialized assembly kernels into main branch proves extraordinarily difficult.

Upstream maintainers must prioritize cross-platform stability; touching low-level memory layouts risks segmentation faults on older architectures. While upstream code updates rapidly, execution efficiency on specific consumer GPUs often hovers at a lukewarm baseline.

Naturally, independent engineers began carving their own paths.

Over the past six months, dedicated niche engines sprouted everywhere.

Some leverage Swift and Zig to exhaust Apple Silicon unified memory; others tune static kernel fusion specifically for mobile NPUs.

Yet within this sea of bespoke forks, several questionable trends emerged.

The most glaring involve hyper-aggressive forks chasing inflated benchmarks on social media, such as Strata or Basalt.

They claim to run hundred-billion parameter mixture-of-experts models at astonishing speeds on 16GB cards.

Is there really such a thing as a free lunch in compute?

Peeking into their code reveals the shortcut: to inflate throughput numbers, they quietly corrupt the router logic of MoE models.

When a model determines a token needs an expert residing in system RAM or NVMe storage, the engine overrides the weights and forcefully shunts the token to whatever expert happens to sit in VRAM.

Tokens fire out like a machine gun.

The generated answers, however, turn into complete gibberish with fractured context and broken reasoning.

Trading model thinking quality for hollow benchmark speed offers zero tangible value to real engineering workflows.

Would you dare entrust production code generation to an engine with hijacked routing logic?

I certainly would not.

---

## Mapping the Patch Footprint: Wavefront Scheduling and On-Chip Cache

The actual modification footprint of `llama-cpp-rdna-boosts` is refreshingly narrow.

The quantization tables were untouched.

The mixture-of-experts router was untouched.

The token samplers were untouched as well.

Modifications concentrate in two specific hotspots: the Wave32 path for GEMM matrix multiplication, and local on-chip SRAM staging during attention forward passes.

This stems from architectural differences across AMD hardware lines:

AMD datacenter CDNA chips, such as the MI100 and MI300 series, run on traditional Wave64 wide wavefronts;

Consumer RDNA GPUs, ranging from RX 6000 to RX 7000 series, feature native Wave32 hardware with dual-issue ALUs.

The official ROCm backend largely relies on generic HIP compatibility layers.

Many GEMM kernels structure threadblocks for wider wavefronts.

While that works well on enterprise accelerators, on an RX 7900 XTX it triggers severe wavefront dispatch latency: memory controllers stall waiting for fetches while compute cores sit starved of operands.

The `llama-cpp-rdna-boosts` patchset rewrites the core GEMM kernels.

Wavefront width is locked strictly to native Wave32.

Simultaneously, the attention forward pass redesigns data staging through local memory channels. Intermediate tensors linger inside on-chip scratchpad SRAM adjacent to compute units, avoiding constant round trips back to global VRAM.

Hardware monitoring plots reveal the immediate payoff:

Board power consumption climbed from barely over two hundred watts up to the rated power ceiling;

VRAM read throughput formed a tall, stable plateau.

---

## Grounded Test Rig and Containerized Deployment

Any benchmark omitting full environment details and version chains is deceptive.

My testing rig utilizes an external eGPU expansion enclosure setup:

- GPU Core: AMD Radeon RX 7900 XTX with 24GB GDDR6 VRAM, architecture codenames Navi 31 and gfx1100.
- Host Drivers: AMD ROCm 6.2.4 userland packages running on Linux kernel 6.11.
- Interface Topology: Thunderbolt 4 eGPU enclosure negotiating physical PCIe 4.0 x4 link, recognized as an isolated GPU render node.
- Topology Bus Penalty: Thunderbolt 4 provides approximately 3.2 GB/s payload throughput. During token decode, model weights permanently reside in VRAM, making bus bandwidth penalty virtually zero; during prompt prefill, transporting massive context tokens from host to device encounters bus friction, yielding roughly an 8% prefill throughput penalty compared to direct PCIe slot mounting.
- Evaluation Model: Swift-1.5-Qwen3.8-27B-GGUF released by ukisai, quantized to Q6_K_L, total weight footprint around 21.4 GB.
- Speculative Decoding: Enabled with speculative parameter `--draft 2`.

One versioning detail warrants clarification:

The container image tag reads `server-rocm-10.1`. Here, 10.1 represents the author automated CI pipeline build revision, not an official AMD ROCm release. Inside the container sits a ROCm 6.2 runtime compiled targeting gfx1100, which interfaces cleanly with host ROCm 6.2.4 drivers.

Pull the image:

```bash
podman pull ghcr.io/stew675/llama-cpp-rdna-boosts:server-rocm-10.1
```

Mount the model storage and pass through the GPU device nodes:

```bash
podman run --rm -it \
    --network=host \
    --device /dev/kfd \
    --device /dev/dri \
    --group-add video \
    -v "$HOME/models:/models" \
    "ghcr.io/stew675/llama-cpp-rdna-boosts:server-rocm-10.1" \
    /llama-server \
    -m /models/Swift-1.5-Qwen3.8-27B-Q6_K_L.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    -ngl 99 \
    -c 16384 \
    --ubatch-size 512 \
    --flash-attn \
    --draft 2
```

A practical trap encountered during rollout:

On my initial invocation, I omitted `--flash-attn`. The terminal promptly printed a fallback warning:

```text
ggml_cuda_op_flatten: flash attention disabled, falling back to unoptimized rocm attention kernel
```

Token decode crawled at only 38 tokens per second.

Only when explicitly passing `--flash-attn` did the tailored Wave32 attention kernel actually engage.

At that moment, `rocm-smi` captured the following active status:

```text
======================= ROCm System Management Interface =======================
=================================== Temperatures ===================================
GPU[0] : Temperature (Edge): 68.0c | (Junction): 82.0c | (Memory): 74.0c
====================================== Usage =======================================
GPU[0] : GPU use: 98% | Memory use: 94% (22630MB / 24560MB)
====================================== Power =======================================
GPU[0] : Average Graphics Package Power: 312.0W
====================================================================================
```

---

## Performance Ledger and Verification Cross Checks

Here is the empirical test ledger.

On the identical hardware rig, comparing vanilla upstream llama.cpp b4120 against the patched container image across five consecutive runs, taking median values:

| Engine Version | Prompt Prefill Speed | Token Decode Speed | Resident VRAM Footprint | Code Regression Pass Rate |
| :--- | :--- | :--- | :--- | :--- |
| Upstream Vanilla llama.cpp b4120 | 712 t/s | 35.2 t/s | 21.8 GB | 100% |
| rdna-boosts Specialized Patch | 1318 t/s | 55.4 t/s | 22.1 GB | 100% |

Looking at the plain empirical facts:

Prefill prompt processing rose from 712 to 1318 tokens per second;

Token generation speed jumped from 35.2 to 55.4 tokens per second;

Resident VRAM rose slightly from 21.8GB to 22.1GB, an added 300MB attributable to static scratch workspaces allocated for Wave32 wavefronts.

Crucially: these figures reflect strictly this single card, Q6_K_L quantization, and 16k context window. Extrapolating them indiscriminately to all models or extreme context lengths would be irresponsible.

On output quality, I executed a test suite of 50 challenging Python algorithmic problems and type constraint tasks:

Both engines yielded byte-identical abstract syntax trees with a 100% pass rate;

Across a two-hundred query technical evaluation set, perplexity variation remained below 3e-4.

Excluding subtle edge degradation entirely will require deeper logits differential sampling over 32k contexts.

Nonetheless, for current daily development, no semantic drift occurred.

---

## Operating Boundaries and the Cost of Version Pinning

This branch is best suited for version-pinned operation.

Changes remain tightly bounded to RDNA3 GEMM and cache staging, without touching routing or quantization, keeping troubleshooting straightforward.

The trade-off, however, is unmistakable:

Patches bind directly to specific upstream commits, such as tag b4120.

Upstream llama.cpp re-architects core internals almost weekly. When upstream overhauls KV cache memory pools or alters batch scheduling, out-of-tree patches demand painful rebasing.

For production or daily engineering environments, I recommend three safeguards:

First, pin verified container digests or tags, rather than tracking latest builds blindly;

Second, if running via a direct PCIe 4.0 x16 motherboard slot, prefill speeds will likely gain an extra 8% to 10% over eGPU enclosures, making dedicated workstation deployment worthwhile;

Third, whenever model architecture updates break custom patches, fall back gracefully to the rock-solid upstream main image.

Would you maintain an independent patch container image on your workstation just to gain an extra twenty tokens per second?

As upstream engines march forward, how long can hardware-specific out-of-tree patches maintain practical vitality?

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/amd-rdna-llama-cpp-boosts-inference-benchmark-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

