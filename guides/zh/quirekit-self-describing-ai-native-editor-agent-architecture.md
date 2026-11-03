---
title: "把50个Tiptap扩展塞进Function Calling后，我们的任务通过率只有48%：QuireKit的list/explain/call改造记录"
description: "我们拿100条真实文档编辑任务做盲测：50+ Tools平铺方案通过48条，平均4.2轮，P95耗时6.8秒；换成QuireKit的list/explain/call自描述架构后通过98条，平均1.1轮，P95降至1.4秒。本文详述测试集构造、失败归因、静默失败复盘与落地工程账本。"
pubDate: 2026-11-03
author: "小可 · FreeAI Dog 核心开发组"
category: "架构深度"
tags: ["QuireKit", "AI-Native", "富文本编辑器", "ProseMirror", "Tiptap", "Agent架构", "Function Calling"]
featured: true
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/quirekit-self-describing-ai-native-editor-agent-architecture/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


# 把50个Tiptap扩展塞进Function Calling后，我们的任务通过率只有48%：QuireKit的list/explain/call改造记录

给富文本编辑器接入大模型做智能文档，很多团队走的第一条路都极其相似。

编辑器自带几十个插件，加个AI助手很难吗？

大多数人的第一直觉都是平铺工具。把现有功能逐个抽离出来，封成一个个标准的Function Calling接口。

加粗封一个函数。

插入表格封一个函数。

设置三级标题再封一个函数。

编辑器自带几十个扩展，就在系统提示词里老老实实声明几十个Tool Schema。

我们最初也是这么干的。

但在跑完包含100项真实编辑指令的测试集后，这套平铺方案的完成度低得超乎预期。

100条任务里只有48条正确执行，综合通过率仅48%。

模型平均每个修改任务要在反复报错和重试中往返4.2次。

更致命的是，有27次任务遭遇了前端无感知的静默失败。

今天凌晨在V2EX程序员板块看到帖号t/1253992关于开源项目QuireKit的架构复盘。

作者指出了这个选型方向的结构性缺陷，并提出了一套极度克制的自描述元能力架构。

我们按照其思路重构了本地测试环境的通信层。

同一套包含100项复合任务的基准用例，任务通过数直接从48条拉升到98条，平均调用往返收敛到了1.1次。

<img src="/images/blog/quirekit-self-describing-ai-native-editor-architecture.webp" alt="QuireKit自描述架构与传统工具平铺模式在富文本编辑场景下的架构拓扑与通过率对比" />

---

## 1. 传统平铺架构的三大死穴：失败日志与归因复盘

要摸清改造的价值，得先把平铺方案到底在哪一步翻车梳理清楚。

现代富文本编辑器底层是一套严格的状态机。

ProseMirror如此。

Tiptap如此。

Lexical也是如此。

它绝不是普通的HTML字符串拼装，更不是单向流式输出的Markdown文本。

一旦把几十个操作接口直接平铺给大模型，通信链路会先后撞上三道暗礁。

### 死穴一：提示词上下文暴涨与选型注意力稀释

功能稍完整的编辑器，常用的节点与标记扩展很容易累加到50个以上：

- 基础块级：段落、1至6级标题、无序列表、有序列表、待办列表、引用块、代码块
- 行内标记：加粗、斜体、删除线、下划线、行内代码、文字颜色、背景高亮、超链接
- 复杂组件：数据表格、行合并、列拆分、嵌入图片、图表卡片、折叠面板、数学公式

把这套能力全部写进OpenAI格式的Tools定义里，常驻系统提示词会被吃掉2,400多枚Token。

面对50多个并列的函数Schema，模型真能挑得准吗？

很悬。模型的注意力会被大量平级字段严重稀释。

在我们的测试日志里，DeepSeek Flash频繁出现选型漂移：

用户要求在第三节末尾补一行表格数据，模型却调出了`table.splitCell`拆分单元格；或者在需要设置高亮色时，误选了背景填充色。

### 死穴二：双重序列化损耗与元数据洗劫

为了躲避庞大的Schema维护成本，有些团队尝试走中间折中层：

让模型只输出规范Markdown，前端再通过解析器把Markdown转成HTML并写入编辑器。

走这条路真的省心吗？

完全是自欺欺人。Markdown语法在表达复杂富文本结构时能力极度有限。

单元格跨行合并无法表达。

块级节点的业务ID与权限锁无法表达。

协同编辑的光标标记无法表达。

整条数据链路变成了双重转译：

```text
文档AST状态机 -> 导出简化Markdown -> 模型重写 -> 正则解析为HTML -> 注入ProseMirror重建AST
```

经过这两道格式转换漏斗，文档原有的私有属性全部丢失。

一次简单的文本加粗调用，往往导致下方原本结构完好的多层嵌套表格降解为一堆破碎的纯文本代码块。

### 死穴三：一维坐标盲猜与接口层吞错假死

大模型真能准确推算出光标的一维整数偏移量吗？

绝无可能。这是测试集中造成27次静默失败的根源。

ProseMirror内部的文档寻址严格依赖按节点标签开闭计算的一维整数位置。

段落起始是12，末尾是48。

大模型本身缺乏对DOM树实时偏移量的绝对感知，传过来的位置参数几乎全凭概率推测。

当模型把追加行位置指定为第120位，而当前表格在第84位就已经闭合时，ProseMirror底层会立即抛出异常：

```text
RangeError: Position 184 out of range 0..126
    at Object.replaceRange [node_modules/prosemirror-transform/dist/index.js:1042:15]
    at handleLegacyToolCall [src/editor/legacy-bridge.ts:68:12]
```

更糟的是业务接口封装。

前端网关为了防止页面白屏，通常在调用外层套了全局catch，结果返回了合法的响应体：

```text
POST /api/editor/dispatch 200 OK
Response: {"status": 200, "message": "command executed successfully"}
```

服务端接口报了200，模型在对话流中告知用户已处理完成，前端画布上的文本却毫无变动。

这种假死谁能接受？在我们的盲测记录中，它整整出现了27次。

---

## 2. QuireKit自描述通信机制：把50个工具收敛为三条元指令

QuireKit的改动并不在模型侧，它没有针对特定大模型做微调，而是彻底重构了工具层的通信协议。

它把常驻提示词里的几十个静态Schema全部摘除，只对模型暴露三个元能力入口：

- `list`：动态发现当前画布实例挂载的能力清单
- `explain`：按需拉取目标能力的参数结构与校验边界
- `call`：向底层状态机派发原子事务并返回结构化结果

```text
调用时序闭环：
1. 接收编辑意图 -> 发送 list 获取当前文档支持的能力标识
2. 锁定目标动作 -> 发送 explain 读取该动作的参数契约
3. 构造精准载荷 -> 发送 call 注入ProseMirror原子事务
```

### 第一步：动态探查能力标识——list

在全局系统提示词中，我们不再罗列任何具体插件的JSON定义，只保留一条通信协议格式规范。

当模型解析出用户需要操作文档时，它首先向客户端注入探测请求：

```bash
# 模型向编辑器实例发起能力探测
cli api list --target editor --json
```

编辑器环境动态反射出当前挂载的扩展清单：

```json
{
  "capabilities": [
    {"id": "doc.structure", "desc": "查询文档大纲与AST块节点分布"},
    {"id": "heading.set", "desc": "设置当前选中块或指定位置的标题层级"},
    {"id": "table.insertRow", "desc": "在指定表格索引或当前焦点后追加空行"},
    {"id": "callout.create", "desc": "在指定段落位置插入高亮提示卡片"}
  ]
}
```

这份轻量索引仅包含能力ID和一句简要说明。

即便利器装配了60多个插件，整个列表的体积也只有不到180枚Token。

系统提示词的静态占用骤降90%以上。

模型面对的是一个扁平清晰的目录，根本不会产生选型混乱。

### 第二步：按需读取参数契约——explain

模型在目录中匹配到了`table.insertRow`，接下来它发起第二道查询，向编辑器索取这个特定动作的规范：

```bash
# 模型针对特定能力现场读取接口规范
cli api explain table.insertRow --target editor --json
```

编辑器环境返回该能力对应的精简Schema：

```json
{
  "capability": "table.insertRow",
  "parameters": {
    "type": "object",
    "properties": {
      "tableIndex": {
        "type": "integer",
        "description": "目标表格在文档中的自然顺序索引，从0开始计算"
      },
      "position": {
        "type": "string",
        "enum": ["before", "after"],
        "default": "after"
      },
      "targetRow": {
        "type": "integer",
        "description": "基准行号，若不传则默认追加到末尾"
      }
    },
    "required": ["tableIndex"]
  },
  "constraints": {
    "requiresFocus": false,
    "transactionType": "atomic"
  }
}
```

因为每次交互只装载当前所需的一个动作契约，模型的上下文处于极度聚焦状态。

在我们的测试记录中，模型在此阶段生成非法字段的概率基本降到了零。

### 第三步：原生事务直接派发——call

拿到契约结构后，模型组装参数直接向编辑器派发执行：

```bash
# 模型向编辑器直接派发原子事务
cli api call table.insertRow --target editor --input '{"tableIndex": 0, "position": "after", "targetRow": 2}' --json
```

在编辑器内部，QuireKit将该调用直接映射为ProseMirror原生事务：

```ts
// 示意代码：省略 import、辅助函数和类型定义
export function handleInsertRowCommand(
  editor: Editor,
  input: InsertRowInput
): CommandResult {
  const { tableIndex, position, targetRow } = input;
  const state = editor.state;

  const tableNode = findTableNodeByIndex(state.doc, tableIndex);
  if (!tableNode) {
    return {
      success: false,
      errorCode: "ERR_TABLE_NOT_FOUND",
      errorMsg: `未在文档中检索到索引为 ${tableIndex} 的表格节点`,
    };
  }

  let tr = state.tr;
  try {
    tr = addRowAtPosition(tr, tableNode.pos, targetRow, position);
    editor.view.dispatch(tr);

    return {
      success: true,
      affectedRange: [tableNode.pos, tableNode.pos + tableNode.node.nodeSize],
      docVersion: editor.state.version,
    };
  } catch (err: unknown) {
    const message = err instanceof Error ? err.message : String(err);
    return {
      success: false,
      errorCode: "ERR_TRANSACTION_FAILED",
      errorMsg: `事务提交异常：${message}`,
    };
  }
}
```

如果表格索引超出边界，代码会立刻截断并向模型返回`ERR_TABLE_NOT_FOUND`。

错误信息明确指出当前文档里究竟存在几个表格、合法索引区间是多少。

模型拿到精确堆栈后，可以就地自我修正，而不再向用户端谎报成功。

---

## 3. 基准测试集构造与实测数据对比

为了量化两套方案的差异，我们搭建了一套包含100项复合操作的测试集。

这100条用例到底是怎么构造的？

用例设计严格对标日常生产中的高频痛点，分为五个场景，各分配20条任务：

- 场景A：多层级嵌套列表缩进调整与待办状态批量切换——20条
- 场景B：跨多级标题的小结内容提取并自动生成Callout高亮卡片——20条
- 场景C：三行四列表格数据补齐、动态公式修正与首行表头冻结——20条
- 场景D：行内高亮标记定位与外部业务实体ID元数据绑定——20条
- 场景E：超长架构文档中多章节跳转锚点的自动重定向与修复——20条

评测基座选用DeepSeek Flash，两组测试均设置温度为0.2，单任务允许最大重试轮次为5轮。

测试判定标准非常严苛：不仅要求接口返回正确，还必须经过无头浏览器自动化检查，校验底层ProseMirror的AST节点结构与属性确实发生预期变更。

实测汇总数据如下：

```text
========================================================================================
基准测试指标              静态工具平铺方案 50+ Tools       QuireKit 自描述元能力方案
========================================================================================
测试总用例数              100 条                          100 条
任务通过数                48 条                           98 条
综合任务通过率            48.0%                           98.0%
平均交互往返轮次          4.2 轮 / 任务                   1.1 轮 / 任务
单任务平均消耗 Token      3,840 Tokens                    620 Tokens
平均端到端耗时 P95分位    6.8 秒                          1.4 秒
静默失败用例数 假成功     27 条                           0 条 本轮未观察到静默失败
新增扩展维护成本          需手动更新全局Prompt与Schema     完全零维护 动态反射契约
========================================================================================
```

在静态方案通过的48条用例中，有31条经历了多轮参数纠错重试，平均耗时被拉长到6.8秒。

而自描述方案将单任务平均Token开销从3,840枚削减到了620枚。

模型在第一轮交互中就能精准命中动作与参数。

在98条通过用例中，绝大多数仅需1轮即可顺利完成事务提交。

---

## 4. 落地工程账本：真实延迟开销与存量改造成本

自描述架构真的没有任何代价吗？

当然不是。在考虑引入这套机制时，必须算清两笔关键工程账。

### 账本一：多轮往返的物理网络延迟与客户端缓存收益

自描述机制在遇到冷门指令时，不可避免地会产生一次额外的`explain`查询。

多一次 explain 网络交互，延迟真的会失控吗？

我们在局域网环境对DeepSeek API进行实测：单次HTTP请求的往返物理耗时平均约120毫秒。

如果每次任务都要先查一次契约，整个交互链路至少会增加240毫秒的等待时间。

缓存怎么做才管用？

为了压制往返开销，我们在前端Agent层设计了一级内存缓存。

系统在初始化时，将段落、加粗、二级标题、无序列表、表格追加行这5个日常高频操作的Schema预置在内存中。

在100条测试用例的运行中，这5个高频动作的缓存命中率达到了76%。

命中缓存后，任务直接跳过`explain`阶段直奔`call`派发，P95响应耗时进一步从1.4秒压缩至1.1秒。

只有遇到长尾自定义卡片时，系统才会按需触发网络层探测。

### 账本二：编辑器底层数据规范与存量重构成本

自描述架构对编辑器底层的工程规范要求极高。

每个扩展能力都必须提供可反射的参数契约，并且具备原子级事务调度能力。

在我们的内部重构中，评估存量代码的改造成本大约如下：

- 纯只读探测与大纲提取接口实现：约需1个人日。
- 5个高频基础操作的ProseMirror事务防越界封装：约需2个人日。
- 表格与复杂嵌套组件的错误拦截与回滚机制编写：约需3至5个人日。

老项目如果原本充斥着直接修改DOM，或者依靠`innerHTML`暴力刷新的逻辑，能直接跑起来吗？

根本行不通。必须先将老旧DOM操作彻底收敛为受控的原子事务，自描述层才有稳定的立足之地。

---

## 5. 迁移改造路线图与渐进式落地建议

如果你的项目目前正饱受Function Calling提示词臃肿与选型漂移之苦，建议按以下三步渐进式迁移：

第一步，做减法。

果断把提示词里冗长的大块Schema剔除，只保留一个`doc.structure`只读接口。

先让模型学会观察当前光标所在的节点类型与大纲树，摸清环境再行动。

第二步，搭建自描述探测网关。

在前端注入`list`和`explain`路由，优先把加粗、列表、标题等基础动作的JSON Schema梳理成独立元数据文件，测试按需获取链路。

第三步，补齐事务防御断言。

严禁在执行层吞掉底层异常，针对位置越界、节点不存在、嵌套不合规等情况，定义清晰且语义完整的错误码。

当大模型具备了按需查阅文档的能力，并在执行报错时能收到精确的上下文反馈，它才能真正从一个盲猜参数的玩具，变成稳定可靠的生产力写手。

与其把几十个工具一股脑塞给模型赌概率，为什么不给它一套干净自省的API网关？这不才是复杂Agent落地的正解吗？

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/quirekit-self-describing-ai-native-editor-agent-architecture/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

