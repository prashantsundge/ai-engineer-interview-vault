# Lessons Learned — RAG Pipeline Tuning & LLM Prompt Engineering
### SharePoint AI Chatbot Project | June 2026

---

## Overview

Building a RAG system that retrieves the right document is only half the battle. The harder problem is getting the LLM to produce answers that are actually useful — specific, grounded, and formatted correctly — rather than generic templates that describe documents instead of answering questions. This document captures every tuning decision made, what failed, what worked, and why.

---

## Problem 1 — LLM Describing Documents Instead of Answering Questions

**Symptom:** User asked "please provide me Lincoln Electric site information". The system retrieved the correct runbook. The LLM returned:

> *"The document titled 'Lincoln Electric1 Runbook.V2' contains site information. It includes sections on Customer Location, IP addresses, and Contact details."*

This describes the document. The user wanted the actual site address, IP ranges, and contacts.

**Root Cause:** The generation prompt used a rigid template:
```
Use this format:
Document Found
Summary: <answer>
Key Evidence: - fact 1 - fact 2
Source Documents: - file name - page number
```

When you give the LLM a template, it fills the template. It summarises the document structure because that fits "Summary:" better than extracting specific data values.

**Fix — Rewrite the prompt with action-oriented instructions:**
```
Answer the SPECIFIC question asked — do not describe what the document contains.
Extract and present the ACTUAL DATA: IP addresses, contacts, configurations.
If the question asks for site information, provide ALL site details found.
Start immediately with the answer — no preamble.
```

**Lesson:** Template-based prompts produce template-shaped answers. For enterprise Q&A, the prompt must instruct extraction of specific data values, not document summarisation. Remove all output format templates and replace with content-quality rules.

---

## Problem 2 — "I May Not Have Sufficient Information" Replacing Valid Answers

**Symptom:** The system retrieved the correct documents with relevant content. The LLM generated a good answer. But the user saw:

> *"I may not have sufficient information in the available documents to confidently answer this question. Would you like me to refine the search or check other related documents?"*

**Root Cause:** `followup_node.py` contained:
```python
if state.get("is_low_confidence"):
    state["answer"] = "I may not have sufficient information..."
    return state
```

This **overwrote the LLM's answer** with a hardcoded fallback whenever confidence scored below threshold. The confidence threshold was also miscalibrated — spreadsheet chunks scored low even when they contained the exact answer.

**Two separate bugs:**

1. `followup_node.py` was designed to add follow-up suggestions but was also silently replacing the main answer
2. `confidence_node.py` had a threshold of 0.60 with `stability="low"` which triggered for almost every spreadsheet query

**Fix:**
```python
# followup_node.py — NEVER touch state["answer"]
def followup_node(state, llm_service=None):
    if state.get("is_low_confidence"):
        # Suggest refinements but KEEP the answer
        state["followups"] = ["Try specifying document name...", ...]
        return state  # answer unchanged
```

```python
# confidence_node.py — calibrated thresholds by content type
has_spreadsheet = any(m in ("spreadsheet", "ocr") for m in modalities)
threshold = 0.38 if has_spreadsheet else 0.45  # lower for structured data
```

**Lesson:** Never let a scoring/routing node overwrite the generation node's output. Keep concerns separated — confidence is a signal for the UI, not an override mechanism for the answer. The generation node owns the answer; everything downstream should only annotate it.

---

## Problem 3 — Confidence Scoring Too Aggressive for Spreadsheet Content

**Symptom:** Confidence scored 27-45% for queries that returned clearly correct Excel spreadsheet answers. The system displayed "Low confidence" even when source data was exact.

**Root Cause:** The confidence formula relied heavily on semantic similarity scores (`avg_similarity * 0.40`). Spreadsheet chunks contain short, structured cell values ("London | 192.168.1.0/24 | John Smith") that have low cosine similarity with natural language queries ("Lincoln Electric site information"). The mismatch is structural, not a quality problem.

**Fix — Spreadsheet-aware confidence formula:**
```python
if has_spreadsheet:
    # Weight grounding score higher — did the answer use context words?
    confidence = (
        avg_similarity * 0.30 +   # lower — structure mismatch expected
        rerank_avg * 0.20 +
        grounding_score * 0.35 +  # higher — key signal for spreadsheets
        ocr_avg * 0.15
    )
    threshold = 0.38  # lower threshold for structured data
```

The grounding score (overlap between answer tokens and context tokens) is a better signal for spreadsheets because cell values appear verbatim in both context and answer.

**Lesson:** Confidence metrics must account for content modality. A threshold that works for prose documents will fail for spreadsheets, OCR, and structured data. Add modality detection and tune thresholds per content type.

---

## Problem 4 — Context Window Too Small for Multi-Sheet Excel Files

**Symptom:** "Lincoln Electric site information" retrieved chunks from sheets 1, 3, 5, 6, 13 — but site info was on sheet 4. With top-K=5, sheet 4 was ranked 6th and excluded from context.

**Root Cause:** `context_node.py` used a fixed top-K of 5 chunks regardless of content type. Excel workbooks spread data across many sheets — a 15-sheet runbook might have the most relevant data on any sheet.

**Fix — Adaptive top-K based on content type:**
```python
has_spreadsheet = any(m == "spreadsheet" for m in modalities)
if has_spreadsheet:
    top_k = min(len(chunks), 10)  # take all available for spreadsheets
elif stability == "high":
    top_k = 5
elif stability == "medium":
    top_k = 7
else:
    top_k = min(len(chunks), 10)
```

Also increased `retrieval_node.py` initial retrieval from 5 to 10 vector results so more candidates reach the reranker.

**Lesson:** One-size top-K doesn't work for heterogeneous enterprise content. Spreadsheets need more chunks. Single-document queries need fewer. Make top-K dynamic based on the content type and retrieval stability score.

---

## Problem 5 — Markdown Tables Rendered as Pipe-Separated Text

**Symptom:** The LLM correctly generated a markdown table for the Day 2 Turnover Checklist:
```
| SLNo | Name | Status |
|------|------|--------|
| 1 | Avatar Monitoring | ✅ |
```

But the UI displayed: `| SLNo | Name | Status | |------|------|--------| | 1 | Avatar Monitoring |`

**Root Cause:** Two separate issues:

1. `ReactMarkdown` does not support GFM (GitHub Flavored Markdown) tables by default — it needs the `remark-gfm` plugin
2. The LLM was sometimes generating malformed tables (missing separator row `|---|---|`) because the prompt didn't specify that the separator is mandatory

**Fix — Frontend:**
```bash
npm install remark-gfm
```
```tsx
import remarkGfm from "remark-gfm";

<ReactMarkdown
  remarkPlugins={[remarkGfm]}
  components={{
    table: ({ children }) => (
      <div style={{ overflowX: "auto", borderRadius: "8px" }}>
        <table style={{ borderCollapse: "collapse", width: "100%" }}>
          {children}
        </table>
      </div>
    ),
    th: ({ children }) => <th style={{ padding: "10px 14px", fontWeight: 600, color: "#1d4ed8" }}>{children}</th>,
    td: ({ children }) => <td style={{ padding: "9px 14px", color: "#334155" }}>{children}</td>,
  }}
>
  {content}
</ReactMarkdown>
```

**Fix — Prompt:**
```
FORMATTING RULES:
For ANY tabular data, ALWAYS use proper markdown table format:
| Column 1 | Column 2 |
|----------|----------|    ← separator row is MANDATORY, never omit it
| value 1  | value 2  |
```

**Lesson:** `react-markdown` needs `remark-gfm` for tables. The LLM needs explicit instruction that the separator row is mandatory — without it, it sometimes skips the `|---|---|` line which breaks the parser silently.

---

## Problem 6 — Follow-up Questions Searching Wrong Document

**Symptom:** User asked "nutreco runbook contacts" → got the Nutreco runbook with contacts table. Then asked "site information?" → system retrieved a completely different document (PDC LLD) instead of staying in the Nutreco runbook.

**Root Cause:** The system had no conversation memory. Every query was independent — the backend received only the current query string, with no knowledge of what was being discussed. "Site information?" matched many documents semantically, none of which was the Nutreco runbook.

**Fix — Conversation memory system:**

1. Frontend tracks `activeDocument` per chat session, updated from each response's metadata
2. Frontend sends `conversation_history` (last 6 messages) and `active_document` with every request
3. `intent_node.py` detects follow-up queries (short queries, vague references, single-topic words)
4. `retrieval_node.py` applies document-scoped Qdrant filter when follow-up detected
5. `generation_node.py` injects conversation history into the prompt

```python
# intent_node.py — follow-up detection
def is_followup_query(query, conversation_history):
    if not conversation_history:
        return False
    word_count = len(query.strip().split())
    if word_count <= 3:
        return True  # very short with prior context = follow-up
    # Pattern match: "site information?", "what about contacts?"
    for pattern in FOLLOWUP_PATTERNS:
        if re.search(pattern, query, re.IGNORECASE):
            return True
    return False
```

```python
# retrieval_node.py — scoped search
if is_followup and active_document:
    search_filter = Filter(must=[
        FieldCondition(key="file_name", match=MatchText(text=active_document))
    ])
    results = vector_repository.search(query_vector, top_k=10, filters=search_filter)
```

**Lesson:** Multi-turn conversation requires explicit state management. The LangGraph state dict must carry `conversation_history` and `active_document` as first-class fields. The retrieval node must honour document scope when context exists. Without this, every follow-up question feels like starting over.

---

## Problem 7 — SSE Stream Metadata Not Reaching Frontend

**Symptom:** Backend logs confirmed: `Streaming final metadata with 4 sources`. Sources had valid `web_url` values. But the frontend showed no source cards — sources array was always empty.

**Root Cause:** The SSE parser was buggy. The `event: metadata\ndata: {...}` block arrives as:
```
event: metadata
data: {"sources": [...]}
```

The old parser split the buffer on `\n\n` (correct) but then called `.trim()` on the entire block before splitting on `\n` to get lines. This turned the multiline block into one string starting with `"event: metadata"` — which triggered `currentEvent = "metadata"` and then `continue` — skipping to the next loop iteration without ever processing the `data:` line.

**Fix:**
```typescript
for (const block of blocks) {
  let eventType = "";
  let dataLine = "";

  // Split block into lines FIRST, then extract event type and data separately
  for (const line of block.split("\n")) {
    const trimmed = line.trim();
    if (trimmed.startsWith("event:")) {
      eventType = trimmed.replace("event:", "").trim();
    } else if (trimmed.startsWith("data:")) {
      dataLine = trimmed.replace("data:", "").trim();
    }
  }

  // Process after reading the complete block
  if (dataLine && eventType === "metadata") {
    const parsed = JSON.parse(dataLine);
    // update sources, confidence, etc.
  }
}
```

**Lesson:** SSE parsing must process each `\n\n`-separated block as a unit. Extract event type and data from the block's lines together before processing — not line by line with continue statements that skip sibling lines.

---

## General Prompt Engineering Principles Learned

### 1. Be prescriptive about extraction, not format
❌ Wrong: *"Use this format: Document Found / Summary / Key Evidence"*  
✅ Right: *"Extract the actual IP addresses, contacts, and configuration values"*

### 2. Explicitly forbid the bad behaviour
❌ Wrong: *"Answer from the context"*  
✅ Right: *"NEVER say 'I may not have sufficient information' if context contains relevant data"*

### 3. Modality-specific formatting instructions
For spreadsheet data, you must tell the LLM:
- "Recreate tabular data as markdown tables"
- "The separator row `|---|---|` is MANDATORY"
- "Never write pipe-separated data on a single line"

### 4. Conversation history positioning matters
Put conversation history AFTER the rules but BEFORE the context. The LLM reads sequentially — rules must be established before it encounters historical context.

### 5. Trim history aggressively
Long conversation histories consume tokens that should go to document context. Cap at 6 messages (3 exchanges) and truncate each to 400 characters.

### 6. Confidence thresholds must be empirically tuned
Start with 0.5 and observe. Different content types (prose vs spreadsheet vs OCR) require different baselines. Log every confidence score with its modality and calibrate from real data.

---

## Performance Observations

| Query Type | Avg Response Time | Notes |
|-----------|-------------------|-------|
| First request (cold) | 12-15 seconds | Embedding model + reranker loading |
| Subsequent requests | 8-12 seconds | Models cached in memory |
| Spreadsheet queries | 10-13 seconds | More chunks, larger context |
| Follow-up queries | 6-9 seconds | Smaller Qdrant search scope |

**Bottlenecks identified:**
1. BM25 index rebuild on every cold start (~3s for 2260 chunks)
2. Cross-encoder reranker loads model on first request (~5s)
3. Synchronous `graph.invoke()` blocks the event loop (mitigated with `run_in_executor`)

---

*Document maintained by: Pravin Kumar Sundge | CONNX 2026 | June 2026*
