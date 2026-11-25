---
title: "Small Local Models Crash in Agent Coding: A Hands-on Guide to Harny Loop Guard and Observation Masking"
description: An architectural breakdown of Harny, an open-source VS Code extension designed to tame local models. Testing Loop Guard infinite retry circuit breakers, Analyze mode dry-run diffs, and observation masking.
pubDate: 2026-11-25T14:05:00.000Z
author: Chenxian
category: Model Evaluation & Practical Tests
tags:
  - Harny
  - AI Coding Agent
  - VS Code
  - Ollama
  - Local LLM
  - Engineering Lessons
heroImage: /images/blog/free-ai-hero-card.webp
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/harny-ai-coding-agent-vs-code-loop-guard-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


Let us address the pain point first. Running agentic coding on small local models fails eight times out of ten.

Many developers run Qwen2.5-Coder-7B Q4_K_M or Llama-3.1-8B-Instruct on Mac or 16GB VRAM machines via Ollama or llama.cpp. Single-turn code completion or chat works decently. The moment you give it tool access inside VS Code to modify files autonomously, chaos quickly follows.

Two typical failure modes occur:

First, infinite loops. Small models spin in place on identical errors, calling the same tool repeatedly until context memory collapses.

Second, destructive file overwrites. A minor comprehension flaw causes the model to replace an entire source file with a broken skeleton, leaving you with no easy recovery.

The open-source extension Harny tackles these issues directly. Located on GitHub at spltek/harny under the MIT license with current tested version v0.3.2, it lives natively in the VS Code and VSCodium sidebar. It avoids exaggerated claims about model intelligence, adding two mechanical safeguards: Loop Guard and Analyze dry-run diff mode.

Can local models handle daily refactoring? Can these defensive safeguards reliably hold the line?

---

## The Structural Weaknesses of Local Models in Agentic Tasks

Frontier cloud models with hundreds of billions of parameters possess strong self-correction and planning. When a tool fails, they shift strategy within two turns.

What happens with lightweight 7B to 14B models?

Small models fail in agent workflows for three compounding reasons:

The mildest issue is attention drift. As turns accumulate, command outputs and file contents fill the context. After a few thousand tokens, the model forgets initial instructions. A 32K context window on a model card is a benchmark claim, not a production guarantee.

A more damaging issue is repeating identical mistakes. When a tool call fails, small models struggle with backtracking. They invoke the exact same command with identical parameters, outputting the exact same explanation. Greedy sampling traps the model inside a local minimum.

The most severe issue is uninspected file overwrites. Most integrated agents write directly to the workspace. When small models hallucinate midway, they overwrite working code, causing immediate workspace damage.

Harny adopts a pragmatic stance: if the model lacks general intelligence, the external runtime harness must enforce deterministic guardrails.

---

## Breakdown One: Breaking Deadlocks with Loop Guard

Across agent evaluations, repetitive loops consume tokens and waste developer time.

Harny integrates a real-time Loop Guard inside its execution runtime. Rather than hoping the model realizes its mistake, the harness tracks a sliding operation fingerprint table.

```text
Model emits tool call
       │
       ▼
┌────────────────────────┐
│ Compute Trace Hash     │  Tool name + argument hash + output similarity
└──────────┬─────────────┘
       │
       ▼
       / Matches Loop Pattern? \
     <                          > ── No ──▶ Allow normal execution
       \          Yes           /
       │
       ▼
┌────────────────────────┐
│ Loop Guard Triggered   │
└──────────┬─────────────┘
       │  Escalating intervention
       ├─▶ Level 1: Inject system corrective prompt warning of repetition
       ├─▶ Level 2: Intercept tool call and roll back execution stack
       └─▶ Level 3: Terminate session, returning control to terminal
```

In testing, I assigned local Ollama running Qwen2.5-Coder-7B to fix a missing TypeScript type error. The terminal emitted this real error:

```text
src/index.ts:14:7 - error TS2322: Type 'string' is not assignable to type 'number'.
npm run build exited with code 2
```

The model made an incorrect adjustment. The second run failed identically. Without reconsidering, it initiated the exact same command a third time.

Normally, the developer watches the terminal cycle fruitlessly. Just before the third invocation began, Harny displayed an interception notice:

```text
[Loop Guard] Detected repeated tool execution: npm run build
Action blocked. Injecting steering prompt to switch investigation approach.
```

The guard seized tool execution directly, injecting a corrective prompt that forced the model to read tsconfig.json and inspect source declarations instead.

This single intervention severed the infinite retry loop, preserving remaining context space.

---

## Breakdown Two: Analyze Mode Dry-Run Isolation

Code adjustments risk unintended destruction.

Harny splits interaction into two operational modes: **Analyze Mode** and **Build Mode**.

Sessions default to Analyze Mode. In this mode, any tool call attempting to modify, write, or delete files runs in dry-run simulation.

What does this mean in practice?

A physical wall separates reasoning from disk writes. Under Analyze Mode, the model believes it modified code while files on disk remain untouched. Could a stumbling model delete your files? Not a single character changes.

Harny generates a standard Unified Diff in the sidebar. You inspect changes line by line like a pre-commit code review, verifying modified variables and functions.

Once you confirm diff logic is sound, manually toggle to Build Mode. Only then do disk modifications execute.

Retaining disk write authorization in human hands provides genuine peace of mind.

---

## Breakdown Three: Observation Masking and Context Scratchpads

Beyond deadlocks, running agents locally strains VRAM and context limits. Running an 8K context on an Apple Silicon M2 Pro 16GB machine quickly leads to VRAM pressure once massive logs accumulate.

Observation Masking is an established technique. Anthropic research on context engineering and JetBrains studies on agent context tuning demonstrate that condensing stale tool outputs into brief summaries while preserving only the latest responses maintains agent performance across extended tasks.

Harny implements this logic directly:

First, proactive masking of stale observations. Across multiple turns, only the most recent command outputs remain intact. Older verbose logs in conversation history become concise status summaries, retaining factual outcomes while pruning redundant text.

Second, persistent structured scratchpads. The model maintains an isolated scratchpad throughout extended tasks, tracking modified files and current verification stages. The previous twenty turns of raw discussion get discarded entirely.

Testing on an 8K context quantized model showed this mechanism supported ten consecutive turns of multi-file debugging without losing earlier instructions.

---

## Practical Setup: Connecting Ollama to Harny

Setting up a safe local Harny environment takes a few minutes:

### 1. Ready the Local Model Endpoint

Verify that your local Ollama or llama.cpp endpoint is running. For Ollama, the default listener runs on port 11434:

```bash
ollama run qwen2.5-coder:7b
```

### 2. Configure Harny Extension

Search and install Harny from the VS Code Marketplace, then open sidebar settings:

- **API Protocol**: Select OpenAI Compatible
- **Base URL**: `http://localhost:11434/v1`
- **API Key**: Enter any placeholder string, such as `ollama`
- **Model Name**: `qwen2.5-coder:7b`

### 3. Recommended Debugging Workflow

When debugging with local models, follow this discipline:

Start exclusively in Analyze Mode. State your objective, letting the model inspect project files and propose a Unified Diff without disk access. Can you skip this preview step? Never.

Scan diff boundaries line by line. Verify unrelated files remain untouched and ensure existing comments survive.

Switch to Build Mode to apply changes. Once confirmed, toggle to Build Mode for one-click application, then run tests locally to close the loop.

---

## Conclusion

With frontier cloud models dominating, why bother running local agents?

Developers choose local execution for three clear reasons: keeping code private, working without internet access, and avoiding recurring monthly API bills. None of these sound flashy, but in practice, none are easy to give up. That is why local coding agents remain relevant.

Harny provides a practical takeaway: while model weights transition toward general capability, disciplined runtime harnesses and defensive architectures effectively bridge the gap left by smaller parameter scales.

If you have spare VRAM on your desk, why not give it a contained sandbox to work in?

Install the brakes, enforce the boundaries, and local models can perform reliable work within safe limits.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/harny-ai-coding-agent-vs-code-loop-guard-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

