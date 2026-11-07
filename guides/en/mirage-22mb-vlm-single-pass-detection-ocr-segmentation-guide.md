---
title: "Single-Pass Vision Model under 22MB for Detection and OCR: How Mirage Works"
description: "Compressing multi-task vision into a 22MB footprint while achieving 74ms latency on mobile NPUs. A technical deep dive into Mirage by Cactus Compute, breaking down its 51.3M parameter allocation, Hadamard convolution channels, dedicated spatial heads, and the reality of 5.8-second CPU inference without accelerators."
pubDate: 2026-11-07
author: "Xiao Ke"
category: "Hands-on Guide"
tags: ["Mirage", "Edge-AI", "Vision-Language-Model", "Multi-Task-Architecture", "NPU", "Compact-Models", "Local-Deployment"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/mirage-22mb-vlm-single-pass-detection-ocr-segmentation-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Single-Pass Vision Model under 22MB for Detection and OCR: How Mirage Works

Running multi-modal models on edge hardware or lightweight virtual machines consistently strains memory and bandwidth.

Mainstream models routinely demand 8GB of VRAM on a single card. Processing a standard 1080p interface screenshot can easily burn through over two thousand visual tokens during patch tokenization. Feeding these long sequences into an autoregressive decoder forces the fan to spin at maximum speed while the cursor stays frozen for several seconds. Often, the pipeline only needs to extract two bounding boxes for a power switch alongside a short string from a door sign.

Keeping a heavy model occupying dozens of gigabytes continuously resident in memory strains server budgets. It also drains mobile batteries rapidly.

On edge nodes without dedicated GPUs, what engineers genuinely need is a weight file compressed below several dozen megabytes that isolates entity localization, OCR text extraction, and spatial relations. Can conventional autoregressive architectures realistically achieve this?

For a long time, the answer has been no.

An intriguing minimal architecture recently emerged in the open source ecosystem. The team at Cactus Compute released Mirage, an edge multi-modal model with open weights licensed under Apache 2.0.

Community discussion and firsthand engineering feedback are tracked on Reddit:
https://www.reddit.com/r/LocalLLaMA/comments/1xpm8mw/

The entire model package requires only 22MB of disk space. Total parameter count is capped around 51.3M. More importantly, it outputs scene descriptions, bounding boxes, text recognition, segmentation masks, and spatial topology in a single forward pass.

How does such a compact model pack so many vision capabilities into one forward pass? Is it genuinely that lightweight?

I had plenty of doubts before testing it myself.

---

## Hardware Budget and 51.3M Parameter Distribution

Conventional multi-modal models grow massive because engineers expect a single decoder to handle everything.

The decoder is forced to generate descriptive prose while simultaneously emitting floating-point coordinates inside a continuous text stream. This approach squanders the limited capacity of compact models on low-efficiency coordinate character generation.

Mirage approaches the challenge from a completely different angle. Instead of stacking redundant self-attention layers onto the decoder, the architecture strictly partitions the 51.3M parameters across distinct components:

- Image Encoder. Around 7.8M parameters, utilizing an HConvGLU module with fixed Hadamard transforms to mix channel features alongside lightweight learned convolution kernels;
- Language Decoder. Around 40.8M parameters, built on Simple Attention combined with Hadamard MLPs, derived from the Needle and Whistle sequence architectures;
- Dedicated Output Heads. Around 2.6M parameters, consisting of dedicated text reading heads and spatial geometry heads.

Reviewing this parameter allocation clarifies the mechanism.

Spatial coordinates and segmentation masks are never delegated to the text decoder for autoregressive guessing. Parallel dedicated heads spanning 2.6M parameters emit structural results directly. The language decoder is left to focus strictly on semantic entity labels, basic properties, and causal spatial relations.

Text recognition follows a matching pattern. The model incorporates a dedicated text scanning pathway, extracting character features before establishing semantic alignment.

This division of labor allows the 40.8M language decoder to operate without unnecessary weight.

For scene descriptions, the model supports English, German, French, Spanish, Italian, Dutch, and Polish. The output remains concise and direct.

---

## 74 Milliseconds on Mobile NPU versus 5.8 Seconds on CPU

Evaluating compact model performance requires transparent hardware context.

The benchmark figures highlighted by the author are measured on a Samsung Galaxy S26 phone:

- Running on the onboard neural processing unit NPU, single-pass multi-task inference requires approximately 74 milliseconds;
- Running on the phone CPU, single-pass inference takes approximately 107 milliseconds.

Sub-100ms response times provide adequate speed for on-device photo sorting or camera autofocus assistance.

However, does the experience hold up when deployed on unaccelerated cloud VMs or dated office PCs? Are there performance trade-offs?

Community developer MagoViejo tested the weights on a standard x86 personal computer without hardware accelerators. Relying entirely on dual-core CPU floating-point calculations, parsing a single image stretched to 5.8 seconds.

Going from 74 milliseconds on a phone NPU to 5.8 seconds on a standard PC represents nearly an eighty-fold latency increase. The root cause lies entirely in the compute units: the Galaxy S26 relies on dedicated NPU hardware for fixed-point tensor math, whereas an older x86 machine executes full floating-point matrix operations on CPU cores without dedicated vector instruction acceleration.

These numbers establish clear hardware boundaries. Hardware behavior does not lie.

If your target hardware includes a modern NPU, tens of milliseconds of latency provide genuine practical value.

Conversely, if you intend to run real-time video surveillance on a cheap single-core virtual machine, a throughput of 0.2 frames per second will cause pipelines to fall behind immediately. Facing nearly six seconds of latency, who could integrate this into a live video stream? What production workload would tolerate such delays?

I tested it on an idle low-spec Linux box. The cooling fan spun up immediately.

Positioning Mirage for background offline image clustering, cold-start pre-filtering, or low-overhead perception for lightweight agents without dedicated GPUs matches engineering reality. Running offline batch workloads saves considerable cloud spending.

---

## Community Scrutiny and Container Environment Pitfalls

Every open source project faces intense community scrutiny before hitting production.

Following the release of Mirage, two main points of debate emerged.

The first concern originated from an automated community moderation bot. The account stylome-ai-detection flagged the release post as potentially AI-generated text. While the author quickly engaged in technical discussions to clarify implementation details, promotional wording understandably raised caution among meticulous developers.

Is this genuine engineering work or marketing packaging?

The second debate centered on comparisons with traditional YOLO object detection pipelines.

In raw bounding box throughput on industrial PC CPUs, stripped-down YOLO nano models remain considerably faster. Mirage distinguishes itself not by competing on raw FPS, but by bundling character extraction and multi-entity spatial semantics alongside bounding boxes within a 22MB footprint.

Validating the pipeline locally is straightforward. Under Python 3.10 or higher, install the official package:

```bash
pip install cactus-compute
```

Invoke the built-in inference pipeline via the Python interface:

```python
import cactus_compute

pipeline = cactus_compute.MiragePipeline.from_pretrained("cactus-compute/mirage-v1")
outputs = pipeline.analyze_image("test_sample.jpg")

print("Detected Boxes:", outputs.boxes)
print("OCR Text:", outputs.text_segments)
print("Spatial Topology:", outputs.relationships)
```

During my first run inside a clean Ubuntu 22.04 container, the terminal threw an immediate error: ImportError: libGL.so.1: cannot open shared object file: No such file or directory.

Execution halted on the spot.

Checking environment logs revealed an underlying dependency on graphic rendering libraries. Inside minimal containers, running apt-get update && apt-get install -y libgl1 resolves the missing dependency, allowing execution to proceed smoothly.

Installing the missing system library clears the blocker.

Inference ran reliably once that initial hurdle was addressed.

---

## Technical Boundaries and Selection Criteria

Analyzing Mirage establishes clear operational criteria for 22MB multi-task models.

Suitable deployment scenarios include:
- Mobile and wearable hardware. Devices with low-power NPUs requiring offline parsing of camera signage text and physical object boundaries;
- Edge agent workload filtering. Deploying a local front-end filter on Raspberry Pi or industrial hardware to discard blank or uninformative frames before sending anomalies to cloud models;
- Offline private photo indexing. Building search tags locally without uploading personal imagery to cloud endpoints.

Unsuitable deployment scenarios remain equally clear:
- High frame-rate industrial inspection. Workloads requiring dozens of frames per second for defect identification should rely on YOLO or dedicated detection heads;
- Open-domain deep visual comprehension. Reading intricate chart structures, artwork metaphors, or multi-page financial documents requires API access to frontier cloud multi-modal models.

Embedding structural priors into the problem formulation and using dedicated spatial heads to offload language decoders is a sound architectural strategy.

At 22MB, however, engineers should not expect an all-knowing visual intelligence. Treating it as a compact, focused edge utility knife keeps production systems resilient when traffic spikes arrive.

To run this pipeline smoothly on edge hardware, have you budgeted for six seconds of processing latency on CPU systems? Or does your target deployment already carry a dedicated NPU?

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/mirage-22mb-vlm-single-pass-detection-ocr-segmentation-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

