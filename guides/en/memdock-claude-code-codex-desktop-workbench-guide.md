---
title: "Managing Claude Code and Codex Memory and Skills: Hands-On with the MemDock Desktop Workbench"
description: "Dissecting the practical architecture of MemDock, a desktop workbench for Claude Code and Codex. Solving CLAUDE.md and AGENTS.md rule rot via staging reviews, bi-directional session transfers, Git Worktree parallel isolation, and private GitHub repository sync."
pubDate: 2026-11-09
author: "Xiao Ke"
category: "Hands-on Benchmarks"
tags: ["MemDock", "Claude Code", "Codex", "AI Coding", "Git Worktree", "CLAUDE.md", "Open Source Tools"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/memdock-claude-code-codex-desktop-workbench-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Managing Claude Code and Codex Memory and Skills: Hands-On with the MemDock Desktop Workbench

If you code with Claude Code or Codex, have you ever felt disgusted by memory pollution?

In many project root folders, rule files linger indefinitely: `CLAUDE.md` or `AGENTS.md`.

At the beginning, everything feels great. You pack them with project rules, testing workflows, and build scripts. Two weeks of heavy hacking later, your testing framework shifted from Jest to Vitest; build commands evolved completely. Starting a fresh session, the model does not become smarter; it stubbornly attempts running deprecated terminal commands.

Updated the test commands in prose? The model simply ignores you.

Outdated advice from past sessions remains trapped in the auto-memory cache, derailing subsequent conversations. Does constant troubleshooting get exhausting? Thoroughly exhausting.

Dual-tool fragmentation makes things even messier.

Claude Code inside the terminal provides lightning-fast execution. Codex with its multi-column graphical interface makes inspecting large multi-file diffs far easier on the eyes. Yet plugins and memory operate in isolated silos. Install a suite of custom Skills in Claude Code, and switching to Codex forces manual directory rummaging to configure everything again. Switching laptops resets both toolchains to zero.

Developer cturing published the open-source MemDock workbench. Code repository hosted on GitHub: `LingyiChen-AI/MemDock`.

Let us establish what it does not do.

It does not generate code, does not write prompts for you, and does not route to third-party models. Install it, and your Claude Code remains the exact same Claude Code, terminal command remains `claude`, with a single `.memdock/` folder added to your project root storing historical rule revisions. Revert when things break; that is its sole function.

It manages three specific jobs:

Which rules deserve to stay. Which interface hosts the session. How to migrate configurations across machines.

Just those three; it touches nothing else.

## Memory Anti-Rot: Why a Staging Review Buffer is Non-Negotiable

The fatal flaw of most automatic memory implementations is permitting models to write directly into persistent rule files.

Late into the night, you hack a temporary patch to bypass a transient error. The model adopts that desperate workaround as an inviolable truth, silently appending it to the project rule cache. Project code quality deteriorates under that toxic memory; hitting identical traps on subsequent runs becomes pure agony.

MemDock takes a conservative approach: establishing a staging review buffer.

The software indexes rules globally and per-project, searching across `CLAUDE.md`, `AGENTS.md`, and Claude Code local memories. When stale rules appear, split-pane editing allows instant manual corrections with full revision rollback.

The standout feature is the weekly experience extraction workflow.

The system scans recent conversations, identifying technical takeaways worth preserving alongside outdated rules requiring revision.

None of these candidates write to disk automatically.

Extracted suggestions drop directly into a review staging queue. Sitting at your monitor, you inspect old rules on the left, proposed additions in the middle, and original chat transcripts on the right.

You click accept, and it commits to disk; you click reject, and toxic memories vanish immediately.

During my initial extraction run, I hit a snag: build logs from `dist/` were parsed as candidate rules. Adding `dist/` and `coverage/` to MemDock ignore lists cleaned up the extraction pipeline completely.

Paired with memory dependency graphing, active rules link backward to the exact dialogue snippet where they originated. Auditing configuration rules no longer feels like wandering in the dark.

## Bridging the Gap: Bi-Directional Session Transfers Between Claude Code and Codex

Which tool should you standardize on?

Only beginners obsess over binary choices; seasoned developers switch back and forth constantly.

Terminal interactions respond immediately, making Claude Code ideal for targeted test runs, single-file edits, and bulk refactors. Architectural design over complex modules, meanwhile, benefits from multi-pane graphical diffs inside Codex.

MemDock functions as a bi-directional translation bridge.

Chatting with Claude Code in the CLI and finding multiple modules tangled in cross-cutting edits? Convert the session into Codex with one click and continue visually.

The reverse works equally well. Outline specifications visually in Codex, then transfer the session back to the native terminal for batch execution.

Skill plugin management receives the same unified treatment.

The interface enumerates global and project-level Skills across both ecosystems, organizing Superpowers plugins cleanly by package. Install packages directly from GitHub URLs, local paths, or ZIP archives.

Dialed in a solid debugging Skill inside Claude Code? Click copy, and it mirrors into Codex without digging through buried configuration folders.

## Parallel Isolation: The Real Cost of Git Worktrees and Cloud Sync

Running AI assistants inside a shared workspace invites file-locking chaos.

One session tasks the assistant with database schema overhauls, while another prompt attempts tweaking frontend forms. Changes collide inside a single working directory, uncommitted edits overwrite each other, and merge states deadlock.

MemDock embeds native Git Worktree workflows directly.

Spawn three branches simultaneously, running `git worktree add ../feat-schema feat/schema` across distinct directories. Querying status in the terminal shows:

```text
$ git worktree list
/Users/cool/work/project-core       b39a1c2 [main]
/Users/cool/work/feat-schema        d7e4120 [feat/schema]
/Users/cool/work/feat-ui-auth       a199f03 [feat/ui-auth]
```

Do not expect it to eliminate conflicts.

It shifts conflicts from writing time to merge time. Merging branches last week still threw conflict markers across two files, though resolving them stayed manageable because changes were confined to thirty schema lines.

The true benefit is context isolation. One directory runs tests while another edits UI components, preventing cross-contamination across `node_modules` folders.

The trade-off requires upfront budgeting. Running three worktrees doubles disk consumption. Each folder requires its own package installations, eating SSD space rapidly.

Migrating across laptops remains practical. Export memory rules and Skill lists into structured JSON, pushing them to a private GitHub repository.

Security caveats demand strict attention:

Inspect `CLAUDE.md` carefully before pushing to private GitHub repositories. Developers frequently leave internal staging IPs, test credentials, or private API path prefixes inside rule files. Private repositories do not constitute an internal network security perimeter; uploading credentials invites serious security incidents.

Additionally, MemDock operates under a source-available license; free for personal development, but commercial repackaging or redistribution remains prohibited.

## The Real Workflow and Daily Routine

My current setup looks like this: Claude Code handles test runs and single-file modifications, Codex reviews diffs, while rules sit in the review queue for weekly clearance. Friday afternoon pushes updates to a private repository, pulled onto my home MacBook seamlessly.

Far from perfect. Switching machines still requires manually re-entering tokens; keychain integration was promised by the author, but has not shipped yet.

How do you currently manage memory across these coding tools? If you have simpler workflows, drop a comment below.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/memdock-claude-code-codex-desktop-workbench-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

