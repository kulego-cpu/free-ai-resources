---
title: "5.9GB 塞进 27B 编码模型：Ternary Bonsai 2.5 三元量化实测与 llama.cpp 四大踩坑复盘"
description: "实测 PrismML 发布的 Ternary Bonsai 2.5 27B 1.58-bit 三元量化模型，核对 5.93GB 体积下的 SWE-bench 75.6 跑分与 RTX 4090 实测速率，拆解旧版 llama.cpp 静默乱码、默认四槽爆显存等四大部署陷阱。"
pubDate: 2026-11-25T13:35:00.000Z
author: 小可
category: 极客实操
tags:
  - Ternary Bonsai
  - 三元量化
  - llama.cpp
  - 显存优化
  - Qwen3.8
  - 本地编码
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/ternary-bonsai-2-5-27b-5gb-vram-agentic-coding-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


8GB 显存跑 27B 参数的代码智能体，真能跑通吗？

我一开始也怀疑。把 53.8GB 的 FP16 模型硬压到 5.93GB，每个权重只剩 1.76 bit。这难道不会直接退化成胡言乱语？

11 月 24 日，PrismML 放出了 `Ternary Bonsai 2.5 27B`。上线当天直接冲上 Hacker News 首页近 400 赞。

我翻了官方白皮书，又拉了仓库 `prism-ml/Ternary-Bonsai-2.5-27B-GGUF` 的说明。

结论：能跑。但别拿旧版 `llama.cpp` 直接拉，会当场踩坑。

---

## 为什么这次三元量化没崩？

以前的低比特量化一碰多轮工具调用就废。

上代 Bonsai 2 27B 在 DeepSWE 上直接吃鸭蛋，0 分。AppWorld 多步骤任务也只有 51.2 分。工具一旦报错，模型就原地打转，甚至自己编个结果宣布完工。

Bonsai 2.5 底座依然是 `Qwen3.8-27B`。语言主干 24.35B。视觉塔 0.47B。词表嵌入 2.54B。

存储格式其实没动，依然是 g128 分组缩放的 1.58-bit 三元权重。参数只取 `-1`、`0`、`+1` 三个值。离线阶段把分块 Hadamard 旋转矩阵直接折叠进权重里，推理时只需旋转激活值，不占额外显存带宽。

分数到底掉了多少？看官方按 `xhigh` 思考模式跑出的对照表：

| 评测维度与基准 | 体积对比 | Qwen3.8 FP16 | GSQ-RCO IQ2_XS | Bonsai 2 27B | Bonsai 2.5 27B |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 权重体积 | GB | 53.8 | 8.4 | 5.93 | 5.93 |
| 每权重比特数 | bpw | 16.0 | 2.50 | 1.76 | 1.76 |
| SWE-bench Verified | 编码 | 80.6 | 74.4 | 60.8 | 75.6 |
| DeepSWE v1.1 | 编码 | 30.1 | 17.7 | 0.0 | 27.0 |
| Terminal-Bench 2.1 | 终端 | 70.8 | 53.4 | 52.8 | 60.1 |
| AppWorld | 工具链 | 93.2 | 46.5 | 51.2 | 90.5 |
| τ³-Bench | 多轮工具 | 72.8 | 67.5 | 69.1 | 73.9 |

留意 `GSQ-RCO IQ2_XS` 那列。体积比 Bonsai 2.5 大四成，到了 AppWorld 直接摔到 46.5 分。Bonsai 2.5 五项工具调用平均拿到 77.2 分，离 FP16 只差 0.5 分。

这算全面超越原版吗？

不算。Terminal-Bench 2.1 比 FP16 低了 10.7 分。BFCL v4 也落后 3.6 分。HN 评论区还有人吐槽被前代死循环折磨过，怀疑新版本是不是对评测集做了定向优化？

1/9 显存换回九成编码能力，这买卖我认为值，至少值到我拿它顶了两天日常补全。

---

## 速度到底多快？别把 MTP 当基础版

很多转载帖拿 204 tok/s 到处吹，却不交代测试前提。

仓库其实拆出了两套文件。基础版不带草稿头。尾部带 `-mtp.gguf` 的版本，额外塞了一层多 Token 预测草稿头。

不加 MTP 草稿头，纯跑 `PTQ1_0` 原生推理是什么水平？看官方数据：

- RTX 4090：`PTQ1_0` 解码 112 tok/s；预填充 3,452 tok/s
- RTX 5090：`PTQ1_0` 解码 163 tok/s。预填充 4,198 tok/s
- Apple M5 Pro：Metal 仅测 `PQ2_0`。解码 30.2 tok/s

只有换上 `-mtp.gguf` 并加载 `presets/llama.cpp/mtp-cuda.json` 预设，RTX 4090 才能借投机解码冲上 204 tok/s。视觉塔 `mmproj-Q8_0.gguf` 也被独立拆了出去。纯写代码不加载它，又能省下 630MB 显存。

---

## 跑分表扫一眼就行，先说要命的四个坑

真到自己机器上跑，下面这四个坑才要命，每一个都能让你白折腾一晚上。

### 下了 5.99GB 的 PTQ1_0，原版 llama.cpp 直接拒载？

仓库挂了三种打包格式，各自绑定了不同运行时：

- `PTQ1_0`：5.99 GB，1.76 bpw 密集三元封装，必须用 `PrismML-Eng/llama.cpp` 分支
- `PQ2_0`：7.25 GB，每三元占 2-bit 槽位。专供 Apple Metal，同样依赖魔改分支
- `Q2_0`：7.67 GB，标准 `llama.cpp` 原生格式

懒得编译魔改分支，直接拿 `Q2_0` 跑原版行不行？

行。但有版本死线：**必须升级到 `b11556` 或更新版本**。这个版本才合入 Hadamard 变换算子。

更坑的是，旧版 `llama.cpp` 加载 `Q2_0` 时**根本不报错**。它会静默加载完，然后吐出一堆乱码。动手前先敲 `llama-cli --version` 核对版本，别直接盲跑。

### 4 个默认槽位把 10GB 显存吃穿

模型才 6GB 左右，为什么 10GB 显存的 RTX 3080 还是差点爆掉？

槽位。`llama-server` 默认开 4 个共享 KV 缓存槽，显存全给你算进去。我一开始也以为是模型本身的问题，查了半天才定位到 `-np`。Hugging Face 讨论区也有人踩过同样的雷：不限并发直接起服务，RTX 3080 整卡显存飙到 9.5 GB。桌面差点卡死。

加个 `-np 1` 锁死单槽位，显存立刻回落到 7.8 GB。

### min-p 0 忘了传，采样直接跑偏

`llama.cpp` 默认自带 `min_p=0.05` 截断。Bonsai 2.5 沿用 Qwen3.8 推荐采样，思考模式必须关掉 `min_p` 截断。漏传这个参数会怎样？长链条写代码极易陷入死板重复。

### -n 256 根本不够它想

这是个强思考模型，默认档位是 `xhigh`。而且**不支持 `low` 档位**。强行传 `low`，内部照样按接近 `xhigh` 的长度吐推理链。

终端随手给个 `-n 256`，模型光在 `<think>` 里分析报错就能耗光配额。最后一行有效代码都吐不出来。

---

## 可直接复制的端侧启动脚本

配好 PrismML 分支或 `b11556` 以上原版 `llama.cpp` 后，按下面参数起服务：

```bash
# 方案 A：使用 PrismML 分支跑 5.99GB 最小体积 PTQ1_0
git clone https://github.com/PrismML-Eng/llama.cpp && cd llama.cpp
cmake -B build -DGGML_CUDA=ON && cmake --build build -j

hf download prism-ml/Ternary-Bonsai-2.5-27B-GGUF Ternary-Bonsai-2.5-27B-PTQ1_0.gguf --local-dir .

./build/bin/llama-server -m Ternary-Bonsai-2.5-27B-PTQ1_0.gguf \
  -ngl 99 -fa on -np 1 -c 32768 -n 16384 \
  --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0 \
  --host 127.0.0.1 --port 8080
```

跑在 16GB 内存的 Mac 上，模型换成 `Ternary-Bonsai-2.5-27B-PQ2_0.gguf`。去掉 `-DGGML_CUDA=ON` 走 Metal。不想折腾编译？拉 7.67GB 的 `Q2_0`。喂给 `b11556+` 原版 `llama-server` 照样能跑通。

---

## 到底值不值得塞进日常工具链？

手里只有 8GB 显卡或 16GB 笔记本，以前连 Qwen3.8-27B 的边都摸不到。现在靠 6GB 磁盘体积换回七成半 SWE-bench 实力。离线补全多了一张底牌。

代价也很直白：1.76 bpw 绝不是全精度平替，Terminal-Bench 掉的 10 分就是硬伤。真拿这种极低比特模型跑几十轮复杂重构，到底能不能稳住不翻车？我还在用自己的私有仓库继续压测。

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/ternary-bonsai-2-5-27b-5gb-vram-agentic-coding-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

