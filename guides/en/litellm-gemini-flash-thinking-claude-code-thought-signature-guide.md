---
title: "LiteLLM Solves the Gemini Thought Signature Black Hole: Mounting 1M Context Thinking Streams to Claude Code and Cursor for 0 Dollars"
description: "Dissecting LiteLLM commit da296ee fixing Google Gemini thought signature pass-through: resolving the 400 thought_signature drops inside Claude Code multi-turn tool calling, with full local proxy SOP and cost accounting."
pubDate: 2026-11-17
author: "Xiao Ke"
category: "Architecture"
tags: ["LiteLLM", "Gemini", "ClaudeCode", "Cursor", "ReasoningModels", "Troubleshooting"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/litellm-gemini-flash-thinking-claude-code-thought-signature-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# LiteLLM Solves the Gemini Thought Signature Black Hole: Mounting 1M Context Thinking Streams to Claude Code and Cursor for 0 Dollars

Two nights ago, I was refactoring an older project spanning hundreds of source files.

Claude Code was running inside my terminal.

Official Claude 3.7 Sonnet with extended thinking is undeniably capable. Its reasoning chains are sharp. However, burning through tokens feels like feeding bills into a shredder.

Output pricing sits high at $15 per million tokens.

A few hours of heavy refactoring easily burns dozens of dollars. Who wants to sustain that indefinitely?

I decided to pivot my approach. Google AI Studio provides Gemini 2.5 Flash with generous free allowances. It offers a 1M token context window. Streaming thinking chains are included at zero cost.

Mounting 1M context at zero cost to power Claude Code seemed like an obvious match.

I spun up a local LiteLLM proxy and redirected the Anthropic Messages endpoint toward Gemini.

The initial handshake ran cleanly.

The model streamed back thousands of tokens of detailed reasoning.

Yet the moment multi-turn interaction began, things fell apart. Claude Code triggered Bash commands to compile code or called FileEdit to modify lines. The terminal promptly spat back a fatal error.

The entire session crashed.

A Google API error blocked the terminal screen:

```text
400 Invalid argument: Function call is missing a thought_signature
```

At other times, it skipped the 400 error entirely, degenerating into infinite single-character repetition loops and freezing in place.

The free compute calculation had barely begun before getting knocked back by a 400 error.

What exactly is this thought_signature mechanism?

Why did single-turn text prompts work while tool execution crashed immediately?

---

## 1. Protocol Mismatch: Google Cryptographic Signatures vs Anthropic Payload Schemas

Diagnosing this required capturing and slicing raw HTTP payloads on both sides.

The root cause stems from internal validation rules inside Google Gemini reasoning models.

Within Google reasoning architecture, models generating Function Call parameters bind a cryptographic token named thought_signature.

This signature records which chain-of-thought derivation path produced the specific tool invocation.

Under Google API conventions, when clients return tool execution outputs on subsequent turns, they must echo back the previous thought_signature untouched.

What happens if the signature goes missing? Google backend endpoints reject the request immediately. At best, you receive an immediate 400 error. At worst, model context corrupts, causing it to loop single characters indefinitely.

How did this signature disappear inside the LiteLLM proxy layer?

The vulnerability lies within the Anthropic Messages API specification.

Claude Code was built natively against Anthropic standards, communicating over the /v1/messages endpoint.

Inside the Anthropic schema, tool interaction is restricted to tool_use and tool_result blocks.

The JSON schema provides no recognized field for third-party proprietary signature tokens.

LiteLLM initially attempted two workarounds: stuffing the signature into custom extension fields, or appending it to the tool_use ID string. Neither worked.

The Claude Code client strictly validates incoming structures against official Anthropic schemas before forwarding, discarding unrecognized non-standard fields.

When the client finishes executing local commands and bundles tool_result payloads back, those proprietary signature fields are long gone.

LiteLLM receives no signature and forwards the raw payload to Google.

Google notices the missing signature and halts execution immediately.

The multi-turn proxy session is deadlocked in this protocol gap.

---

## 2. Patch Anatomy: Session-Level In-Memory Caching and Egress Injection

Relying on the client is futile. We cannot expect Claude Code to preserve vendor-specific fields—it only retains official tool_use and tool_result blocks, stripping everything else. The LiteLLM proxy layer must retain the state itself.

In commit da296ee4bfa706e2924ef4294d5b3d9802a2aa38, LiteLLM overhauled signature handling inside the Anthropic adapter.

The fix operates cleanly without expecting client-side round-tripping.

Change 1: Server-side tool call caching.

When Gemini emits a tool call block containing a thought_signature, LiteLLM intercepts it; the proxy allocates an in-memory session cache indexed by the tool_call_id and session context hash.

The signature token remains preserved inside the proxy memory space.

Change 2: Egress interception and signature re-injection.

When Claude Code posts tool execution results back to /v1/messages, the adapter matches the tool_call_id against its internal table.

It extracts the retained thought_signature from memory and injects it into the outgoing payload sent to Google.

The client never needs to know the signature exists.

The data flow topology operates as follows:

```text
Claude Code Terminal Client [Strict Anthropic Protocol]
      │
      ▼  POST /v1/messages [Emits tool_result without signature]
LiteLLM Dynamic Proxy [Running commit da296ee patch]
├─ Matches tool_call_id in memory cache
├─ Retrieves and injects thought_signature token
└─ Translates into Google AI Studio native REST payload
      │
      ▼
Google Gemini 2.5 Flash Backend [Signature verified]
├─ Reasoning stream advances [1M context fully preserved]
└─ Returns subsequent code generation and tool call responses
```

Change 3: Fallback safety hatch.

What happens if an in-memory entry is lost during extreme asynchronous churn? LiteLLM provides a skip_thought_signature_validator fallback. It generates a placeholder signature to prevent hard crashes. The tradeoff is that Google backend flags the turn as lacking valid derivation context, meaning thinking coherence may degrade. Do not rely on this fallback; preserving the cache remains essential.

This combination fully unlocks multi-turn tool calling without deadlocks.

---

## 3. Local Setup: Connecting Claude Code and Cursor in Five Minutes

With the mechanics understood, local setup requires straightforward configuration.

Ensure LiteLLM includes the latest patch updates:

```bash
pip install -U "litellm[proxy]>=1.63.1"
```

If you prefer building from source:

```bash
git clone https://github.com/BerriAI/litellm.git
cd litellm
pip install -e .
```

Create a config.yaml service configuration file:

```yaml
model_list:
  - model_name: claude-3-7-sonnet-20250219
    litellm_params:
      model: gemini/gemini-2.5-flash
      api_key: os.environ/GEMINI_API_KEY
      thinking:
        type: enabled
        budget_tokens: 4096

general_settings:
  master_key: sk-litellm-local-dev-key
```

Notice the alias mapping convention: the model name is aliased to claude-3-7-sonnet-20250219.

This prevents Claude Code startup checks from failing, forwarding requests smoothly to Gemini.

Export your Google AI Studio API key and launch the proxy daemon:

```bash
export GEMINI_API_KEY="AIzaSyYourRealKeyHere"
litellm --config config.yaml --port 4000
```

Open a separate terminal shell and export the redirect variables for Claude Code:

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:4000"
export ANTHROPIC_API_KEY="sk-litellm-local-dev-key"
claude
```

Send a refactoring command instructing Claude Code to inspect the repository and run unit tests.

The live terminal output shows smooth streaming reasoning:

```text
╭── Claude Code ───────────────────────────────────────────────╮
│ Model: claude-3-7-sonnet-20250219 [via LiteLLM proxy :4000]   │
╰──────────────────────────────────────────────────────────────╯
> Check unit tests and fix tokenizer AST definitions
Thinking:
  Need to inspect failing cases. Running cargo test first.
$ cargo test --test parser_test
  Compiling parser v0.1.0 [/Users/cool/code/demo]
  Running unittests src/lib.rs
test parser_test ... ok
Thinking:
  Test passed. Now editing src/parser/ast.rs to update Node enum.
Tool call: Edit(file: "src/parser/ast.rs", ...)
  Updated 1 line in src/parser/ast.rs
Session continued smoothly across 14 turns without thought_signature errors.
```

Over fourteen consecutive rounds of file inspections, test executions, and edits completed without a single 400 signature error.

Configuring Cursor is equally simple: under Settings > Models > OpenAI API, enter Base URL as http://127.0.0.1:4000/v1 and API Key as sk-litellm-local-dev-key.

---

## 4. Cost Accounting: Zero Dollars vs Commercial Token Bills

Technical architectures must prove viable on real balance sheets.

Comparing both operational models:

Official Anthropic Claude 3.7 Sonnet:
Input pricing is $3 per million tokens; output pricing is $15 per million tokens.
A full-stack engineer actively refactoring projects easily consumes 500K input tokens and 100K reasoning output tokens daily.
Daily spend fluctuates between $3 and $8. Over a month, API expenses surpass hundreds of dollars. Can individual budgets absorb this? Rarely without hesitation.

Google AI Studio routed through LiteLLM:
Gemini 2.5 Flash free tier provides generous RPM limits and millions of daily tokens.
Personal project refactoring fits within free usage allowances. Direct invocation cost drops to zero.
Crucially, Gemini 2.5 Flash natively features a 1M token context window, far exceeding Sonnet 200K ceiling.
Tens of thousands of lines can be ingested at once without mid-session truncation.

I checked system utilization on an M3 Mac via Activity Monitor:
The LiteLLM Python proxy occupies roughly 320MB of resident RAM with CPU idling near zero, occasionally peaking at 5% during JSON serialization.
It imposes minimal overhead on development machines, though remember it remains a resident Python process—avoid launching multiple instances when memory is constrained.

---

## 5. Operational Boundaries, Rate Limits, and Self-Check Checklist

Despite smooth ergonomics, practical deployments face technical constraints.

First, Google free tier rate limits.

Google AI Studio enforces limits of 15 requests per minute and 1500 calls per day on free tiers.

Launching uncontrolled autonomous subagents with dozens of parallel tool calls risks hitting 429 Resource Exhausted errors.

Use this setup for focused single-agent tasks, reserving paid quotas for high-concurrency automated teams.

Second, proxy restarts causing orphaned sessions.

Because commit da296ee caches thought_signature in memory, restarting LiteLLM mid-session clears the lookup table.

Subsequent tool returns will throw 400 errors due to missing signatures.

Running LiteLLM under process supervisors like pm2 or nohup avoids accidental mid-session reloads.

In your daily coding workflow, which matters more to you?

Is it the nuanced reasoning precision of native Claude models?

Or is it the zero-cost exploration enabled by a 1M context window?

If Google tightens free tier reasoning quotas, could multi-key rotation pools provide sustainable headroom?

We recommend testing this three-step setup inside an experimental repository to confirm multi-turn stability before everyday adoption.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/litellm-gemini-flash-thinking-claude-code-thought-signature-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

