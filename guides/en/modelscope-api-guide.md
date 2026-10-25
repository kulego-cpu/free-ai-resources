---
title: "ModelScope API Evaluation: 250 MoLi Daily Free Allowance, 900+ Open Models, Mainland Direct Connection & Native CORS"
description: "In-depth guide to Alibaba's premier ModelScope open-source AI community Serverless API Inference! Learn about the 250 MoLi daily free allowance (~125-500 calls), 900+ models (Qwen, DeepSeek-V3/R1, GLM), native CORS support, real-name requirements, and 5 essential engineering gotchas."
pubDate: 2026-10-19
author: "FreeAI Hub Editorial Team"
category: "国内直连"
tags: ["ModelScope", "Alibaba", "Qwen", "DeepSeek", "OpenSource", "CORS", "DailyRefresh", "NoCreditCard"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/modelscope-api-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# ModelScope API Evaluation: 250 MoLi Daily Free Allowance, 900+ Open Models, Mainland Direct Connection & Native CORS

In China's thriving open-source AI ecosystem, **ModelScope**, initiated by Alibaba Cloud alongside prominent research institutes and industry partners, stands as the nation's premier open model hub. The platform hosts thousands of cutting-edge open weights ranging from Alibaba's flagship Qwen series to the globally renowned DeepSeek-V3 and R1, alongside Zhipu GLM, Baichuan, and MiniMax.

For software engineers, hobbyists, and researchers, ModelScope provides a standout **Serverless API Inference free tier**:
- ❌ **No credit card or international banking required**
- ❌ **No prepayment or deposit needed**
- ✅ **Ultra-low latency direct connection within China backed by Alibaba CDN**
- ✅ **250 MoLi granted daily (approx. 125 to 500 inference calls per day)**
- ✅ **Native browser CORS preflight support (`access-control-allow-origin: *`)**

This article delivers a comprehensive breakdown of ModelScope's quota rules, MoLi pricing units, model coverage, and 5 crucial engineering gotchas.

---

## 1. Core Policy & Allowance: The Daily "MoLi" Currency

ModelScope utilizes an internal computational currency called **"MoLi" (魔粒)**. Developers claim their daily allowance simply by accessing the platform:

| Dimension | Measured Status & Official Policy | Key Takeaway & Recommendation |
| :--- | :--- | :--- |
| **Free Tier Type** | ✅ **`daily_refresh` (Daily Automated Reset)** | Resets every midnight at 00:00 (UTC+8) |
| **Daily Allowance** | **250 MoLi / day** | 200 base for daily login + 50 for linking a verified Alibaba Cloud account |
| **Estimated Calls** | **~125 to 500 requests / day** | Dependent on the model complexity tier |
| **Expiration Rules** | **Short-term MoLi expires in 24 hours** | Daily 250 MoLi does not roll over; 50 long-term MoLi granted upon initial signup (valid 90 days) |
| **Credit Card** | ❌ **Not required** | Complete freedom from surprise bills or credit card validation |
| **Real-Name Verification** | ⚠️ **Mandatory** | Requires linking an Alibaba Cloud account verified with mainland Chinese citizen ID (via Alipay) |
| **CORS Support** | 🟢 **Natively Open (`*`)** | Preflight OPTIONS returns `access-control-allow-origin: *` for direct web fetch |
| **OpenAI Compatibility** | 🟢 **Fully Compatible** | Standard endpoint at `https://api-inference.modelscope.cn/v1` |
| **Overage Behavior** | Returns `HTTP 429 Too Many Requests` | Clean cutoff when daily MoLi is exhausted; zero debit risk |

---

## 2. MoLi Consumption Tiers: Maximizing Value

The rate of MoLi consumption is categorized by parameter scale and computational intensity:

### 1. Lightweight Tier: 0.5 MoLi / call (~500 calls/day)
- **Representative Models**: `Qwen/Qwen2.5-7B-Instruct`, `Qwen/Qwen2.5-0.5B`, `Qwen/Qwen2.5-1.5B`.
- **Ideal For**: Text formatting, classification, intent routing, multi-turn simple dialogues, and lightweight code completions.

### 2. Mainstream Tier: 1.0 MoLi / call (~250 calls/day)
- **Representative Models**: `deepseek-ai/DeepSeek-V3`, `deepseek-ai/DeepSeek-R1-Distill-Qwen-32B`, `ZhipuAI/GLM-4-9B-Chat`.
- **Ideal For**: Deep logical reasoning, long document synthesis, complex code generation, and enterprise RAG question-answering.

### 3. Flagship Tier: 2.0 MoLi / call (~125 calls/day)
- **Representative Models**: `Qwen/Qwen2.5-72B-Instruct` and other 70B+ parameter flagship models.
- **Ideal For**: Complex agent task planning, nuanced translation, and academic benchmark evaluations.

---

## 3. 5 Essential Engineering Gotchas

### 1. ⚠️ Real-Name ID Requirement (International Developer Barrier)
ModelScope API Inference operates under mainland regulatory frameworks. Users must link an Alibaba Cloud account that has completed real-name identity verification (typically completed in seconds using a mainland Chinese national ID card via Alipay). **International developers without mainland credentials cannot unlock the API Inference quota.**

### 2. ⚠️ Short-Term MoLi Expires in 24 Hours Without Roll-Over
The daily 250 MoLi allowance constitutes "short-term MoLi." It expires precisely at 00:00 UTC+8 daily. Unused tokens cannot be accumulated over time. When tokens are deducted, the platform automatically spends the earliest-expiring MoLi first.

### 3. ⚠️ Native Browser CORS Headers (Direct Frontend Fetch)
Unlike many domestic cloud providers that block browser cross-origin requests, ModelScope's gateway returns `access-control-allow-origin: *` on OPTIONS preflight requests. You can execute `fetch()` calls directly from static web apps, React/Vue SPAs, and browser extensions without deploying a backend proxy.

### 4. ⚠️ Strictly Non-Commercial Terms & Dynamic Throttling
Official terms state that API Inference is intended for developer evaluation, prototyping, and academic research. **It carries no production SLA.** Concurrency is dynamically restricted according to overall community server load; peak hours frequently encounter `HTTP 429` rate-limit responses.

### 5. ⚠️ Model Names Must Include Organization Namespace
When issuing requests through the OpenAI SDK, the `model` argument must specify the full namespace path:
- Incorrect: `model: "Qwen2.5-72B-Instruct"` ❌
- Correct: `model: "Qwen/Qwen2.5-72B-Instruct"` ✅
- Correct: `model: "deepseek-ai/DeepSeek-V3"` ✅

---

## 4. Quickstart Code Snippets

### 1. Python (OpenAI SDK Standard)

```python
import os
from openai import OpenAI

# Access Token copied from modelscope.cn/apikey
client = OpenAI(
    api_key=os.environ.get("MODELSCOPE_API_KEY", "your_modelscope_access_token"),
    base_url="https://api-inference.modelscope.cn/v1"
)

response = client.chat.completions.create(
    model="Qwen/Qwen2.5-72B-Instruct",
    messages=[
        {"role": "system", "content": "You are Qwen 2.5 72B hosted on Alibaba ModelScope."},
        {"role": "user", "content": "Summarize the primary benefits of ModelScope API Inference in 3 bullet points."}
    ],
    temperature=0.7,
    max_tokens=512
)

print(response.choices[0].message.content)
```

### 2. cURL Terminal Execution

```bash
curl -X POST "https://api-inference.modelscope.cn/v1/chat/completions" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-R1-Distill-Qwen-32B",
    "messages": [{"role": "user", "content": "Which number is greater: 9.11 or 9.8?"}],
    "temperature": 0.6
  }'
```

---

## 5. 3-Step Onboarding Guide

1. **Sign Up & Link Account**: Visit [modelscope.cn](https://modelscope.cn/) and link your verified Alibaba Cloud account in settings;
2. **Retrieve API Token**: Navigate to [modelscope.cn/apikey](https://modelscope.cn/apikey) and copy your auto-generated Access Token;
3. **Configure & Run**: Set Base URL to `https://api-inference.modelscope.cn/v1`, paste your token, choose `Qwen/Qwen2.5-72B-Instruct` or `deepseek-ai/DeepSeek-V3`, and enjoy 250 daily free MoLi!

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/modelscope-api-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

