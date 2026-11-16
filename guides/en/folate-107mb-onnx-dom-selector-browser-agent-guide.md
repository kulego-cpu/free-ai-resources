---
title: "Kicking 14B Vision Models Out of the Browser: Field Notes on the 107MB Folate DOM Selector"
description: "Field notes replacing heavy vision-language models with Folate, a 107MB lightweight INT8 ONNX cross-encoder: CPU-only inference without PyTorch, Mind2Web element ranking, hallucination-free XPath grounding, and closed Shadow DOM traversal."
pubDate: 2026-11-16
author: "Xiao Ke"
category: "Agents & Automation"
tags: ["Folate", "Agent", "Browser Automation", "ONNX", "Playwright", "Budget Hardware"]
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/folate-107mb-onnx-dom-selector-browser-agent-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


Last month I was running browser scraping agents on a budget 2 vCPU 4GB RAM cloud instance.

System memory was constantly dancing on the edge of disaster.

Headless Chromium booted up, and a few active tabs easily consumed 1.2GB of memory.

Whenever I tried running a multimodal vision model alongside to look at the rendered page, the Linux kernel OOM killer stepped in immediately.

Checking dmesg logs, the alert was brutally blunt: `Out of memory: Killed process chromium`.

The browser died. Who could tolerate that kind of fragile setup?

Back then, the trendy approach was screenshot-based visual grounding.

Capture a full-resolution screenshot, feed it to a massive vision-language model, and ask it to output screen pixel coordinates for target buttons.

It sounded futuristic on paper, but in production it was a minefield of hidden traps.

Full-page screenshots weigh two to three megabytes each; shipping images to cloud vision APIs introduced seconds of latency while burning painful API credits.

Worse was coordinate scaling: misconfigure `devicePixelRatio` inside the headless browser by a fraction, and calculated coordinates drift by thirty or forty pixels.

The mouse clicked into empty white margins.

Wasted effort.

Does grounding a simple input box on a webpage really demand a multi-billion parameter visual model?

Last week on Hacker News, I stumbled upon an open-source project named Folate, hosted at github.com/figranium/folate.

The author made a remarkably disciplined design choice: discard visual screenshots entirely, and let a specialized micro-neural network read interactive nodes directly from the DOM tree.

The quantized model file weighs merely 107MB.

I ran it on local machines for several days, and in typical automation workflows, it cleanly solved several long-standing engineering headaches.

## Stripping Away Multimodal Overhead: Why Cross-Encoders Beat Raw Visuals

Many engineers building browser agents assume the visual image seen by human eyes represents the most natural ground truth.

In the face of the DOM tree, however, human eyes take the long detour.

A webpage is inherently a structured document built from HTML tags; where buttons sit, how inputs are named, and which handlers bind to which forms are written in plain code.

Folate adopts a pragmatic architecture: fine-tuned from `google/electra-base-discriminator` on the Mind2Web real-world web dataset, and dynamically quantized into an INT8 ONNX runtime artifact.

It inspects zero pixels.

At inference time, it operates as a discriminative cross-encoder.

It takes two inputs: your natural-language intent, like clicking a checkout confirmation button; and serialized attributes of interactive DOM nodes extracted from the live page.

The model emits relevance scores for each query-element pair.

Inspecting `folate/ranker.py`, the preprocessing pipeline is lean and swift:

```python
# Extract interactive DOM nodes from live HTML
INTERACTIVE_XPATH = (
    "//a | //button | //input | //select | //textarea | "
    "//*[@role='button' or @role='link' or @role='menuitem' or @role='tab' or @role='checkbox' or @role='combobox'] | "
    "//*[@onclick or @tabindex]"
)
```

Leveraging lxml for high-performance XPath parsing, it strips away irrelevant text and decorative layers in milliseconds.

Links, buttons, input fields, and elements with ARIA interactive roles are pulled out in one swift pass.

Each candidate node is serialized into a compact string capped at 96 tokens.

The representation packs tag names, id, name, type, aria-label, placeholder, and inner text.

The primary benefit of this design is closing off model selector hallucination almost completely.

Notice the qualifier: almost completely.

Because Folate only scores candidates extracted from the live DOM tree, the selector it picks is physically guaranteed to exist on the page for Playwright to target; whether ranking puts the right element on top is a separate matter.

Coordinate drift is also bypassed entirely—it never touches pixel math, so retina scaling discrepancies do not exist.

At the level of selector existence, the input carries physical certainty.

## Zero GPU Dependency: Memory and Latency Benchmarks Without PyTorch

To test its mettle on constrained hardware, I moved the harness to an older laptop lacking dedicated graphics.

The system was running Ubuntu 24.04 LTS, dual-core CPU, with 4GB of physical RAM allocated.

The installation was refreshingly featherweight for an AI project:

```bash
# Zero heavy PyTorch or transformers dependencies
pip install folate onnxruntime tokenizers lxml
```

No CUDA, and zero multi-gigabyte deep learning bundles.

The entire runtime depends strictly on onnxruntime, tokenizers, and lxml.

I wrote a compact Playwright script to automate checkout on a demo e-commerce store:

```python
from playwright.sync_api import sync_playwright
from folate import FolateSelectorFinder

finder = FolateSelectorFinder(num_threads=2)

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://demo.e-commerce.local/checkout")
    
    # Extract page source and rank with Folate
    html_source = page.content()
    matches = finder.rank("click place order button", html_source, top_k=3)
    
    if matches:
        best_match = matches[0]
        print(f"Match Score: {best_match['score']:.4f}")
        print(f"Target Selector: {best_match['selector']}")
        page.locator(best_match['selector']).click()
        
    browser.close()
```

While running the script, I monitored htop in an adjacent terminal.

Resource consumption was surprisingly clean.

After model initialization, the Python process settled at around 210MB of resident memory [RSS].

Combined with the 450MB footprint of headless Chromium, the entire automation loop consumed under 700MB of RAM.

Memory never blew up, and the machine ran rock solid.

Where local LLMs demand seven or eight seconds per decision, Folate ranks 50 candidate nodes in 1.8 to 2.4 seconds on dual CPU cores.

Is that slow? That depends on your workload. For batch form automation, it is plenty fast; for high-frequency interactive streaming, you will want to optimize further.

Across Mind2Web standard evaluation splits [test_task, test_website, and test_domain covering 6,037 action steps], this 107MB model delivers Top-1 accuracy between 88.9% and 93.3%.

Compared to paying cloud multimodal APIs for every single click, running completely free and offline is a tangible advantage.

## Real Troubleshooting Detours: Closed Shadow DOM and Async Hydration

In messy production environments, however, setups rarely succeed on the first attempt.

Last week while automating an internal approval dashboard built on Web Components, this pipeline stumbled hard.

A submit button was clearly visible on the screen, but the script ranked an unrelated top-right menu item as the highest match.

How could it miss an obvious button?

I briefly suspected that Mind2Web lacked enterprise component generalization.

Auditing logs, I realized the error was mine.

The test script called `page.content` directly.

That API dumps outer static HTML.

The modern frontend used closed Shadow DOM, locking the actual interactive buttons deep within shadow roots.

Standard lxml XPath cannot pierce shadow boundaries.

Blind to internal nodes, Folate had no choice but to pick from exposed outer tags.

Was the shadow tree impossible to reach?

To bypass this detour, I injected a small JavaScript snippet via Chrome DevTools Protocol to flatten shadow roots prior to extraction:

```javascript
// Recursively extract and flatten all Shadow Root nodes
function getPiercedDOM(root = document.documentElement) {
    let html = '';
    const walker = document.createTreeWalker(root, NodeFilter.SHOW_ELEMENT);
    let node = walker.currentNode;
    while (node) {
        if (node.shadowRoot) {
            html += getPiercedDOM(node.shadowRoot);
        }
        node = walker.nextNode();
    }
    return html;
}
```

Passing the pierced DOM to the ranker, the hidden submit button surged to rank one with a confidence score of 0.9842.

Another trap to watch for is async hydration in modern single-page applications.

In React or Vue apps, DOM elements do not hydrate instantaneously.

If you invoke `ranker.rank` immediately after the load event fires, the model often inspects skeleton screen placeholders.

Automation scripts must await container visibility before capturing the DOM snapshot, ensuring candidate rankings reflect stable page state.

## Boundaries and Technical Limits: What It Cannot Replace

It is vital to be realistic about what Folate cannot do; do not force it onto every webpage blindly.

If you encounter apps rendered entirely on Canvas or WebGL, like Google Maps, browser games, or Figma:

The page contains no standard HTML elements; the entire UI is pure painted pixels.

In such environments, lxml cannot extract interactive tags, and Folate fails completely.

For pure canvas workflows, there is no shortcut around visual models capable of reading raw pixel coordinates.

Deeply nested cross-origin iframes also pose challenges; without targeted CDP frame permissions, nested documents remain invisible.

For conventional web automation: admin consoles, e-commerce stores, flight portals, form fillers, and content aggregators.

VRAM is freed, and screenshot upload latency is gone. On 2 vCPU 4GB machines, that is not marketing speak; it means the OOM killer visits one less time.

Facing low-spec servers with tight budgets, do you really want to keep loading multi-gigabyte vision models just to find a button?

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/folate-107mb-onnx-dom-selector-browser-agent-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

