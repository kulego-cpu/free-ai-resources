---
title: "月之暗面 Moonshot Kimi API 实测指南：国内实名送 15 元代金券、国际站 $1 门槛与预付费零扣费全解析"
description: "深度评测清华系 AI 独角兽月之暗面（Moonshot Kimi）开放平台！全面拆解「国内站送 15 元代金券」与「国际站无免费层需充值 $1」的双轨真相、新旗舰 K3 准入门槛、Tier 0~5 阶梯限流、CORS 精确匹配与 5 大工程避坑指南。"
pubDate: 2026-10-19
author: "FreeAI Hub 编辑部"
category: "国内直连"
tags: ["月之暗面", "Moonshot", "Kimi", "长文本", "代金券", "预付费", "国内直连"]
featured: false
---

# 月之暗面 Moonshot Kimi API 实测指南：国内实名送 15 元券、国际站充值门槛与预付费零风险解析

作为中国大模型领域由清华大学杨植麟博士领衔创立的头部独角兽，**月之暗面（Moonshot AI）** 旗下的 **Kimi** 凭借在超长文本无损理解、高精度网络搜索整合以及学术级长篇资料解析上的亮眼表现，成为无数开发者、产品经理和科研人员的日常首选。

然而，在开发者 API 生态中，很多初次接触 Moonshot 的同学经常产生严重误区：**“Kimi API 到底是免费的还是纯付费的？”、“为什么我注册了却提示余额不足无法调用？”**

经过 2026 年 10 月最新实测，我们发现 Moonshot 实际上采取了极为严谨的**「境内境外双轨制」**运营策略：
- 🇨🇳 **国内站（`platform.moonshot.cn`）**：使用大陆手机号注册并完成身份证实名认证后，**自动到账 15 元 API 代金券（等值约 1,500 万 Tokens）**，可直接零充值调用 `moonshot-v1` 全系列！
- 🌐 **国际站（`platform.moonshot.ai`）**：采用纯美元计价与严格预付费模式，**完全没有免费体验层**，必须预先充值最低 $1 激活 Tier 0 才能调用。
- 🔒 **资金安全保障**：无论是国内站还是国际站，均采用**纯预付费（Prepaid）架构**，余额或代金券耗尽后干净返回 `HTTP 402 Payment Required`，**绝无信用卡静默扣费或透支天价账单风险**！

本文将为你深度理清国内站与国际站的核心规则、15 元代金券可用模型范围、Tier 0~5 分层限制及 5 大关键避坑细节。

---

## 一、 核心政策与双轨机制：15 元代金券 vs $1 起充

| 评估维度 | 国内站 (`platform.moonshot.cn`) | 国际站 (`platform.moonshot.ai`) | 核心解析与建议 |
| :--- | :--- | :--- | :--- |
| **免费体验类型** | 🟢 **`voucher`（实名送 15 元代金券）** | ❌ **`none`（无免费层）** | 国内开发者务必走国内站注册 |
| **免费起步门槛** | 手机号注册 ＋ **中国大陆身份证实名** | 需充值最低 **$1** 成为 Tier 0 | 国际站面向海外用户，无免费羊毛 |
| **可用模型范围** | **`moonshot-v1-8k/32k/128k` 全系列** | 全系模型（依充值 Tier 解锁） | **注意：15 元代金券不可直接用于 K3** |
| **后续代金券** | 暂无固定自动赠券 | 累计充值满 $5 赠送 $5 代金券 | 代金券不计入账户累充 Tier 升级 |
| **扣费与风控** | 严格预付费，余额耗尽返 402 | 严格预付费，余额耗尽返 402 | 零信用卡自动扣款风险，用完即止 |
| **CORS 跨域** | ⚠️ 精确匹配 Origin（非通配符 `*`） | ⚠️ 精确匹配 Origin（非通配符 `*`） | 预检回显来源 Origin，纯前端需配置 |
| **OpenAI SDK** | 🟢 完全兼容 `/v1/chat/completions` | 🟢 完全兼容 `/v1/chat/completions` | 仅需切换 Base URL 与 API Key |

---

## 二、 关键模型可用性与分档

### 1. 15 元代金券可用模型（主流生产级长文本）
新用户完成实名后赠送的 15 元代金券，可以全额抵扣以下经典基座模型：
- **`moonshot-v1-8k`**（输入 ¥0.012/k tokens，输出 ¥0.012/k tokens）：通用对话主力，中文理解细腻，15 元约可支撑百万级 Token 交互；
- **`moonshot-v1-32k`**：中长篇研报、小说章节或长文档的解析与问答；
- **`moonshot-v1-128k`**：高达 128K 上下文的全库代码审计与多篇论文横向对比。

### 2. 需首次现金充值解锁的模型
- **`kimi-k3`**：月之暗面最新一代推理旗舰，官方政策明确标注**新用户赠送的 15 元代金券不可直接用于 K3**。需在开放平台至少完成一次现金充值（最低 ¥10 或 $1）后方可开放权限；
- **`kimi-k2.7-code`**：编程特化模型，代码补全与复杂算法推演表现优异。

---

## 三、 5 大工程避坑指南 (Gotchas)

### 1. ⚠️ 务必走国内站认证以解锁 15 元代金券
许多开发者从 GitHub 或第三方文档直接访问了 `platform.moonshot.ai`（英文国际站），发现系统强制要求充值 $1 才能生成 Key 并发起调用，误以为平台已经全面取消免费体验。**请务必认准国内站 `platform.moonshot.cn`**，注册后进入「账号管理」完成身份证实名核验，15 元代金券便会自动到账！

### 2. ⚠️ 15 元代金券无法调用 Kimi K3
如果你调用 `kimi-k3` 报错提示权限不足或额度受限，并非代金券失效，而是平台对新一代旗舰设立了门槛保护。15 元代金券仅支持 `moonshot-v1-8k/32k/128k`。如需体验 K3 的深度推理能力，需在控制台充值一次最低档位（¥10）。

### 3. ⚠️ 代金券不计入 Tier 阶梯累计充值
月之暗面 API 设有 Tier 0 至 Tier 5 的严格流控阶梯（决定并发数、RPM 与 TPM）。平台的晋级规则是按照「**累计实际现金充值金额**」计算的，平台赠送的 15 元代金券不计入累充总额，因此使用代金券期间默认享受基础并发保护。

### 4. ⚠️ 消费端 Kimi App 与开发者 API 完全隔离
在 kimi.com 网页端或手机 App 上的个人免费额度，与开发者开放平台的 API 是完全独立结算的。App 端的免费对话或会员权益**不能**转换为 API Tokens，反之亦然。

### 5. ⚠️ CORS 采用 Origin 精确回显机制
实测 `api.moonshot.cn` 在处理前端 OPTIONS 预检请求时，下发的 `access-control-allow-origin` 为发起请求的具体域名，而非所有来源的通配符 `*`。纯前端单页应用（SPA）或浏览器扩展直连时需注意跨域域名安全校验。

---

## 四、 快速上手与配置代码

### 1. Python (OpenAI SDK 标准兼容)

```python
import os
from openai import OpenAI

# 填入在 platform.moonshot.cn 获取的 sk- 开头密钥
client = OpenAI(
    api_key=os.environ.get("MOONSHOT_API_KEY", "your_moonshot_api_key"),
    base_url="https://api.moonshot.cn/v1"
)

response = client.chat.completions.create(
    model="moonshot-v1-8k",
    messages=[
        {"role": "system", "content": "你是月之暗面 Kimi 大模型，擅长长文本分析与精准中文表达。"},
        {"role": "user", "content": "请用三句话概述月之暗面 Kimi 的核心技术特点。"}
    ],
    temperature=0.3
)

print(response.choices[0].message.content)
```

### 2. cURL 命令行极速测试

```bash
curl -X POST "https://api.moonshot.cn/v1/chat/completions" \
  -H "Authorization: Bearer $MOONSHOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "moonshot-v1-8k",
    "messages": [{"role": "user", "content": "9.11 和 9.8 哪个数更大？"}],
    "temperature": 0.3
  }'
```

---

## 五、 三步入坑指引

1. **注册国内站**：访问 [platform.moonshot.cn](https://platform.moonshot.cn/) 使用中国大陆手机号注册，并完成身份证实名认证；
2. **核验证金并领 Key**：在「费用中心」确认 15 元代金券已到账，在「API Key 管理」生成专属 sk- 密钥；
3. **配置客户端开跑**：将 Base URL 设为 `https://api.moonshot.cn/v1`，填入 Key，模型选用 `moonshot-v1-8k` 或 `moonshot-v1-32k`，零充值畅享 200K 长文本理解！

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/blog/moonshot-kimi-api-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/radar/)
