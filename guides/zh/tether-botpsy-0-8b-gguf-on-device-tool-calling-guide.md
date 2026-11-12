---
title: "530MB 跑端侧工具调用：Tether BotPsy-0.8B GGUF 实测口径与模板默认坑"
description: "拆解 Tether AI Research 开源的 BotPsy-0.8B 端侧 Agent 模型：基于 Qwen3.5-0.8B 微调、Q4_K_M 与 Q8_0 量化体积与掉分实账、embedded chat template 默认关闭思考模式的启动坑，以及 0.8B 参数在 BrowseComp-Plus 仅 4.5 分的真实能力边界。"
pubDate: 2026-11-12
author: "小可"
category: "端侧与量化"
tags: ["BotPsy-0.8B", "GGUF", "llama.cpp", "端侧 Agent", "Qwen3.5"]
featured: false
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/tether-botpsy-0-8b-gguf-on-device-tool-calling-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


发稳定币的 Tether 居然跑来开源端侧小模型了。名字叫 `BotPsy-0.8B`，基于 `Qwen3.5-0.8B` 微调，主打在手机和低配笔记本上本地跑函数调用。

模型小到什么程度？`Q4_K_M` 量化版只有 `0.53 GB`。准确地说是 `504.8 MiB`。`Q8_0` 版本也才 `0.81 GB`。

BFCL v4 拿到 `44.0` 分，`ToolSandbox` 64.6 分，把 `2.5` 倍体量的 `Qwen3.5-2B-Think` 压在身下。分数是真好看。可你要现在就把 `gguf` 拖进手机，大概率跑出一堆废 JSON。把主仓库和 `GGUFs` 附注啃完，有三个地方得先看。

信源就三处：`huggingface.co/qvac/BotPsy-0.8B`、`huggingface.co/qvac/BotPsy-0.8B-GGUFs` 模型卡，外加官方博客 `huggingface.co/blog/qvac/botpsy`。没拿安卓真机测 `32768` 上下文内存峰值。耗电与吞吐数字不写。没条件测。

## 为什么 0.8B 能干翻 2B？别只看总分

`BotPsy-0.8B` 的底座是 `Qwen3.5-0.8B`。混合 `Gated DeltaNet` 加门控注意力，纯文本调用，Apache 2.0 开源。

对比基座 `Qwen3.5-0.8B-Think`：`BFCL v4` 从 `23.4` 拉到 `44.0`。`ACEBench` 从 `30.1` 跳到 `58.7`。`τ-bench` 三项均值从 `23.2` 升到 `51.5`。基本贴着基座翻倍。这不是泛化涨点，是微调把工具调用这条路径单独训出来了。在幻觉拒答测试上——没工具时能不能拒绝调用——`BFCL v4` 拿了 `86.1` 分。

往下拉到知识类基准，短板马上露出来。

`GPQA Diamond` 只有 `38.1` 分。`Qwen3.5-2B-Think` 可是 `59.9` 分。`MMLU-Pro` 也只有 `48.9` 分对 `65.5` 分。更扎眼的是 `BrowseComp-Plus`——最高才 `4.5` 分。开放式长链网页检索别想了。它只适合“意图明确、工具集合有限”的短链调用。

## 启动第一坑：内嵌模板默认关着思考模式

这是最容易让新人测出垃圾结果的地方。

`BotPsy-0.8B` 训练和评测全程开着思考模式。可官方仓库明确写了一句：内嵌 Chat Template 默认关闭思考。直接拿 `llama-cli` 加载而不手动传参会怎样？模型会跳过推理步骤直接吐 JSON，工具调用准确率直接打折。

官方给的 `llama.cpp` 启动命令把采样参数卡得很死：

```bash
hf download qvac/BotPsy-0.8B-GGUFs botpsy-0.8b-q8_0.gguf --local-dir .

llama-server -m botpsy-0.8b-q8_0.gguf --jinja -c 32768 \
    --temp 0.3 --top-p 0.95 --top-k 20 --presence-penalty 1.5 \
    --chat-template-kwargs '{"enable_thinking": true}' --port 8080
```

注意两个反常参数。一个是 `--chat-template-kwargs '{"enable_thinking": true}'`。少了它，跑出来的根本不是评测表里那个 `BotPsy`。另一个是 `--presence-penalty 1.5`。常规小模型部署很少把重复惩罚拉到这么高。沿用客户端默认的 `0` 或 `1.0`，小参数模型极容易在思考段复读卡死。

顺带一提，GGUF 导出版本把原底座的多词预测层 `MTP` 剥掉了。而且官方推荐仅用英文指令。拿中文复杂业务单据硬喂，效果会掉多少？模型卡里一个字没提。

## 量化账本：Q8_0 比 BF16 省一半，引擎差异却更大

仓库里一共放了三个 GGUF 文件。按文件清单和评测表核出来的账，选型其实没啥悬念。

BF16 那个 `1446.5 MiB`，约 `1.52 GB`，十项均分 `50.4`，居然比 Q8_0 还低 `0.3`。别下，1.5G 纯占地方，除了给评测做基线对照没啥用。真部署就两条路：桌面端和 8G 手机直接上 `botpsy-0.8b-q8_0.gguf`，`774.2 MiB`、KL 散度 `0.001`、均分 `50.7`，体积砍半而分数无损。实在抠内存再考虑 `botpsy-0.8b-q4_k_m-imat.gguf`，体积 `504.8 MiB`，官方加了 `imatrix` 校准把 KL 从普通 `Q4_K_M` 的 `0.048` 压到 `0.036`，可均分掉到 `48.9`，`GPQA Diamond` 和 `MMLU-Pro` 各掉三四分——省那两百多兆去换知识退化，我不干。

这里有个反直觉的地方。同一套无损 BF16 权重，vLLM 和 `llama.cpp` CUDA `b12169` 的 `τ-bench` 差 `4.0` 分，`Claw-Eval` 差 `3.1` 分，`IFBench` 反倒高 `2.8` 分。推理引擎和采样波动的误差，比从 BF16 降到 Q8_0 还大。迷信小数点后一位的榜单，意义大吗？

## 什么活能交给它，什么活别碰

这东西定位很清晰。它不是桌面端 `7B`、`14B` 的替代品。

如果你在做端侧语音助手、日程触发器或本地文件整理——工具固定、入参规整、不用跨十几轮——花 `800MB` 内存换一个能离线跑、带 `Apache 2.0` 授权的引擎，这笔账算得过。

指望 `0.8B` 去接复杂代码重构、中文长文档解析或者跨网检索？`BrowseComp-Plus` 那 `4.5` 分已经把天花板画在墙上了。

动手拉权重之前，先核对这三件事：

1. 客户端必须能往 `llama-server` 注入 `enable_thinking: true`。只用默认模板，跑出来的根本不是一回事。
2. 内存够 `1GB` 余量就选 `botpsy-0.8b-q8_0.gguf`。省那两百多兆去用 `Q4_K_M` 赌知识退化，图啥？
3. 控制好输入语种与窗口长度。示例给了 `-c 32768`，端侧设备拉满 32K 加上长思考输出，延迟和内存真扛得住吗？先拿短上下文压测再说。

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/tether-botpsy-0-8b-gguf-on-device-tool-calling-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

