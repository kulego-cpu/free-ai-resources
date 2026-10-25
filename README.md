# ⚡ Awesome Free AI Resources & High-Speed Endpoints
> 全球免费大模型 API、免绑卡/免 Key 渠道与端侧部署实操指南（持续更新）
> Curated list of 100% free AI models, zero-key endpoints, student perks, and local deployment hacks.

[![Awesome](https://awesome.re/badge.svg)](https://awesome.re)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/kulego-cpu/free-ai-resources/pulls)

---

## 🎁 一、 热门福利与学生认证专项

| 项目 | 核心权益 | 实操指南 | 状态 |
| :--- | :--- | :--- | :--- |
| **Google Gemini AI Pro** | 免费 1 年 Pro 会员 + 5TB Google One 空间 + 深度研究功能 | [SheerID 学信网认证与免税州全流程指南](https://freeaidog.com/blog/google-gemini-1-year-student-pro-sheerid-guide/) | 🟢 官方进行中 (至12月31日) |
| **商汤 SenseNova** | 日日新 6.8 Flash-Lite，全量适配 Harness，低延迟推理 | [商汤大模型端点测评与免费调用指南](https://freeaidog.com/blog/sensenova-points-strategy/) | 🟢 公测免费中 |
| **SiliconFlow 硅基流动** | Qwen-2.5-7B、DeepSeek-V3 等数十款开源模型无限制公测 | 注册赠送额度，海量模型免充值可用 | 🟢 稳定可用 |

---

## ⚡ 二、 免 Key / 免绑卡直接调用渠道汇总

适合直接放进沉浸式翻译、NextChat、LobeChat、Cline 等客户端使用：

- **商汤日日新 SenseNova**：`https://token.sensenova.cn/v1`（免费支持 Harness 模式）
- **智谱 GLM-4-Flash**：个人开发者免 Token 费，超长上下文
- **讯飞星火 Spark-Lite**：全量免费无门槛 API
- **腾讯混元 Hunyuan-Lite**：支持高并发并发调用，0 门槛

> 💡 **实时健康度监测**：以上端点若遇波动，可随时通过 [FreeAI Dog 实时雷达](https://freeaidog.com/radar/) 查看秒级探活状态。

---

## 💻 三、 个人设备极客部署与端侧加速

没有高昂算力预算？用手头现有的轻量设备跑满性能：

1. **Mac mini 16G 极限榨干计划**：
   - 利用 Apple Neural Engine (ANE) 进行硬件级 NPU 矩阵计算加速，完全不占用 CPU/GPU；
   - 详细指南：[Mac mini 16G 释放 ANE 神经引擎：Qwen-3.8B 狂飙实操](https://freeaidog.com/blog/mac-mini-16g-apple-neural-engine-anemll-qwen-guide/)
2. **8GB 显卡跑 262K 超长上下文**：
   - 巧用 llama.cpp `-ot` 异构卸载与 `fa` FlashAttention 技术，解决小显存跑长文本 OOM 痛点；
   - 详细指南：[Qwen-Flash-Next-Coder 8GB 显存畅跑 262K 实操](https://freeaidog.com/blog/qwen-flash-next-coder-8gb-gpu-262k-context-llamacpp-guide/)

---

## 🌐 四、 推荐在线检测工具与数据源

- **[FreeAI Dog (AI 模型与探活雷达)](https://freeaidog.com)**: 全球免费 AI 接口实时可用率探测、真实端到端延迟测速（TTFT / TPS）、支持中英日韩四语。
- **[Astro 现代静态架构](https://astro.build)**: 极速 SEO 友好的开源博客框架。

---

## 🤝 贡献与交流

欢迎提交 Issue 和 Pull Request 分享你发现的新免费端点或优质折腾教程！
