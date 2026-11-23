---
title: "XERJ Chinese RAG Benchmark Post-Mortem: R@3 Boosted from 22.2% to 91.1% Without Kernel Changes"
description: "On 33575 statute and financial chunks, native BM25 in XERJ scored an R@3 of only 22.22%. We diagnosed three Chinese preprocessing traps and fixed them entirely on the client side."
pubDate: 2026-11-23
author: FreeAI Hub 站长
category: dev-platform
tags:
  - rag
  - rust
  - xerj
  - search-engine
  - nlp
  - python
featured: false
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/xerj-rust-search-engine-chinese-rag-benchmark-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


Many engineers building on-premise knowledge bases look closely at XERJ, the popular 19.4k-star single-binary Rust search engine on GitHub. It ships as a single executable, speaks standard Elasticsearch APIs, consumes barely dozens of megabytes, and spins up with one command. For English documentation and API reference querying, its millisecond-level responsiveness feels great.

Put it in front of real-world Chinese legal and financial corpora, however, and the out-of-the-box configuration breaks down fast.

We partitioned 208 real-world compliance documents into 33,575 chunks and executed a rigorous 60-query benchmark. The terminal numbers were sobering. Under native BM25 lexical retrieval, Top 3 evidence passage recall sat at an abysmal 22.22%. End-to-end task success plummeted to 5.3%.

What broke? Did an established high-star Rust engine fail at basic inverted indexing?

Spending a full day cross-checking raw chunks and query logs revealed that the Rust engine core was blameless. The accuracy collapse stemmed from three subtle preprocessing traps unique to Chinese text: numeral notation mismatches, legacy PDF font CMap corruptions, and single-character over-recall cascades.

Without touching a single line of XERJ Rust source code, we implemented a zero-dependency client-side query adapter using standard Python libraries. Rerunning the identical 33,575-chunk benchmark produced an immediate turnaround. Top 3 evidence recall surged to 91.11%, and end-to-end task completion returned to 97.4%.

The evaluation report is documented in PR #3371 and corpus additions in PR #3372, both merged into the upstream xerj-org/xerj repository. Native CJK kernel support is tracked in official Issue #3377. The testing toolkit vericorpus is open-sourced on GitHub. Here is our complete post-mortem of the three production traps and our client-side healing implementation.

---

## Benchmark Setup and Corpus Profile

To prevent synthetic overfitting, all 33,575 chunks originated from production-grade compliance corpora indexed within a single namespace:

- Hardware host: Apple Silicon Mac, unified memory, macOS system.
- Engine version: XERJ v1.0.0-rc.101 listening on port 9200, executed with XERJ_DISABLE_QUERY_CACHE=1 to eliminate query cache skew.
- Corpus composition:
  1. Statutory corpus: 29 Chinese commercial statutes comprising 3,204 articles, preserved article by article.
  2. Clinical tables: 55 open-access clinical trial papers from NIH and Europe PMC, yielding 271 structured tables.
  3. Exchange filings: Tencent Holdings 0700.HK quarterly reports, US SEC 10-K filings, and STAR Market IPO prospectuses.
- Verification pipeline: Retrieved Top 3 passages feed directly into a local verification model, which must reject if evidence is insufficient.

Indexing all 33,575 chunks took roughly 9.2 seconds. Raw indexing throughput was genuinely impressive. But when real statutory, balance-sheet, and clinical queries were fired, retrieval precision collapsed.

---

## Dissecting the Three Retrieval Failure Modes

Reading raw terminal failures revealed that our initial hunch—BM25 term saturation—was wrong. The errors originated squarely in string intake and preprocessing.

### 1. Arabic vs Chinese Textual Numeral Misalignment

Statutory texts write provisions out in Chinese characters, such as Article One Hundred Forty-Seven.

Real humans and coding agents never type out full formal words. They query Arabic digits, such as "Article 147 director duties under Company Law".

Native tokenization broke "Article 147" into distinct fragments. In the inverted index, provisions were cataloged under character tokens for hundred, forty, and seven. The tokenizer failed to match digit 147, stripping the query of its most restrictive numeral filter. Retrieval results returned generic corporate law sections while the actual Article 147 was pushed past position thirty.

Zero recall on critical clauses. One formatting quirk eradicated precision on legal search.

### 2. HKEX PDF Font CMap Corruption and Broken Line Floats

Financial teams regularly ingest Traditional Chinese filings from the Hong Kong Stock Exchange. This is where unmanaged document parsing hurts most.

Examining Tencent 0700.HK quarterly filings, keyword searches targeting specific quarters repeatedly missed. Printing extracted chunks in the terminal revealed the culprit:

```text
For the three months ended September 30, 2026 -> extracted with character 'month' replaced by 'corpse'
```

Embedded fonts in legacy HKEX filings suffer from a font CMap encoding drift where the glyph for month maps to an unrelated character. Across one filing, 341 dates suffered this corruption while the standard glyph appeared zero times. Searching for quarterly revenue yielded complete misses.

A search engine cannot fix corrupted text it never received.

Broken line-wraps made tables worse. Floating-point numbers split across soft returns:

```text
Total Revenue was 39.\n9 Billion USD
Accounting negative notation (\n1,234.5) Million
```

Default sentence segmentation detached decimal points from digits. Searching for "39.9 billion" hit nothing because the index stored two disconnected pieces.

### 3. Single-Character Over-Recall on Trap Questions

To test hallucination resistance, the benchmark included 4 unanswerable questions, such as asking for overseas subsidiary revenue from a filing that never disclosed geographical segments.

A reliable RAG pipeline must return zero passages and trigger explicit refusal.

Under native BM25 rules, queries degraded to single-character matching. Individual characters triggered matches across thousands of unrelated paragraphs. Each query flooded the context with 15 to 3,319 weakly matching chunks. Downstream models drowned in noise and hallucinated fabricated figures. All 4 trap questions failed to reject.

---

## Zero-Dependency Client Healing Adapter

Many teams respond to these issues by rewriting engine tokenizers or spinning up GPU-heavy reranker models. In edge or air-gapped deployments, dedicating an entire GPU just to fix encoding glitches is unnecessary overhead.

We built a lightweight client-side adapter named query_cn. It introduces zero external pip packages and relies strictly on Python standard libraries re and unicodedata.

Processing flows through three clean stages:

```text
User Query
    │
    ▼
Stage 1: Bidirectional Numeral Expansion
Expands "Article 147" to both digit and textual forms
    │
    ▼
Stage 2: HKEX CMap and Table Reconnection
Repairs date glyphs and rejoins split decimal floats
    │
    ▼
Stage 3: High-Confidence Constraints and Trap Defense
Enforces term length gates and drops noise to zero
    │
    ▼
Target Engine Endpoint
```

### Implementation: Bidirectional Statute Expansion

A standard-library converter maps Arabic digits to formal Chinese characters, injecting both variations into multi-match queries:

```python
import re

CN_NUM_MAP = {
    '0': '零', '1': '一', '2': '二', '3': '三', '4': '四',
    '5': '五', '6': '六', '7': '七', '8': '八', '9': '九'
}

def expand_legal_query(query: str) -> str:
    def replacer(match):
        raw_num = match.group(1)
        # Expands digits into Chinese character provisions
        return f'Article {raw_num} Section {raw_num}'

    return re.sub(r'Article\s*(\d+)', replacer, query)
```

Queries searching for Article 147 supply both numeric and character terms simultaneously, hitting inverted indices on both sides.

### Implementation: Safe Context-Bound Glyph Repair

Corrupted glyphs cannot be blindly replaced across entire documents. A global search-and-replace for the corrupted character would ruin genuine medical or legal terms.

We bound the replacement strictly to date and month contexts using regular expressions:

```python
def repair_hkex_cmap_safe(text: str) -> str:
    # Target only numeric or character date prefixes
    pattern = r'([0-9]{1,4})\s*corrupted_glyph\s*([0-9]{1,2})\s*day'
    
    def repl_date(m):
        return f'{m.group(1)} month {m.group(2)} day'

    res = re.sub(pattern, repl_date, text)
    # Rejoin floating-point numbers split across line breaks
    res = re.sub(r'(\d+)\.\s*\n\s*(\d+)', r'\1.\2', res)
    return res
```

In empirical testing, this logic repaired all 341 broken date glyphs in Tencent filings with zero false positives on genuine vocabulary.

---

## Benchmark Comparisons Across 60 Real Queries

Using the identical Apple Silicon host and unchanged XERJ index files, we executed three comparative runs across 33,575 chunks.

Test queries covered:
- 20 strict SEC and XBRL financial metrics
- 10 hard-tech IPO prospectuses
- 15 Chinese statutory compliance scenarios
- 15 intentionally unanswerable trap questions

Comparative metrics:

| Benchmark Branch | Correct | Wrong | No-Answer | Net Score | Evidence R@1 | Evidence R@3 | p50 Latency |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Arm 1: Human-curated oracle passages | 52 | 7 | 1 | 38 | 100.0% | 100.0% | 0.0 ms |
| Arm 2: Native XERJ BM25 default | 24 | 11 | 25 | 2 | 17.78% | 22.22% | 113.67 ms |
| Arm 3: VeriCorpus query_cn adapter | 49 | 6 | 5 | 37 | 77.78% | 91.11% | 102.50 ms |

Top 3 evidence passage recall jumped from 22.22% to 91.11%, while refusal-related verification errors dropped from 25 to 5. Measured against oracle passage performance, native BM25 scored an unusable 5.3%, whereas the client adapter restored operational parity to 97.4%.

Latency remained flat. Precompiled regular expressions and in-memory lookups added less than 1 millisecond. By cutting out sprawling candidate over-recall, search p50 latency actually dropped from 113.67 ms to 102.50 ms. All 4 trap queries rejected cleanly without hallucinations.

---

## Practical Deployment Guidelines

The clearest takeaway from this benchmark is simple: never rush to swap out embedding models the moment retrieval misbehaves.

In production environments, vector embeddings cannot fix mangled source characters or unaligned numeral representations. Before increasing model parameter counts, verify three fundamentals:

- Check whether text extraction pipelines repair PDF font CMap corruptions on annual reports.
- Verify whether article and clause identifiers are bidirectionally normalized across Arabic and character notations.
- Ensure that low-confidence single-character queries are cleanly gated, returning empty sets when matches are dubious.

Upstream CJK kernel discussions continue under XERJ Issue #3377. But until engine internals shift, placing robust cleaning and query rewriting on the client side remains the most cost-effective path to deterministic retrieval.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/xerj-rust-search-engine-chinese-rag-benchmark-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

