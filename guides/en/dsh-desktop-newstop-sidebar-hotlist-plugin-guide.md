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
Inference often requires waiting two to three minutes.
Switching to a primary web browser to check feeds?
Attention quickly drifts away.
By the time focus returns, the exact diff review context is lost.
What if a lightweight collapsible sidebar lived inside the AI client?
Watch terminal console logs on the right, skim community discussions on the left.
Fold away when done.
Zero clutter on the main workspace.
Clean.

The open-source plugin DS-NewsTop targets this workflow.
Crafted specifically for DSH Desktop, it leverages the Cordis modular architecture to dock multi-platform feeds directly inside the client activity bar.

<img src="/images/blog/ds-newstop-cover.webp" alt="DS-NewsTop sidebar multi-platform feed interface" />

---

## 1. Architecture Breakdown: Same-Origin Gateway and Zero Polling Trade-offs

What is the biggest risk when embedding feed plugins into a heavy developer client?
Uncontrolled background polling loops.
Many generic feed extensions poll dozens of endpoints every few minutes to claim real-time accuracy.
If the workstation is simultaneously running local LLM inference, continuous polling steals CPU cycles and network sockets.
It can degrade UI responsiveness.

DS-NewsTop takes a restrained architectural approach across three layers:

### 1. Same-Origin Gateway and In-Memory Cache
The sidebar webview never makes cross-origin requests directly to remote platforms.
The plugin host runtime registers a single internal endpoint in lib/index.js:
`/dsh-hot-list/items?id=sourceID`
The frontend exclusively pulls data from this local endpoint.
The host maintains a 60-second in-memory cache.
Opening or toggling panels within one minute hits memory directly, emitting zero outbound HTTP traffic.
Fast execution.

### 2. Upstream Aggregation and Content Sanitization
Data flows through two primary pipelines:
Eighteen mainstream technical platforms like GitHub, Hacker News, V2EX, and Zhihu route through the open-source NewsNow aggregation service.
A dedicated parser handles the Appinn community discovery channel Markdown feed, filtering pinned moderation guidelines and extracting top shared tools.
Host code hardcodes upstream targets, refusing arbitrary query URLs to prevent the endpoint from becoming an open proxy.
Secure boundary.

### 3. Restrained On-Demand Refresh Design
The extension completely eliminates automated polling timers.
Opening the panel loads only actively enabled platform cards.
Closed panels issue zero network calls.
Manual refreshes bypass local 60-second caches to pull fresh upstream data.
If an upstream node fails, the UI preserves previously cached items and displays an inline retry button instead of failing with a blank screen.
Stable behavior.

---

## 2. Compatibility Matrix and Measured Hardware Footprint

Running the extension across a three-hour production session yielded these verified runtime metrics:

### Benchmark Environment
- Host Platform: DSH Desktop 2.0.18, underlying core compatibility 0.2.1-rc.2 and 0.2.1-rc.3
- Workstation Hardware: MacBook Pro with M2 Max chip, 32GB unified memory
- Extension Version: dsh-hot-list 1.3.2 release

### Measured Performance and Resource Ledger

| Evaluation Metric | Multi-Tab Browser Feeds | DS-NewsTop Sidebar | Notes and Technical Boundary |
| :--- | :--- | :--- | :--- |
| Resident RAM Footprint | Browser tabs take 1.2GB | Host RAM rises by roughly 45MB | Bypasses heavy browser engines and trackers |
| Idle CPU Utilization | Background tabs take 1% to 3% | Completely silent, 0% CPU delta | No background polling intervals |
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

Verify JSON syntax to ensure no trailing comma corrupts the profile schema.

### 3. Restart and Verify Sidebar Entrance
Completely quit and reopen DSH Desktop.
The hotlist icon appears in the primary left activity bar.
Click to launch the sidebar panel.
The top-right gear icon allows toggling 19 data sources, with 14 active by default.
Preferences persist directly to local desktop storage.
Ready for daily use.

---

## 4. Acceptance Checklist

After completing deployment, verify these four operational checkpoints:
1. Verify resident memory: confirm host RAM delta stays well within 50MB.
2. Test offline resilience: disconnect network and click an active card, confirming cached lists remain visible with retry prompts rather than blank panes.
3. Validate theme synchronization: switch DSH to dark mode, ensuring borders, background tones, and typography update instantly.
4. Confirm zero background traffic: monitor network packet capture while idle, ensuring outbound HTTP traffic remains zero when sidebar is hidden. Ready for work.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/dsh-desktop-newstop-sidebar-hotlist-plugin-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

