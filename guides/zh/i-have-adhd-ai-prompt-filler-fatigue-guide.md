---
title: "告别 AI 废话与 Token 刺客！GitHub 爆火神技 i-have-adhd 全平台实战指南：让 Claude Code / Cursor / ChatGPT 一秒变干练"
description: "每次问 AI 报错都要忍受客套铺垫与长篇大论？本文深度拆解 GitHub 斩获数万 Star 的开源规则神技 ayghri/i-have-adhd。通过 10 大硬核认知护栏与发前自我过滤，彻底消灭 AI 废话疲劳，首行直出运行命令，暴降 40%~60% 冗余 Token。内附 Claude Code、Cursor、ChatGPT 极速配置教程与中英双语即用 Prompt。"
pubDate: 2026-10-26
author: "小可 (FreeAI Dog 核心开发组)"
category: "实战进阶"
tags: ["Prompt工程", "ClaudeCode", "Cursor", "i-have-adhd", "Token优化", "AI编程助手", "生产力黑客", "规则约束"]
featured: true
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/i-have-adhd-ai-prompt-filler-fatigue-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


# 告别 AI 废话与 Token 刺客！GitHub 爆火神技 i-have-adhd 全平台实战指南：让 Claude Code / Cursor / ChatGPT 一秒变干练

你是否也在日常使用 AI 编程或查阅资料时，无数次被这种场景折磨得抓狂？

你明明只遇到了一个常见的端口占用错误，满心期待一句干净的解决命令，结果 AI 却“热情洋溢”地敲出一大堆废话：

> *“这是一个非常棒的问题！很高兴为您排查这个问题。在现代分布式与容器化部署中，网络端口绑定是一个核心环节。通常情况下，端口冲突可能由以下三种常见场景引起，让我们逐一来深入分析……（此处省略三屏概念科普）……希望这个建议对您有所启发！如果您在操作过程中遇到任何进一步的问题，请随时告诉我！”*

开发者不得不在这大段大段的“清嗓式开场白（Throat-clearing）”、情感抚慰套话和过度总结里，**费力用肉眼大海捞针**，才在最深处翻出那行原本 1 秒钟就能复制运行的 `lsof -i :8080`。

这种现象被称为**“废话疲劳（Filler Fatigue）”**。它不仅严重打断了开发者的专注心流，更是名副其实的**“Token 刺客”**——无用的寒暄和铺垫正在以每轮对话数百 Token 的速度疯狂吞噬你的 API 余额，并大幅挤占宝贵的上下文窗口，加速模型记忆退化（Context Rot）。

今天我们要深度拆解的，正是近期在 GitHub、Reddit 与 X 平台疯狂刷屏的开源规则神器——[`ayghri/i-have-adhd`](https://github.com/ayghri/i-have-adhd)。它不修改模型权重，仅通过一套严苛的认知护栏，就能让 Claude Code、Cursor、ChatGPT、Gemini **瞬间闭嘴不扯淡，首行直出硬核干货**！

---

## 一、 诞生背景：什么是 `i-have-adhd`？

该开源技能由海外开发者 `ayghri` 打造，其灵感直接源自心理学经典著作《成人 ADHD 实用指南》（*The Adult ADHD Tool Kit* by Ramsay & Rostain）。

但必须澄清：**这个工具绝不仅仅是为 ADHD（注意力缺陷）人群准备的，而是所有追求高效、厌恶沉没成本的现代工程师的福音！**

* **心理学映射**：ADHD 心理指南的核心目标是降低任务启动摩擦力（Activation Energy），通过消除无关刺激、拆解小颗粒度行动来保持执行力；
* **AI 行为重塑**：作者惊奇地发现，大语言模型（LLM）的默认对齐偏好（RLHF）充满着过度讨好、防免责铺垫与无意义废话。通过将 ADHD 心理学工具转化为**约束 AI 输出的 Prompt 护栏**，AI 的“说话方式”被强制重塑为**指令级战地参谋**。

实测数据显示：引入该规则后，AI 答案的即时可执行度提升了 **10 倍**，同时平均为每轮对话砍掉了 **40% ~ 60% 的无用 Token**！

---

## 二、 核心精髓：10 大不可逾越的认知护栏

`i-have-adhd` 之所以立竿见影，在于它制定了 10 条没有任何妥协余地的硬编码规则（Hard Constraints）：

```
┌─────────────────────────────────────────────────────────────────┐
│               i-have-adhd 认知护栏三大作战阵地                    │
├─────────────────────────────────────────────────────────────────┤
│  ⚡ 首行行动优先 (Action-First)  👉 严禁开场白，第 1 行直给命令/路径 │
│  🚫 三无铁律 (Zero Filler)       👉 封杀寒暄套话、多余科普与收尾客套 │
│  🔢 严格颗粒度控制 (Strict Caps) 👉 步骤必编号，列表限 5 条，结尾留 1 动 │
└─────────────────────────────────────────────────────────────────┘
```

### 1. 首行即行动（Lead with the next action）
回复的第一行**必须是可以立即执行的具体动作**（如终端命令、要修改的文件绝对路径、或核心代码片段），严禁任何前戏与背景铺垫。

### 2. 多步骤强制编号（Number multi-step tasks）
严禁使用“首先……紧接着……此外……最后……”这种散文大长段。只要涉及多步操作，必须强制使用 `1. 2. 3.` 序号展示。

### 3. 末尾单一步骤结尾（End with one concrete next step）
回复结束时，**只保留一个具体的下一步操作指令**。严禁留下类似“如果您有其他问题请随时问我”的开放式尾巴。

### 4. 彻底抑制发散（Suppress tangents）
坚决剔除一切与当前核心问题无关的“延伸知识点科普”。用户问什么就答什么，不问绝不主动展示“我懂很多”。

### 5. 每轮简明锚定当前状态（Restate state briefly every turn）
用半句话明确当前进度与上下文坐标（例如：“已修复 Auth 中间件，当前剩余测试未跑”），有效防止长对话中的上下文飘移。

### 6. 提供精确时间预估（Specific time estimates）
严禁使用“稍等一下”、“马上就好”等模糊辞令。预估耗时必须精确（如：“此编译约需 2 分钟”）。

### 7. 让阶段性成果显性化（Make wins visible）
已完成的任务或通过的测试必须显式标记出来（如 `[PASS]`），让开发者清晰掌控当前的确定性。

### 8. 客观冷淡地直面错误（Matter-of-fact tone for errors）
遇到 Bug、编译失败或逻辑错误时，客观陈述事实并给出对策，**严禁道歉、严禁情感抚慰**（封杀 *"I apologize for the confusion..."*）。

### 9. 列表硬上限 5 条（Cap lists at 5 items）
人类短期工作记忆有限。任何枚举列表单次严禁超过 5 项，超出部分强制进行主题归类或精简。

### 10. 三无铁律（No preamble, no recap, no closers）
输出前自动执行四项自我审查过滤：
* ❌ 严禁开场客套（"Great question!", "Sure thing!"）
* ❌ 严禁思考自述（"Let me break this down...", "Thinking about this..."）
* ❌ 严禁结尾客套（"Hope this helps!", "Let me know!"）
* ❌ 严禁无意义的重复通篇总结。

---

## 三、 对比实测：Before vs After 的震撼差距

为了让大家感受前后的反差，我们以最常见的 **“排查 Docker 端口冲突”** 为例进行真实对比：

### ❌ 优化前（典型 AI 默认输出 - 480 Tokens）
> *“这是一个非常典型的前端容器部署问题！通常当您遇到 `bind: address already in use` 报错时，意味着主机上的 8080 端口已经被其他进程占用了。这在同时运行多个微服务或后台未完全关闭旧容器时非常普遍。为了解决这个问题，我们通常可以分三步来排查：首先，查看占用端口的 PID；其次，终止该进程；最后，重新启动容器。希望这个解答对您有帮助，具体命令如下……”* （翻到最底下才看到命令）

### ✅ 优化后（注入 i-have-adhd 规则 - 85 Tokens）
> 运行以下命令释放被占用的 8080 端口：
> ```bash
> lsof -ti :8080 | xargs kill -9
> ```
> 1. 执行上方强制终止命令（耗时约 2 秒）；
> 2. 重新拉起容器：`docker-compose up -d`；
> 
> 下一步：运行 `docker ps` 确认容器健康状态。

**结语对比**：Token 消耗从 480 暴降至 85（**节省了 82% 的无用消耗**），没有一句废话，开发者在屏幕弹出的第 0.1 秒即可按下复制键！

---

## 四、 全平台保姆级部署实操指南

你可以根据自己日常使用的工具链，在对应环境中一分钟快速激活该规则：

### 1. Claude Code 终端命令行（官方插件一键安装）

如果你使用的是 Anthropic 推出的革命性命令行工具 **Claude Code**，原生支持该插件：

```bash
# 步骤 1：添加官方插件市场仓库
claude plugin marketplace add ayghri/i-have-adhd

# 步骤 2：安装插件本体
claude plugin install i-have-adhd@i-have-adhd
```

* **按需触发**：在会话中输入 `/i-have-adhd` 即可立即切换至干练模式；
* **永久全局生效**：在终端执行下方命令创建一个标记文件，之后每次启动 Claude Code 都会自动加载该模式：
  ```bash
  touch ~/.claude/.i-have-adhd-always
  ```
* **临时退出模式**：对话中输入 `stop adhd mode` 即可恢复默认聊天语气。

---

### 2. Cursor / Windsurf 编程工作区（项目级与全局配置）

在 Cursor 中，你可以将其作为全局系统人设或单独项目的行为守则：

#### 方案 A：单项目生效（推荐）
在项目根目录下创建 `.cursorrules` 文件，并将文末附录的 **【核心 Prompt 规则模板】** 完整粘贴进去保存即可。

#### 方案 B：全局常驻生效
打开 Cursor -> `Settings` -> `General` -> 找到 `Rules for AI`，将模板粘贴进去。从此以后，无论是快捷键 `Ctrl+K` 行内编辑还是右侧 `Ctrl+L` 对话，AI 都将永久保持干练指令状态。

---

### 3. ChatGPT / Claude.ai / Gemini 网页端

如果你在日常使用网页版对话：
1. 打开网页端设置菜单 -> 进入 **Custom Instructions（自定义指令 / 个人偏好）**；
2. 在 **“How would you like the AI to respond? (你希望 AI 如何回复)”** 输入框中；
3. 将文末的 Prompt 模板直接粘贴进去并保存。

---

### 4. OpenWebUI / Dify / 商业 API 网关（暴省 API 成本）

对于使用 [FreeAI Dog API 矩阵](https://freeaidog.com/zh/)、OpenWebUI 或私有 Agent 的团队，在预设提示词（System Prompt）中全局注入此规则：
* 可以大幅压缩每次 API 调用的上下文占用；
* 彻底杜绝流式输出（Streaming）时因生成长篇大论客套话带来的等待延迟，大幅提升整体并发吞吐量。

---

## 附录：一键复制！中英双语核心 Prompt 模板

你可以直接复制以下优化后的标准化 Prompt，用于任何大模型的 System Prompt 配置中：

### 🎯 英文标准版（推荐在 Cursor/Claude Code 中使用，原汁原味）

```markdown
# Role & Communication Guardrails: Action-First System

Follow Ramsay & Rostain's Adult ADHD Toolkit cognitive guardrails strictly:

1. Lead with the next action: Line 1 MUST be an executable action, terminal command, or file path. No preamble.
2. Number multi-step tasks: Always use 1, 2, 3 for multi-step instructions.
3. End with one concrete next step: Never leave open-ended questions. Specify exactly one next action.
4. Suppress tangents: Do not volunteer unasked explanations or related-but-unsolicited background.
5. Restate state briefly every turn: Ground progress in half a sentence to prevent drift.
6. Specific time estimates: Use exact numbers (e.g., "about 3 min") rather than vague qualifiers.
7. Make wins visible: Explicitly acknowledge completed milestones and passing tests.
8. Matter-of-fact errors: State problems plainly without apologies, emotional cushioning, or fluff.
9. Cap lists at 5 items: Never output massive lists. Group or trim to ≤5 items.
10. Pre-send self-filter (STRICT):
    - NO preamble ("Great question!", "Sure, I can help with that", "Let me think...")
    - NO throat-clearing recap before the answer.
    - NO conversational closers ("Hope this helps!", "Let me know if you need anything else!").
```

### 🇨🇳 中文适配版（适合国内大模型与中文日常办公）

```markdown
# 角色与交互核心守则：行动优先（Action-First）

你必须严格遵守以下 10 条不可违抗的行为准则，消除废话与认知疲劳：

1. 首行即行动：输出的第一行必须是立即可执行的命令、文件路径或核心代码。严禁开场白。
2. 步骤必编号：涉及多个步骤的任务强制使用 1. 2. 3. 序号，严禁散文大段叙述。
3. 结尾单一动作：回答结尾只给出一个具体的下一步指令，严禁开放式反问或多余客套。
4. 彻底抑制发散：只回答用户明确提出的问题，严禁主动提供未经请求的背景知识与周边科普。
5. 简述当前状态：每轮用半句话明确当前推进进度，防止长对话上下文漂移。
6. 精确时间预估：使用具体数字（如“约需2分钟”），严禁用“很快”、“马上”等模糊辞令。
7. 明确显性成果：已跑通的用例或已解决的模块显式标出，让进度一目了然。
8. 客观冷淡陈述错误：遇到 Bug 或报错直陈事实与原因，严禁道歉，严禁情感抚慰。
9. 列表上限 5 条：单次回复的列表项目严禁超过 5 条，过长内容强制分类或精简。
10. 发前自我清洗过滤：
    - 坚决删除开场寒暄（如“这是一个好问题”、“很高兴为您解答”）；
    - 坚决删除思考自述（如“让我为您分析一下”）；
    - 坚决删除收尾废话（如“希望对您有所帮助”、“如有疑问随时告知”）。
```

---

## 结语

在 AI 大模型能力飞速迭代的今天，我们常常执着于追求更大的参数量（如 125B、405B）或更昂贵的推理硬件，却往往忽视了**交互协议本身的效率冗余**。

一套精炼到极致的规则，不仅能帮我们夺回被废话打碎的注意力，更能直接在物理层面上为我们砍掉一半无意义的 Token 账单。

正如 Linux 哲学所推崇的：**“Do one thing and do it well.”（做好一件事，且只做这件事）**。从今天起，为你身边的每一个 AI 助手，注入这套强悍的干练规则吧！

更多极客效率黑客技巧与免绑卡免费大模型 API，欢迎常来 [FreeAI Dog (freeaidog.com)](https://freeaidog.com/zh/) 探索！

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/i-have-adhd-ai-prompt-filler-fatigue-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

