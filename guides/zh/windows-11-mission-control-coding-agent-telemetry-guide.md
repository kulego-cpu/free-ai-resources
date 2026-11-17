---
title: "Windows 11 逆向工程实锤：MissionControlAggregator 深度监控 Coding Agent 进程与防护实战"
description: "基于 Windows 11 Build 26100 逆向工程分析核心动态链接库 MissionControlAggregator.dll，解剖微软针对 Claude Code、Cursor、Codex 等 AI 编码智能体的静默扫描机制，拆解 ETW 诊断报告链路，并提供组策略与注册表彻底阻断遥测回传的实战防御 SOP。"
pubDate: 2026-11-17
author: "小可 · FreeAI Dog 核心开发组"
category: "架构剖析"
tags: ["Windows11", "CodingAgent", "逆向工程", "开发者隐私", "遥测防护", "避坑指南"]
featured: true
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/windows-11-mission-control-coding-agent-telemetry-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


# Windows 11 逆向工程实锤：MissionControlAggregator 深度监控 Coding Agent 进程与防护实战

前几天夜里，我在本地调一个自动化重构脚本。

跑着跑着，机箱风扇突然狂转。

我顺手切出任务管理器。

CPU 占用率不高，大约只有 8%。

但是磁盘写入速率反常，突然拉到每秒将近 20MB。

我没急着下结论。

随手打开 Process Monitor 抓事件。

把过滤条件设为进程名包含 svchost.exe 与 DiagTrack.exe。

抓出来的事件清晰显示，系统诊断服务正在高频刷写几个临时跟踪文件。

我试着在管理员终端敲下命令：

```cmd
sc stop DiagTrack
```

命令返回成功的一瞬间，磁盘写入立刻跌回 0MB/s。

两分钟后，风扇从 3800 转慢悠悠落回 1600 转静音状态。

重新拉起服务，写入再次出现。

因果对上了。

但诊断服务大半夜到底在记录什么？

昨天翻 Hacker News，看到独立研究员徐胜发的逆向审计帖。

原帖讨论已经超过五百点赞。

原帖地址我放这里：https://xusheng.dev/posts/windows-coding-agent-telemetry/

感兴趣可以直接看。

社区讨论串在：https://news.ycombinator.com/item?id=50524736

两处线索完全吻合。

作者在 Windows 11 预览版镜像里，抓到了针对 AI 编码工具的特征扫描。

测试样本来自 Windows 11 Build 26100 与 26200 镜像。

我没直接轻信网上的文字结论。

我从本地系统目录拷出同名文件，在沙箱虚拟机里完整过了一遍。

底层究竟有没有这份硬编码名单？

用 Sysinternals 工具包里的 strings 工具直接过一遍只读字符串：

```cmd
strings -el %SystemRoot%\System32\MissionControlAggregator.dll | findstr /i "claude codex cursor copilot"
```

终端屏幕立刻吐出了一串明文字符。

字符串完全能对上。

---

## 一、 二进制取证：系统 DLL 里的硬编码名单

事情的源头在 Windows 11 系统目录里。

文件位于 C 盘 Windows 目录下 System32 文件夹内的 MissionControlAggregator.dll。

对外公开的描述，只是模糊的任务控制与系统指标聚合器。

我们在 Binary Ninja 里加载这个 DLL 文件，直接跳到只读数据段。

底层的判定字串是毫无遮掩的明文宽字符。

不需要脱壳。

也不存在任何动态混淆。

<img src="https://img.freeaidog.com/images/blog/windows-11-coding-agent-telemetry-binary-ninja.webp" alt="Binary Ninja 逆向工程显示 MissionControlAggregator.dll 硬编码的 Coding Agent 进程名" />

反汇编窗口一打开，名单被清晰划分成两个独立的匹配槽位。

第一个槽位是命令行 AI 智能体。

代码直接比对四款可执行文件：
- copilot.exe
- claude.exe
- codex.exe
- agy.exe

如果用户走的是 WSL 虚拟机环境呢？

比对函数会自动剥离 exe 后缀。

它专门抓取 Linux 环境下的原生执行体。

第二个槽位是桌面端智能体与集成开发环境，包含五款目标：
- github.exe
- claude.exe
- codex.exe
- cursor.exe
- antigravity.exe

除了这些最近大火的 AI 工具，匹配器里还密密麻麻列出了一长串传统工具链。

比如 gcc.exe。

还有 clang.exe。

gdb.exe、javac.exe、dotnet.exe 全在里面。

python.exe 与 node.exe 也没落下。

Visual Studio 的 devenv.exe 赫然在列。

VS Code 的主程序 code.exe 同样在名单里。

微软甚至对部分常用运行时做了签名校验。

例如命中 node.exe 时顺带核对 Node.js 官方签名。

命中 python.exe 时核对 Python 基金会签名。

而 GCC 与 Clang 这类开源工具，只要名字匹配就会直接记录。

控制这个特征开关的内部标识符非常直白，就叫 CLICodingAgentUsage。

对应的内部功能编号是 62580080。

一个底层的操作系统诊断模块，为何要专程盯防开发者正在敲哪款 AI 辅助工具？

难道这是排查驱动崩溃所必须收集的数据吗？

往下看具体的订阅源。

---

## 二、 监控链路拆解：从内核事件到诊断上报

我没看到常驻的高 CPU 轮询线程。

在 Process Monitor 与内核调用栈里，它走的是事件驱动架构。

函数 Aggregator::OnLoad 初始化时，直接订阅了 Windows 操作系统内核已有的几组事件通道。

第一条通道针对原生 Windows 进程。

它订阅了内核事件 KernelProcess.AppStateChangeSummary。

同时订阅了图形交互事件 Win32kTraceLogging.AppInteractivitySummary。

每次新进程启动、挂起、切焦点或者退出，内核都会向总线派发一条轻量状态变更摘要。

聚合模块坐在事件总线旁，接收内核派发的应用身份通知。

拿到镜像名称后，直接去比对那份预设名单。

第二条通道针对 Linux 开发者。

它订阅了 WSL 内部的专用管理通道 Microsoft.Windows.Lxss.Manager.Exec 以及 ExecCritical。

这是针对子系统的内核探针。

只要你在 WSL 终端敲下 claude 或者 codex，驱动层向宿主汇报的 imageName 字段就会触发匹配。

命中非常迅速。

当可执行文件名命中上述规则后，模块内部会执行一段形如 std::_Func_impl_no_alloc 的回调。

匹配成功的状态并不会立刻发起网络外连。

它把结果写进内存里的一个聚合位图。

整个事件流转结构如下：

```text
Aggregator::OnLoad 初始化订阅
├─ WSL 虚拟机环境
│  └─ Lxss.Manager.Exec 事件派发 → CLI 智能体匹配器 [copilot, claude, codex, agy]
├─ 原生 Windows 环境
│  └─ KernelProcess.AppStateChangeSummary → CLI 智能体匹配器
└─ 桌面图形交互
   └─ Win32k.AppInteractivitySummary → 桌面级智能体匹配器 [cursor, claude, codex]

Aggregator::OnRelativeTimerExpire 内部定时器周期触发
└─ MissionControlAggregatorLogging::ReportCohorts<12,4,5>
   └─ 转化为 Segments.SubCohorts 位图
      └─ 打包进 ETW 事件并交由系统诊断服务落地
```

数据归集之后，模块内部定时器周期性触发。

函数 ReportCohorts<12,4,5> 将当前周期内的观测结果压缩成一组位图。

随后向系统派发名为 Segments.SubCohorts 的 TraceLogging 事件。

所谓的 12、4、5 究竟代表什么？

它们分别对应十二类常规开发工具、四个命令行智能体位、五个桌面端智能体位。

系统不回传冗长路径。

只把对应比特位置为 1。

例如，过去几小时内只要 Codex 运行过，属于它的比特位就会变高。

二进制标记写入磁盘，交由 DiagTrack 服务在后台统一上传。

函数调用链我只跟到 ReportCohorts 这一层。

至于更深处的云端打标与服务下发逻辑，属于微软云端未公开实现，这里不瞎猜。

---

## 三、 默认开启的遥测：你的日常调用是如何溜走的

我们看看系统安装时的初始向导。

在全新安装或大版本更新向导中，绝大多数区域设置默认勾选了什么？

默认勾选的正是包含可选诊断数据。

微软在官方服务协议里明确写明了定义。

可选诊断数据包括有关浏览的网站、如何使用应用和功能、增强型错误报告等数据。

刚才拆解的 CLICodingAgentUsage 特征，正是挂靠在可选诊断数据管道上的。

除非你在装机时手动切到仅发送必需数据。

否则系统在出厂状态下就已经默认授权了这套画像抓取。

个人写小脚本，这或许只是一张报表上的冷门数字？

但企业核心源码与工程选型呢？

研发活跃时段呢？

各业务团队内部的 AI 工具渗透率与生产力工具选型呢？

这种静默收集放在合规要求极高的场景下，属于明确的外部可观测性敞口。

天天琢磨代码防外泄，很多人根本想不到系统底层早把开发工具画像摸得清清楚楚。

---

## 四、 本地实战：彻底封死三层数据通道的防御 SOP

既然搞清了链路，在本地分三步阻断即可。

### 第一步：在图形界面关死可选遥测开关

打开设置面板。

依次进入隐私和安全性，再点开诊断和反馈。

检查诊断数据一栏。

把开关从发送可选诊断数据彻底切到仅发送必需的诊断数据。

随后把下方的定制体验关闭。

点击删除诊断数据按钮，清空微软本地缓存的设备使用轨迹。

但之前有开发者反馈过：系统推送累积更新后，个别界面配置项是否会被静默重置？

确实存在这种风险。

光关界面开关还不够稳妥，继续下重手。

### 第二步：组策略与注册表硬性阻断

如果是专业版、企业版或教育版，按 Win + R 键输入 gpedit.msc 打开本地组策略。

依次展开计算机配置、管理模板、Windows 组件、数据收集和预览版本。

右侧双击打开允许诊断数据策略。

勾选已禁用。

或者勾选已启用后在下方菜单选诊断数据已关闭。

保存退出。

家庭版没有组策略怎么办？

或者你想批量执行脚本？

以管理员身份启动终端，执行两条注册表命令：

```cmd
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v MaxTelemetryAllowed /t REG_DWORD /d 0 /f
```

这两条配置是系统诊断服务的最高约束。

数值设为 0，代表强制将遥测级别压死到关闭状态。

底层模块即使算出位图，也无法继续往上传递。

注意这里的管理边界：如果机器加入了企业 Active Directory 域或受 Intune 托管，域分发策略会覆盖本地注册表吗？

答案是肯定的，此时需由域管理员在组织单元层面统一下发阻断组策略。

### 第三步：禁用后台诊断服务与屏蔽上报域名

管子堵死了，负责打包推送的服务留着也是浪费内存。

在管理员终端里彻底停掉它们：

```cmd
sc stop DiagTrack
sc config DiagTrack start= disabled

sc stop dmwappushservice
sc config dmwappushservice start= disabled
```

如果你的环境对出网审计极其严苛，还可以在本地 hosts 文件里把已知回传域名全部引向回环地址：

```text
127.0.0.1 v10.events.data.microsoft.com
127.0.0.1 v20.events.data.microsoft.com
127.0.0.1 watson.telemetry.microsoft.com
127.0.0.1 telemetry.microsoft.com
```

这里同样存在一个边界条件：如果系统启用了基于 HTTPS 的 DNS 解析服务，即 DoH 协议，hosts 文件的静态条目会不会被绕过？

确实会被绕过。

因此严苛环境建议在网关防火墙上针对上述域名配置 DNS 阻断或丢弃对应 IP 段。

---

## 五、 验收与后续观察

配置完成后重启主机。

进桌面后打开管理员命令提示符，执行验证三连击：

```cmd
logman query providers | findstr /i "MissionControl"
sc query DiagTrack
sc query dmwappushservice
```

核对回显输出：
MissionControl 相关的追踪会话处于未激活状态，DiagTrack 与 dmwappushservice 的当前状态均为 STOPPED，启动类型均为 DISABLED。

你在实际环境中跑完这组检查，两个服务的状态是否也彻底停掉了？

回到 Process Monitor 里观察十分钟，先前高频的临时跟踪文件写入没有再次出现。

机箱风扇转速稳定在低位。

后续系统月度累积更新如果再次推送，会不会偷偷重置这些注册表项？

建议在每月补丁日后重新跑一次上述查询命令进行核对。

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/windows-11-mission-control-coding-agent-telemetry-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

