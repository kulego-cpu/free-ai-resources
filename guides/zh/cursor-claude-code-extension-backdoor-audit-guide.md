---
title: "你在 Cursor 市场装的 Claude Code 扩展可能带 SSH 后门：事件复盘与全平台排查清除清单"
description: "近日有报道声称“Claude Code 扩展被植入后门”，引发开发者圈恐慌。本文第一时间深度还原技术真相：被攻破的是 Cursor IDE 市场中的篡改版（v2.1.313），而非 Anthropic 官方扩展。由于该恶意扩展包含系统级持久化后门，单纯在编辑器中“卸载扩展”无法将其清除。本文提供针对 macOS、Linux、Windows 的完整指纹核验与一键清除排查指南。"
pubDate: 2026-10-30
author: "小可 (FreeAI Dog 核心开发组)"
category: "实战进阶"
tags: ["Cursor", "Claude Code", "供应链安全", "安全防护", "SSH后门", "后门排查", "网络安全"]
featured: true
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/cursor-claude-code-extension-backdoor-audit-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


# 你在 Cursor 市场装的 Claude Code 扩展可能带 SSH 后门：事件复盘与全平台排查清除清单

过去 24 小时内，Reddit r/LocalLLaMA 与安全社区掀起了一场轩然大波：一条高赞爆料称“Claude Code 官方 VS Code 扩展被攻破并被植入持久化 SSH 后门”，瞬间引发了大量正在使用 AI 辅助编程开发者的集体恐慌。

然而，**标题往往具有严重的误导性**。

经过 FreeAI Dog 团队与安全社区的交叉比对与官方声明核验，真相迅速厘清：**被植入恶意外联后门的并不是 Anthropic 官方发布的扩展，而是 Cursor IDE 独立扩展市场上架的一个同名修改版（Modified Version）**。根据 Cursor 官方团队发布的应急公告，受影响用户规模预估在 5,000 人以内。

但问题的严峻性在于：**恶意扩展在被激活后，会立即在操作系统底层写入持久化保活机制（Persistence Mechanism）。如果你或你的团队曾经在 Cursor 市场中安装过此扩展，仅仅在 Cursor 界面中点击“卸载（Uninstall）”根本无法清除后门！**

为了帮助所有开发者彻底排查隐患，本文将完整复盘事件脉络，并给出跨平台的逐行排查与彻底清除指南。

---

## 一、 事件技术真相还原：谁被黑了？谁是安全的？

在展开排查之前，我们必须首先拨开社交媒体上的恐慌迷雾，实事求是地看清攻击范围。

### 1. 并非 Anthropic 官方 VS Code 扩展
Anthropic 官方维护并上架在微软官方 Visual Studio Marketplace 上的正版扩展没有任何被篡改的迹象。微软官方市场拥有严格的代码签名与发布者身份认证机制，直接在 VS Code 官方市场下载的用户无需恐慌。

### 2. 事故源头：Cursor IDE 独立市场的供应链投毒
Cursor 作为基于 VS Code 开源核心（Code - OSS）构建的分支 IDE，由于无法直接无缝使用微软专属的官方插件市场协议，采用了自建的扩展分发体系（Cursor Marketplace）。

攻击者正利用了这一分发通道的审核缝隙：
- **恶意版本号**：`2.1.313`（上架于 cursor.com/marketplace）。
- **受影响范围**：仅限在 Cursor 市场中手动搜索并安装过该特定版本的开发者（≤ 5,000 人）。
- **Cursor 官方声明**：*"a modified Claude Code VS Code extension accessed through our Cursor IDE extension marketplace, impacting at most 5,000 users. This malicious version of the extension was not developed by Anthropic."*

### 3. 为什么“卸载扩展”完全没用？
常规的 IDE 扩展在卸载后，其插件目录会被删除。但该恶意版本在首次运行时，会执行一段 Node.js 安装钩子（Post-install Hook），直接越过编辑器沙箱，向用户的系统关键配置中注入外部公钥与自启动守护进程：
1. **注入 SSH 免密通道**：向 `~/.ssh/authorized_keys` 强行追加一段攻击者的公钥，开辟外部反弹 SSH 隧道。
2. **植入自启动保活脚本**：在用户目录下伪装成系统守护程序（如伪装成 Apple 系统组件），在重启或开机时自动复活。

---

## 二、 全平台排查与彻底清除指南

请各位使用 Cursor 的开发者立即打开终端，按照自己所使用的操作系统，逐项运行以下命令进行审计。

### 1. 通用检查：核对 SSH 授权密钥（必须优先执行）

无论你使用的是 macOS、Linux 还是配置了 OpenSSH Server 的 Windows，首先检查你的授权公钥列表。

#### 步骤 A：检查是否包含 `svc-telemetry` 标识
在终端运行：
```bash
grep -n "svc-telemetry" ~/.ssh/authorized_keys
```
如果输出为空，说明未直接匹配到攻击者默认的注释标签；但为了防止攻击者后续变异注释，**必须进行步骤 B 的指纹比对**。

#### 步骤 B：使用 `ssh-keygen` 核验公钥指纹
运行以下命令列出所有授权公钥的 SHA256 指纹：
```bash
ssh-keygen -lf ~/.ssh/authorized_keys
```
**🚨 重点核对恶意指纹**：
```text
SHA256:0Z6R+DF64Do2nwQic8a7kkfDLNNk4IiK+2U2Rci4htk
```
如果终端列表中出现了上述哈希值，证明你的机器已经被写入后门，请立即执行清除：

```bash
# 备份当前 authorized_keys
cp ~/.ssh/authorized_keys ~/.ssh/authorized_keys.bak.$(date +%F)

# 移除包含恶意注释或恶意公钥的行
sed -i '' '/svc-telemetry/d' ~/.ssh/authorized_keys 2>/dev/null || sed -i '/svc-telemetry/d' ~/.ssh/authorized_keys
```
*(如果恶意行公钥注释已被篡改，请直接使用文本编辑器打开 `~/.ssh/authorized_keys`，手动删除对应指纹的那整行公钥)*。

---

### 2. macOS 专属排查与清除

在 macOS 环境下，该后门利用了伪装的 LaunchAgent 自启动项。

#### 步骤 A：排查伪造的 Apple 守护服务
攻击者将保活文件命名为 `com.apple.sysmond.plist`，试图混淆视听。

> ⚠️ **关键安全常识**：
> 苹果原装正版的系统守护程序位于只读系统目录：`/System/Library/LaunchDaemons/com.apple.sysmond.plist`，那是 macOS 原生监控组件，**绝对不要尝试去动它**！
> 恶意后门文件位于**用户个人目录**：`~/Library/LaunchAgents/com.apple.sysmond.plist`。

运行以下命令检查恶意文件是否存在：
```bash
ls -la ~/Library/LaunchAgents/com.apple.sysmond.plist
ls -la ~/.config/.smt
```

#### 步骤 B：立即停止并彻底清除
如果上述任一文件存在，请顺序执行以下命令卸载并删除：
```bash
# 1. 停止并注销恶意 LaunchAgent
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.apple.sysmond.plist 2>/dev/null || launchctl unload ~/Library/LaunchAgents/com.apple.sysmond.plist 2>/dev/null

# 2. 彻底删除恶意 plist 描述文件
rm -f ~/Library/LaunchAgents/com.apple.sysmond.plist

# 3. 删除恶意载荷核心本体
rm -rf ~/.config/.smt

# 4. 检查当前是否残留恶意运行进程
ps aux | grep -i "\.smt" | grep -v grep
```
如果 `ps aux` 发现仍然有 `.smt` 相关进程存活，使用 `kill -9 <PID>` 强制终止。

---

### 3. Linux 专属排查与清除

在 Linux 发行版中，该恶意程序主要通过计划任务（Crontab）实现开机复活。

#### 步骤 A：检查 `@reboot` 定时任务
运行命令查看当前用户的计划任务：
```bash
crontab -l
```
检查输出中是否存在指向 `~/.config/.smt` 或包含类似如下字段的命令：
```text
@reboot ~/.config/.smt >/dev/null 2>&1
```

#### 步骤 B：清理 Crontab 与载荷文件
```bash
# 1. 备份现有 crontab
crontab -l > /tmp/crontab.bak

# 2. 剔除包含 .smt 的恶意行并重新载入
crontab -l | grep -v "\.config/\.smt" | crontab -

# 3. 彻底删除恶意载荷文件
rm -rf ~/.config/.smt

# 4. 终结潜在后台进程
pkill -f "\.config/\.smt"
```

---

### 4. Windows 专属排查与清除

在 Windows 系统下，后门主要隐藏于计划任务库（Task Scheduler）或系统启动文件夹中。

#### 步骤 A：检查计划任务与启动项
以管理员身份打开 PowerShell，运行以下命令：
```powershell
# 1. 查询是否存在名为 SysMon 或伪装命名的异常计划任务
Get-ScheduledTask | Where-Object { $_.TaskName -like "*sysmon*" -or $_.TaskPath -like "*sysmon*" }

# 2. 检查用户启动目录是否存在恶意脚本
Get-ChildItem "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup" | Where-Object { $_.Name -like "*sysmon*" }
```

#### 步骤 B：删除恶意任务与脚本
如果发现异常项（例如指向非微软官方 Sysinternals 工具路径的 `sysmon.cmd` 或未知脚本）：
```powershell
# 停止并删除计划任务（假定任务名为 SysMonTask）
Unregister-ScheduledTask -TaskName "SysMon" -Confirm:$false

# 删除启动项脚本
Remove-Item "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\sysmon.cmd" -Force -ErrorAction SilentlyContinue
```

---

## 三、 深度止损：如果发现中招，你还需要做哪些处置？

安全排查遵循“深度防御”与“假设已被攻破”原则。如果你在排查中确实发现了上述恶意指纹或 `.smt` 进程，除了删除本地后门文件外，必须执行以下二次止损操作：

1. **轮换 SSH 密钥对**：
   攻击者在拿到目标机权限后，通常会顺手读取 `~/.ssh/id_rsa`、`~/.ssh/id_ed25519` 等本地私钥文件。如果你的机器存有用于连接生产服务器、GitHub、GitLab 的私钥，**请立即生成全新的密钥对并更新线上所有代码托管平台与云服务器的公钥授权**。
2. **轮换本地环境变量中的 API 密钥**：
   开发者本地机器往往配置了 `OPENAI_API_KEY`、`ANTHROPIC_API_KEY`、AWS/GCP 凭证或各类数据库连接串。一旦发现被种后门，应视所有储存在 `~/.zshrc`、`~/.bashrc`、`~/.env` 中的凭证处于泄露状态，立即在服务商后台重置。
3. **彻底卸载 Cursor 中的残留问题版本**：
   在 Cursor 扩展列表中，确认已经彻底卸载 `Claude Code (Modified)`，并等待官方清理后的安全通道通知。

---

## 四、 行业反思：AI 时代 IDE 插件供应链的防御铁律

从数月前的 Open VSX 投毒，到如今 Cursor 独立市场的篡改版插件，这已经不再是孤立偶然的个案，而是**针对 AI 辅助编程生态的一类典型供应链攻击模式**。

现代 AI 编程助手（如 Cursor、Windsurf、Trae 等）正在以惊人的速度迭代，但它们往往无法直接接入微软 VS Code 封闭的官方扩展生态，只能依赖第三方镜像仓库或自建市场。这种“生态割裂”恰恰成了黑客投毒的黄金温床。

作为高频与 AI 协同的开发者，我们建议建立以下三条防御铁律：

| 防御层级 | 核心策略 | 实操建议 |
| :--- | :--- | :--- |
| **来源核验** | 认清官方市场与第三方镜像差异 | 在非官方 IDE 中安装涉及敏感网络/终端权限的扩展前，必须先核验发布者（Publisher）域名与 GitHub 源码链接。 |
| **权限隔离** | 本地代码与高权限系统环境分离 | 涉及外部不可信脚本、新上架扩展的开发环境，优先在 Docker 容器、Dev Containers 或精简沙箱中运行。 |
| **持续审计** | 定期对系统保活与 SSH 资产巡检 | 将 `~/.ssh/authorized_keys`、`LaunchAgents`、`crontab` 纳入个人终端安全巡检清单，杜绝幽灵凭证。 |

保持警惕，定期排查，方能在拥抱最先进 AI 生产力的同时，守住代码与数字资产的底线安全！

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/cursor-claude-code-extension-backdoor-audit-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)

