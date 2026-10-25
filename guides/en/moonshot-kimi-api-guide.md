---
title: "Moonshot Kimi API Deep Dive: Mainland 15 RMB Voucher, Global $1 Tier Gate & Prepaid Zero-Risk Policy"
description: "Comprehensive guide to Tsinghua-founded unicorn Moonshot AI (Kimi)! Unpack the dual-site structure (Domestic site 15 RMB voucher vs Global site $1 threshold), K3 model access prerequisites, Tier 0-5 rate tiers, CORS exact reflection, and 5 essential engineering gotchas."
pubDate: 2026-10-19
author: "FreeAI Hub Editorial Team"
category: "国内直连"
tags: ["Moonshot", "Kimi", "LongContext", "Voucher", "Prepaid", "DirectConnection", "ChinaAI"]
featured: false
---

# Moonshot Kimi API Deep Dive: Mainland 15 RMB Voucher, Global $1 Tier Gate & Prepaid Zero-Risk Policy

Founded by Dr. Yang Zhilin of Tsinghua University, **Moonshot AI (月之暗面)** and its flagship conversational assistant **Kimi** have gained immense popularity across Asia and worldwide for groundbreaking lossless ultra-long context understanding (up to 2 million tokens in consumer apps, 128k+ in API), real-time web retrieval integration, and academic-grade document analysis.

However, developers integrating Moonshot's Developer Platform often encounter confusing discrepancies: **"Is Kimi API truly free or purely paid?", "Why does my freshly created API key fail with insufficient quota errors?"**

In our October 2026 benchmark verification, we discovered that Moonshot operates a strict **dual-track domestic vs. international operational model**:
- 🇨🇳 **Domestic Portal (`platform.moonshot.cn`)**: Registering with a Mainland China phone number and completing national ID real-name verification grants an **automatic 15 RMB API voucher (~1.5M tokens)**. You can immediately call the `moonshot-v1` family without depositing a single cent!
- 🌐 **Global Portal (`platform.moonshot.ai`)**: Built exclusively for international developers, priced in USD, and operates on a strict prepaid basis **with zero free tier**. Activating Tier 0 requires an initial minimum deposit of $1.
- 🔒 **Zero Billing Risk**: Both domestic and global portals run on a **pure prepaid balance model**. When vouchers or balances hit zero, the API gracefully halts with `HTTP 402 Payment Required`. **There is absolutely no risk of silent credit card auto-debits or runaway overdraft bills!**

This guide provides a comprehensive evaluation of domestic vs. global portal policies, 15 RMB voucher model eligibility, Tier 0–5 concurrency gates, and 5 crucial engineering gotchas.

---

## 1. Core Policy & Dual-Track Matrix: 15 RMB Voucher vs. $1 Minimum Recharge

| Dimension | Domestic Portal (`platform.moonshot.cn`) | Global Portal (`platform.moonshot.ai`) | Key Takeaway & Recommendation |
| :--- | :--- | :--- | :--- |
| **Free Tier Type** | 🟢 **`voucher` (15 RMB voucher upon real-name)** | ❌ **`none` (No free tier)** | Use domestic portal if you have mainland identity credentials |
| **Activation Prerequisite** | China mobile + **Mainland Citizen ID Verification** | Minimum **$1 deposit** to unlock Tier 0 | Global portal has no free allowance |
| **Eligible Models** | **`moonshot-v1-8k/32k/128k` full series** | All unlocked models according to deposit Tier | **Notice: 15 RMB voucher cannot be used for K3** |
| **Subsequent Credits** | No fixed recurring grants | Deposit cumulative $5 to receive $5 voucher | Promotional vouchers do not count toward tier upgrades |
| **Overage Behavior** | Strict prepaid: returns `HTTP 402` on exhaustion | Strict prepaid: returns `HTTP 402` on exhaustion | Zero auto-debit risk, completely safe for experiments |
| **CORS Preflight** | ⚠️ Exact Origin Reflection (not wildcard `*`) | ⚠️ Exact Origin Reflection (not wildcard `*`) | Echoes incoming Origin header; configure frontend origins |
| **OpenAI SDK** | 🟢 100% compatible with `/v1/chat/completions` | 🟢 100% compatible with `/v1/chat/completions` | Drop-in replacement via Base URL and API Key |

---

## 2. Model Availability & Access Tiers

### 1. Models Eligible for 15 RMB Voucher (Mainstream Production Long-Context)
The 15 RMB voucher granted upon real-name verification fully subsidizes calls to the following production foundation models:
- **`moonshot-v1-8k`** (¥0.012 / 1K input tokens, ¥0.012 / 1K output tokens): General conversational backbone, responsive and nuanced in Chinese/English; 15 RMB provides ~1.25M tokens of interactive dialogue.
- **`moonshot-v1-32k`**: Medium-to-long reports, financial sheets, and deep document extraction.
- **`moonshot-v1-128k`**: Full-scale code repository audits and multi-paper comparative synthesis up to 128K context.

### 2. Models Requiring Cash Deposit Unlock
- **`kimi-k3`**: Moonshot's next-generation reasoning flagship. Official policy strictly mandates that **the 15 RMB new-user voucher cannot be used for K3**. Users must complete at least one cash deposit (minimum ¥10 or $1) to unlock K3 endpoints.
- **`kimi-k2.7-code`**: Code generation specialization model engineered for competitive algorithmic synthesis and refactoring.

---

## 3. 5 Essential Engineering Gotchas

### 1. ⚠️ Register on the Domestic Portal for the 15 RMB Voucher
Developers who navigate to `platform.moonshot.ai` (the English global site) will find that API keys cannot be used without depositing $1, leading them to believe the free tier was cancelled. **Navigate to `platform.moonshot.cn`**, register with an eligible mobile number, complete real-name verification in Account Settings, and the 15 RMB voucher will instantly appear.

### 2. ⚠️ 15 RMB Voucher Cannot Call Kimi K3
If calling `kimi-k3` returns quota exhaustion or insufficient permissions, your voucher has not expired; rather, Moonshot places guardrails around its flagship reasoning model. The 15 RMB voucher supports `moonshot-v1-8k/32k/128k`. Testing K3 requires a minimal deposit of ¥10.

### 3. ⚠️ Free Vouchers Do Not Advance Tier Rankings
Moonshot enforces strict rate-limiting tiers from Tier 0 to Tier 5 based on **cumulative actual cash deposits**. The 15 RMB complimentary voucher does not count towards cumulative spend, so accounts operating solely on the voucher remain on baseline Tier concurrency protections.

### 4. ⚠️ Consumer Kimi App Quota is Completely Isolated from Developer API
Personal free usage, search credits, and VIP subscriptions on the consumer `kimi.com` web app or mobile app are completely distinct from the developer API platform. Web app benefits cannot be converted into API tokens, nor vice-versa.

### 5. ⚠️ CORS Uses Exact Origin Reflection
Live testing confirms that `api.moonshot.cn` does not send wildcard `access-control-allow-origin: *` during OPTIONS preflights. Instead, it echoes the exact requesting `Origin`. In browser single-page applications (SPAs) or web extensions, ensure your origin headers are properly formatted.

---

## 4. Quickstart Code Snippets

### 1. Python (OpenAI SDK Drop-in)

```python
import os
from openai import OpenAI

# Initialize client with your Moonshot API Key (sk-...)
client = OpenAI(
    api_key=os.environ.get("MOONSHOT_API_KEY", "your_moonshot_api_key"),
    base_url="https://api.moonshot.cn/v1"
)

response = client.chat.completions.create(
    model="moonshot-v1-8k",
    messages=[
        {"role": "system", "content": "You are Kimi, an AI assistant developed by Moonshot AI specializing in long-context analysis."},
        {"role": "user", "content": "Summarize the key advantages of Moonshot Kimi in three bullet points."}
    ],
    temperature=0.3
)

print(response.choices[0].message.content)
```

### 2. cURL Command-Line Verification

```bash
curl -X POST "https://api.moonshot.cn/v1/chat/completions" \
  -H "Authorization: Bearer $MOONSHOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "moonshot-v1-8k",
    "messages": [{"role": "user", "content": "Which number is larger: 9.11 or 9.8?"}],
    "temperature": 0.3
  }'
```

---

## 5. Three-Step Onboarding Guide

1. **Register on Domestic Portal**: Visit [platform.moonshot.cn](https://platform.moonshot.cn/), register with a mainland mobile number, and complete citizen ID verification;
2. **Confirm Voucher & Generate Key**: Verify the 15 RMB voucher in "Billing Center", then generate an `sk-` credential in "API Key Management";
3. **Configure & Start Calling**: Point your Base URL to `https://api.moonshot.cn/v1`, select `moonshot-v1-8k` or `moonshot-v1-32k`, and enjoy high-accuracy 128K context processing with zero upfront cost!

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/en/blog/moonshot-kimi-api-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/en/radar/)
