---
title: "Compromised Claude Code Extension in Cursor Marketplace Drops SSH Backdoor: Full Incident Postmortem and Removal SOP"
description: "Alarms were raised over claims that the Claude Code extension was backdoored. Here is the verified technical reality: the compromised release was a modified third-party clone (v2.1.313) hosted on Cursor's independent marketplace, not Anthropic's official extension. Because this payload installs OS-level persistence, simply clicking 'uninstall' inside Cursor does not remove the threat. Here is the full cross-platform audit and cleanup guide for macOS, Linux, and Windows."
pubDate: 2026-10-30
author: "Xiao Ke (FreeAI Dog Core Team)"
category: "实战进阶"
tags: ["Cursor", "Claude Code", "Supply Chain Security", "Security Audit", "SSH Backdoor", "DevSecOps", "Cybersecurity"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/cursor-claude-code-extension-backdoor-audit-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Compromised Claude Code Extension in Cursor Marketplace Drops SSH Backdoor: Full Incident Postmortem and Removal SOP

Over the past 24 hours, viral headlines reverberated across Reddit r/LocalLLaMA and developer security forums claiming that the "official Claude Code VS Code extension was breached with a persistent SSH backdoor." 

However, **the sensational headlines are misleading**.

Following cross-verification by the FreeAI Dog technical team and official disclosures from Cursor, the precise scope of the breach has been clarified: **The compromised package was not Anthropic's official VS Code extension, but an unauthorized modified third-party version published to Cursor IDE's independent extension marketplace.** According to Cursor's security incident announcement, the impact is strictly confined to approximately 5,000 or fewer users who installed this specific build.

The critical danger, however, lies in persistence: **Once loaded, this malicious extension immediately injects root-level SSH authorized keys and background startup daemons into the operating system. If you or your engineering team installed this package from the Cursor marketplace, simply clicking "Uninstall" within Cursor will NOT neutralize the backdoor.**

This guide presents the complete postmortem alongside an actionable, copy-pasteable audit and removal checklist across macOS, Linux, and Windows.

---

## 1. Incident Breakdown: What Happened and Who Is Affected?

Before initiating terminal audits, developers should understand the exact vector of attack.

### Not Anthropic's Official VS Code Extension
The genuine Claude Code extension maintained directly by Anthropic on Microsoft's official Visual Studio Marketplace remains completely unaffected. Microsoft's marketplace enforces publisher verification, rigorous code signing, and strict namespace ownership. Developers using vanilla VS Code downloading directly from marketplace.visualstudio.com are safe.

### Attack Vector: Cursor IDE's Independent Extension Marketplace
Cursor, built upon the open-source VS Code core (Code - OSS), operates its own extension distribution infrastructure (Cursor Marketplace) rather than connecting natively to Microsoft's proprietary marketplace endpoints.

Attackers exploited vetting blindspots on this third-party channel:
- **Compromised Version**: `2.1.313` (distributed via `cursor.com/marketplace`).
- **Blast Radius**: Limited strictly to users who explicitly searched and installed this modified package inside Cursor (≤ 5,000 users).
- **Official Cursor Statement**: *"a modified Claude Code VS Code extension accessed through our Cursor IDE extension marketplace, impacting at most 5,000 users. This malicious version of the extension was not developed by Anthropic."*

### Why Is "Uninstalling" Ineffective?
Standard IDE plugins are sandboxed to their respective user extension directories. However, this payload executed a Node.js post-install script on initial load that escaped the editor runtime:
1. **Backdoor SSH Ingress**: Appended an attacker-controlled public key to `~/.ssh/authorized_keys` with the label `svc-telemetry`.
2. **OS-Level Persistence**: Spawned a disguised background daemon (mimicking native Apple or system monitoring processes) that revives itself on system reboot.

---

## 2. Cross-Platform Audit and Cleanup SOP

Open your terminal immediately and execute the relevant diagnostic commands for your environment.

### 1. Universal SSH Key Audit (Run This First on All Systems)

Whether running macOS, Linux, or Windows with an active OpenSSH Server, inspect your authorized keys file first.

#### Step A: Check for the `svc-telemetry` Marker
```bash
grep -n "svc-telemetry" ~/.ssh/authorized_keys
```
If this returns empty, the default attacker comment was not found. However, because attackers frequently alter comment flags, **you must verify the key fingerprint in Step B**.

#### Step B: Verify Key Fingerprints with `ssh-keygen`
List the SHA256 fingerprints of every authorized key on your machine:
```bash
ssh-keygen -lf ~/.ssh/authorized_keys
```
**🚨 Flagged Malicious Fingerprint**:
```text
SHA256:0Z6R+DF64Do2nwQic8a7kkfDLNNk4IiK+2U2Rci4htk
```
If your output contains this exact hash, your machine has been compromised. Clean it immediately:

```bash
# 1. Back up current keys
cp ~/.ssh/authorized_keys ~/.ssh/authorized_keys.bak.$(date +%F)

# 2. Strip entries containing the known marker
sed -i '' '/svc-telemetry/d' ~/.ssh/authorized_keys 2>/dev/null || sed -i '/svc-telemetry/d' ~/.ssh/authorized_keys
```
*(If the comment was randomized, open `~/.ssh/authorized_keys` in an editor and delete the line corresponding to the malicious SHA256 fingerprint).*

---

### 2. macOS Audit and Remediation

On macOS, the backdoor abuses a malicious `LaunchAgent` disguise.

#### Step A: Inspect the Rogue Apple Daemon
The payload disguises its plist file as `com.apple.sysmond.plist`.

> ⚠️ **Critical macOS Distinction**:
> The legitimate Apple monitoring daemon resides in the read-only system partition: `/System/Library/LaunchDaemons/com.apple.sysmond.plist`. **DO NOT touch or attempt to delete Apple's native binary!**
> The malicious rogue file is planted strictly in your **user directory**: `~/Library/LaunchAgents/com.apple.sysmond.plist`.

Check for presence of the payload files:
```bash
ls -la ~/Library/LaunchAgents/com.apple.sysmond.plist
ls -la ~/.config/.smt
```

#### Step B: Immediate Unload and Purge
If either file exists, execute the cleanup sequence:
```bash
# 1. Unload and unregister the malicious LaunchAgent
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.apple.sysmond.plist 2>/dev/null || launchctl unload ~/Library/LaunchAgents/com.apple.sysmond.plist 2>/dev/null

# 2. Delete the rogue plist definition
rm -f ~/Library/LaunchAgents/com.apple.sysmond.plist

# 3. Purge the malicious binary payload
rm -rf ~/.config/.smt

# 4. Check for surviving background processes
ps aux | grep -i "\.smt" | grep -v grep
```
If an active process matching `.smt` is returned, kill it immediately with `kill -9 <PID>`.

---

### 3. Linux Audit and Remediation

On Linux distributions, persistence is primarily established via recurring `crontab` jobs.

#### Step A: Inspect User Crontab for `@reboot` Triggers
```bash
crontab -l
```
Look for lines targeting `~/.config/.smt` such as:
```text
@reboot ~/.config/.smt >/dev/null 2>&1
```

#### Step B: Purge Crontab and Artifacts
```bash
# 1. Back up active crontab
crontab -l > /tmp/crontab.bak

# 2. Filter out .smt triggers and reload
crontab -l | grep -v "\.config/\.smt" | crontab -

# 3. Delete the backdoor payload
rm -rf ~/.config/.smt

# 4. Kill active processes
pkill -f "\.config/\.smt"
```

---

### 4. Windows Audit and Remediation

On Windows, persistence typically hooks into the Windows Task Scheduler or User Startup folder.

#### Step A: Query Scheduled Tasks and Startup Folders
Launch PowerShell as Administrator and run:
```powershell
# 1. Search for rogue tasks named sysmon
Get-ScheduledTask | Where-Object { $_.TaskName -like "*sysmon*" -or $_.TaskPath -like "*sysmon*" }

# 2. Inspect User Startup directory
Get-ChildItem "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup" | Where-Object { $_.Name -like "*sysmon*" }
```

#### Step B: Remove Tasks and Scripts
```powershell
# Unregister task if found (adjust task name if necessary)
Unregister-ScheduledTask -TaskName "SysMon" -Confirm:$false

# Delete startup script artifact
Remove-Item "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\sysmon.cmd" -Force -ErrorAction SilentlyContinue
```

---

## 3. Post-Incident Response: Mandatory Credentials Rotation

Security engineering requires operating under the assumption of breach. If your system was confirmed to contain the malicious fingerprint or active `.smt` binaries, you must perform credential rotation:

1. **Rotate All Local SSH Private Keys**:
   Adversaries gaining command execution frequently harvest `~/.ssh/id_rsa` and `~/.ssh/id_ed25519`. Generate completely fresh keypairs and update your public keys on GitHub, GitLab, and enterprise bastion hosts immediately.
2. **Invalidate Environment Variables and API Keys**:
   Workstations used for AI engineering frequently house tokens like `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, AWS secrets, and database strings in `~/.zshrc`, `~/.bashrc`, or project `.env` files. Consider these compromised and regenerate them through their respective cloud consoles.
3. **Completely Purge the Extension from Cursor**:
   Ensure the rogue package is removed from your Cursor extension list, and rely strictly on verified Anthropic repositories moving forward.

---

## 4. Architectural Lessons for the AI IDE Supply Chain

From recent Open VSX namespace hijackings to this Cursor marketplace injection, supply chain attacks targeting AI-native development environments are becoming an aggressive trend.

Modern IDE forks like Cursor, Windsurf, and Trae offer immense developer velocity, but because they are cut off from Microsoft's proprietary extension marketplace, they rely on custom repositories or community mirrors. Attackers understand that developers trust their editor environment implicitly.

### Core Defensive Tenets for AI-Assisted Engineers
- **Verify Publisher Provenance**: Before installing packages in alternative IDEs, double-check the publisher identity, associated GitHub repository, and stars against official vendor domains.
- **Isolate Local Runtimes**: Isolate unvetted coding agents, autonomous CLI extensions, and experimental packages inside Docker containers or Dev Containers with restricted host volume mounts.
- **Audit Local Persistence Regularly**: Incorporate `~/.ssh/authorized_keys`, `LaunchAgents`, and `crontab` into your personal scheduled security hygiene routine.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/cursor-claude-code-extension-backdoor-audit-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

