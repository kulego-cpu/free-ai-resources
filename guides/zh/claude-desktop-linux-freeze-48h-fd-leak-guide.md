---
title: "排查 Claude 桌面端定期冻结：27 字节无读响应与 Chromium 句柄泄露"
description: "复盘 Linux 环境下 Claude 桌面端长时间运行无故挂起的完整链路：从 1.5TB 崩溃转储分析、Chromium 共享内存管道与 Cache-Control 分流逻辑，到利用 Caddy 本地改写标头的规避方案。"
pubDate: 2026-11-15
author: "小可"
category: "踩坑与架构"
tags: ["Claude Desktop", "Chromium", "Linux", "Electron", "逆向工程", "Bug 排查"]
featured: false
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/claude-desktop-linux-freeze-48h-fd-leak-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


周三下午，工作站卡住了。

鼠标大概每秒两帧。敲键盘要等三四秒才出字。

打开终端敲 uptime。回车后等了二十秒。系统负载输出为 46.34。这台机器 16 核，等于每个核的队列里堆了近三个任务。什么脚本能跑成这样？我第一反应是编译失控了，ps aux 一看，没有。

检查内核 I/O 阻塞压力指标。运行检查命令：

```bash
# 查看内核 I/O 阻塞压力指标
cat /proc/pressure/io
```

输出里的 full 十秒平均值冲到了 85.3。整块 NVMe 固态硬盘被连续写盘堵死。

切到屏幕右边的 Claude 桌面端。界面早已经白了，只剩骨架屏占位块：

<img src="https://img.freeaidog.com/images/blog/claude-desktop-frozen-skeleton.webp" alt="Claude 桌面端无响应冻死后的骨架屏现场" />

按鼠标毫无反应。点哪都没动静。

找了一圈，进程列表里在狂写盘的是 systemd-coredump 守护进程。Claude 的渲染进程崩了。内核发出了 SIGILL 终止信号。系统在把整个进程镜像写盘。五分钟狂写 92.4GB，最后超时被内核砍了，连个完整的调试文件都没能留下。这怎么排查？

在 Linux 上跑这个客户端，大概每两三天死一回。要么白屏，要么冻死。日志翻来覆去只有一句空话：主页面无响应，即将杀死并重载。它从来没重载成功过。

我花了两周打补丁、挂调试器，顺着线索揪出了这起由 27 字节冷门响应引发的句柄堆积。

## 为什么崩溃转储文件能膨胀到 1.5TB？

渲染进程为什么能有 1.5TB？这是 Chromium 虚拟地址空间保留机制惹的祸。

客户端基于 Electron 44 打包。内核是 Chrome 152。启动时 V8 预留 1TB 沙箱空间。前后分别垫上 64GB 与 288GB 保护区。加上内存分配器与 GC 堆，虚拟空间直接过了 1.3TB。

正常情况下，Linux 可以调 madvise 传入 DONTDUMP 标记。它告诉内核这块虚拟地址不要写盘。

但渲染进程有 seccomp 白名单。翻开 baseline_policy.cc 源码，策略只放行六种特定的 advice 标记：DONTNEED、WILLNEED、RANDOM、REMOVE、NORMAL、FREE。MADV_DONTDUMP 并不在放行列表里。标记不写盘的操作被内核直接打回 EPERM 权限拒绝。

崩溃发生时，内核走管道灌流。管道不能跳过空洞。虚拟地址变成了实打实的零字节写盘。

后来看原作者 Vladimir Petrzhikovskii 在 blog.dfb.sh/blog/claude-desktop-freeze 的记录，他用 Rust 写了个走 lseek 数据寻洞特性的工具，才把 1.5TB 的稀疏文件检索压缩到了零点七秒。

## 抽丝剥茧：942 个共享内存区域与 27 字节心跳

分析内存转储时，渲染进程持有的文件描述符堆了将近一千个：

```bash
# 查看当前渲染进程持有的文件句柄数量
ls -l /proc/$RENDERER_PID/fd | wc -l
```

其中 942 个是 2MB 的 POSIX 共享内存段。strings 扫过那 938 个段，内容完全一致，全是一行 27 字节的 JSON：

```json
{
  "refresh_after_seconds": 0
}
```

为了这 27 个字节，进程开了九百多个 2MB 的共享内存段。我盯着这个数字看了挺久：一个清空在场状态的响应，为什么非要带刷新指令？

顺着虚函数表与 V8 堆内存反查，这批响应来自 Code 标签页。

标签页设计了在场心跳机制。窗口最小化或后台闲置时，前端发一条清理心跳：

```http
POST /v1/code/channels/session-id/client/presence HTTP/1.1
Host: api.anthropic.com
Content-Type: application/json

{"client_id": "client-token", "clear": true}
```

请求带有 keepalive 标记。告诉云端服务器当前用户离开了，接口返回一个包含 0 秒刷新指令的 27 字节响应体。

因为是发后即忘的上报任务，前端代码在发起请求后根本没读响应体：fetch 挂了个空的 catch 完事。后台挂着的时候，这种请求以每小时大概几十个的频次静默累加。

三十多小时过去，句柄数量直奔上千个。

## 责任归属：从 no-store 机制到 IPC 消息死锁

服务端回了 no-store，前端又没读流。

Chromium 这边按规范得把数据扣在管道里等人来取，不会主动排空。Cache-Control: no-store 的语义本来是防止落盘，结果在这里变成了防止回收。我写了个最小复现，手动触发一次 GC，包装对象确实被回收了，/proc/$PID/fd 的计数没动。按常理推断，垃圾回收器清空对象后，底层管道和句柄难道不该自动释放？

翻开 fetch_manager.cc 代码。响应到达时，Blink 原本会调缓冲器。它把数据从底层管道读入内存，读完关闭管道、释放句柄。逻辑很规整。

但源码有特殊分支。响应头若带 Cache-Control: no-store，系统直接跳过缓冲器。

初衷本来很美好。服务端既然禁止存储，为防内存堆积，内核选择不缓冲。

代价是什么？数据必须停在 Mojo 管道里。它必须等前端主动读取，或者显式 cancel 取消。

服务端加了 no-store 标头，前端又完全不读，两头对不上。如果前端一直不读呢？那就一直占着。

Chromium 坚守规范不肯提前排空管道，V8 的垃圾回收器管不到底层的 Mojo C++ 资源释放。于是每一个无人在意的 27 字节响应，就各自死死占着一个 2MB 共享内存块与一个系统文件句柄。

Linux 下普通渲染进程的软句柄上限通过系统配置被限制为 1024：

```bash
# 查看系统的软句柄限制
ulimit -Sn
```

管道堆到一千个左右，可用句柄彻底见底。

此时主线程正准备通过本地存储执行一条常规数据读写，底层向浏览器主进程派发了一条 IPC 消息。主进程试图通过套接字将一个新句柄移交给渲染进程。

但内核在接收时遇到了句柄耗尽，给套接字打上了截断标记 MSG_CTRUNC。

Chrome 152 的旧版 Mojo 处理不了截断。接收端发现句柄少了，就无限期卡在队列里。

后续所有 IPC 指令全堵在后面。主线程僵死。画面冻结。窗口成了毫无反应的僵尸。

## 官方工单反馈：预期行为与机器人自动关闭

排查出这个硬伤后，作者 Vladimir 向两个上游团队分别提交了缺陷报告。

第一站是 Chromium 官方缺陷追踪平台。

作者提交了编号为 582417463 的工单，建议当渲染进程遇到文件句柄耗尽时，应当让请求直接抛出常规的类型异常，而不是让整个浏览器渲染引擎在底层死锁。

Chromium 团队当天就将工单标记为重复项并直接关闭。工单关联到的历史工单 378822137 已经挂在需求池里很长时间。这算不算预期行为？

对应负责人在评论区的答复很干脆：如果你有能力读取响应却自己选择不读，我们认为这属于符合预期的正常行为。浏览器无法预知你未来是否会去读取它，因此必须将它一直保留在内存管道中。

第二站是 Anthropic 的官方开源仓库 anthropics/claude-code。

既然底层浏览器引擎不肯妥协，在应用层修复总该很容易。代码改动极其轻微，只需要在发后即忘的请求末尾补上一句对响应流执行 cancel 取消的操作，让管道及时释放即可。

作者在仓库提交了编号为 107887 的工单。

结果没有任何人类工程师出面沟通。几分钟后，仓库的自动化管理机器人打上了 invalid 标签，并留下一句话将其关闭：当前暂时关闭——该问题看起来与 Claude Code 无关。

一边是浏览器团队认为句柄泄露符合规范，一边是大模型团队的自动化脚本把崩溃问题拒之门外。

两头踢皮球。买单的只能是天天强退客户端的用户。这算哪门子预期行为？

## 免重编译落地：本地标头改写规避实操

既然两头官方都不打算动，本地规避反而是最快的路子。

修复思路就两种：要么在前端拦截 fetch 对响应流调用 cancel 取消，要么在网络层把这个响应头的 no-store 扒掉。

Claude 桌面端在运行时从线上动态拉取前端界面，我们无法直接去改写二进制包里的静态脚本。但在操作系统网络层，治理这种无意义的吸血心跳轻而易举。

如果本地跑着 Caddy，直接改写响应头即可。以轻量级 Caddyfile 为例：

```caddyfile
api.anthropic.com {
    reverse_proxy https://api.anthropic.com {
        header_up Host {upstream_hostport}
        @presence path /v1/code/channels/*/client/presence
        handle_response @presence {
            header -Cache-Control
        }
    }
}
```

将标头从原点响应中剥离后，Chromium 的内部缓冲机制会在 250 毫秒内自动将数据排空并关闭管道。文件句柄数量将全天候稳定在几十个的低位，再也不会发生无限累加。

另一种更轻量的本地做法是利用调试端口启动客户端：

```bash
# 启动客户端时附加本地远程调试端口
claude-desktop --remote-debugging-port=9222
```

挂上补丁之后连着跑了七天，句柄数在 36 到 41 之间浮动，没再往上走。RSS 稳定，load 一直在 0.1 以下。

两个上游的 issue 都还关着。582417463 挂着 duplicate，107887 挂着 invalid。我估计短期不会有动静。

Caddyfile 那段就在我 dotfiles 的 claude-fix.caddy 里，换机器直接引用。真要等他们修，48 小时的时钟还得走几轮？

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/claude-desktop-linux-freeze-48h-fd-leak-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

