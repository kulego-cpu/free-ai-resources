---
title: "Packing 50 Tiptap Extensions into Function Calling Yielded Only 48% Pass Rate: QuireKit List, Explain, and Call Postmortem"
description: "We benchmarked 100 real document editing tasks: exposing 50+ tool schemas passed only 48 tasks with 4.2 roundtrips and 6.8s P95 latency; migrating to QuireKit self-describing list, explain, and call protocol passed 98 tasks with 1.1 roundtrips and 1.4s P95 latency. Engineering notes on test suite design, failure classification, silent dropouts, and real deployment costs."
pubDate: 2026-11-03
author: "Xiao Ke · FreeAI Dog Core Dev Team"
category: "Architecture Deep Dive"
tags: ["QuireKit", "AINative", "RichTextEditor", "ProseMirror", "Tiptap", "AgentArchitecture", "FunctionCalling"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/quirekit-self-describing-ai-native-editor-agent-architecture/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Packing 50 Tiptap Extensions into Function Calling Yielded Only 48% Pass Rate: QuireKit List, Explain, and Call Postmortem

Frontend teams building AI powered document editors almost universally stumble into the exact same pitfall.

When product managers ask for an in-editor AI copilot, the instinct is straightforward: take every single editor capability and turn it into an individual Function Calling endpoint.

Bold formatting gets an endpoint.

Table creation gets an endpoint.

Level 3 heading formatting gets another endpoint.

If the editor ships with fifty extensions, developers faithfully declare fifty separate tool schemas inside the system prompt.

We took this path initially.

Yet after running a benchmark of 100 realistic editing operations against this setup, the completion rate was dismayingly poor.

Only 48 out of 100 tasks executed successfully, yielding a flat 48% pass rate.

The model required an average of 4.2 roundtrips per modification task, caught in endless loops of parameter retries.

Even worse, 27 tasks suffered silent failures where the interface reported success while the canvas canvas state never mutated.

A postmortem thread on V2EX under topic t/1253992 by the authors of open source project QuireKit diagnosed this structural bottleneck and proposed a self-describing meta capability architecture.

We refactored our test environment communication layer following their principles.

Across the identical 100 composite tasks, successful runs jumped from 48 to 98, with average roundtrips collapsing to 1.1 calls per task.

<img src="https://img.freeaidog.com/images/blog/quirekit-self-describing-ai-native-editor-architecture.webp" alt="QuireKit self-describing architecture vs traditional flat tool calling architecture benchmark topology and pass rates" />

---

## 1. Three Fatal Flaws in Flat Tool Architectures: Error Logs and Root Cause Analysis

To evaluate the architectural shift, one must pinpoint where flat tool calling breaks down inside a rich text engine.

Modern rich text editors rely on a strict abstract syntax tree state machine.

ProseMirror works this way.

Tiptap works this way.

Lexical works this way as well.

It is never crude HTML string concatenation, nor is it raw streaming Markdown text.

Dumping fifty flat endpoints onto an LLM immediately collides with three engineering walls.

### Flaw 1: Context Bloat and Severe Tool Selection Drift

In a full fledged editor, the combination of node and mark extensions easily exceeds fifty items:

- Block nodes: paragraphs, headings level 1 through 6, bullet lists, ordered lists, task items, blockquotes, code blocks
- Inline marks: bold, italic, strike, underline, inline code, text color, highlight, links
- Rich widgets: tables, cell merging, column splitting, images, chart cards, details disclosure, math blocks

Expressing all these capabilities in standard OpenAI tools format consumes upwards of 2,400 prompt tokens on every request.

Faced with fifty peer function schemas, model attention is heavily diluted across flat fields.

Can a model reliably pick the right tool among fifty options?

Unlikely. In our telemetry, DeepSeek Flash repeatedly suffered selection drift:

When asked to append a row of metrics to a table in section three, the model dispatched `table.splitCell` instead of row insertion; or when prompted for text highlighting, it accidentally invoked background color fills.

### Flaw 2: Double Serialization Loss and Metadata Stripping

To escape the burden of maintaining fifty schemas, some teams adopt an intermediate compromise:

Instruct the model to emit clean Markdown, and then parse that Markdown back into HTML before writing to the editor.

Is this route genuinely low maintenance?

It is self deception. Markdown has limited expressive power for rich structures.

Merged table cells cannot be represented.

Custom block business identifiers and access locks cannot be represented.

Collaborative multiplayer cursors cannot be represented.

The entire pipeline suffers two consecutive translation funnels:

```text
Document AST State Machine -> Export Simplified Markdown -> Model Rewrites Text -> Regex Parses to HTML -> Injected into ProseMirror AST
```

After passing through two lossy conversions, original document metadata is scrubbed clean.

A request to bold a single phrase frequently triggers a total document rewrite, turning nested data tables below into broken plain text code snippets.

### Flaw 3: Coordinate Hallucination and Silent Failures

This issue accounted for all 27 silent failures in our benchmark.

ProseMirror document addressing relies on strict one dimensional integer offsets based on node opening and closing boundaries.

A paragraph begins at position 12 and ends at position 48.

Can an LLM accurately calculate one dimensional integer offsets inside a DOM tree?

Not reliably. The model has no spatial awareness of real DOM byte positions and simply predicts positions based on token probabilities.

When the model targets an insertion at position 120, while the actual table closes at position 84, the ProseMirror runtime throws an immediate exception:

```text
RangeError: Position 184 out of range 0..126
    at Object.replaceRange [node_modules/prosemirror-transform/dist/index.js:1042:15]
    at handleLegacyToolCall [src/editor/legacy-bridge.ts:68:12]
```

To prevent UI crashes, frontend wrapper layers commonly wrap dispatches in generic catch blocks, returning an HTTP 200 payload:

```text
POST /api/editor/dispatch 200 OK
Response: {"status": 200, "message": "command executed successfully"}
```

The server returns 200, the model cheerfully tells the user the update is complete, yet the editor canvas remains completely unchanged.

Who can accept this phantom success behavior? In our benchmark records, it occurred 27 times.

---

## 2. QuireKit Self-Describing Protocol: Collapsing 50 Tools into Three Meta Primitives

QuireKit did not fine tune a model; its intervention targets the communication protocol.

It removes all fifty static schemas from persistent prompts, exposing only three meta primitives:

- `list`: dynamically discovers capabilities active on the current editor canvas
- `explain`: on-demand retrieval of schema parameters and constraints for a single action
- `call`: dispatches an atomic transaction into the state machine and returns structured diagnostics

```text
Interaction Lifecycle:
1. Receive editing intent -> Send list to discover available capabilities with minimal tokens
2. Select target action -> Send explain to read parameter contracts on demand
3. Construct payload -> Send call to dispatch ProseMirror atomic transaction
```

### Step 1: Dynamic Capability Discovery via list

The base system prompt no longer enumerates individual plugin schemas, carrying only a single unified protocol definition.

When the model detects an editing requirement, it sends an initial probe:

```bash
cli api list --target editor --json
```

The editor reflects the summary list of currently enabled capabilities:

```json
{
  "capabilities": [
    {"id": "doc.structure", "desc": "Query document outline and AST block hierarchy"},
    {"id": "heading.set", "desc": "Apply heading level to selection or block position"},
    {"id": "table.insertRow", "desc": "Append empty row after table index or focus"},
    {"id": "callout.create", "desc": "Insert warning or note callout at block position"}
  ]
}
```

This lightweight index contains only IDs and brief descriptions.

Even across sixty plugins, the payload stays below 180 tokens.

Prompt token overhead drops by over 90%, presenting the model with an organized directory rather than a chaotic wall of schemas.

### Step 2: Contract Inspection via explain

Once the model matches `table.insertRow`, it queries the editor for the exact schema required:

```bash
cli api explain table.insertRow --target editor --json
```

The editor returns the focused schema:

```json
{
  "capability": "table.insertRow",
  "parameters": {
    "type": "object",
    "properties": {
      "tableIndex": {
        "type": "integer",
        "description": "Natural zero-indexed order of table in document"
      },
      "position": {
        "type": "string",
        "enum": ["before", "after"],
        "default": "after"
      },
      "targetRow": {
        "type": "integer",
        "description": "Reference row index, defaults to end of table"
      }
    },
    "required": ["tableIndex"]
  },
  "constraints": {
    "requiresFocus": false,
    "transactionType": "atomic"
  }
}
```

Because context loads only one specific schema at a time, model attention is laser focused.

In our telemetry, parameter hallucination dropped to zero during this stage.

### Step 3: Direct Transaction Dispatch via call

Equipped with the contract, the model constructs the payload and executes:

```bash
cli api call table.insertRow --target editor --input '{"tableIndex": 0, "position": "after", "targetRow": 2}' --json
```

Inside the editor, QuireKit maps this directly to a native ProseMirror transaction:

```ts
// Illustration: imports, helpers, and types omitted
export function handleInsertRowCommand(
  editor: Editor,
  input: InsertRowInput
): CommandResult {
  const { tableIndex, position, targetRow } = input;
  const state = editor.state;

  const tableNode = findTableNodeByIndex(state.doc, tableIndex);
  if (!tableNode) {
    return {
      success: false,
      errorCode: "ERR_TABLE_NOT_FOUND",
      errorMsg: `Table node at index ${tableIndex} not found in document`,
    };
  }

  let tr = state.tr;
  try {
    tr = addRowAtPosition(tr, tableNode.pos, targetRow, position);
    editor.view.dispatch(tr);

    return {
      success: true,
      affectedRange: [tableNode.pos, tableNode.pos + tableNode.node.nodeSize],
      docVersion: editor.state.version,
    };
  } catch (err: unknown) {
    const message = err instanceof Error ? err.message : String(err);
    return {
      success: false,
      errorCode: "ERR_TRANSACTION_FAILED",
      errorMsg: `Transaction dispatch error: ${message}`,
    };
  }
}
```

If the table index is invalid, execution halts and returns `ERR_TABLE_NOT_FOUND`.

The payload clarifies valid table ranges.

Armed with actionable error details, the model corrects itself in place instead of claiming false success.

---

## 3. Benchmark Suite Design and Empirical Results

To measure both approaches rigorously, we constructed a benchmark suite of 100 compound editing tasks.

How were these 100 test cases constructed?

Cases were split evenly across five high difficulty operational categories, each holding 20 tasks:

- Category A: Multi-level nested list re-indentation and bulk task item state toggling——20 tasks
- Category B: Section summary extraction across headers into warning callout cards——20 tasks
- Category C: Three by four table data completion, dynamic formulas, and header freezing——20 tasks
- Category D: Inline text mark application and external entity ID metadata binding——20 tasks
- Category E: Document wide cross-reference anchor re-linking and broken slug repairs——20 tasks

We used DeepSeek Flash as the evaluation model, locked temperature to 0.2, and allowed up to 5 retries per task.

Success criteria required headless browser assertions verifying actual AST mutations inside ProseMirror.

Empirical evaluation summary:

```text
========================================================================================
Benchmark Metric          Flat Tool Calling (50+ Schemas) QuireKit Self-Describing Protocol
========================================================================================
Total Test Cases          100 Tasks                       100 Tasks
Successful Tasks          48 Tasks                        98 Tasks
Overall Pass Rate         48.0%                           98.0%
Average Roundtrips        4.2 calls / task                1.1 calls / task
Average Token Usage       3,840 Tokens                    620 Tokens
P95 End-to-End Latency    6.8 seconds                     1.4 seconds
Silent Failures Observed  27 Tasks                        0 Tasks (None Observed)
Plugin Maintenance Cost   Manual prompt engineering       Zero maintenance via reflection
========================================================================================
```

Among the 48 tasks passed under the flat scheme, 31 required multiple corrective retries, pushing P95 latency to 6.8s.

The self-describing protocol slashed average token consumption from 3,840 to 620 tokens.

Because the model accurately identifies target actions on the first attempt, the vast majority of the 98 passing tasks completed within a single roundtrip.

---

## 4. Engineering Ledger: Latency Overhead and Migration Costs

Adopting a self-describing architecture is not without tradeoffs.

### Ledger 1: Network Roundtrip Overhead and Client Side Caching

Does an extra explain roundtrip risk runaway latency?

When handling cold or uncommon actions, an extra network roundtrip is unavoidable.

Under local network benchmarking against DeepSeek endpoints, single HTTP roundtrips average roughly 120ms.

An un-cached explain step adds approximately 240ms of latency.

How can one optimize this?

We implemented a client side memory cache inside the Agent runtime.

On initialization, schemas for five high frequency primitives——paragraphs, bold marks, level 2 headings, bullet lists, and table row additions——are pinned in memory.

Across our 100 benchmark tasks, client cache hit rates reached 76%.

When cached, calls skip the explain step directly into call dispatch, lowering P95 latency from 1.4s to 1.1s.

### Ledger 2: State Machine Hygiene and Legacy Refactoring Costs

Can legacy editors relying on raw innerHTML injection migrate smoothly?

No. The architecture demands atomic transactions and runtime introspection from the underlying editor.

In our migration assessment, engineering effort breaks down as follows:

- Read-only outline inspection interface: roughly 1 engineer day
- Transaction defensive bounds checks for 5 core actions: roughly 2 engineer days
- Error interception and rollback for tables and nested blocks: roughly 3 to 5 engineer days

If an editor relies on un-tracked DOM mutations, this architecture cannot function out of the box.

Teams must encapsulate DOM changes into verifiable ProseMirror transactions before self-describing layers can operate reliably.

---

## 5. Migration Roadmap and Production Recommendations

For teams struggling with prompt context bloat and tool selection confusion, we recommend a three stage migration:

First, prune schemas.

Remove bulky tool definitions from prompts, leaving only a read-only `doc.structure` interface so models learn to inspect document outlines before acting.

Second, deploy discovery endpoints.

Implement `list` and `explain` routes, externalizing JSON schemas for headings, lists, and bold text into modular descriptors.

Third, enforce defensive error codes.

Stop swallowing backend exceptions. Return precise error codes when coordinates overflow or nodes are missing.

When models can inspect APIs dynamically and receive structured error traces upon failure, document editing copilots transform from brittle toys into dependable production utilities.

Rather than dumping fifty schemas into prompts and hoping for the best, why not provide clean, self-describing API gateways? That is how robust agents are built.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/quirekit-self-describing-ai-native-editor-agent-architecture/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

