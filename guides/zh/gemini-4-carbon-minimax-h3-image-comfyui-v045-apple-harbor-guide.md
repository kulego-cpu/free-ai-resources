---
title: "24G 卡跑 H3-Image 与 Gemini 4 Carbon 选型账：先拆权重体积，Harbor 暂别下单"
description: "对照 Hugging Face 文件清单与 ComfyUI v0.45.0 PR 记录，核算 MiniMax-H3-Image 从 135.32GB 原版压到 29.41GB 的运行条件；拆解 Gemini 4 Carbon 推广期后的成本与榜单短板，并给出苹果 Harbor 预览版的硬件观望理由。"
pubDate: 2026-11-11
author: "小可"
category: "工程拆解"
tags: ["Gemini 4 Carbon", "MiniMax-H3-Image", "ComfyUI", "Apple Harbor", "显存优化"]
featured: false
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/gemini-4-carbon-minimax-h3-image-comfyui-v045-apple-harbor-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


昨晚翻 `MiniMax-H3-Image` 文件树时，算完体积差点直接关网页。

33.1B 主干拆成了 12 个 BF16 分片。同库还放着 `transformer_diffusers/` 与两份 VAE。隔壁仓库又挂着 66.7 GB 文本编码器。不看列表直接 `git clone`？两百多 GB 固态硬盘当场见底。

同一天 11 月 10 日，还有几件关联极紧的事落地。DeepMind 放开了 `Gemini 4 Carbon` 全量接口。ComfyUI `v0.45.0` 合了 PR `#17655` 给 H3-Image 减负。9to5Mac 顺带挖出了苹果内测应用 `Harbor` 的雷电 5 内存池消息。

全摊在一张算力账上：云端大模型怎么压住 token 费，桌面卡怎么啃下百 GB 权重。下面只核一手文档、文件体积与 PR 记录。本地没拿裸机掐表实跑，出图秒数我一个字不瞎编。

## 135GB 的 H3-Image：不靠 ComfyUI v0.45.0 量化包，消费卡根本动不了

MiniMax 没另起新骨架。他们把 H3-Omni 主干砍掉音频支路，把文生图与图像编辑塞进了同一个 checkpoint。

原仓 `README.md` 标出的权重体积很沉：

- DiT 主干 33.1B 参数，50 层单流架构，隐藏维度 5376。BF16 占 66.28 GB。Embedding 与输出头留 FP32，外挂 12 MB 分段头文件。
- 文本编码器沿用 `Qwen3-VL-32B`。DiT 只读第 50 层输出。单这一项就是 66.7 GB。
- VAE 为 CNN 编码配 24 层 ViT 解码，16 倍空间压缩。官方 FP32 版占 2.34 GB。

三块硬加就是 **135.32 GB**。拉取前请务必过滤冗余目录，别白耗宽带：

```bash
hf download MiniMaxAI/MiniMax-H3-Image --exclude "transformer_diffusers/*" "vae_diffusers/*"
hf download MiniMaxAI/MiniMax-H3 --include "text_encoder/*"
```

权重虽胖，推理步数反而短。Model Card 写明该版本已蒸馏至 **6 步采样**。同时取消了 CFG，`guidance_scale` 锁死在 1。原生画幅直出 1600 万像素。1:1 可出 4096x4096，21:9 直出 6272x2688。

下面这张是官方发布的本地 SGLang 与线上 API 同配置对照：

<img src="https://img.freeaidog.com/images/blog/minimax-h3-image-local-vs-api-2026.webp" alt="MiniMax H3-Image 本地 SGLang 推理与官方 API 4736x3552 生图输出对比" />

真往内网部署时，先避开两处暗坑。

先看参数校验。无论跑 SGLang 还是 vLLM-Omni，推理步数只认 6 或留空。引导系数同样只能传 1。按 SDXL 老习惯填 20 步？后端立刻丢 `HTTP 400`。`negative_prompt` 也完全拦截，避雷词只能写进正面长句。

另一处坑藏在授权协议里。点开 `LICENSE` 第一条第 5 款就明白：**欧盟、英国、韩国及美国**全在排除适用区里。哪怕处于适用区域，年营收超 100 万美元也得另签授权。对外搭托管 API 同样得先申请书面许可。接出海商用项目，法务先过一遍条款。

消费卡拿什么跑通？靠当天发布的 **ComfyUI v0.45.0**。作者 `@kijai` 提交了 PR `#17655`，合进无音频版模型类与 PDD 调度器。配套还上了支持 16 张参考图的条件节点。

Comfy-Org 镜像仓直接重新打包了裁剪量化权重：

| 模型组件 | 镜像仓库文件名 | 量化格式 | 文件大小 | 本地存放目录 |
| --- | --- | --- | --- | --- |
| DiT 主干无损裁剪 | `minimax_h3_image_pruned_bf16.safetensors` | Pruned BF16 | 40.23 GB | `models/diffusion_models/` |
| DiT 主干均衡档 | `minimax_h3_image_pruned_int8_convrot.safetensors` | Pruned INT8 | 20.98 GB | `models/diffusion_models/` |
| DiT 主干极限档 | `minimax_h3_image_pruned_w4a8.safetensors` | Pruned W4A8 | 12.55 GB | `models/diffusion_models/` |
| 文本编码器 INT8 | `qwen3vl_32b_minimax_h3_int8_convrot.safetensors` | INT8 | 27.14 GB | `models/text_encoders/` |
| 文本编码器 FP4 | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | NVFP4 AWQ | 15.69 GB | `models/text_encoders/` |
| 轻量扩写器替代 | `qwen3.5_9b_w4a8_gptq.safetensors` | W4A8 GPTQ | 7.34 GB | `models/text_encoders/` |
| 图像 VAE | `minimax_h3_image_vae_fp16.safetensors` | FP16 | 1.17 GB | `models/vae/` |

最小组合一加是 **29.41 GB**。看 PR 备注时有个疑点很扎眼：`@kijai` 用单卡 RTX 5090 测通了 BF16 与 INT8 的 2048 图生图。单卡只有 32 GB 显存，怎么硬塞进 40.23 GB 的 BF16 权重？

说穿了就是分阶段卸载加 CPU offload。编码器算完先踢出显存。接着载入 DiT 走 6 步 Euler `simple` 采样，超容权重靠 PCIe 在内存间倒页。最后读进 1.17 GB VAE 出图。主机内存低于 64 GB 的机器硬冲 BF16 必卡无疑。普通 24 GB 卡直奔 W4A8 组合就行。

多图编辑时拉高画幅还是爆 OOM 怎么办？先压尺寸。官方文生图模板故意压在 4 MP 即 2048x2048 起步。多图连线前务必开 `Resize Image/Mask` 节点限制边长。此外 `v0.45.0` 顺带进了 Qwen 3.x 并行线性权重加速 `#17361`。它还修了 FP8 文本崩溃 `#17620`。SeedVR2 也改成按剩余显存切片 `#17508`。顺手升一版很值。

## Gemini 4 Carbon 选型账：长文本能打，仓库重构别盲切

10 月 28 日先给了安全团队小范围公测。11 月 10 日，DeepMind 全面放开了 `Gemini 4 Carbon`。现已接入 AI Studio 付费层、Android Studio 与 Antigravity。

表面报价看似温和。推广期沿用 Argon 费率：输入每百万 2 美元，输出 10 美元，缓存输入 0.1 美元。

别见 2 美元就改全量路由。脚注把涨价写得很直白：推广窗结束后单价翻倍。届时输入 4 美元，输出 20 美元。真正咬肉的在思考耗量——Carbon 专为长周期排查优化。单轮思考 token 吐得比 Argon 多一截。日常增删改查塞给它？钱真划不来。

直接看 DeepMind 放出的基准总表，短板也全摆在明面上：

<img src="https://img.freeaidog.com/images/blog/gemini-4-carbon-benchmarks-2026.webp" alt="Gemini 4 Carbon 与 Gemini 4 Argon、GPT-6 Astra、Claude Opus 5.5 的官方基准评测对比表" />

选型结论一句就够。Carbon 在 `AA Index v5.0` 拿了 59 分，长文本与防注入极强。可编码和桌面操作各有落后项：

- 百万字溯源与安全审计优先切 Carbon。`GraphWalks` 256k 到 1M 拿到 96.2% F1。对手被甩开二十余分。`SEC-bench Pro` 达 88.2%，Gray Swan 间接注入失手率为 0.0%。挂进 `/swarm` 当总控架构师，下面配轻模型写单测，性价比最稳。
- 复杂仓库级补丁生成别盲切它。`FrontierSWE 2` 上 Carbon 只拿 61.2%。这个成绩比 `GPT-6 Astra` 低了 4.3 个点。
- 桌面 GUI 连续操作也差一截。`OSWorld-2.1` 离线子集它拿 81.5%，落后 `Claude Opus 5.5` 5.7 个点。按路由切场景，别把总分当成万能券。

## 苹果 Harbor 预览版曝光：先别为了雷电 5 内存池掏钱

当天 `r/LocalLLaMA` 转热了另一条苹果消息。9to5Mac 引述 `@aaronp613` 的挖掘称，苹果正以 Claris 品牌内测 Mac 应用 **Harbor**。

点子简单粗暴。界面搜模型、点击下载、拉成私有服务。彻底甩开 Conda 环境与集群脚本。对标 135 GB 体重的，正是它的 **Thunderbolt 5** 互联。雷电线接起多台 Mac，Harbor 自动合并统一内存。

以前拿两台 Mac Studio 跑 66 GB 编码器有多麻烦？得手搓 MLX 环网配 SSH 隧道。多数开发卡在第一步就弃坑了。Harbor 想把门槛压到插线即用。

买机器的念头先按住。它还在 Claris 私有预览阶段，公开时间没影。底层开不开 OpenAI 兼容接口也没消息。更扎心的差距在计算速度。统一内存能堆出 128 GB 甚至 256 GB 容量，却追不上独显张量算力。拿 M 系 GPU 硬算 1600 万像素 DiT 重活，对上 RTX 5090 的 FP4 核心，出图耗时会好看吗？等社区贴出两台 Mac Studio 串联的实机日志再掏钱。现在下单是不是太早了？

---

本文涉及的核心数值、体积与条款，均可从以下一手链接直接核对：

| 核验条目 | 一手出处链接 |
| --- | --- |
| Gemini 4 Carbon 发布公告与定价脚注 | https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-4-carbon-now-available/ |
| Gemini 4 Carbon 基准数据原帖 | https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-carbon/ |
| MiniMax-H3-Image 模型卡与参数 | https://huggingface.co/MiniMaxAI/MiniMax-H3-Image |
| MiniMax 社区许可除外区域条款 | https://huggingface.co/MiniMaxAI/MiniMax-H3-Image/blob/main/LICENSE |
| ComfyUI PR #17655 节点合并记录 | https://github.com/Comfy-Org/ComfyUI/pull/17655 |
| ComfyUI 官方部署指南与模板要求 | https://docs.comfy.org/tutorials/image/minimax/minimax-h3-image |
| Comfy-Org 裁剪量化权重文件清单 | https://huggingface.co/Comfy-Org/MiniMax-H3-Image |
| 苹果 Claris Harbor 曝光报道 | https://9to5mac.com/2026/11/10/apple-has-new-mac-app-coming-to-make-running-local-ai-models-easy/ |

手头只有 24 GB 卡的话，升到 ComfyUI v0.45.0 先挂 W4A8 组合。先跑一遍 2048 峰值内存。云端代码流里，把长上下文审查拆给 Carbon，常规小改留在便宜模型上。这样账单最稳。你这周跑本地模型卡在哪一关——显存装不下，还是换页慢得受不了？

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/gemini-4-carbon-minimax-h3-image-comfyui-v045-apple-harbor-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

