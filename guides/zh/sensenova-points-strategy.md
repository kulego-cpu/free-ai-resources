---
title: "商汤日日新“体力自动回血”详解：每 5 小时 12 万积分公测抢鲜手册"
description: "AI 四小龙商汤的公测大羊毛！双积分池每周送 120 万积分，原生看图多模态特化。"
pubDate: 2026-10-11
author: "FreeAI Hub 编辑部"
category: "国内直连"
tags: ["商汤", "SenseNova", "积分制", "公测福利"]
featured: false
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/blog/sensenova-points-strategy/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站。


# 商汤日日新：像玩游戏一样“定时回血”的神仙机制

商汤科技（SenseTime）的「日日新 SenseNova」大模型开放平台目前正处于 **Token Plan 免费公测阶段**。它的免费机制在国内所有大模型厂商中独树一帜：**双积分池体力自动恢复制**。

---

## 一、 为什么叫“体力自动回血”？

绝大部分厂商的赠送额度都是“一次性”的（比如送 10 元，花光就只能充钱）。
而商汤日日新采用了**周期性重置积分池**：

1. **通用积分池**：每 5 小时自动恢复 **60,000 积分**（每周上限 60 万积分）；
2. **Flash-Lite 专属池**：每 5 小时额外专享 **60,000 积分**；
3. **合计回血速度**：**每 5 小时净回血 12 万积分，每周免费赠送高达 120 万积分（折合现金约 240 元）！**
4. **🎁 独创 1:1 消费反哺**：每消耗 1 积分 Flash-Lite 专属池，系统以 1:1 比例返还 1 通用积分（按日汇总每小时到账，30 天有效且不占用滚动额度上限），相当于变相免费额度翻倍！

只要你的调用频率不是疯狂并发轰炸，基本上“每隔几小时积分就会重新满格”，实现了真正的长期持续免费！

---

## 二、 7 款官方免费大模型全景

商汤日日新不仅有自研模型，还一站式聚合了当前最热门的开源与国产顶流：

1. **`sensenova-6.8-flash-lite`**：商汤自研主力轻量多模态智能体，针对图表、文档、办公场景专项深度优化。
2. **`sensenova-u1.5-fast` & `u1.5-lite`**：图像生成与精细化编辑一体模型，公测期调用传入 `watermark: false` 可免费免除平台水印！
3. **`deepseek-v4-flash` & `deepseek-flash`**：聚合 DeepSeek 强力深度思考推理大模型，直接扣通用池积分。
4. **`glm-5.2`**：聚合智谱 GLM 5.2 旗舰开源模型，通用语言理解极佳。
5. **`kimi-k3`**：聚合月之暗面 Kimi K3 超长文本模型，长文档检索分析标杆。

---

## 三、 3 步开通指引

1. 访问商汤日日新控制台直达链接：
   👉 [https://platform.sensenova.cn/console/keys](https://platform.sensenova.cn/console/keys)
2. 手机号输入验证码登录，系统自动为你开通 **0元/月的公测档位**；
3. 进入「API 密钥」管理页面创建密钥（支持最多 20 个 Key）；
4. 接口全面兼容 OpenAI 规范（Base URL: `https://token.sensenova.cn/v1`），官方文档更原生支持 Hermes Agent、OpenClaw 与各类开源客户端无缝接入。

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/blog/sensenova-points-strategy/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/radar/)
