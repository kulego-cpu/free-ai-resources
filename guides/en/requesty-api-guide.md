---
title: "Requesty Developer Guide: Highest 200 Free Requests/Day, $10 Signup Bonus, 13 Free Models & 5% Lowest Platform Fee"
description: "A comprehensive developer guide to Requesty, the next-gen high-availability AI router! Explore 200 free requests/day (4x OpenRouter), a $10 signup credit (for Claude 3.5 Sonnet / GPT-4o), 13 permanent $0 models (Nemotron 1M & Gemma 4), wildcard CORS (*), dual OpenAI & Anthropic SDK endpoints, and EU Frankfurt data residency."
pubDate: 2026-10-19
author: "FreeAI Hub Editorial Team"
category: "Overseas Line"
tags: ["Requesty", "AI Router", "Nemotron", "Gemma 4", "Free API", "1M Context", "OpenAI Compatible", "Global Network"]
featured: false
---

# Requesty Developer Guide: Highest 200 Free Requests/Day, $10 Signup Bonus, 13 Free Models & 5% Lowest Platform Fee

In the rapidly evolving AI gateway and routing market, **Requesty** has quickly established itself as a developer favorite, widely praised as **the most cost-effective AI router in the industry**.

Compared to traditional API aggregators, Requesty delivers standout advantages:
- 🎯 **200 Free Requests/Day (Market High)**: Without requiring a credit card, newly created organizations receive **200 free requests every day** (at 20 RPM), a daily allowance **4x larger** than OpenRouter's initial free tier (50 requests/day).
- 🎁 **$10 Instant Signup Credit**: Simple email verification unlocks a $10 credit usable across **600+ premium paid models**, including Claude 3.5 Sonnet, GPT-4o, and Gemini 2.0 Flash.
- 📦 **13 Permanent $0 Foundation Models**: Curated access to NVIDIA Nemotron 3 Ultra 550B (1.0M ultra-long context), Google Gemma 4 31B, InclusionAI Ling 3.1 Flash (with prompt caching), Meta Muse Glimmer, and Poolside coding models.
- 🌐 **Full Wildcard CORS Support (`*`)**: Both global and EU endpoints return `access-control-allow-origin: *`, enabling frictionless browser frontend integrations.
- 💰 **Industry-Low 5% Platform Fee**: Paid usage incurs only a 5% platform fee (vs OpenRouter's 5.5%), supports Bring-Your-Own-Key (BYOK), and requires **$0 minimum deposit**.

This guide covers Requesty's free tier mechanics, detailed specs for all 13 free models, Python implementation examples, and 5 vital engineering gotchas.

---

## 1. Core Policies & Technical Specs

| Evaluation Dimension | Official Rules & Test Results | Key Analysis & Advice |
| :--- | :--- | :--- |
| **Free Tier Category** | 🟢 **`daily_refresh` (Daily Auto-Reset)** | **200 free requests/day** (permanent refresh) + **$10 signup credit** |
| **Credit Card Requirement** | 🟢 **No credit card required** | Email signup only; no phone or identity verification required |
| **Rate Limits** | ⚡ **20 RPM (Requests Per Minute)** | Applies across all 13 free models; HTTP 429 returned if exceeded |
| **Auto-Billing Risk** | 🟢 **Zero auto-billing risk** | Safe cut-off on free models; paid models require explicit prepaid credits |
| **CORS Cross-Origin** | 🟢 **Full wildcard (`*`) supported** | `access-control-allow-origin: *` returned, browser frontend friendly |
| **SDK Compatibility** | 🔄 **Dual OpenAI & Anthropic SDKs** | Native endpoints for OpenAI `/v1` and Anthropic `/anthropic/v1/messages` |
| **Platform Service Fee** | 💰 **5% (Lowest in class)** | Below OpenRouter's 5.5%; BYOK included with $0 minimum deposit |
| **EU Data Residency** | 🇪🇺 **Frankfurt (All Tiers)** | Dedicated EU endpoint `router.eu.requesty.ai` for complete GDPR compliance |

---

## 2. 13 Permanent $0 Free Foundation Models

Requesty offers 13 models provided by NVIDIA, Google, Meta, Mistral, InclusionAI, and Poolside designated as **$0.00 / 1M Tokens**:

| # | Model ID | Provider | Context Window | Key Capabilities | Recommended Use Cases |
| :---: | :--- | :--- | :---: | :--- | :--- |
| 1 | `nvidia/nemotron-3-ultra-550b-a55b` | NVIDIA | **1.0M** | Tools, Think Mode | Massive MoE reasoning, agentic planning, and logic |
| 2 | `nvidia/nemotron-3-super-120b-a12b` | NVIDIA | **1.0M** | Tools, Think Mode | Cost-effective general reasoning on 1M token contexts |
| 3 | `nvidia/nemotron-3.5-lightning-30b-a3b` | NVIDIA | **1.0M** | Tools, Think Mode | Ultra-fast responses for lightweight agent dispatch |
| 4 | `nvidia/nemotron-3-nano-30b-a3b` | NVIDIA | 262K | Tools, Think Mode | Low-latency tasks and edge logic processing |
| 5 | `nvidia/nemotron-3-nano-omni-30b-a3b-reasoning` | NVIDIA | 131K | **Vision Multimodal**, Think | Chart understanding and multimodal chain-of-thought |
| 6 | `nvidia/nemotron-3.5-content-safety` | NVIDIA | 131K | **Vision Multimodal**, Think | Text and visual moderation guardrail pipeline |
| 7 | `inclusion/ling-3.1-flash` | InclusionAI | 262K | **Prompt Cache**, Tools | Ultra-fast document comprehension with prompt caching |
| 8 | `inclusion/ling-3.0-tiny` | InclusionAI | 262K | **Prompt Cache**, Tools | Ultra-low latency short-context Q&A and extraction |
| 9 | `google/gemma-4-31b-it` | Google | 262K | **Vision Multimodal**, Tools | Google open weights for coding, math, and vision tasks |
| 10 | `meta/muse-glimmer-30b` | Meta | 131K | **Vision Multimodal**, Tools | Multimodal open weights for image recognition & QA |
| 11 | `mistralai/leanstral-1-5` | Mistral AI | 262K | Tools, Strict JSON | Structured JSON output and function calling |
| 12 | `poolside/laguna-m.1` | Poolside | 33K | Code Specialist, Think | Code generation, automated testing, and refactoring |
| 13 | `poolside/laguna-xs.2` | Poolside | 33K | Code Specialist, Think | High-frequency shell generation and autocomplete |

---

## 3. Code Implementation: OpenAI & Anthropic SDKs

### 1. Python (OpenAI SDK Standard Setup)

Point `base_url` to `https://router.requesty.ai/v1` to invoke 1.0M context models with standard OpenAI code:

```python
from openai import OpenAI

client = OpenAI(
    api_key="req_xxxxxxxxxxxxxxxxxxxx",
    base_url="https://router.requesty.ai/v1"
)

response = client.chat.completions.create(
    model="google/gemma-4-31b-it",
    messages=[
        {"role": "system", "content": "You are a distributed systems architect."},
        {"role": "user", "content": "Explain the architectural advantages of an AI router over direct provider APIs."}
    ],
    temperature=0.7,
    max_tokens=1024
)

print(response.choices[0].message.content)
```

### 2. Python (Anthropic Messages API Setup)

Developers using Claude Code, Cline, or Cursor can leverage Requesty's dedicated Anthropic endpoint directly:

```python
from anthropic import Anthropic

client = Anthropic(
    api_key="req_xxxxxxxxxxxxxxxxxxxx",
    base_url="https://router.requesty.ai/anthropic/v1"
)

message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Write an asynchronous Python HTTP client with automatic retries and exponential backoff."}
    ]
)

print(message.content[0].text)
```

---

## 4. 5 Critical Engineering Gotchas

### 1. Market-Leading 200 Daily Free Requests
While OpenRouter provides 50 free requests per day, Requesty provides 200 requests/day at 20 RPM right out of the box. This provides a 4x buffer for local automated testing and agent development loops.

### 2. $10 Signup Credit Enables Paid Model Testing
Your $10 signup bonus works with 600+ closed-source models. This provides roughly 1 Million output tokens on Claude 3.5 Sonnet or 2 Million tokens on GPT-4o, giving you ample runway to test commercial architectures for free.

### 3. Full Wildcard CORS Support (`*`)
Verified via curl OPTIONS, Requesty endpoints return `access-control-allow-origin: *`. Single-page web apps, browser extensions, and client-side demos can connect directly without proxy infrastructure.

### 4. EU Data Residency in Frankfurt
For teams with strict GDPR or data locality requirements, Requesty provides `https://router.eu.requesty.ai/v1`. All prompt routing and processing remain strictly within the European Union.

### 5. Proxy Recommended for Mainland China
With core routing nodes in US East and Frankfurt, traffic from mainland China may experience routing jitter. Using an overseas proxy ensures stable sub-300ms latency.

---

## 5. 3-Step Quickstart Guide

1. **Sign Up for Free & Claim $10 Credit**: Visit [Requesty Sign-up](https://app.requesty.ai/api-keys), register with your email address (no credit card needed), and claim your $10 starting balance.
2. **Generate API Key**: Go to the API Keys tab in your dashboard and generate your `req_...` key.
3. **Set Base URL to Start Querying**: Point your SDK base URL to `https://router.requesty.ai/v1` and enjoy 200 daily requests across 13 free foundation models!

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/en/blog/requesty-api-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/en/radar/)
