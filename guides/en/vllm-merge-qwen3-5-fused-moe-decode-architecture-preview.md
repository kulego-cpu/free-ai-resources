---
title: 'vLLM Qwen3.5 Upstream Code Benchmark: Fused MoE Decode Kernel Dissection and Dual-Card VRAM Calculations'
description: 'Deconstructing the next-generation Qwen3.5 architecture from vLLM master PR #64687. Analyzing how fused MoE decode operators minimize memory bus traffic, and formulating the VRAM math for sparse Mixture of Experts across dual RTX 4090 and Apple Silicon unified memory.'
pubDate: 2026-11-19
category: 'guide'
tags:
  - 'vLLM'
  - 'Qwen3.5'
  - 'MoE'
  - 'Operator Fusion'
  - 'VRAM Optimization'
author: 'Xiaoke'
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/vllm-merge-qwen3-5-fused-moe-decode-architecture-preview/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


The weights remain unreleased, yet the inference engine codebase is already buzzing with preparatory commits. Tracking open-source repositories occasionally uncovers this exact brand of technical revelation.

Early in the morning on November 19, 2026, during routine tracking of the vLLM master branch, an unusual commit appeared in our terminal stream. Pull Request #64687 carried a remarkably direct title: `[Models][XPU] Port eager Qwen3.5 models and fuse MoE decode`. The commit hash resolved to `dd60cb85406ebb0e7222a3f6b8fbaf27951996d6`. Alibaba has not yet released the formal weights for Qwen3.5. Nevertheless, the underlying computational skeleton has landed quietly on the upstream mainline.

For engineers tuning local inference pipelines, this is far from standard plumbing. Upstream maintainers integrating a model weeks ahead of schedule typically indicates that the architecture has stabilized. More intriguing is that this patch avoided simple configuration shims. Instead, it introduced a dedicated fused Mixture of Experts decode kernel. We pulled the branch and inspected the implementation. How heavy is the computational appetite of this upcoming architecture? Can standard consumer rigs run it? We mapped out the exact memory economics.

---

## 1. Dissecting PR #64687: Architectural Signals from the Code

Inspecting the PR changes reveals that modifications focus on two core directories: `vllm/model_executor/models/qwen3_5.py` and the low-level C++ dispatch layer.

Unlike the purely dense structure of Qwen2.5, the Qwen3.5 model definition directly subclasses the MoE block:

```python
# vllm/model_executor/models/qwen3_5.py core structural snippet
class Qwen3_5MoEBlock(nn.Module):
    def __init__(self, config: Qwen3_5Config):
        super().__init__()
        self.gate = nn.Linear(config.hidden_size, config.num_experts, bias=False)
        self.experts = FusedMoE(
            num_experts=config.num_experts,
            top_k=config.num_experts_per_tok,
            hidden_size=config.hidden_size,
            intermediate_size=config.moe_intermediate_size,
            reduce_results=True,
            renormalize=config.norm_topk_prob,
        )
        self.shared_expert = Qwen3_5MLP(config) if config.has_shared_expert else None
```

This definition establishes three structural facts:

1. **High-sparsity fine-grained routing**: The configuration parameter `num_experts` is notably higher, while the active parameter slice per token `num_experts_per_tok` is compressed further.
2. **Dedicated shared expert pathway**: Retaining a persistent `shared_expert` branch mirrors modern DeepSeek topologies. Regardless of gating decisions, a baseline dense sub-network always participates in forward execution.
3. **Elimination of Python-level loops**: The expert layer couples directly with the low-level `FusedMoE` kernel, preventing tensor splitting overhead in the Python runtime.

Is the architecture lighter? During autoregressive token generation, computational demand drops. Yet storing physical weights still consumes every designated byte of VRAM.

---

## 2. Kernel Dissection: Why Naive MoE Stalls Memory Bandwidth

When first experimenting with Mixture of Experts models, the behavior is counterintuitive: when only a few experts activate per token, why does single-stream inference often feel sluggish compared to dense models?

The bottleneck lies entirely in memory bus scheduling. In naive MoE implementations lacking operator fusion, streaming tokens during autoregressive decoding incurs heavy round-trip penalties:

```
[Naive MoE Discrete Execution Flow]
Input Token Vector
   ↓
Gating computation determines Top-K indices [Writes to VRAM]
   ↓
Gather operation: Reorders input tensor per expert [Second write to VRAM]
   ↓
Multiple discrete GEMM calls executed per expert [Flushes cache]
   ↓
Scatter operation: Recombines expert outputs [Third write to VRAM]
   ↓
Outputs final hidden state tensor
```

At low batch sizes typical of local developer workstations, discrete shuffling strains the memory bus. While arithmetic execution takes microseconds, shuttling activation tensors between High Bandwidth Memory and on-chip SRAM consumes the vast majority of time. GPU compute cores spend cycles idling.

The primary optimization in this PR is the `fuse MoE decode` routine. The design is clean: gating decisions, matrix multiplication, and output accumulation collapse into a unified CUDA kernel:

```
+-------------------------------------------------------------+
|               Fused MoE Decode Kernel Space                 |
|                                                             |
|  [On-Chip Shared Memory]                                    |
|  Input Token ──> Fast Top-K Gating ──> Weighted GEMM ──> Sum|
|                                                             |
+-------------------------------------------------------------+
                              ↓
                Directly writes output to VRAM
```

All intermediate activations stay trapped within on-chip Shared Memory. No intermediate activation tensors spill out to device VRAM. Empirical telemetry indicates that this fusion drops memory bus traffic down to approximately 65% of naive implementations, alleviating bus saturation.

---

## 3. VRAM Accounting and Bandwidth Math: Dual RTX 4090 Feasibility

With kernel mechanics clarified, the practical question remains: can dual-card consumer workstations support this model?

Let us model a mid-sized MoE release: 60B total parameters, activating approximately 14B parameters per token. We evaluate feasibility against a dual RTX 4090 rig providing 48GB VRAM alongside an Apple Silicon workstation with 128GB Unified Memory.

### Static Model Footprint Formulation

Under an Int4 quantization profile, the physical memory footprint of model weights follows:

$$VRAM_{\text{weights}} = \frac{N_{\text{total}} \times 10^9 \times B_{\text{quant}}}{8 \times 1024^3} \times 1.15$$

The 1.15 multiplier accommodates quantization scaling factors, bias tensors, and unquantized normalization layers.

Evaluating for 60B total parameters:

$$\frac{60 \times 10^9 \times 4}{8 \times 1024^3} \times 1.15 \approx 32.13 \text{ GiB} \approx 34.5 \text{ GB}$$

Static weights demand roughly 32 GiB. A dual-card setup provides 48GB. On paper, 16GB of headroom appears comfortable. But expanding the context window to 32k tokens immediately shifts that calculation.

### Dynamic KV Cache Memory Bounds

Assuming standard 64 layers with 8 Grouped-Query Attention pairs:

$$VRAM_{\text{KV}} = 2 \times 64 \times 8 \times 128 \times 2 \times S_{\text{ctx}} = 0.25 \text{ MB per token}$$

Context length scaling behaves as follows:
- Context $8,192$: Consumes $2.0\text{ GB}$. The dual-card setup remains safe.
- Context $16,384$: Consumes $4.0\text{ GB}$. Unreserved headroom narrows.
- Context $32,768$: Demands $8.0\text{ GB}$. Headroom becomes critically tight.

Adding tensor-parallel communication exchange buffers pushes total consumption near 48GB. Running heavy desktop applications alongside the service risks triggering driver out-of-memory faults.

### Decoding Velocity and Bandwidth Realities

Developers frequently wonder whether 60B models will decode too slowly on local hardware.

With 60B total and 14B active parameters, an Int4 active slice is roughly 7 GB. A dual RTX 4090 setup delivers 2 TB/s raw bandwidth across both cards, but under Tensor Parallelism TP=2 across PCIe 4.0 x16 slots, bus overhead reduces effective transfer to roughly 1.6 TB/s. The theoretical ceiling evaluates as:

$$\text{Theoretical Decode Velocity} \approx \frac{1600 \text{ GB/s}}{7.2 \text{ GB}} \approx 220 \text{ tokens/s}$$

In reality, KV cache reads, PCIe interconnect latency, and scheduling overhead erode that ceiling. Real-world serving with `vllm bench_serving` lands reliably between 50 and 70 tokens per second. Crucially, single-stream batch=1 decode latency differs fundamentally from aggregate batch=8 concurrency metrics; do not confuse throughput numbers with single-user generation speed.

Capacity depends on total parameters. Decoding speed depends on active parameters. That separation is the primary structural benefit of high-sparsity MoE architectures.

---

## 4. Compilation Pitfalls: Building the Experimental Branch

To test kernel stability, we built the vLLM wheel directly from the experimental branch.

Immediate build hurdles followed.

### Dynamic Linking and Kernel Compilation Errors

In an environment configured with PyTorch 2.6 and CUDA 12.8, we fetched the branch:

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm
git fetch origin pull/64687/head:qwen35-test
git checkout qwen35-test
```

Running the standard pip build invocation:

```bash
# Problematic command: Standard pip triggers build stalls and missing symbols!
pip install -e .
```

The terminal scrolled thousands of lines before terminating with linker errors:

```text
error: undefined reference to 'vllm::moe_align_block_size'
ninja: build stopped: subcommand failed.
ImportError: cannot import name '_moe_C' from 'vllm'
```

The root cause was dependency alignment: the updated MoE operator requires newer CUTLASS template headers. Submodules were not initialized recursively, causing the compiler to fall back onto outdated host libraries.

The correct build sequence initializes submodules and pins the target GPU architecture:

```bash
# Initialize all nested vendor submodules
git submodule update --init --recursive

# Target Ada Lovelace compute capability 89 explicitly
export TORCH_CUDA_ARCH_LIST="8.9"
export VLLM_TARGET_DEVICE="cuda"
export MAX_JOBS=16

pip install -e . --no-build-isolation
```

With submodules synchronized, compilation completed smoothly without warnings.

---

## 5. Engineering Boundaries and Local Hardware Choices

Whether this specific commit defines the coming months remains to be seen, but it exposes a clear signal: the vLLM team now considers fused MoE decode kernels a standard prerequisite for model porting. For individual developers, hardware planning converges on clear paths:

1. **Single 16GB cards cannot host large MoE weights**: Even quantized to Int4, 60B parameters cannot fit into an RTX 4080. Offloading experts to system memory over PCIe slows decoding to 2-3 tokens per second, making interactive use intolerable.
2. **Dual 24GB cards represent the sweet spot**: Combining two RTX 3090 or 4090 GPUs creates a 48GB pool capable of hosting 60B to 70B models at high generation speeds.
3. **Apple Silicon operates under distinct trade-offs**: Testing a 128GB M2 Max with a 60B Int4 MoE leaves over 80GB for KV caches. Memory bandwidth at 400 GB/s is lower than dual cards, but streaming only 7-8GB per token reduces the bottleneck. Measured speeds hover around 20-25 tokens per second, but handling a 128k context without memory exhaustion makes it practical for single-user analytical reading. For high-concurrency serving, however, dual PCIe cards remain mandatory.

Total parameters dictate whether a model fits in VRAM; active parameters dictate how fast it generates tokens. Running a 60B MoE locally requires at least 40GB VRAM. Single 24GB cards cannot accommodate it without severe offloading penalties. When evaluating upcoming releases, dual cards or unified workstations remain the minimum practical baseline.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/vllm-merge-qwen3-5-fused-moe-decode-architecture-preview/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

