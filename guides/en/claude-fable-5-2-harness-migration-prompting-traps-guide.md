---
title: "Before Moving Old Prompts to Claude Fable 5.2: 400 Prefix Checks and Behavioral Shifts"
description: "Migration notes from Anthropic's official Prompting Claude Fable 5.2 docs: account-wide thinking-block prefix checks throwing 400 errors, effort overrides vs prompt-level suppression, accumulated memory rules causing approval loops, and reasoning_extraction refusals."
pubDate: 2026-11-12
author: "小可"
category: "Architecture & Migration"
tags: ["Claude Fable 5.2", "Claude Mythos 5.2", "Prompt Engineering", "Agent Harness", "Anthropic"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/claude-fable-5-2-harness-migration-prompting-traps-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


Swap an old `Claude Fable 5.1` or `Opus 5.5` system prompt straight onto `Claude Fable 5.2`, and two things are likely to break first.

Resuming a stored conversation throws a `400` error from the API. Or your coding agent stalls mid-run, asking for approval just to delete a temp file.

Anthropic just posted `prompting-claude-fable-5-2` to its docs. Reading through it, the biggest shift in this generation is how several quiet harness assumptions changed underneath existing workflows. Everything below comes strictly from that official guide; I have not run full production long-session regressions here, so no fabricated token bills or latency curves appear below — no way to measure them honestly.

## Editing a System Prompt Throws 400: Account-Wide Prefix Checks

On earlier upgrades, engineers routinely tweaked two lines in `system` and resumed stored conversations from a database.

That habit hits a wall on `Claude Fable 5.2`. The API enforces the thinking-block `prefix check` for every account. Change `system`, `tools`, or any earlier message while passing back saved thinking blocks, and the Messages API rejects the request with a `400` error.

Prompt edits belong in new conversations or in the newest user turn. Need to adjust instructions mid-thread?

Anthropic provides a beta escape hatch: add the `thinking-binding-controls-2026-08-01` header and set `thinking.block_binding.prefix_mismatch_behavior` to `"drop_block"`. There is an immediate cost. In Anthropic's testing, dropping earlier thinking on every turn makes the model re-deliberate and pushes output tokens higher. Outside explicit compaction or context editing, keeping message history append-only stays the cheapest path.

## "Don't Overthink" Prompts Stop Working at Medium and Above

Many `Fable 5.1` prompts carry a line like `"answer directly without deliberating"` to cut latency on simple turns.

On `Fable 5.2`, that line fails. The docs state plainly that in Anthropic's own tests, at `medium` effort and above, the model still thinks before almost every reply — even a simple greeting. Adding that prompt instruction does not stop it, and `Fable 5.2` defaults to `high` effort.

Lowering `effort` directly is the only reliable lever. The guide notes that `Fable 5.2` at `low` effort scores close to `Fable 5.1` at higher effort on coding evals, though benchmark names and sample sizes are omitted there — I cannot reproduce them from that line alone, so treat it as a directional signal rather than an SLA. Switching top-level `effort` mid-thread busts the prompt cache; to keep the cache warm while stepping effort up or down per turn, use the `mid-conversation-output-config-2026-07-01` beta header for per-message effort overrides.

Watch for early stopping at `low` and `medium` effort too. On ordinary timeouts or empty tool results, the model may list remaining work and end the turn unless you add explicit persistence instructions to the system prompt. Who wants an unattended run stopping halfway?

## Accumulated Memory Rules Turn Into Approval Loops

This generation also follows rules far more literally.

Teams running `Fable 5.1` or `Opus 5.5` often piled defensive notes into Memory and rule files: `"never retry"`, `"never delete files"`, `"stop and review at the first error"`. Older models flexed around them; `Fable 5.2` treats any instruction containing `"never"` as an absolute boundary.

Across multi-agent runs, stale cautions accumulate and trigger constant approval prompts for routine work.

Adding a vague "be bolder" line at the end will not fix it. Audit and prune old memory files: delete stale `"never retry"` notes, and rewrite `"never delete files"` as `"ask for approval before deleting files"` so routine reads and edits stay autonomous.

Subagent spawning behaves similarly. At `xhigh` and `max` effort, `Fable 5.2` spawns subagents to review its own code — something `Fable 5.1` did not do on the same tasks. Every spawned subagent bills against its configured model. The guide cites an Anthropic `high`-effort comparison where spelling out subagent cost and delegation criteria in the tool description roughly halved subagent launches; task distributions and sample counts are not listed, so treat that as a trend until your own pipeline tests it.

Progress updates carry a token tax as well. Adding the doc's opening-line, mid-turn-update, and closing-recap template raised output tokens by roughly `20%` on coding tasks — worth it when a human is watching, pure overhead for headless batch jobs.

## New `reasoning_extraction` Refusals: Stop Asking for Reasoning in Prose

Old skills or tool descriptions sometimes tell the model to write out its step-by-step reasoning inside the visible reply.

On `Claude Fable 5.2`, that instruction can trigger `stop_reason: "refusal"`.

Refusals now surface `stop_details.category` across six values: `cyber`, `bio`, `frontier_llm`, `reasoning_extraction`, `general_harms`, and `null`. Asking the model to reproduce internal reasoning in response prose trips `reasoning_extraction`. Remove that prompt rule and set `display: "summarized"` to inspect thinking blocks instead.

For legitimate security or life-sciences workflows blocked by `cyber` or `bio`, Anthropic routes verified organizations to `Claude Mythos 5.2` — matching `Fable 5.2` capabilities behind the Cyber and Life Sciences verification programs.

Before flipping production traffic, run through four checks:

1. Benchmark `medium`, `high`, and `xhigh` effort in fresh sessions instead of copying `Fable 5.1` settings.
2. Audit existing system prompts, skills, and memory files to strip `"answer directly without deliberating"`, stale `"never"` bans, and prose reasoning demands.
3. Verify long-session resume paths keep `system`, `tools`, and message prefixes strictly append-only so `prefix check` does not throw `400`.
4. Dry-run your longest agent pipeline to spot early stops, excessive subagents, or approval loops — without clearing those old landmines first, would you really cut 100% of traffic over blind?

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/claude-fable-5-2-harness-migration-prompting-traps-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

