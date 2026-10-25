# ⚡ Awesome Free AI Resources & High-Speed Endpoints
> 全球免费大模型 API、免绑卡/免 Key 渠道与端侧硬件加速实操指南（持续探活更新）
> Curated directory of 100% free AI models, zero-key endpoints, student perks, and local deployment hacks.

[![Awesome](https://awesome.re/badge.svg)](https://awesome.re)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/kulego-cpu/free-ai-resources/pulls)
[![Languages: zh-en-ja-ko](https://img.shields.io/badge/Languages-zh%20%7C%20en%20%7C%20ja%20%7C%20ko-orange.svg)](https://freeaidog.com)

---

## 🗺️ 一、 FreeAI Dog 平台全功能矩阵导航

[FreeAI Dog (freeaidog.com)](https://freeaidog.com) 是一个完全免费、无广告的公益 AI 基础设施导航与探活平台。以下是核心功能直达入口：

| 模块 | 核心功能 | 直达链接 | 状态 |
| :--- | :--- | :--- | :--- |
| 🛰️ **可用性雷达** | 200+ 免费/公测 API 秒级探活、免Key/免绑卡门槛过滤、端到端延迟监控 | [👉 前往雷达监测站](https://freeaidog.com/radar/) | 🟢 实时秒级探活 |
| 🧪 **前沿模型库** | 60+ 主流大模型（DeepSeek-V3/R1、Qwen-2.5、Llama-3.3等）参数规格与上下文窗口 | [👉 查看前沿模型全景](https://freeaidog.com/models/) | ⚡ 动态基准收录 |
| 🛠️ **开发者工具箱** | 在线连通性自测 (API Tester)、故障无缝热备迁移配置、Token 成本计算 | [👉 启动实用站长工具](https://freeaidog.com/tools/) | 💡 纯前端免登录 |
| 📚 **极客与羊毛博客** | 硬件潜能榨干（Mac mini ANE、8G显卡长文本）、SheerID认证避坑实操 | [👉 阅读深度实操专栏](https://freeaidog.com/blog/) | 📖 中英日韩四语 |

---

## 🎁 二、 热门福利与学生认证专项

| 厂商 / 项目 | 核心权益 | 实操指南 | 准入门槛 |
| :--- | :--- | :--- | :--- |
| **Google Gemini AI Pro** | 免费 1 年 Pro 会员 + 5TB Google One 空间 + 深度研究功能 | [SheerID 学信网认证与免税州全流程指南](https://freeaidog.com/blog/google-gemini-1-year-student-pro-sheerid-guide/) | 💳 需身份认证 (至12/31) |
| **商汤日日新 SenseNova** | 日日新 6.8 Flash-Lite，全量适配 Harness，超低延迟推理 | [SenseNova 端点评测与 Harness 适配攻略](https://freeaidog.com/blog/sensenova-points-strategy/) | 🎁 免绑卡即用 |
| **SiliconFlow 硅基流动** | Qwen-2.5-7B、DeepSeek-V3 等数十款开源模型无限制调用 | [硅基流动代金券使用技巧与端点优化](https://freeaidog.com/blog/siliconflow-voucher-tips/) | 🎁 注册即赠额度 |

---

## ⚡ 三、 零门槛免 Key / 免绑卡直接调用渠道汇总

适合直接放进沉浸式翻译、NextChat、LobeChat、Cline 等客户端使用：

- **商汤日日新 SenseNova**：`https://token.sensenova.cn/v1`（免费支持 Harness 模式）
- **智谱 GLM-4-Flash**：个人开发者免 Token 费，128K 超长上下文
- **讯飞星火 Spark-Lite**：全量免费无门槛 API
- **腾讯混元 Hunyuan-Lite**：高并发调用，免预充值

> 💡 **实时状态提醒**：第三方车队与公益端点易发生波动，调用前可前往 [FreeAI Dog 可用性雷达](https://freeaidog.com/radar/) 确认端点健康度红绿灯。

---

## 💻 四、 个人设备极客部署与端侧加速

没有高昂算力预算？用手头现有的轻量设备跑满性能：

1. **Mac mini 16G 极限榨干计划**：
   - 利用 Apple Neural Engine (ANE) 进行硬件级 NPU 矩阵计算加速，完全释放 CPU/GPU；
   - 详细指南：[Mac mini 16G 释放 ANE 神经引擎：Qwen-3.8B 狂飙实操](https://freeaidog.com/blog/mac-mini-16g-apple-neural-engine-anemll-qwen-guide/)
2. **8GB 显存畅跑 262K 超长上下文**：
   - 巧用 llama.cpp `-ot` 异构卸载与 `fa` FlashAttention 技术，解决小显存长文本 OOM 痛点；
   - 详细指南：[Qwen-Flash-Next-Coder 8GB 显存畅跑 262K 实操](https://freeaidog.com/blog/qwen-flash-next-coder-8gb-gpu-262k-context-llamacpp-guide/)
3. **老卡焕新：RTX 3090 算力解锁**：
   - [RTX 3090 深度解锁 BF16 硬件加速与 Kernel 调优](https://freeaidog.com/blog/rtx-3090-gpu-unlock-bf16-kernel-module-guide/)

---

## 🛠️ 五、 开发者在线实用工具箱

- **[在线 API 连通性测试 (Tester)](https://freeaidog.com/tools/tester/)**: 免装 cURL，一键测试任意 OpenAI 兼容端点的响应速度与返回有效性。
- **[故障无缝热备迁移配置器 (Failover)](https://freeaidog.com/tools/failover/)**: 自动生成双端点智能降级与重试配置代码。
- **[Token 消耗与成本计算器 (Calculator)](https://freeaidog.com/tools/calculator/)**: 实时测算不同大模型在中英文长文本下的 Token 开销。

---

## 🤝 贡献与交流

欢迎提交 Issue 和 Pull Request 分享你发现的新免费端点、公益车队或折腾技巧！

- 官方网址：[https://freeaidog.com](https://freeaidog.com)
- 语言支持：中文 (zh) / English (en) / 日本語 (ja) / 한국어 (ko)
