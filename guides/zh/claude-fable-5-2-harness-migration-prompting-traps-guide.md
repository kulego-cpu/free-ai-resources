---
title: "旧提示词迁上 Claude Fable 5.2 先别急：400 前缀校验与几处行为变动"
description: "基于 Anthropic 官方 Prompting Claude Fable 5.2 文档拆解迁移注意项：全账号强制思考块 prefix check 引发 400 报错、medium 以上档位压不住思考、Memory 旧规则触发过度审批与 reasoning_extraction 拒答分类。"
pubDate: 2026-11-12
author: "小可"
category: "架构与迁移"
tags: ["Claude Fable 5.2", "Claude Mythos 5.2", "Prompt Engineering", "Agent Harness", "Anthropic"]
featured: false
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/claude-fable-5-2-harness-migration-prompting-traps-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


拿 `Claude Fable 5.1` 或 `Opus 5.5` 的老提示词切到 `Claude Fable 5.2`，很容易在两处翻车。

恢复旧对话时，API 直接甩回 `400` 报错。或者 Agent 跑着跑着突然停下。连删个临时文件都要找人审批。

Anthropic 刚把 `prompting-claude-fable-5-2` 挂上官网。翻完才明白，这一代把旧 Harness 的几条潜规则全改了。全篇只核这份官方文档。没拿生产长会话跑回归。token 账单和耗时曲线不编——没条件测。

## 改一句系统提示就 400：全账号强制前缀校验

以前升级版本，顺手改两行 `system` 提示词再正常不过。改完把数据库存的老会话捞出来接着聊。

这招在 `Claude Fable 5.2` 上会撞墙。新版本对所有账号强制执行 `prefix check`。只要请求里改过 `system`、`tools` 或早期消息，又把保存的 thinking blocks 传回去，Messages API 默认拒收。直接返回 `400`。

新指令只能放到新对话，或者塞进最新一轮消息。非要中途改历史上下文怎么办？

官方留了 Beta 逃生门：加上 `thinking-binding-controls-2026-08-01` 请求头，把 `prefix_mismatch_behavior` 设为 `"drop_block"`。代价马上跟上。每轮丢掉早期思考块，模型就会重算思考。输出 token 跟着往上蹿。除了压缩与上下文编辑，保持历史只追加最省钱。

## 写“别多想”不管用了：medium 以上照样先思考

从 `Fable 5.1` 迁过来的老提示词里，很多人爱加一句 `"answer directly without deliberating"`，用来压简单问答的延迟。

这句在 `Fable 5.2` 上废了。文档写得很直白：在 Anthropic 自己的测试里，只要 `effort` 开在 `medium` 及以上，发一句打招呼它照样先思考。加那句提示词也拦不住。偏偏 `Fable 5.2` 默认 `effort` 就是 `high`。

想省简单轮次的开销，靠提示词硬拦已经没戏。只能降 `effort` 档位。文档顺手提了一句：`low` 档编码评测已接近 `Fable 5.1` 的 `high`。基准名和样本量原文没展开。我这没法复现。先记个趋势，别拿它当 SLA。同一段会话里既想快速闲聊、又想在关键步骤拉深，直接切顶层 `effort` 会把 Prompt Cache 打穿。要保住缓存怎么办？得启用 `mid-conversation-output-config-2026-07-01` Beta 头，走单条消息级别的 `effort` 覆盖。

另外得盯紧 `low` 和 `medium` 档的“提前停工”毛病。低档位遇到普通超时或空结果，很容易列个剩余清单就结束当轮。不补持久化推进约束，长链自动化任务半路就断片。谁受得了跑一半自己停下来？

## Memory 积灰的副作用：把旧规则当成绝对禁令

这一代还有个容易忽略的毛病：太死板。

以前用 `Fable 5.1` 或 `Opus 5.5`，工程师习惯往 Memory、Rules 或笔记里塞补丁：`"never retry"`、`"never delete files"`、`"stop and review at the first error"`。老版本有时会变通，新版本会把带 `"never"` 的句子当成绝对红线。

多 Agent 协作跑几轮之后，老会话攒下的谨慎规则越叠越高。结果就是常规读写也停下来找人审批。

光在提示词末尾加一句“大胆一点”救不回来。得把旧记忆翻出来清仓。过时的 `"never retry"` 直接删。把 `"never delete files"` 改写成 `"ask for approval before deleting files"`，只卡破坏性动作，把日常操作权还回去。

子 Agent 调度也一样。在 `xhigh` 和 `max` 档下，`Fable 5.2` 会自发拉起子 Agent 去审自己刚写的代码。以前 `Fable 5.1` 跑同款任务可没这习惯。听着很聪明，账单却全算在子 Agent 头上。文档提到 Anthropic 在 `high` 档做过一组对比：把子 Agent 成本和适用条件写进工具描述后，拉起次数比短描述大约减半。具体任务分布和样本数原文没给。先按趋势看，等自己流水线空跑一轮再定。

顺带算算进度汇报的税。嫌长任务半天不吐字？按官方模板要求“开场说一句、隔几步报一声、结尾附总结”，编码任务输出 token 大约多出 `20%`。有人盯屏值得开，后台静默批处理开了纯烧钱。

## 新增 reasoning_extraction 拒答：别逼它正文吐推理

旧 Skill 或工具说明里，不少人爱写“把推理步骤直接写进正文”。

换上 `Claude Fable 5.2`，这句随时触发 `stop_reason: "refusal"`。拒答了。

新版本把拒答细分进 `stop_details.category`：一共六类：`cyber`、`bio`、`frontier_llm`、`reasoning_extraction`、`general_harms` 与 `null`。逼它把内部推理搬进回复正文，正好踩中 `reasoning_extraction`。想看推理摘要？删掉旧指令。改传 `display: "summarized"` 读思考块。

常规安全与生命科学研发要是被 `cyber` 或 `bio` 误杀呢？Anthropic 把通道分给了企业认证版 `Claude Mythos 5.2`。能力与 `Fable 5.2` 相同，只对通过认证的机构开放。

上线之前，先把这四步排查过一遍：

1. 新会话分别跑一遍 `medium`、`high` 与 `xhigh` 档 `effort`。旧档位别直接照搬。
2. 翻一遍现存 System Prompt、Skills 与 Memory。删掉 `"answer directly without deliberating"`。顺手清理带 `"never"` 的过期禁令，以及正文吐推理的旧句子。
3. 查长会话恢复链路。确保历史消息与 `system`、`tools` 只追加不改前缀，避开 `400` 报错。
4. 拿最长流水线空跑一轮。看它卡在提前停工、乱开子 Agent，还是卡在无意义审批。没摸清旧规则里的地雷，直接全量切流你真敢吗？

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/claude-fable-5-2-harness-migration-prompting-traps-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

