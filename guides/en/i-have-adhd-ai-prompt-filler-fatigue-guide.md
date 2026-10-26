---
title: "Stop AI Throat-Clearing & Cut Token Waste by 50%! The Complete Guide to ayghri/i-have-adhd: Transform Claude Code, Cursor, and ChatGPT into Action-First Machines"
description: "Tired of AI burying the one-line fix beneath paragraphs of pleasantries and preambles? Deep dive into the viral open-source framework ayghri/i-have-adhd. Enforce 10 strict cognitive guardrails and pre-send self-filtering to eradicate filler fatigue, surface executable commands on line one, and slash redundant token costs by 40% to 60%. Includes setup guides for Claude Code, Cursor, and ready-to-use prompt templates."
pubDate: 2026-10-26
author: "Xiao Ke (FreeAI Dog Core Team)"
category: "Hands-on Practice"
tags: ["PromptEngineering", "ClaudeCode", "Cursor", "i-have-adhd", "TokenOptimization", "AIAssistant", "ProductivityHacks", "SystemPrompts"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/i-have-adhd-ai-prompt-filler-fatigue-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Stop AI Throat-Clearing & Cut Token Waste by 50%! The Complete Guide to ayghri/i-have-adhd: Transform Claude Code, Cursor, and ChatGPT into Action-First Machines

Have you ever found yourself in this infuriating situation while coding with AI?

You encounter a mundane Docker port binding error. All you need is the single terminal command to kill the zombie process. But instead, the AI responds with this familiar essay:

> *"Great question! I would be more than happy to help you diagnose this network binding issue. In modern containerized and microservice workflows, port allocation is a critical component. Typically, a `bind: address already in use` error can be attributed to three primary scenarios. Let's break down each one in detail... [three screens of generic conceptual exposition omitted]... I hope this explanation proves insightful! If you run into any further hurdles, please feel free to ask!"*

By the time you scroll past the pleasantries, the emotional cushioning, and the unsolicited recap, you have wasted precious mental bandwidth just to locate the one-line command (`lsof -ti :8080 | xargs kill -9`) buried at the very bottom.

This widespread developer dilemma is known as **"Filler Fatigue"** or **"Throat-Clearing"**. It shatters your cognitive flow state and acts as a silent **Token Assassin**—squandering hundreds of output tokens per turn, inflating your API billing, and bloating the context window until memory drift (Context Rot) sets in.

Today, we dive into the viral open-source project making waves across GitHub, Reddit, and X: [`ayghri/i-have-adhd`](https://github.com/ayghri/i-have-adhd). Without altering model weights, this lightweight instruction ruleset instantly cures AI verbosity, enforcing direct, command-level outputs right on Line 1.

---

## 1. The Origins: What is `i-have-adhd`?

Created by open-source developer `ayghri`, the skill draws inspiration directly from clinical psychology—specifically Ramsay & Rostain’s acclaimed handbook, *The Adult ADHD Tool Kit*.

Let's clear up a major misconception immediately: **You do not need an ADHD diagnosis to benefit from this tool. It is an indispensable gift for any engineer who values focus, speed, and zero sunk cost!**

* **The Psychology Analogy**: The clinical toolkit aims to minimize "activation energy" by eliminating extraneous distractions and breaking overwhelming objectives into atomic, sequential tasks;
* **Re-architecting LLM Behavior**: Default RLHF alignment programs modern models to be overly polite, apologetic, and defensive. By transposing cognitive accessibility guardrails into **rigid prompt constraints**, the AI is transformed from a rambling tutor into a concise field engineer.

Empirical community benchmarks reveal that applying this framework increases direct actionability by **10x** while slashing token consumption by **40% to 60%** across standard programming sessions!

---

## 2. The 10 Golden Cognitive Guardrails

The reason `i-have-adhd` is so remarkably effective is that it replaces soft suggestions with 10 uncompromising, hard-coded rules:

```
┌─────────────────────────────────────────────────────────────────┐
│               i-have-adhd Three Strategic Frontlines            │
├─────────────────────────────────────────────────────────────────┤
│  ⚡ Action-First Protocol       👉 Line 1 MUST be executable action/cmd│
│  🚫 Zero Filler Discipline      👉 Ban preambles, recaps, and closers  │
│  🔢 Cognitive Granularity Caps  👉 Number steps, list ≤ 5, one next step│
└─────────────────────────────────────────────────────────────────┘
```

### 1. Lead with the Next Action (Action-First)
Line 1 of the response **must be an actionable command, a file path, or the concrete code diff**. No preamble, no greetings, no introductory fluff.

### 2. Number Multi-step Tasks
Never output sprawling paragraphs with prose transitions (*"First... next... after that... finally..."*). Multi-step sequences must strictly use `1. 2. 3.` formatting.

### 3. End with Exactly One Concrete Next Step
Every response must terminate with **one single, unambiguous next action**. Never ask open-ended questions like *"Would you like me to explain anything else?"*.

### 4. Suppress Tangents
Ruthlessly eliminate unprompted background trivia. Answer exactly what was asked—do not volunteer unsolicited knowledge.

### 5. Restate State Briefly Every Turn
Anchor progress with half a sentence (e.g., *"Auth middleware patched; remaining integration tests unexecuted"*). This prevents context drift during long sessions.

### 6. Specific Time Estimates
Never use vague qualifiers like *"this shouldn't take long"* or *"soon"*. Always provide exact numbers (e.g., *"compilation takes ~2 min"*).

### 7. Make Wins Visible
Explicitly acknowledge passing tests or finished tasks (e.g., `[PASS]`). Visual checkpoints reinforce certainty.

### 8. Matter-of-Fact Tone for Errors
Report bugs, failures, or failed assertions objectively without emotional cushioning or apologies (strictly ban *"I apologize for the confusion..."*).

### 9. Cap Lists at 5 Items
Working memory is fragile. Group or trim all enumerations to a maximum of 5 items per message.

### 10. Strict Pre-Send Self-Filter
The model must mentally scrub its output before transmission:
* ❌ Strip greetings (*"Great question!"*, *"Sure thing!"*);
* ❌ Strip cognitive narration (*"Let me analyze this problem..."*);
* ❌ Strip conversational closers (*"Hope this helps!"*, *"Let me know if you need anything!"*);
* ❌ Strip redundant recapitulations.

---

## 3. Real-World Benchmark: Before vs. After

Here is an authentic before-and-after comparison diagnosing a typical Docker container port collision:

### ❌ Default Model Output (480 Tokens - High Cognitive Friction)
> *"That's a very common Docker deployment issue! When you encounter `bind: address already in use`, it typically indicates that another process on your host is actively listening on port 8080. This frequently happens if you have duplicate services running or an orphaned container from an earlier test. To resolve this gracefully, we can follow a three-pronged approach: first, inspect the offending PID using network utilities; second, issue a termination signal; third, relaunch your container stack. I hope this helps you get back on track! Here are the steps..."* (actual command buried below)

### ✅ With `i-have-adhd` Injected (85 Tokens - Instant Execution)
> Run this command to kill the process occupying port 8080:
> ```bash
> lsof -ti :8080 | xargs kill -9
> ```
> 1. Execute the termination command above (~2 seconds);
> 2. Restart your stack: `docker-compose up -d`;
> 
> Next step: Run `docker ps` to verify container health.

**Result**: Token consumption drops from 480 to 85—**an 82% reduction in waste**. The developer can hit the copy button within 0.1 seconds of output generation.

---

## 4. Step-by-Step Multi-Platform Installation Guide

### 1. Claude Code CLI (Native Marketplace Plugin)
If you use Anthropic’s terminal tool **Claude Code**, install it directly via the official marketplace:

```bash
# Step 1: Add marketplace source
claude plugin marketplace add ayghri/i-have-adhd

# Step 2: Install the plugin
claude plugin install i-have-adhd@i-have-adhd
```

* **On-Demand Activation**: Type `/i-have-adhd` in your active session;
* **Always-On Mode**: Run the following command in terminal to enable it automatically across all future sessions:
  ```bash
  touch ~/.claude/.i-have-adhd-always
  ```
* **Deactivation**: Type `stop adhd mode` anytime to revert to standard conversational tone.

---

### 2. Cursor / Windsurf (Project & Global Configurations)

#### Option A: Per-Project `.cursorrules` (Recommended)
Create a `.cursorrules` file in your repository root and paste the **Core Prompt Template** provided in the Appendix below.

#### Option B: Global System Prompt
In Cursor, navigate to `Settings` -> `General` -> `Rules for AI`, and paste the template. Your `Ctrl+K` inline edits and `Ctrl+L` chat panels will permanently enforce action-first discipline.

---

### 3. ChatGPT / Claude.ai / Gemini Web Apps
1. Open user settings -> Navigate to **Custom Instructions**;
2. Locate the prompt field: **"How would you like the AI to respond?"**;
3. Paste the Appendix template and hit **Save**.

---

### 4. OpenWebUI / Dify / API Gateways (Slashing Team Token Bills)
For engineering teams utilizing [FreeAI Dog API Endpoints](https://freeaidog.com/en/), OpenWebUI, or private LangChain/Dify agents:
* Injecting this prompt into the global system persona slashes streaming latency;
* Eliminating pleasantries saves up to 50% on enterprise token consumption while dramatically expanding effective context limits.

---

## Appendix: Ready-to-Use Prompt Templates

Copy and paste this standardized template directly into your preferred tool:

```markdown
# Role & Communication Guardrails: Action-First System

Follow Ramsay & Rostain's Adult ADHD Toolkit cognitive guardrails strictly:

1. Lead with the next action: Line 1 MUST be an executable action, terminal command, or file path. No preamble.
2. Number multi-step tasks: Always use 1, 2, 3 for multi-step instructions.
3. End with one concrete next step: Never leave open-ended questions. Specify exactly one next action.
4. Suppress tangents: Do not volunteer unasked explanations or related-but-unsolicited background.
5. Restate state briefly every turn: Ground progress in half a sentence to prevent drift.
6. Specific time estimates: Use exact numbers (e.g., "about 3 min") rather than vague qualifiers.
7. Make wins visible: Explicitly acknowledge completed milestones and passing tests.
8. Matter-of-fact errors: State problems plainly without apologies, emotional cushioning, or fluff.
9. Cap lists at 5 items: Never output massive lists. Group or trim to ≤5 items.
10. Pre-send self-filter (STRICT):
    - NO preamble ("Great question!", "Sure, I can help with that", "Let me think...")
    - NO throat-clearing recap before the answer.
    - NO conversational closers ("Hope this helps!", "Let me know if you need anything else!").
```

---

## Conclusion

In the rapid race of generative AI, we often fixate on larger parameter sizes (125B, 405B) and expensive hardware upgrades while ignoring the **sheer communicative inefficiency of the default interaction protocol**.

A rigorously crafted ruleset does more than preserve your focus—it physically halves your token billing. As Unix philosophy famously preaches: **"Do one thing and do it well."**

Upgrade your AI assistants today, stop the throat-clearing, and reclaim your flow state!

For more cutting-edge developer productivity hacks and free API hubs, explore [FreeAI Dog (freeaidog.com)](https://freeaidog.com/en/)!

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/i-have-adhd-ai-prompt-filler-fatigue-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

