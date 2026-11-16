---
title: "Breaking a Coding Assistant by Jacking max_num_seqs from 4 to 128"
description: "Field notes debugging a dual-node 8x RTX 4090 cluster on vLLM: continuous batching preemption storm, PCIe All-Reduce bottlenecks, INT4 indentation collapses, and speculative decoding temperature decay."
pubDate: 2026-11-16
author: "Xiao Ke"
category: "Model Deployment & Evaluation"
tags: ["vLLM", "RTX4090", "Inference Engineering", "Continuous Batching", "PD Disaggregation", "Throughput Tuning"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/llm-inference-efficient-frontier-latency-throughput-disaggregation-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


Last winter I took over an inference cluster powered by two nodes with eight RTX 4090 cards.

The Grafana dashboard looked impeccable: VRAM hovered around 70%, and GPU util consistently floated near 90%.

I was feeling quite pleased with myself back then, thinking all eight GPUs were being squeezed for every drop of compute.

Until developers on the frontend integration team started blowing up my chat group.

The cursor stopped dead on line three. That was the very first complaint.

My immediate reaction was that the WebSocket connection had dropped, so I asked them to capture a HAR log.

The packet capture revealed that the network connection remained fully alive; packet drop rate sat at absolute zero; yet the initial output token simply refused to land.

One engineer timed it on his phone: from hitting Enter to seeing the first token appear on screen took long enough for him to walk over to the breakroom and brew instant coffee.

Did the datacenter switch fail? I spent two full days inspecting network infrastructure.

Capturing packets, testing NICs, auditing optical transceivers... every bit of it was a waste of time.

Only on the third night did I finally discover where the actual deadlock lived.

## The Mirage on the Dashboard: 480ms Mean vs 2s at p99

To probe the true capacity ceiling of this hardware, I built a benchmark using Locust.

There was zero fluff in the harness.

I spawned 64 asynchronous coroutines, feeding prompt snippets from HumanEval concurrently, capping generation at 512 tokens, and sustained the load for ten full minutes to capture steady-state numbers.

In the launch flags, I cranked concurrency straight to maximum:

```bash
# Launch vLLM with full concurrency batching
vllm serve Qwen/Qwen2.5-Coder-32B-Instruct \
  --tensor-parallel-size 4 \
  --max-num-seqs 128 \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.92
```

The headline metrics looked wildly seductive: raising `max_num_seqs` from 4 to 128 drove aggregate system throughput from 320 tok/s all the way to 1280 tok/s.

A clean 4x gain in aggregate tokens. Could anyone possibly call that a failure?

Staring only at the primary monitoring panel, you could not spot a single defect.

What I neglected at the time was a fundamental detail: the default curve rendered on the dashboard showed mean time to first token [TTFT].

The mean sat gracefully at 480ms, looking totally composed.

When I finally pulled percentile distributions from Prometheus, cold sweat broke out: p90 climbed to 1.2 seconds, and p99 blew completely past 1.95 seconds.

Opening the container standard output logs, the terminal was spamming line after line of bright yellow warnings:

```text
WARNING [vllm.core.scheduler] Sequence group 1422 is preempted by PreemptionMode.RECOMPUTE.
```

The warnings flooded the console.

Four 24GB RTX 4090 cards per node yield 96GB of aggregate physical VRAM.

Loading Qwen2.5-Coder-32B in native bfloat16 consumes roughly 64GB strictly for model weights.

The VRAM headroom left for KV cache totals 24GB at best; this accommodates around 80,000 to 90,000 contextual tokens.

Once concurrency climbs to 128, as soon as each sequence exceeds six or seven hundred tokens of history, the KV cache pool empties out.

The memory reserve evaporated.

To prevent CUDA OOM crashes, the vLLM scheduler is forced into recomputation preemption.

It evicts older sequence caches completely; once earlier sequences finish, it restarts prefill from scratch.

Those frontend engineers waiting two seconds were the victims caught right as the scheduler evicted their sessions from VRAM.

Who could tolerate coding with that kind of stutter?

That night I rolled `max_num_seqs` back to 24 in a rolling update, ran the load test again, and p99 stabilized at 340ms.

Nobody complained in the chat group anymore.

As for finding a magical balance at 128 concurrency that never evicts cache while maximizing tokens, I have still not found one on this rig; I will log it here until we test on larger memory servers.

## The Missing NVLink Bridge: PCIe Bottlenecks Under Tensor Parallelism

When tuning parallelism strategies, I made another naive assumption.

If one card runs slowly, spreading compute across multiple cards should intuitively make it faster, right?

Countless architecture articles claim that Tensor Parallelism [TP] compresses latency to the floor, often citing several hundred gigabytes of NVLink bandwidth as proof.

On consumer desktop cards, that premise does not exist.

Starting with the Ada Lovelace generation, Nvidia physically severed the NVLink bridge from 40-series gaming cards.

Consumer cards have zero NVLink fingers.

Running topology inspection on the host:

```bash
nvidia-smi topo -m
```

The resulting matrix showed pure `SYS` across all entries.

This means inter-card communication cannot traverse high-speed peer-to-peer links; data must bounce through the CPU root complex, motherboard PCIe 4.0 x16 slots, and host memory.

Bidirectional theoretical bandwidth peaks at merely 64GB/s.

Worse, under stock Linux drivers lacking hardware P2P enablement, inter-card data bounces back through host system RAM by default.

Back then, seeking raw speed on the 32B model, I forced TP to 4 across four local cards.

The benchmarked inter-token latency [ITL] regressed from 22ms back to 58ms.

Compute cores quadrupled, yet token emission slowed by more than half. I was utterly baffled.

I captured a forward pass trace using Nsight Systems, and the truth became glaringly clear.

After every attention block and feed-forward layer in the Transformer stack, all four GPU workers must execute an `all_reduce` collective communication call.

Kernel computation took 1.2ms, but stalling on PCIe bus synchronization ate 3.8ms.

CUDA cores sat completely idle waiting for bus transfers. Routing everything over cheap bus lanes, how could it not choke?

Without high-speed fabric, running high TP degrees on consumer silicon means slamming lightweight buses with brutal collective communications.

I subsequently re-architected the node into pairs running TP=2; between the two pairs, traffic routed via HTTP request-level data parallelism. Per-token generation latency dropped back to 26ms immediately.

Anyone working with 4090s must ignore hyperscaler supercomputer recipes; calculate the physical bandwidth of your motherboard slots instead.

## The Quantization Cliff: 28-Point HumanEval Drop on Broken Indentation

When VRAM tightens, quantization is the first lever engineers pull.

Many articles depict quantization as frictionless magic: cut VRAM in half, double generation speed, and benchmarks drop by tenths of a point.

Can you really trust that marketing narrative?

In strict coding environments, reality looks entirely different.

Under identical test harnesses, we evaluated Qwen2.5-Coder-32B across three quantization tiers on HumanEval pass@1:

```text
HumanEval 50-Problem Automated Code Generation Benchmark [commit: 4e7b8a1]
- Native FP16 Baseline: 82.0%
- FP8 Dynamic Quant: 80.4% [micro-drop of 1.6%]
- Standard INT4 Format: 54.2% [cliff collapse of 27.8%]
```

Stepping down to FP8, whether on static weights or by enabling `--kv-cache-dtype fp8`, proved extraordinarily solid.

VRAM footprint fell by half.

KV cache slots doubled.

With memory bandwidth pressure relieved, token output speed jumped by almost 80%.

Once you cross FP8 down into generic INT4 weights, logical reasoning collapses off a cliff.

A 54.2% pass rate means failing every other programming task.

Was saving those 6GB of VRAM worth the destruction?

I analyzed the diffs across all 20 failed test cases individually.

Exactly 17 failures collapsed directly on nested indentation.

Whenever Python logic nested deeper than three levels of loops or conditionals, the INT4 model turned four spaces into six spaces, or terminated scopes prematurely in nested try-except blocks.

The remaining 3 errors stemmed from quoted f-strings, where the model fell into infinite token loops repeating identical lines.

Code like that fails basic flake8 linting in CI before tests even run.

Saving 6GB of VRAM cost us the entire utility of the assistant.

If your workload involves summarizing internal docs or vector reranking, INT4 savings make economic sense.

For real-time developer pair programming, FP8 is the absolute floor.

Unless upstream weights undergo distilled quantization using modern microscaling formats like NVFP4 or MXFP4, running raw INT4 on long code contexts is asking for trouble.

## Disaggregating Prefill and Decode: Why Mixed Serving Stalls Output

If single-node tuning feels like playing whack-a-mole, what problem does P/D disaggregation actually solve?

Its physical premise is straightforward: segregate conflicting computational workloads across distinct physical nodes.

The prefill phase is pure compute-bound matrix multiplication; it hinges on Tensor Core TFLOPS, swallowing thousands of input tokens in single parallel forward passes.

The decode phase is pure memory-bandwidth-bound autoregression; it hinges on memory bandwidth in GB/s, reading the entire parameter set and accumulated KV cache for every emitted token.

Shoving both phases onto the same cards is the root cause of cursor freezes.

While smoothly generating tokens at 50 tok/s for one user, another client hits the node with a 20,000-line OpenAPI schema.

The scheduler halts decoding to compute attention projections for that massive input.

Active decode streams are suspended instantly.

To the user watching the screen, the cursor freezes for a second and a half.

Who wants to work with a frozen cursor?

P/D disaggregation resolves this by erecting two distinct clusters:

```text
Client Requests
    │
    ▼
[API Load Gateway]
    │
    ├─► [Prefill Specialized Cluster]: Dense compute cards swallowing prompt matrices
    │         │
    │         ▼ [KV Cache transfers via RDMA / High-Speed Fabric]
    └─► [Decode Specialized Cluster]: High memory bandwidth nodes [H100 3.35TB/s] emitting tokens
```

Once prefill finishes on compute nodes, intermediate KV states route across internal networks to decode nodes, which emit tokens without interruption.

In multi-turn agentic workflows, physical segregation eliminates more than 60% of p99 latency jitter.

The trade-off is equally real: you must manage distributed KV network transfers, and provision 20% spare capacity for traffic skew.

Large tech companies love this pattern, but it belongs to high-volume operations.

For teams running small clusters, avoid distributed KV transfer traps; use chunked prefill to tame input spikes instead.

## The Reality of Speculative Decoding: Acceptance Rates vs Temperature Decay

The final technique capable of pushing the efficiency frontier outward is speculative decoding.

Many engineers try speculative decoding in conversational chat and abandon it in frustration.

Open-ended queries are divergent; small draft models guess target outputs with acceptance rates hovering near 30%.

Every misprediction forces target model verification rollbacks, wasting compute and memory cycles.

Is it not counterproductive?

In controlled code completion, data regularity changes the equation completely.

Code follows strict syntax: class declarations, package imports, closed brackets, and uniform indentation.

Testing a lightweight Qwen2.5-Coder-0.5B draft model alongside the 32B target:

Vocabulary tokenizers must maintain exact physical alignment, or draft predictions fail instantly.

With temperature locked down to 0.2, mean acceptance length reached 3.8 to 4.1 tokens.

Instead of four sequential forward passes, the target model verifies four tokens in one batch forward pass.

Skipping three full VRAM round trips unlocks real hardware efficiency.

Do not celebrate prematurely.

Speculative decoding exhibits a brutal temperature decay curve in code generation.

At temperature 0.2, acceptance holds around 70%, nearly doubling perceived speed.

The moment a developer raises temperature to 0.7 seeking creative code, acceptance collapses to 1.6 tokens.

Worse is transient memory amplification.

During verification, the target model processes K speculative draft branches simultaneously, expanding effective batch size by K times.

This sudden spike pushes tight VRAM pools toward the brink.

If base KV cache usage already sits near 90%, enabling speculative decoding triggers earlier preemption recomputations or hard OOM crashes.

Is turning off the draft head not cleaner in high-temperature setups?

Our dual-node cluster currently runs two production workers.

We pinned `max_num_seqs` to 24; enabled FP8 weights; left dynamic FP8 KV cache on; and disabled speculative decoding for now, because new endpoints enforce temperature 0.75, which wastes VRAM.

Building an intelligent proxy that toggles draft heads dynamically based on request temperature remains on my backlog.

If you maintain an internal coding assistant, pull last week's p99 metrics; see what your engineers are actually waiting for when staring at their screens.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/llm-inference-efficient-frontier-latency-throughput-disaggregation-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

