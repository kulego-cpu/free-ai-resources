---
title: "Squeezing a 26B Looped MoE into 5.4GB VRAM: Hands-On LookAtMyQuant 2 Compression of Inkling-2-Mini"
description: "Deconstructing ShinMegamiBoson's open-source quantization compiler LookAtMyQuant 2. Benchmarking Fisher information bit allocation and trellis encoding to fit the 26.4B physical parameter Inkling-2-Mini into 5.38GB memory, debugging Apple Silicon Metal kernels and mapping accuracy limits."
pubDate: 2026-11-22
author: FreeAI Hub 站长
category: 架构剖析
tags:
  - LookAtMyQuant
  - Inkling-2-Mini
  - Looped Architecture
  - Model Quantization
  - VRAM Optimization
  - Apple Silicon
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/lookatmyquant-2-inkling-2-mini-5gb-vram-quantization-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


A few days ago Thinking Machines Lab quietly pushed weights for Inkling-2-Mini to open-source hubs.

Crafted by former OpenAI engineering talent, the model features an unusual architectural skeleton. It employs a looped mixture of experts topology where total physical parameters reach 26.4B, while active parameters per token remain around 3.4B. Several intermediate transformer layers are looped through forward computation three consecutive times, squeezing deeper reasoning capabilities without expanding physical weight footprints.

The concept is compelling. Weighing the native FP16 release on disk, however, clocks in at 18.89GB.

Workstation cards with 8GB VRAM or lightweight MacBook Airs with 8GB unified memory cannot even load the base tensors. They halt right at the doorstep.

Just as I prepared to shut down for the night, Hacker News featured an active thread. Developer ShinMegamiBoson open-sourced the LookAtMyQuant compiler toolkit, demonstrating a deployment that crushed Inkling-2-Mini into 5.4GB of memory. Average weight precision plummeted to 1.63 bits per weight.

What does 1.63 bpw mean in practice? Standard GGUF formats pushed down to IQ2 or IQ1 frequently collapse into incoherent gibberish. Can 5.4GB produce functional responses, or is this another theoretical synthetic benchmark?

I cloned the toolkit locally and ran exhaustive hardware benchmarks across two separate machines.

---

## 1. Aggressive Compression Mechanics: Why Standard Quantization Collapses at 1.63 Bits

Many engineers wonder: why does AWQ or GPTQ remain stable around 4-bit, yet degrade into complete nonsense once pushed below 2-bit?

The reason lies in the extreme variance of individual weight significance across matrices.

Conventional uniform quantization rounds all tensors across a fixed grid. At higher bit widths ample redundancy masks quantization noise. Once precision drops below 2-bit, the critical minority of weights maintaining attention trajectory are wiped out. The reasoning engine breaks immediately.

LookAtMyQuant 2 adopts a compiler-driven pipeline. Inspecting its codebase reveals two core mechanisms: Fisher information dynamic bit allocation and trellis state-graph optimization.

### 1.1 Fisher Information Sensitivity Allocation

LAMQ rejects the assumption that all transformer layers are created equal.

The compiler first passes short calibration sequences from C4 and code repositories to compute the second-order curvature of the loss surface with respect to every weight block, formally known as the Fisher information matrix.

High curvature signifies that even minimal precision perturbation will trigger severe semantic drift. Flat curvature regions, by contrast, tolerate aggressive trimming with negligible output disruption.

Based on this curvature profile, LAMQ partitions the precision budget dynamically:

Token embedding layers, output projections, and early attention matrices receive strict 3-bit or 4-bit protection;

Rarely invoked micro-expert feed-forward layers inside the recurrent loop are aggressively pruned down to 1-bit or 1.5-bit.

Weighted across the entire network, the global average precision settles at 1.63 bpw.

### 1.2 Trellis Encoding and Layer-Wise Distillation Recovery

Rather than relying on nearest-neighbor rounding, LAMQ implements trellis state-path search.

The algorithm executes dynamic programming across a trellis graph to locate weight quantization paths that cancel out cumulative rounding errors globally. Once encoded, an iterative post-training distillation pass fine-tunes layers against original high-precision outputs to restore representational fidelity.

On paper it looked elegant. Pulling it up for a live run, memory bandwidth hit first. Once bitwise unpacking kicked in, latency piled up layer over layer. The rest was pure grunt work.

---

## 2. Local Environment Setup and Metal Kernel Debugging

Physical testing was conducted across two distinct hardware configurations:

First, a single NVIDIA GeForce RTX 4060 8GB setup running driver 550.127.05 and CUDA 12.4.

Second, an Apple Silicon M3 machine with 16GB unified memory running macOS 15.1.

### 2.1 Source Build and Header Missing Exceptions

LookAtMyQuant provides dedicated Metal kernels tailored for Apple Silicon. Installing the CLI toolchain via pip:

```bash
pip install git+https://github.com/ShinMegamiBoson/lookatmyquant.git
```

Fetching the pre-compiled quantized weights and initiating a test prompt:

```bash
lamq generate shinmegamiboson/Inkling-2-Mini-LookAtMyQuant-5.4GB \
  --prompt "Write a quicksort in Python."
```

After loading several tensor shards, the terminal aborted with a Metal compilation error:

```text
MetalCompilerError: Compilation failed: unknown type name 'half4x4'
  Device::newLibraryWithSource failed to compile custom kernel: lamq_trellis_unpack.metal
```

Execution stalled during the runtime just-in-time compilation of unpacking shaders.

Reviewing macOS 15 changelogs revealed that Xcode Command Line Tools relocated default Metal header search paths under Sequoia. Without an explicit path, the compiler failed to resolve built-in compact matrix structs.

The fix was straightforward. Injecting the toolchain path into the shell environment:

```bash
export METAL_XCODE_TOOLCHAIN_PATH="/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain"
```

Re-running the launch command resolved the issue.

The terminal logged shader cache creation and transitioned into generation mode without warnings.

### 2.2 Physical Memory Footprint Under Load

Profiling actual memory utilization under load yielded the following metrics:

- Static base weights consumed 5.38GB of memory.
- Under a 4,096 token context setting, KV cache reserved approximately 1.15GB.
- Runtime and framework overhead occupied roughly 0.35GB.
- Peak system memory stabilized at 6.88GB.

On the 8GB RTX 4060, memory consumption avoided out-of-memory crashes. On the 16GB Mac, the operating system generated zero swap activity.

The physical footprint held strictly within the promised 5.4GB boundary.

---

## 3. Real-World Throughput: The Latency Penalty of Recurrent Depth

Memory constraints were solved, but what about throughput?

Many assume a 5.4GB model will run with blisteringly fast latency. After running the numbers, I went silent. It fit into 5.4GB, but the speed was far from gentle.

Benchmarking varying prompt lengths across five rounds yielded median performance:

| Context Length | Prefill Throughput | Decoding Throughput | Peak VRAM Footprint |
| :--- | :--- | :--- | :--- |
| 512 Tokens | 74.2 tokens/s | 23.8 tokens/s | 6.42GB |
| 2,048 Tokens | 68.5 tokens/s | 22.4 tokens/s | 6.68GB |
| 4,096 Tokens | 59.1 tokens/s | 20.6 tokens/s | 6.88GB |
| 8,192 Tokens | 46.3 tokens/s | 18.2 tokens/s | 7.35GB |

Why does decoding throughput plateau between 20 and 23 tokens per second?

This is the non-negotiable physical price of looped architectures.

While weights occupy only 5.4GB, generating each individual token requires recurrent layers to compute three consecutive forward passes. The hardware computes an effective forward depth comparable to a massive 70-layer dense model.

Furthermore, unpacking trellis codes demands bitwise operations in compute shaders, adding noticeable dequantization overhead.

It functions reliably, but it is no speed demon. For document summarization or code drafting it is entirely usable; expecting it to power low-latency streaming voice assistants will disappoint.

---

## 4. Empirical Accuracy Degradation Testing

Marketing claims mean nothing without validating real generation quality.

I tested the model across three distinct evaluation tiers to evaluate 1.63 bpw fidelity.

### 4.1 Core Code Synthesis and Logical Explanations

The evaluation prompt requested an asynchronous rate limiter with strict type hints.

The resulting code was clean and well-structured. It imported asyncio.Semaphore correctly, implemented proper type annotations, and handled basic exception edge cases without syntax regressions. For standard software development, core reasoning remained robust.

### 4.2 Numerical Reasoning and Mathematical Derivations

The prompt was switched to a multi-step matrix computation requiring strict formatted outputs.

Here the limitations of 1.63 bpw became evident.

During the third step of calculation, the model introduced a transposed sign error. Re-running the generation three times, schema alignment accuracy dropped from 96% on 4-bit baselines to 88%.

To be blunt, 1.63 bpw simply cannot handle hardcore numerical reasoning. Once matrices go deep, signs drift. Expecting reliable structured output falls apart after a few iterations.

---

## 5. Hardware Recommendations and Architectural Takeaways

Having concluded testing, who is this setup suitable for?

First, developers constrained to 8GB graphics cards or 8GB unified memory Macs who insist on running 26B class reasoning locally will find LAMQ 2 one of the few functional avenues available.

Can it actually fit? It unequivocally fits.

Second, engineers equipped with 16GB or 24GB VRAM should stick to standard 4-bit quantizations. There is no justification for accepting 1.63 bpw trade-offs when memory permits higher precision.

Third, look past synthetic benchmark hype. Storage is saved. The cost? Every token generated forces intermediate looped layers to compute three extra times. Whatever space you saved, you pay back in time with interest.

Whether upstream compilers integrate optimized hand-written assembly kernels remains to be seen. Grounding your pipeline in hardware physics and memory arithmetic remains far wiser than chasing unverified compression claims.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/lookatmyquant-2-inkling-2-mini-5gb-vram-quantization-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

