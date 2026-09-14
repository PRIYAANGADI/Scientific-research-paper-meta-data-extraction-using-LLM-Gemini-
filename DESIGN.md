# Design Document: Research Paper Metadata Extraction Pipeline

## Purpose

This document explains the technical design decisions behind the pipeline — what problem each part solves, why it's built the way it is, and what tradeoffs were made. See `README.md` for setup and usage instructions.

## Problem Statement

Given a folder of scientific PDFs, produce a structured table of 19 bibliographic and thematic attributes per paper (title, authors, DOI, funding, methodology, technology, research theme, etc.), suitable for use in a literature review or bibliometric inventory — without a human manually reading and coding every paper.

Constraints that shaped the design:
- Papers arrive as PDFs of varying length and formatting quality.
- The number of papers needs to scale from a handful to 30+ without a rewrite.
- Runs happen in Google Colab, where sessions can disconnect or time out mid-batch.
- LLM calls are the bottleneck: they're the slowest, most rate-limited, and (for paid tiers) most costly part of the pipeline.
- Extracted values must be traceable to the source text — no silent guessing.

## Architecture Overview

```
PDF folder (Drive)
      |
      v
Load full text per paper  (PyPDFLoader, one pass, no chunk overlap)
      |
      v
Structured extraction      (1 Gemini call per paper, schema-enforced,
      |                      concurrent across papers, retried on failure)
      v
Incremental JSON save      (written after every paper -- resumable)
      |
      v
Excel export                (styled matrix, one row per paper)
```

A parallel, currently-inactive branch chunks the same paper text (with overlap, for retrieval recall) into a Chroma vector store using Gemini embeddings. This exists as scaffolding for a future retrieval step (e.g. cross-paper semantic search) and does not affect the extraction path above.

## Key Design Decisions

### 1. One structured call per paper, not one call per field

**Original approach:** loop over each of 19 fields and issue a separate free-text Gemini call per (paper, field) pair. At 30 papers this is 570 calls.

**Current approach:** define a Pydantic model (`PaperRecord`) covering all 19 fields, and use `llm.with_structured_output(PaperRecord)` to request the entire record in one call. This drops the call count to one per paper (30 total).

**Why:** each call carries the full paper text as context, so 19 separate calls means paying for that context 19 times over for no added benefit — the model doesn't need the fields extracted in isolation from each other. Structured output also removes a manual text-parsing step, since the response deserializes directly into the schema.

**Tradeoff:** a single large call packs more instructions into one prompt (19 field descriptions at once). Field descriptions are kept short and are generated programmatically from `TARGET_FIELDS`, so adding or removing a field doesn't require touching the extraction logic.

### 2. No overlapping-chunk duplication in the LLM context

**Original approach:** split each paper into chunks with `chunk_overlap=256`, then join those chunks back together to form the "full context" sent to the LLM.

**Current approach:** load each paper's raw page text directly (`PyPDFLoader`, joined page by page) with no splitting at all for the extraction step.

**Why:** chunking with overlap exists to help *retrieval* (so no relevant passage falls on a chunk boundary). But the extraction step doesn't retrieve individual chunks — it concatenates all of them back into one string. Overlapping chunks joined this way duplicate ~20-25% of the paper's text in the prompt, which inflates token usage and cost with no accuracy benefit, since the model already sees the whole paper either way.

**Tradeoff:** chunking is still needed for the (currently unused) embeddings/Chroma step, where overlap genuinely helps retrieval recall. That logic is kept as a separate function (`build_chunks_for_embeddings`) so it isn't accidentally reused for the extraction context.

### 3. Concurrency with a bounded thread pool

**Why threads, not async:** the LangChain Gemini client's `.invoke()` call is synchronous and I/O-bound (waiting on the network), which is a good fit for a `ThreadPoolExecutor` without needing to rewrite the call chain as async.

**Why bounded (`MAX_WORKERS`):** per-paper calls are independent, so there's no correctness reason to serialize them. But firing all 30 at once risks tripping the Gemini API's rate limits. A bounded pool (default 5) caps how many calls are in flight at once; this is a tunable knob, not a fixed constant, because the right value depends on your API tier.

### 4. Retry with exponential backoff + jitter

**Why:** at low concurrency (5 papers) transient errors were rare enough to ignore. At 30 papers, concurrent calls make hitting a rate limit (HTTP 429) or a transient timeout far more likely, and a single failed call shouldn't sink an otherwise-successful batch.

**Design:** each call gets up to `MAX_RETRIES` attempts, with the wait time growing exponentially (`2^attempt` seconds) plus a small random jitter, before it's allowed to fail permanently. Jitter avoids many retrying calls waking up at the exact same moment and re-tripping the rate limit together.

**Failure mode:** a paper that exhausts its retries is recorded with `"N/A"` in every field rather than raising and aborting the whole run — one bad paper (corrupt PDF, unusually long text, persistent API error) shouldn't block the other 29.

### 5. Incremental, resumable saves

**Why:** Colab sessions can disconnect mid-run, and a 30-paper batch with retries can take a while. Losing all progress on a disconnect would be costly.

**Design:** the result list is written to `OUTPUT_JSON` after *every* paper completes (success or permanent failure), not just once at the end. On startup, `load_existing_records()` reads that file and treats any paper already present as done, skipping it on re-run. This makes "just run the script again" a safe and sufficient recovery strategy — no manual bookkeeping of which papers already finished.

**Tradeoff:** this means `OUTPUT_JSON` is being opened and rewritten repeatedly (once per completed paper) rather than once at the end. At 30 papers this is a non-issue; at a much larger scale, appending to a line-delimited JSON file (JSONL) instead of rewriting the whole file each time would scale better.

### 6. Conservative extraction instructions ("NA," not a guess)

**Why:** the pipeline's output is meant to be a literature review starting point, not a final verified dataset. A model that fills in plausible-sounding but unstated values would be actively misleading, and harder to catch than an honest "NA."

**Design:** the system prompt and every field's schema description explicitly instruct the model to answer "NA" when a value isn't stated in the text, and interpretive fields (main topic, research theme, etc.) are asked to show brief reasoning before committing to a value, so the reasoning is visible if a result looks off during review.

## Known Limitations / Things Not Yet Done

- **No token-budget guard.** Extremely long papers are sent in full; no truncation or summarization step exists if a paper approaches the model's context window limit.
- **Embeddings/Chroma step is inactive.** It's built and ready to populate but nothing currently queries it — the metadata extraction path is fully independent of it today.
- **No accuracy validation loop.** There's no gold-standard comparison built into this script; that's a manual step recommended in the README before treating output as final.
- **JSON rewrite-per-paper**, noted above, is a scaling limit if this grows well beyond 30-50 papers per run.

## Possible Future Directions

- Batch multiple papers into fewer, larger structured calls if Gemini's context window and rate limits allow it, to further reduce call count.
- Switch `OUTPUT_JSON` to JSONL (append-only) if the paper count grows substantially.
- Activate the embedding store to support cross-paper semantic search or a Q&A layer on top of the corpus.
- Add an automated confidence flag (e.g. "NA"-heavy records, or fields with low-confidence reasoning) to prioritize which extractions get human review first.
