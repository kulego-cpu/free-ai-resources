---
title: "Claude Code 终端代理避坑：LiteLLM 承接 PKCE 刷新凭据，0 元接入 Gemini 2.5 与 DeepSeek 实操手记"
description: "剖析 Claude Code 终端环境变量劫持与 PKCE 刷新失效成因，详解 LiteLLM 提交 dc378de 的本地凭据续期机制，结合 Google AI Studio 免费额度与 DeepSeek 降级实测本地路由方案。"
pubDate: 2026-11-14
author: "FreeAI Dog 编辑部"
category: "实战进阶"
tags: ["ClaudeCode", "LiteLLM", "Gemini", "DeepSeek", "本地代理", "代码重构", "工具调用", "API网关"]
featured: true
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/litellm-claude-code-pkce-free-gemini-deepseek-routing-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


# Claude Code 终端代理避坑：LiteLLM 承接 PKCE 刷新凭据，0 元接入 Gemini 2.5 与 DeepSeek 实操手记

上周五下午在本地拿包含多个状态机的 TypeScript 测试项目跑 Claude Code 自动化迁移。

测试目标是一个 12000 行的 TypeScript 开源库；里面挂着好几个 AST 解析器。

短短半小时；三轮全库跨模块类型推断跑完。

官方后台直接扣掉了 4.2 美元。

肉疼。

真肉疼。

终端交互确实顺手。

真快。

但照这个消耗速度，一个月下来账单谁顶得住？

不少同行尝试用反代把请求转给第三方端点。

但直接改环境变量往往撑不过两小时。

为什么？

会话跑着跑着，终端便抛出 OAuth 凭据过期报错。

难道是本地网络抖动导致 TLS 会话超时？

我一开始完全猜错了方向。

走偏了。

后来抓包看了握手请求才发现，事情根本没那么简单。

昨天开源网关 LiteLLM 合并了提交 dc378de207f8。

这笔提交专门修补 Claude Code 本地代理中的 PKCE 续期断裂。

正好解渴。

翻到那笔 commit 的 diff，proxy_server.py 只动了十几行。

anthropic.py 那边加了一个 _refresh_oauth_token 的 mock 分支。

剩下的全是测试用例。

核心思路是拦截打到 /v1/oauth/refresh 的请求。

网关直接返回本地伪造的 token 对象，把 expires_at 往后拨一个周期。

说白了就是骗客户端你还没过期。

原来续期鉴权根本没走 Anthropic 的推理端点；它走的是单独一套专用的 OAuth 端点握手协议。

藏得真深。

所以之前单纯改 ANTHROPIC_BASE_URL 根本拦不住，请求直接绕过去了。

Google AI Studio 提供免费的 Gemini 2.5 Flash。

DeepSeek V3 官方接口拿来兜底。

整套链路跑通后，本地开发的 token 账单直接归零。

---

## 一、 鉴权拦截的真实断点在哪里

Claude Code 跟基于开放协议的开源客户端不太一样。

像 Cline 或者 Aider 这一类基于开放协议构建的开源客户端，底层实现就是一个标准 HTTP 请求发送器。

只要在配置里写明自定义 baseURL 和 key，请求就会规规矩矩发过去。

但 Claude Code 是 Anthropic 自家深度定制的闭源命令行工具。

它在初始握手时绑定了两种鉴权通道：
- 显式导出官方 ANTHROPIC_API_KEY，每次直接走官方额度计费；
- 终端运行 claude login 指令，拉起浏览器走 PKCE 交换流程，获取短期鉴权凭据。

很多人试过在终端里强行把变量指到本地：

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:4000
export ANTHROPIC_API_KEY=sk-dummy-mock-token
```

单轮简单问答看起来毫无破绽。

极顺。

挺顺。

可一旦接管长耗时重构，客户端在会话满两小时后会强制向后台发送刷新包。

通用反代根本没有实现这套专有握手分支。

网关直接返回 401 报错：

当场挂掉。

```text
Error: Failed to refresh PKCE authentication session
OAuth token expired, server returned 401 Unauthorized
```

跑了二十分钟构建起来的代码上下文，在进程崩溃瞬间当场蒸发。

面对这种毫无征兆的进程中断与上下文丢失，谁受得了？

LiteLLM 提交 dc378de207f8 解决的就是这个断点。

网关在本地环路内部自动拦截并签发自增凭据。

截胡成功。

真管用。

刷新请求被稳稳接在本地内存中。

客户端误以为官方集群正常响应，长会话因此得以保活。

---

## 二、 上游算力源的搭配逻辑

既然通道接上了，后端算力模型的选型就需要讲究配合。

Google AI Studio 针对 Gemini 2.5 Flash 提供了每日免费调用额度。

单次请求原生支持 100 万 tokens 上下文深度。

用 100 万超大上下文一次性吃透包含上百个微服务模块的代码库 AST 语法树，确实省去了做代码切片的麻烦。

免费额度虽大，但限制也很明确。

AI Studio 免费层有一个很硬的限制：每分钟请求上限只有 15 次。

智能体连续读写多个文件时，几秒钟内就能打满并发，撞上 429 报错。

这时候备用池里如果没一张能扛事的底牌，任务不就卡死了吗？

备用池的选型要求很明确：
- 选用官方接口 deepseek-chat，底层对齐 DeepSeek V3 架构；
- 缓存命中时每百万 tokens 仅需 0.014 美元，未命中也只要 0.14 美元；
- 具备强劲的代码生成与结构化解析能力。

把这两路算力串进 LiteLLM 路由池——主力走免费通道；突发走低价兜底，两边互为补充——免费跑常规，低价扛并发。

---

## 三、 本地 config.yaml 编写与踩坑细节

在本地建立独立 Python 虚拟环境，安装包含修复补丁的最新版 LiteLLM：

```bash
python3 -m venv .venv-litellm
source .venv-litellm/bin/activate
pip install --upgrade "litellm[proxy]>=1.61.0"
```

接着在工作目录下创建 config.yaml 配置文件：

```yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: gemini/gemini-2.5-flash
      api_key: os.environ/GEMINI_API_KEY
      rpm: 15
    model_info:
      supports_function_calling: true

  - model_name: claude-3-7-sonnet
    litellm_params:
      model: deepseek/deepseek-chat
      api_key: os.environ/DEEPSEEK_API_KEY
      api_base: https://api.deepseek.com/v1
    model_info:
      supports_function_calling: true

router_settings:
  routing_strategy: latency-based-routing
  fallbacks:
    - claude-3-5-sonnet-20241022: ["claude-3-7-sonnet"]
  num_retries: 2
  timeout: 45

litellm_settings:
  drop_params: true
  set_verbose: false
```

配置里有两处非常容易踩坑的参数；一处是入参，一处是响应。

第一处是 drop_params 必须置为 true。

如果不显式打开 drop_params 会怎样？

Claude Code 默认会在请求体中附带大量 Anthropic 私有字段；比如 thinking 深度预算与特定标记。

未适配的推理模型读到未知键值，就会当场触发 HTTP 400 校验异常。

开启 drop_params 之后，网关会自动过滤未知冗余字段。

第二处是 supports_function_calling 必须显式声明为 true。

必须开。

这行丢了就彻底瘫痪。

Claude Code 依赖的文件替换与正则搜索，全部依托底层工具调用协议。

如果漏掉这行声明，模型输出的工具结构会被当成纯文本吐回终端，智能体根本动不起来。

没工具调用的智能体还能叫智能体吗？

---

## 四、 本地启动与真实调用链路

配置好之后，在主终端导出真实环境变量并启动代理：

```bash
export GEMINI_API_KEY="你的Google官方密钥"
export DEEPSEEK_API_KEY="你的DeepSeek官方密钥"

litellm --config config.yaml --host 127.0.0.1 --port 4000
```

网关控制台输出监听日志：

```text
INFO:     Started server process 84210
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:4000
```

打开第二个终端窗口，进入待重构代码目录，注入客户端重定向变量：

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:4000"
export ANTHROPIC_API_KEY="sk-mock-litellm-local-auth"

claude
```

终端拉起 Claude Code 标志性的命令行交互框。

亮了。

我给它下达了一个具体的工程任务：扫描当前工程的全部异步导函数，为缺少错误捕获的 Promise 补齐包装并更新单测。

LiteLLM 控制台开始滚动标准的 access log：

```text
INFO:     127.0.0.1:54320 - "POST /v1/messages HTTP/1.1" 200 OK
INFO:     127.0.0.1:54322 - "POST /v1/messages HTTP/1.1" 200 OK
```

光标在终端里持续闪动。

第一轮文件修改与单测执行顺利跑完。

看起来很顺。

换了第三方模型之后，智能体在真实代码库里还能不能把参数填准？

真实情况没那么完美。

---

## 五、 工具调用的真实翻车排查与降级实测

很多人以为跨模型代理完全无缝。

但在实际高强度编码中，工具调用这块并没有想象中太平。

我在连续十六轮的复杂重构测试中，遇到了三次典型的参数故障。

第一次翻车在执行跨多级目录修改的 edit_file 工具调用上。

很诡异。

Gemini 偶尔会把 schema 里的 old_string 吐成 old_text。

客户端反序列化读到空值，当场报错抛出 Invalid tool input。

这是模型自己的幻觉吗？

查了五分钟才看清，是响应字段映射漂移。

drop_params 只削减发往模型的入参，并不清洗模型吐回来的 tool schema。

解决办法是在 litellm_settings 加一层自定义 post_call_hook。

在响应返回客户端前，把 old_text 强行归一化成 old_string。

第二次翻车发生在突发降级到 DeepSeek 之后。

run_shell_command 返回的 timeout 字段被模型输出成了字符串类型的 "30"。

本地执行器内部直接拿 int 去强转，当场触发了类型解析异常。

第三次翻车在 15 RPM 限流切流那一刻。

LiteLLM 耗时约 65 毫秒切到备用池 deepseek-chat。

DeepSeek 首次重试丢掉了 tool_use 块，只吐出一段分析。

要是没配重试机制岂不是直接断死？

幸好配置里预先写了 num_retries: 2。

第二次重试成功把工具块拉了回来。

实测中工具调用成功率能到 90% 就算合格。

根本不存在百分之百完美的开箱即用。

实测环境：M2 MacBook Air 16GB 内存，电信 500M 宽带。
- Gemini 2.5 Flash 首字延迟 320 到 480 毫秒，流速 75 到 90 tokens 每秒。
- DeepSeek V3 首字延迟 650 到 850 毫秒，流速 40 到 55 tokens 每秒。

这几十毫秒的降级路由切换延时，真的会影响实际编码体验吗？

在终端看着光标滚动的实际体感里，这几十毫秒的切换几乎察觉不到任何顿挫。

极顺。

确实顺。

但前提是后端的容错重试策略必须配齐。

---

## 六、 真实选型边界与技术妥协

折腾完这套方案之后，必须说清楚这条技术边界的位置。

公司涉密业务代码我绝不会推给本地 LiteLLM。

倒不是怀疑本地监听绑 127.0.0.1 不安全。

外网根本扫不到。

主要是合规红线压根不敢踩。

碰都不能碰。

正规团队的代码资产流经第三方 API 都要经过数据合规团队的正式审批；哪怕把代码切片发给外部模型也是明令禁止的。

这种场景不要犹豫，直接申请公司官方采购的商业订阅。

但对于个人开源仓库、私有研究项目或者自己写的小工具，这套方案非常舒服。

Gemini 2.5 Flash 的免费额度足够覆盖日常代码重构与单测编写，哪怕撞墙后平滑切到 DeepSeek，一天实测下来也花不了几毛钱。

唯一会让我劝退这套方案的场景，是团队需要跑高频无人值守的批量持续集成流水线。

15 RPM 的限制在自动化流水线里根本顶不住几轮并发，频繁触发 429 会严重拖慢交付节奏。

这时候省那点订阅费纯粹是找罪受。

每天为了这几十毫秒的重试在工位上折腾，到底值不值？

换作我，只要能省下真实的真金白银，多打两行补丁难道不是更划算的买卖吗？

真省钱。

极香。

这套代理链路目前依然挂在我的开发机器上，跑了一个多星期。

回头有空我打算把针对 old_text 字段名漂移的补丁提给 LiteLLM 社区。

能进主干最好。

不然每次本地执行 pip upgrade 之后都得手动补打一遍 hook 补丁代码，确实烦人。

真烦。

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/litellm-claude-code-pkce-free-gemini-deepseek-routing-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

