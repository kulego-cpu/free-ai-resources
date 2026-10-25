---
title: "SiliconFlow Voucher & 20-Model Free Matrix: Comprehensive Setup & Pitfalls Guide"
description: "Claim a 16 CNY voucher valid for 180 days via 1-minute face verification. Unlock 20 permanent 0-cost multimodal models, Anthropic API support, and bypass 403/504 errors."
pubDate: 2026-10-18
author: "FreeAI Dog Editorial"
category: "CN Direct"
tags: ["SiliconFlow", "DeepSeek", "Free Voucher", "Setup Guide", "Zero Cost"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/en/blog/siliconflow-voucher-tips/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site.


# SiliconFlow (SiliconCloud): China's Top Open-Source LLM Inference Hub

SiliconFlow (Beijing SiliconFlow Technology Co., Ltd.) is China's premier open-source LLM inference acceleration API platform. Benefiting from multi-line high-availability BGP networks, its average API response latency sits between **180ms and 200ms**, making it a prime choice for developers, researchers, and automated AI agents.

Many developers assume SiliconFlow only offers a brief free trial, or run into confusing `403 Forbidden` or `504 Gateway Timeout` errors. Based on Hermes's latest telemetry and live benchmarks, this guide unpacks how to claim the **16 CNY voucher (valid for 180 days)**, tap into the **permanent 20-model 0-cost multimodal pool**, and avoid common integration traps.

---

## 1. Benefits & Voucher Value Breakdown

### 1. 16 CNY Platform-Wide Universal Voucher (180-Day Validity)
- **Campaign Period**: Official activity runs through **December 31, 2026**.
- **Longest Retention Window**: Valid for **180 days (6 full months)** from the day you claim it.
- **Genuine Purchasing Power**:
  - Full-scale 671B `deepseek-ai/DeepSeek-V3` list price is ~¥2/1M tokens (and only ¥1/1M tokens during off-peak hours).
  - **The 16 CNY voucher covers 16M to 20M tokens**, ample for thousands of complex agent iterations or in-depth document analyses!
  - **No unexpected billings**: Tokens simply stop deducting when the voucher is depleted. Your bank card is never charged automatically.

### 2. Unlimited Referral Program
- In the console's "Activity Center", join the Ambassador program. Every invited user who registers and verifies awards **16 CNY vouchers to both you and your friend**.
- **No invitation ceiling**: High-volume developers can accumulate substantial credits to run persistent RAG clusters at zero operational cost.

---

## 2. The 20-Model Official Permanent Free Tier Matrix

Even when your vouchers are exhausted, SiliconFlow keeps **20 multimodal models 100% free forever**. As long as you specify these Model IDs, the cost is strictly ¥0:

| Category | Model ID | Highlights | Recommended Use Cases |
| :--- | :--- | :--- | :--- |
| **Reasoning** | `deepseek-ai/DeepSeek-R1-0528-Qwen3-8B` | Distilled lightweight reasoning | Code review, math reasoning, logic derivation |
| **Logic Deduction** | `zai-org/GLM-Z1-9B-0414` | Zhipu AI reasoning LLM | Structured reasoning, long-chain agent steps |
| **Large Parameter** | `XingChenAGI/Xing4.0-29B` | Telecom Xingchen **29B parameter** | Rare free 29B model, essay writing, report generation |
| **Flagship Chat** | `Qwen/Qwen3-8B` | Alibaba Qwen latest 8B | Low-latency chat, general conversational bots |
| **Lightweight Chat** | `Qwen/Qwen3.5-4B` / `Qwen/Qwen2.5-7B-Instruct` | 4B fast / 7B reliable | Mobile integration, lightweight tooling |
| **General Purpose** | `zai-org/GLM-4-9B-0414` | Zhipu aligned 9B model | Roleplay, summarization, general assistance |
| **Translation** | `tencent/Hunyuan-MT-7B` | Tencent Hunyuan translation LLM | Cross-border docs, technical translation |
| **Document OCR** | `DeepSeek-OCR` | DeepSeek specialized visual OCR | Complex document parsing, table & formula extraction |
| **Layout OCR** | `PaddlePaddle/PaddleOCR-VL-1.5` | Baidu PaddlePaddle vision-language | Invoices, receipts, industrial OCR text extraction |
| **Text-to-Image** | `Kwai-Kolors/Kolors` | Kuaishou Kolors HD image gen | **100% permanent 0-cost image generation** |
| **Speech-to-Text** | `FunAudioLLM/SenseVoiceSmall` | Alibaba ultra-fast multilingual STT | Meeting transcription, subtitles, voice recognition |
| **Speech-to-Text** | `XingChenAGI/XingChenASR-V3.2-Ultra` | Telecom high-accuracy STT | Noisy audio transcription, batch processing |
| **Speaker Diarization** | `XingChenAGI/XingChenASR-Diarize-V3.0` | Speaker separation STT | Roundtables, interviews, panel discussions |
| **Text-to-Speech** | `XingChenAGI/XingChenGSR-V1.0` | Natural Chinese speech synthesis | Audiobooks, voiceovers, voice alerts |
| **Multilingual Embedding** | `BAAI/bge-m3` | BAAI top embedding architecture | Enterprise RAG vector search, semantic clustering |
| **Search Reranking** | `BAAI/bge-reranker-v2-m3` | Semantic precision reranker | Second-stage RAG retrieval to maximize accuracy |
| **Chinese Embedding** | `BAAI/bge-large-zh-v1.5` | Dense Chinese vector model | Chinese specialized knowledge bases |
| **English Embedding** | `BAAI/bge-large-en-v1.5` | Dense English vector model | English technical docs & GitHub repositories |

> [!TIP]
> **Audio & Image Pitfall Alerts**:
> 1. `Kwai-Kolors/Kolors` is the genuine 100% free image generation model; the FLUX.1 series consumes credits/vouchers.
> 2. `FunAudioLLM/SenseVoiceSmall` is 100% free; CosyVoice2 (¥0.05/1K chars) is paid. Pair **SenseVoiceSmall + XingChenGSR** for a complete 0-cost audio pipeline!

---

## 3. Quick Setup in Three Steps

### Step 1: Register via Welfare Portal
Visit the SiliconFlow Console:
👉 [https://cloud.siliconflow.cn/i/58EXEpfm](https://cloud.siliconflow.cn/i/58EXEpfm)
Enter your mobile number and SMS verification code to sign up in 10 seconds.

### Step 2: Complete Face Verification
1. Click your profile avatar in the upper right and navigate to **"Real-Name Verification"**.
2. Scan the QR code using the Alipay mobile app (Chinese 2nd-gen ID card).
3. The review is automated and completes within 60 seconds.
4. **The 16 CNY voucher is instantly credited**, and your account concurrency jumps to **60 RPM**.

### Step 3: Create API Key
1. Go to **"API Keys"** in the left sidebar.
2. Click **"New API Key"**, give it a project label, and copy your `sk-` token.

---

## 4. Key Traps and Solutions

### Trap 1: 403 Forbidden Error on Paid Models
- **Symptom**: Calling `deepseek-ai/DeepSeek-V3` returns `403 Forbidden` immediately on new accounts.
- **Cause**: Unverified accounts are restricted to free lightweight models to prevent automated abuse.
- **Fix**: Complete the 1-minute face verification. Once the 16 CNY voucher is credited, full-spec models unlock instantly.

### Trap 2: 504 Gateway Timeout on Long Responses
- **Symptom**: The client hangs for 60 seconds then throws `504 Gateway Timeout` when generating long outputs or deep reasoning chains.
- **Cause**: Non-streaming connections exceed reverse proxy gateway keep-alive thresholds.
- **Fix**: **Always set `stream: true` in your client config**. Streaming packets every few milliseconds keeps the connection alive indefinitely.

### Trap 3: Native Anthropic Messages API Support
SiliconFlow natively supports the **Anthropic Messages protocol**!
- In **Claude Code, Cursor, Cline, Roo Code, or Continue**, select the Anthropic provider option:
  - Base URL: `https://api.siliconflow.cn/v1`
  - API Key: `sk-your-key`
  - Model Name: Choose your target model ID.

### Trap 4: Off-Peak 50% Discounts
- **DeepSeek models** are **50% off** between **00:00 - 02:00 AM** and after **20:00 PM** every day.
- Running batch offline jobs or RAG indexing during off-peak hours effectively doubles the lifetime of your 16 CNY voucher to 32 CNY!

---

## 5. Integration Code Samples

### Python (OpenAI SDK with Streaming)

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ.get("SILICONFLOW_API_KEY", "sk-your-key"),
    base_url="https://api.siliconflow.cn/v1"
)

response = client.chat.completions.create(
    model="XingChenAGI/Xing4.0-29B",
    messages=[
        {"role": "user", "content": "Explain RAG architecture best practices with vector reranking."}
    ],
    stream=True
)

for chunk in response:
    content = chunk.choices[0].delta.content or ""
    print(content, end="", flush=True)
```

---

## Summary

With a **16 CNY long-validity voucher**, a **20-model permanent 0-cost multimodal pool**, and **OpenAI + Anthropic dual-protocol compatibility**, SiliconFlow delivers unmatched compute infrastructure.

Connect your SiliconFlow key in [FreeAI Dog](https://freeaidog.com) to enjoy unified, zero-cost high-availability AI services!

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/en/blog/siliconflow-voucher-tips/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/en/radar/)
