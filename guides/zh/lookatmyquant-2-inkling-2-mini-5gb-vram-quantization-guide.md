---
title: "5.4GB 显存硬塞 26B 循环 MoE：LookAtMyQuant 2 极限压缩 Inkling-2-Mini 踩坑实录"
description: "拆解 ShinMegamiBoson 开源量化编译器 LookAtMyQuant 2 的底层机制。实测 Fisher 信息动态位宽与网格编码如何将 26.4B 物理权重的循环 MoE 模型压入 5.38GB 内存，复盘 Apple Silicon Metal 自定义算子编译报错与精度退化边界。"
pubDate: 2026-11-22
author: FreeAI Hub 站长
category: 架构剖析
tags:
  - LookAtMyQuant
  - Inkling-2-Mini
  - 循环架构
  - 模型量化
  - 显存优化
  - Apple Silicon
featured: false
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/lookatmyquant-2-inkling-2-mini-5gb-vram-quantization-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


前几天看到 Thinking Machines Lab 低调推送了 Inkling-2-Mini 权重的开源提交。

这个由前 OpenAI 团队操刀的模型，骨架设计相当罕见。它采用了一种少见的循环混合专家结构。物理总参数量高达 26.4B，但单 Token 激活参数只有 3.4B。中间几层网络在单次前向推理中会被反复循环调用三次。它试图在不堆砌物理显存的前提下，榨出深层的逻辑推演能力。

想法很精妙。但拉下官方的 FP16 原生权重一称，整整 18.89GB。

手头那些 8GB 显存的显卡，或者 8GB 内存的 MacBook，连底模都塞不下。卡死在门外。

正当我准备合上电脑时，Hacker News 首页冒出一条热帖。开发者 ShinMegamiBoson 开源了 LookAtMyQuant 编译器。他把这颗模型压进了 5.4GB 内存。平均位宽直接砍到了 1.63 bpw。

1.63 bpw 是什么概念？常规 GGUF 压到 IQ2 甚至 IQ1 时，输出往往已经语无伦次。5.4GB 到底能不能吐出正常句子？还是又一场纸面参数的自嗨？

我把工具链拉到本地，连夜在两台物理机器上跑了一轮实测。

---

## 1. 极限裁剪机理：为什么常规量化在 1.63 位会当场暴毙

平时用的 AWQ 或者 GPTQ 压到 4-bit 还算好用，为什么跌破 2-bit 就会语无伦次？

因为矩阵里每个权重的重要性天差地别。

传统一刀切的均匀量化，把整张张量按固定网格截断舍入。高位宽下冗余充足，损伤不易察觉。一旦跌破 2-bit，少数维系注意力方向的关键权重会被强行抹平。模型逻辑当场雪崩。

LookAtMyQuant 2 换了一套编译思路。翻看其开源源码，核心依赖两项机制：Fisher 信息量动态位宽分配，以及网格编码寻优。

### 1.1 Fisher 信息敏感度分级

LAMQ 并不假设所有层地位均等。

编译器首先使用少量 C4 文本与开源代码片段做前向校准，计算每一层权重对输出层概率分布的二阶导数曲率，也就是 Fisher 信息矩阵。

曲率越高，意味着权重哪怕稍有扰动，输出语义就会剧烈漂移。曲率平缓的区域，就算大幅裁切，损失也微乎其微。

基于曲率图谱，LAMQ 动态切分权重预算：

词表嵌入层、输出投影层与底层注意力矩阵，强制分配 3-bit 或 4-bit 重点保护；循环体内部那些低频调用的微专家 FFN 层，则直接砍到 1-bit 甚至 1.5-bit。

加权平均算下来，整机位宽刚好卡在 1.63 bpw 上下。

### 1.2 网格编码与逐层蒸馏修复

确定位宽后，它没有采用简单的最近邻舍入，而是引入了 Trellis 编码。

算法在网格状态图上通过动态规划寻找重构路径，抵消舍入误差。量化完成后，再针对高精度输出做一轮逐层蒸馏修复。

纸上算着很美。真拉起来跑，先撞上的是内存带宽。解包那层位运算一开，延迟跟着一层一层往上叠。剩下的全是体力活。

---

## 2. 本地部署现场与 Metal 算子排障记录

本次实测选用了两套物理测试环境：

第一套选用单卡 RTX 4060 8GB 环境。驱动版本 550。搭配 CUDA 12.4。

第二套选用 Apple Silicon M3 芯片搭配 16GB 统一内存，运行 macOS 15.1 系统。

### 2.1 源码编译与头文件缺失暗坑

LookAtMyQuant 针对 Apple Silicon 提供了专门的 Metal 解包算子。敲下命令安装其核心 CLI：

```bash
pip install git+https://github.com/ShinMegamiBoson/lookatmyquant.git
```

拉取已经完成编译的量化模型权重并尝试启动生成：

```bash
lamq generate shinmegamiboson/Inkling-2-Mini-LookAtMyQuant-5.4GB \
  --prompt "Write a quicksort in Python."
```

终端在加载完前几个张量切片后，突然弹出红色编译崩溃：

```text
MetalCompilerError: Compilation failed: unknown type name 'half4x4'
  Device::newLibraryWithSource failed to compile custom kernel: lamq_trellis_unpack.metal
```

报错直接卡在解包算子的即时编译阶段。

翻看更新日志才弄明白。Sequoia 系统下 Xcode Command Line Tools 的 Metal 路径做了调整。未指定路径时，编译器找不到部分紧凑矩阵类型。

排查清楚后，解决起来很直接。在终端中显式补上工具链环境变量：

```bash
export METAL_XCODE_TOOLCHAIN_PATH="/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain"
```

重新敲下启动命令。

终端顺利刷出着色器缓存写入日志，进程顺利进入生成阶段。

### 2.2 显存与内存物理水位

模型跑起来后的物理资源开销如下：

- 底模权重静态占用 5.38GB 空间。
- 在 4,096 Token 上下文设置下，分配 KV Cache 约 1.15GB。
- 运行时与框架开销约 0.35GB。
- 整机峰值显存稳定在 6.88GB。

在 8GB 显存的 RTX 4060 上，显存占用没有触发 OOM 崩溃；在 16GB 的 Mac 上，系统也没有产生任何磁盘 Swap 交换。

物理体积确实压缩到了承诺的 5.4GB 级别。

---

## 3. 真实吞吐性能：循环架构带来的时间代价

显存挤进去了，速度到底怎么样？

很多人误以为 5.4GB 的模型一定跑得飞快。跑完我沉默了。5.4GB 是塞进去了，速度一点都不温柔。

针对不同输入长度进行多轮基准压测，记录吞吐数据如下：

| 输入上下文长度 | Prefill 吞吐速度 | Decode 生成速度 | 峰值显存占用 |
| :--- | :--- | :--- | :--- |
| 512 Tokens | 74.2 tokens/s | 23.8 tokens/s | 6.42GB |
| 2,048 Tokens | 68.5 tokens/s | 22.4 tokens/s | 6.68GB |
| 4,096 Tokens | 59.1 tokens/s | 20.6 tokens/s | 6.88GB |
| 8,192 Tokens | 46.3 tokens/s | 18.2 tokens/s | 7.35GB |

解码速度为什么会被压在每秒 20 到 23 个 Token？

这正是循环 MoE 架构必须支付的物理代价。

模型物理权重虽然只有 5.4GB。但在单步解码中，中间核心层被强制循环前向计算了三次。这意味着生成每个 Token，硬件承担的计算量相当于 70 层以上的深层网络。

此外，Trellis 超低位宽解包需要在着色器中执行位运算还原，引入了额外的反量化开销。

能跑，但绝不是闪电侠。日常文本润色或小段代码生成还算顺手；指望拿它顶高并发 API 或者低延迟流式语音，延迟根本顶不住。

---

## 4. 精度真实退化边界测试

吹得再花哨，终究要回到文字输出质量上。

我挑选了三组不同梯度的真实任务进行验证，直接观察 1.63 bpw 的文字表现。

### 4.1 基础代码与逻辑解释

测试提示词要求编写带类型提示的异步并发请求限制器。

输出代码结构清晰；正确引入了 asyncio.Semaphore 信号量；类型注解、异常捕获、单测用例一应俱全，没有语法硬伤。对于常规软件工程场景，模型的逻辑底座依然扎实。

### 4.2 极端数值与符号推演

把提示词换成连续四步的矩阵推导，要求保持严格格式输出。

在这里，1.63 bpw 的暗伤藏不住了。

在第三步数值推演时，模型出现了一次符号错位。反复测了三次，格式对齐准确率从 4-bit 版本的 96% 下滑到了 88%。

说白了，硬核数值推理这块，1.63 bpw 就是不配。矩阵一深，符号开始飘。想拿它做结构化输出，多跑几轮就露底。

天下没有免费的算力午餐。

---

## 5. 硬件劝退线与选型建议

测试跑完，这套方案适合谁？

第一，手头只有 8GB 显卡或 8GB 内存 Mac 的读者——如果执意想在本地体验 26B 规模的推演，LAMQ 2 确实是目前极少数能塞进去的入场券。

真能塞进去吗？确实塞进去了。

第二，手头如果有 16GB 或 24GB 显存，老老实实去跑 4-bit 量化；没必要为了省空间去硬啃 1.63 bpw。位宽损失对严谨数值推导的侵蚀是客观物理规律。

第三，不要迷信跑分神话。存储是省了。代价呢？生成每个 Token，中间那几层循环全要多算三遍。省下来的空间，时间还回去，还倒贴。

后续是否会有更高效的解包汇编算子合入上游？不妨边跑边看。把算力和物理账本算清，比盲目尝鲜踏实得多。

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/lookatmyquant-2-inkling-2-mini-5gb-vram-quantization-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

