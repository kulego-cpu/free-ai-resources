---
title: "Handing Off from Claude Code to Codex: Atuin Threads Sync Architecture and Credential Isolation"
description: "An engineering breakdown of Atuin 18.24 Threads: parsing local agent session files without hooks, cross-machine handoff between Claude Code and Codex, v18.24.6 daemon fd-hang fixes, and the credential isolation debate on Hacker News."
pubDate: 2026-11-12
author: "小可"
category: "Architecture & Security"
tags: ["Atuin Threads", "Claude Code", "Codex", "MCP", "Session Sync"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/atuin-threads-18-24-cross-machine-agent-session-sync-security-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


You start chasing a disk IO bottleneck with `Claude Code` on a remote Linux dev box, close the lid, and open your MacBook wanting `Codex` to pick up from there. Then you hit the same wall: the two session histories live in separate silos.

Paste the stack trace again? Retype thirty minutes of dead ends into a fresh prompt? Twice is enough to get tired of it.

Atuin shipped `Threads` in `18.24` yesterday with a straightforward pitch. Encrypted sync already solved the five-hundred-line `.bash_history` truncation problem. Agent transcripts left inside home directories by `Claude Code`, `Codex`, and `OpenCode` are just append-only logs too, so they can ride the same sync engine.

Three sources ground everything below: `blog.atuin.sh/threads-agent-sessions/`, the `v18.24.6` release notes on `github.com/atuinsh/atuin`, and Hacker News thread `50453637`. I do not have a multi-node private sync server wired up for load testing here, so you will find zero latency or throughput numbers below — no way to measure them honestly.

## No Hooks: Parsing Local Session Files Directly

At first glance, cross-harness handoff sounded like Atuin had injected plugins into every agent CLI.

Reading maintainer `ellieh` in the comments cleared that up. Right now `Threads` uses zero hooks. Instead, it parses the session files each coding tool already writes locally, splitting turns, shell commands, and error output into a searchable index.

Five harnesses are supported today: `Claude Code`, `Codex`, `Pi`, `Omp`, and `OpenCode`.

Setup takes three commands:

```bash
curl --proto '=https' --tlsv1.2 -LsSf https://setup.atuin.sh | sh
atuin resume
atuin mcp install claude
```

Running `atuin resume` opens an interactive TUI. You can search older sessions by keyword or fork a thread started on a remote dev server onto your laptop under a different harness. With `atuin mcp` enabled, a fresh agent can query past debugging trails on its own.

That convenience comes with an obvious catch. Session transcripts are not versioned database schemas. Each harness shifts its layout whenever it feels like it, and even `Claude Code` shell-edit diffs already needed dedicated patches. If an upstream CLI renames one JSON key tomorrow, will the parser silently drop half a session?

## Reading v18.24.6 Release Notes: Four Concrete Rough Edges

Days after launch, `atuinsh/atuin` is already at `18.24.6`. The changelog says more about the real state of the code than any launch post:

- **Indefinite shell-exit hang**: commit `4668` fixes a daemon bug where inherited file descriptors stayed open before autostart, causing `pty-proxy` to hang forever on shell exit.
- **macOS socket handoff clash**: commit `4669` stops managing the socket via `launchd` because daemon startup clashed with Apple service supervision.
- **Claude Code diffs and key writes**: `threads` needed two targeted patches — capturing diffs from `Claude Code` shell edits and preserving write patches that reference the `atuin` key name.
- **Lagged MCP dependency fix**: commit `4690` bumps `rmcp` for three security advisories, while a SQLite advisory stays pinned until `sqlx` allows the upgrade.

Seeing `4668` made me double-check my own terminal sessions. Anyone keeping long-lived background daemons around hates zombie shells caused by leaked file descriptors. While the daemon is still catching rapid patches like this, trusting it on a primary production workstation without a rollback plan feels premature.

On top of stability, `nzoschke` — who builds Superlogical Rex — raised a blunt signal-to-noise question: a huge share of coding sessions are throwaway experiments, so dumping all of them into one global history risks burying real answers under noise. Another engineer, `justCallMeOmid`, went the opposite route: scoping logs per work item, deleting raw transcripts once done, and storing only short decision summaries in a vector store.

## Where Security Purists Draw the Line

Whenever `Threads` security comes up, most people zoom straight in on end-to-end encryption.

Cryptographically, the design gives you options. Let me lay out the three deployment modes as I see them:

**Local-only** — data stays inside `~/.local/share/atuin` with no server involved. Whatever shares the same host without sandboxing, including third-party scripts you run casually, can still reach it. Good enough if you only want local search and backup.

**Self-hosted server** — an internal VM or a home NAS. Key distribution is on you, and once an agent reads transcripts through `atuin mcp`, it sees plaintext. Encryption protects bytes in flight, not inside an authorized terminal session.

**Hosted E2EE** — encrypted client-side before upload. Atuin cannot read your logs. Still, losing keys on one enrolled machine exposes the entire synced pool. That is not a bug; it is the structural tradeoff of shared history.

Hacker News commenter `petesergeant` drew a hard line here. After spending months isolating agents via separate sandboxes, per-agent encrypted credential stores, and regular transcript scanning, pooling every machine into one cross-searchable store collapses the exact boundary he built.

Tool permissions make that leak path worse.

As `ahmedketata` pointed out in the thread, clicking permanent allow once inside `Claude Code` can persist sensitive commands straight into `.claude/settings.local.json`. Plaintext. Those fragments flow into the session log and join the sync queue. Transport encryption protects the wire, not the endpoint, so a lower-trust agent querying `atuin mcp` can pull that leaked token right out. I stared at that chain for a while and still do not see a clean fix — nobody can audit every local settings file before an agent logs it.

A middle ground does exist. Commenter `dietr1ch` skips centralized sync servers altogether, syncing scrubbed backups over `Syncthing` and searching locally with `grep` and `skim`. You lose instant cross-harness forking, yet every machine keeps a clear boundary.

## When to Adopt It and When to Stay Local

Credit where it is due: `nklmilojevic` shared in the same thread that `Atuin Threads` recovered uncommitted work after accidentally deleting a `git worktree`. Plain `.bash_history` lets you `grep` a single host, nothing more; it cannot reconstruct cross-machine edits after a wiped worktree.

My own rollout stays split down the middle:

- Personal lab boxes and non-sensitive repos: run `Atuin 18.24.6+` with `atuin resume` to bridge `Claude Code` and `Codex` when moving from a Linux box back to a MacBook.
- Machines holding production keys, internal tokens, or client data: keep cross-machine sync off, avoid mounting `atuin mcp` globally for autonomous agents, and scrub logs locally if you archive them at all.

Before turning any of this on, run through three quick checks:

1. Run `atuin --version` and verify `18.24.6` or newer so you avoid the `pty-proxy` fd-leak hang.
2. Inspect `.claude/settings.local.json` and `.env` history before running `atuin mcp install` to make sure live tokens are not sitting in transcripts.
3. If your personal laptop and a work bastion share one sync account, are you actually comfortable pulling bastion session logs back onto your home machine?

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/atuin-threads-18-24-cross-machine-agent-session-sync-security-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

