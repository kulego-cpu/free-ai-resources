---
title: "管好 Claude Code 与 Codex 的记忆和 Skill：MemDock 桌面工作台实战"
description: "深度拆解跨平台 AI 编程工作台 MemDock 的实战架构。解决 CLAUDE.md 与 AGENTS.md 规则腐化痛点，通过每周对话提取入待审核缓冲区、Claude Code 与 Codex 会话双向互转、Git Worktree 并行隔离开发以及 GitHub 私有仓库跨机同步，建立稳健的 AI 编程生产力流。"
pubDate: 2026-11-09
author: "小可"
category: "实操测评"
tags: ["MemDock", "Claude Code", "Codex", "AI 编程", "Git Worktree", "CLAUDE.md", "开源工具"]
featured: true
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/memdock-claude-code-codex-desktop-workbench-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


# 管好 Claude Code 与 Codex 的记忆和 Skill：MemDock 桌面工作台实战

用 Claude Code 或者 Codex 写代码，你有没有被记忆污染恶心过？

很多人的项目根目录下，长年挂着几个规则文件：`CLAUDE.md` 或者 `AGENTS.md`。

刚开始挺好用。你在里面写满了项目规范与打包流程。折腾半个月后，测试框架早就从 Jest 换成了 Vitest；构建脚本也改了。新开一个会话，模型不仅没变聪明，反而照着老规则去敲废弃命令。

改了测试命令？模型压根不听。

老会话留下的错误经验卡在自动记忆库里，新会话继续被带进沟里。来回折腾烦不烦？真烦。

更头疼的是双工具割裂。

终端跑 Claude Code 确实敏捷。看大段代码差异时，Codex 的图形界面更顺眼。但两边的插件与记忆各过各的。给 Claude Code 装了一堆自动化 Skill，切到 Codex 又得手动去翻目录重新配。换台笔记本办公，两套环境直接归零。

开发者 cturing 在社区开源了 MemDock 工作台。仓库托管在 GitHub：`LingyiChen-AI/MemDock`。

先说它不干什么。

不生成代码，不替你写 prompt，不接第三方模型。装完之后，你的 Claude Code 还是原来那个，命令行还是 `claude`，只是项目根目录多了一个 `.memdock/` 文件夹，里面躺着规则的历史版本。改坏了能退回去，就这一个作用。

它管三件事：

哪些规则该留下。会话在哪边跑。换机器怎么搬过去。

就这三件，别的它不管。

## 记忆防腐化：为什么必须设一道【待审核】关卡

大多数所谓自动记忆功能，最大的败笔就是允许大模型自由写入规则文件。

会话进行到深夜，你为了绕过一个临时报错随口写了段脏代码。模型把这段权宜之计当成核心准则，悄悄写进规则库。整个项目的代码质量全被这句毒记忆带偏；以后每次生成都跟着踩坑？真遭罪。

MemDock 的工程解法非常克制：建立待审核缓冲机制。

软件支持全文检索。不管是 `CLAUDE.md`、`AGENTS.md` 还是 Claude Code 的本地自动记忆，都能搜得到。遇到陈旧规则，随时分栏编辑；改错了还能随时调出历史版本回滚。

更关键的是每周经验提取工作流。

系统会自动翻阅近期的实际对话，抓取出那些值得沉淀的技术经验与需要修订的陈旧规则。

但这批新结论绝不会自动落盘。

提取出来的建议全部丢进待审核记忆区。开发者坐在电脑前，左边看旧规则，中间看新提取的建议，右边核对当时的原始对话上下文。

你点采纳，才写回磁盘；点驳回，毒经验作废。

第一次自动提取时我就踩了个坑：它把我 `dist/` 构建目录下的报错日志也硬抓成了候选规则。后来赶紧在 MemDock 忽略名单里加了 `dist/` 与 `coverage/`，提取池才彻底干净。

配合记忆图谱功能，每一条正在生效的规则都能反向追踪到最初产生它的会话片段。改规则不再两眼一抹黑。

## 工具间架桥：Claude Code 与 Codex 双向会话互转

两套工具到底该用哪一个？

小孩子才做选择，真干活的人随时都在两端切换。

终端交互响应迅速，适合让 Claude Code 带着明确目标跑测试、改单文件、批量重构；遇到复杂的业务主干设计，Codex 的图形化多栏窗口审阅代码又更省眼睛。

MemDock 做的核心桥梁，就是双向会话无缝转换。

在终端里跟 Claude Code 聊到一半，发现涉及多个模块交叉修改，一键转换成 Codex 会话，在图形界面里接着推进。

反过来顺手吗？同样顺手。在 Codex 里梳理完规格与架构，一键倒腾回原生命令行，让终端工具去执行繁琐的跑批任务。

两款工具的技能插件管理被收拢到了一处。

工作台集中拉出已安装的全局与项目级 Skill，连带着 Superpowers 插件包也能按模块清晰展示。支持直接从 GitHub 仓库地址、本地目录或者 ZIP 压缩包一键安装。

在 Claude Code 里调顺了一套调试 Skill，点一下复制，就能无损同步进 Codex。再也不用去系统深层目录里倒腾配置文件了。

## 并行开发隔离：Git Worktree 与换机同步的真实代价

单打独斗跑 AI 助手，很容易撞上工作区文件锁死的问题。

一个会话正在让助手大改后端数据模型，另一个会话又想让它去调整前端表单。同一个 Git 工作区里改动混在一起，未提交的文件互相覆盖，最后谁都提交不了。

MemDock 直接集成了 Git Worktree 机制。

三个分支同时开工，各自运行 `git worktree add ../feat-schema feat/schema`，目录分开，互不遮挡。终端里敲一下查询指令：

```text
$ git worktree list
/Users/cool/work/project-core       b39a1c2 [main]
/Users/cool/work/feat-schema        d7e4120 [feat/schema]
/Users/cool/work/feat-ui-auth       a199f03 [feat/ui-auth]
```

但别指望它消灭冲突。

它只是把冲突从写的时候挪到了合的时候。上周合回去，两个文件照样变红，好在冲突点很干净——都在 schema 定义那三十行，改完就过。

真正省事的是上下文隔离。一个分支跑测试，另一个分支改前端，谁也不污染谁的 `node_modules` 依赖。

但代价必须算清楚。开三个 worktree，磁盘占用直接翻倍。每个目录里的依赖都要单独装一份，硬盘吃得飞快。

换电脑继续干活也挺方便。它支持把记忆库与 Skill 打包成 JSON，一键推到 GitHub 私有仓库。

这里必须泼上一盆安全冷水：

把 `CLAUDE.md` 往 GitHub 私有仓库推，必须提前肉眼检查一遍。很多开发者习惯在规则文件里写死测试服务器内网 IP、测试账号或者内网 API 鉴权前缀。私有仓库绝不等于安全内网边界，敏感信息带上去就是安全事故。

另外，MemDock 采用的是 source-available 协议；个人免费用挺顺手，但不要擅自拿去二次分发或者用于商业闭源包装。

## 真实账本与日常工作流

我现在的用法是：Claude Code 负责跑测试和改单文件，Codex 负责看 diff，规则全丢待审核区，一周清一次。周五下班前顺手推到私有仓库，回家那台 MacBook 直接拉。

不算优雅。每次换机还得手动填一次 token，作者说后面会做钥匙串集成，现在还没影。

你们平时是怎么管这两套工具记忆的？要是有更省事的解法，欢迎在评论区聊聊。

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/memdock-claude-code-codex-desktop-workbench-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

