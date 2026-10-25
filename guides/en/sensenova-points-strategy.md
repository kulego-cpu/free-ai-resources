---
title: "SenseNova Auto-Refill Quota Guide: 120k Points Every 5 Hours in Free Beta"
description: "A unique game-like stamina refill system! Dual pools granting ~1.2M points weekly with native vision-multimodal optimization."
pubDate: 2026-10-11
author: "FreeAI Dog Editorial"
category: "CN Direct"
tags: ["SenseTime", "SenseNova", "Points System", "Beta Perks"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/en/blog/sensenova-points-strategy/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site.


# SenseNova: A Gaming-Style Auto-Regenerating Stamina Free Tier

SenseTime's flagship LLM platform **SenseNova** is currently running its **Token Plan Free Public Beta**. Its free tier mechanism stands out from almost every other AI vendor: **an automated dual-pool stamina regeneration system**.

---

## 1. Why Is It Called "Stamina Auto-Regeneration"?

Most cloud providers only give you a one-time grant (e.g. $5 or $10 credit that expires or runs out forever).
SenseNova, however, employs a **periodic auto-refilling pool system**:

1. **General Points Pool**: Automatically restores **60,000 points every 5 hours** (capped at 600,000 points weekly);
2. **Flash-Lite Dedicated Pool**: Automatically adds another **60,000 points every 5 hours**;
3. **Total Regeneration Speed**: **120,000 points refreshed every 5 hours, totaling ~1.2 million free points weekly!**
4. **🎁 1:1 Flash-Lite Rebate Program**: For every 1 point consumed from the Flash-Lite dedicated pool, 1 general point is rebated 1:1 (settled hourly, valid 30 days, exempt from window limits)—effectively doubling your free runway!

Unless you run massive multi-threaded batch jobs, your quota naturally refills back to 100% every few hours, providing true perpetual free usage.

---

## 2. 7 Official Free Models (Proprietary & Aggregated)

SenseNova provides both proprietary multimodal models and top-tier aggregated open-weights models under one API key:

1. **`sensenova-6.8-flash-lite`**: SenseTime's proprietary multimodal agent, specialized in charts, documents, and office workflows.
2. **`sensenova-u1.5-fast` & `u1.5-lite`**: Unified image generation and localized editing models; passing `watermark: false` removes watermarks completely free during the beta!
3. **`deepseek-v4-flash` & `deepseek-flash`**: Aggregated DeepSeek reasoning models, deducting from the general point pool.
4. **`glm-5.2`**: Aggregated Zhipu GLM 5.2 flagship open-source LLM for superb language comprehension.
5. **`kimi-k3`**: Aggregated Moonshot Kimi K3 long-context model for document synthesis.

---

## 3. 3-Step Setup Guide

1. Visit the SenseNova console direct link:
   👉 [https://platform.sensenova.cn/console/keys](https://platform.sensenova.cn/console/keys)
2. Sign up with mobile phone and SMS code. The system automatically activates the **Free Beta Tier (¥0/mo)**;
3. Navigate to the **"API Keys"** section and generate your key (up to 20 keys supported);
4. The endpoint is fully OpenAI-compatible at `https://token.sensenova.cn/v1`, with native configuration support for Hermes Agent, OpenClaw, and Cursor.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/en/blog/sensenova-points-strategy/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/en/radar/)
