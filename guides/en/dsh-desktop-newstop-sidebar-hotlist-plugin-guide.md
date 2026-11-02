---
title: "Monitoring Agent Inference While Browsing News: DS-NewsTop Sidebar Architecture and Setup for DSH Desktop"
description: "Hands-on engineering review of DS-NewsTop, a hotlist sidebar plugin for DeepSeek Harness Desktop. Built on Cordis to aggregate 19 developer and tech platforms, featuring 60-second in-memory caching, zero background polling overhead, and complete installation guide."
pubDate: 2026-11-02
author: "Lao Jiu · FreeAI Dog Lead Architect"
category: "Hands-on Guide"
tags: ["DeepSeek", "DSH", "VSCode", "Plugin Development", "News Aggregation", "Desktop App", "Workflow"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/dsh-desktop-newstop-sidebar-hotlist-plugin-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Monitoring Agent Inference While Browsing News: DS-NewsTop Sidebar Architecture and Setup for DSH Desktop

Running long-running tasks locally.
For example, having an Agent refactor an entire subsystem.
Model inference often requires waiting two to three minutes.
Switching to a primary web browser to check feeds?
Attention quickly drifts away.
By the time focus returns, the exact diff review context is lost.
What if a lightweight collapsible sidebar lived inside the AI client?
Watch terminal logs on the right, skim community discussions on the left.
Fold away when done, leaving the main workspace clear.

The open-source plugin DS-NewsTop addresses this workflow.
Crafted specifically for DSH Desktop, it leverages the Cordis modular runtime to dock multi-platform feeds directly inside the client activity bar.

<img src="/images/blog/ds-newstop-cover.webp" alt="DS-NewsTop sidebar multi-platform feed interface" />

---

## 1. Architecture Breakdown: Same-Origin Gateway and Zero Polling Trade-offs

Before installing this plugin into DSH, I first checked whether it contained any timer loops.

The concern was practical.
Last year I installed a similar feed extension that ran fetches every 90 seconds across a dozen endpoints concurrently.
I was running local 7B model quantization at the time, and cooling fans immediately spun up.
Since then, whenever an extension advertises real-time feeds, I search its source code for setInterval first.

DS-NewsTop has no background polling timers.
It issues network requests only when the panel opens, and goes completely silent once collapsed.
That factor alone determined whether I would keep it docked permanently.

Request pathways are strictly consolidated. The frontend only talks to the local host interface:
`/dsh-hot-list/items?id=sourceID`
The frontend queries this local endpoint exclusively.
The host runtime fetches from fixed upstream targets.
Upstream URLs are defined as hardcoded constants in host code, ignoring arbitrary query parameters.
I did not think much about this initially, until considering the SSRF attack surface:
What if the endpoint accepted arbitrary URLs?
It would become an open proxy, easily abused to scan internal network interfaces.

A 60-second in-memory cache is a welcome convenience.
Opening and closing panels repeatedly hits local memory directly, without issuing extra outbound requests.

Data flows through two primary pipelines:
Eighteen mainstream platforms like GitHub, Hacker News, V2EX, and Zhihu route through the open-source NewsNow aggregation service.
A dedicated parser handles the Appinn community discovery channel Markdown feed, filtering pinned moderation guidelines and extracting top shared tools.

If an upstream node returns a 502 error, the interface preserves previously fetched items and shows an inline retry button.
The first time NewsNow jittered, I was pleasantly surprised that the cached list remained intact rather than breaking into a blank screen.

---

## 2. Compatibility Matrix and Measured Hardware Footprint

After running for three hours, did memory leak?
Was CPU usage noticeable?
Memory and CPU were tracked via macOS Activity Monitor without specialized profilers, so these figures serve as ballpark engineering references.

### Benchmark Environment
- Host Platform: DSH Desktop 2.0.18, underlying core compatibility 0.2.1-rc.2 and 0.2.1-rc.3
- Workstation Hardware: MacBook Pro with M2 Max chip, 32GB unified memory
- Extension Version: dsh-hot-list 1.3.2 release

### Measured Performance and Resource Ledger

In terms of RAM, resident footprint stabilized within a 40MB to 50MB delta after cold start.
Over three hours of continuous use, no memory creep was observed.
By comparison, keeping seven news tabs open in Chrome 128 registered roughly 1.1GB to 1.3GB in Activity Monitor.
However, tab counts and background tab throttling vary widely, so this comparison serves only as a rough magnitude contrast.

For CPU, readings in the sampling window stayed consistently at 0.0.
A reading of 0.0 does not mean absolute zero, merely that utilization fell below the monitor resolution threshold.
What I can confirm is that no visible background spikes occurred, and the machine fan stayed silent all afternoon.

| Evaluation Metric | Multi-Tab Browser Feeds | DS-NewsTop Sidebar | Notes and Technical Boundary |
| :--- | :--- | :--- | :--- |
| Resident RAM Delta | Roughly 1.1GB to 1.3GB | Host RAM rises by roughly 45MB | Bypasses multi-process browser overhead |
| Idle CPU Behavior | Occasional 1% script spikes | Activity Monitor reading stays at 0.0 | Below tool resolution, no activity spikes |
| Network Egress Timing | Periodic polling every two minutes | Requests fire only on manual view/refresh | Conserves bandwidth for Agent traffic |
| UI Theme Adaptation | Requires third-party dark extensions | Automatically inherits DSH CSS variables | Supports light, dark, and auto sync |
| Ecosystem Compatibility | Any standard web browser | Strictly bound to DSH Desktop Cordis | Cannot install directly on VS Code or Cursor |

Note the clear engineering boundary:
This plugin is built around the Cordis modular runtime inside DSH Desktop.
It is not an extension on the VS Code marketplace and cannot run inside generic VS Code forks.
Non-DSH users seeking a similar setup must port the card UI against NewsNow APIs independently.
Do not attempt direct installation elsewhere.

---

## 3. Practical Deployment: Three Steps to Sidebar Integration

To configure the extension inside DSH Desktop 2.0.18, follow these terminal steps:

### 1. Install NPM Package Dependency
Launch DSH Desktop.
Open the internal terminal from the application menu, navigating to the default desktop profile directory:

```bash
cd ~/.dsh/profiles/desktop
pnpm add dsh-hot-list@1.3.2
```

Alternatively, download the release tgz archive from GitHub Releases and install via absolute file path.

### 2. Register Bundle in Profile Configuration
Open package.json in the current profile directory.
Locate the dsh.profile.bundles array, appending dsh-hot-list while keeping default official bundles intact:

```json
{
  "dsh": {
    "profile": {
      "bundles": [
        "@deepseek-ai/dsh-base",
        "@deepseek-ai/dsh-web-app",
        "dsh-hot-list"
      ]
    }
  }
}
```

Verify JSON syntax carefully.
Missing curly braces or trailing commas have tripped up many users, preventing the desktop profile from loading.

### 3. Restart and Verify Sidebar Entrance
Completely quit and reopen DSH Desktop.
The hotlist icon appears in the primary left activity bar.
Click to launch the sidebar panel.
The top-right gear icon allows toggling 19 data sources, with 14 active by default.
Preferences persist directly to local desktop storage.

---

## 4. Acceptance Checklist

Check the following four verification items:
- Verify resident memory: confirm host RAM delta stays within 50MB.
- Test offline resilience: disconnect network and click an active card, confirming cached lists remain visible with retry prompts rather than blank error panes.
- Validate theme synchronization: switch DSH to dark mode, ensuring borders, background tones, and typography update cleanly.
- Confirm zero background traffic: monitor network packet capture while idle, ensuring outbound HTTP traffic remains zero when sidebar is hidden.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/dsh-desktop-newstop-sidebar-hotlist-plugin-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

