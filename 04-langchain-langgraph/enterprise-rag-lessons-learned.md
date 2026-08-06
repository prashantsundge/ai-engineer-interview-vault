## LESSONS LEARNED

### Enterprise SharePoint Multimodal Agentic RAG System

Production Engineering | Security | Pipeline Optimisation | RAG Architecture

## 1. Executive Summary

This document records the complete set of engineering lessons learned during the design, development, and hardening of the CONNX Enterprise SharePoint Multimodal Agentic RAG System — a production-grade AI knowledge assistant built on React 19, FastAPI, LangGraph, Qdrant, GPT-4o-mini, and deployed on ESXi infrastructure.

The system was developed over 6 months, beginning as a prototype and evolving through multiple production incidents into a security-hardened, performant enterprise platform. This document is intended for future engineers inheriting the system, for interview preparation demonstrating production GenAI experience, and as a reference for teams building similar systems.

| Metric                        | Value                                                                 |
|-------------------------------|-----------------------------------------------------------------------|
| Total chunks indexed          | 3,004+                                                                |
| File types supported          | PDF, XLSX, DOCX, PPTX, MKV, MP4, MSG, JPG, PNG, TXT, VTT             |
| Monthly cost (production)     | ~$20/month (OpenAI API only)                                         |
| Infrastructure                | On-premise ESXi — full data sovereignty                               |
| Security incidents resolved    | 5 (credential exposure, mass-processing, auth bypass, token crash, rate limit) |
| Pipeline latency (before)     | 8.4 seconds average                                                   |
| Pipeline latency (after)      | ~2.8 seconds average                                                 |

## 2. Security — Critical Lessons

### 2.1 Credential Sanitizer Architecture

The most persistent and serious issue in the project was credentials from operational runbooks appearing in chatbot answers. The sanitizer was rebuilt three times before achieving complete coverage.

#### The Root Cause

Operational runbooks store credentials in organisation-specific formats that no standard sanitizer library covers. The Nutreco runbook used a bullet table format:

* For AZ-Prod-SBC-Act (UK): L3tm3l0g!n2@s!ms
* For Audio Code SBC: NuTr3c0$

**Password: [REDACTED] C@$Hm0n3y** ← old sanitizer inserted label but left value

| ⚠ Lesson | The old sanitizer inserted [REDACTED] as a label prefix but did NOT consume the actual value after it. 'Password: C@$Hm0n3y' became 'Password: [REDACTED] C@$Hm0n3y' — the label was added but the secret remained visible. |

#### The Fix: Two-Layer Defence

Layer 1 — Context sanitization: strip credentials from retrieved chunks BEFORE sending to LLM. The LLM cannot include what it never receives.

Layer 2 — Answer sanitization: apply 11 regex patterns to LLM output before streaming to the user.

The key insight is that Layer 1 is stronger — if the context is clean, the LLM output is clean. Layer 2 is a safety net for anything Layer 1 misses.

#### Pattern Coverage (Final Sanitizer)

| Pattern                                         | Example                          | Status      |
|-------------------------------------------------|----------------------------------|-------------|
| password/secret label → value                   | Password: C@$Hm0n3y             | ✅ Pattern 2 |
| Already-redacted with trailing value             | **Password: [REDACTED] value    | ✅ Pattern 1 |
| username/userid → value                         | Username: Pro3$3rver             | ✅ Pattern 3 |
| For <device>: <credential> (runbook format)    | * For AZ-Prod: L3tm3l0g!        | ✅ Pattern 4 |
| Bearer JWT tokens                               | Authorization: Bearer eyJ...     | ✅ Pattern 5 |
| OpenAI/Anthropic API keys                       | sk-proj-xK9mP2v...               | ✅ Pattern 6 |
| URL embedded credentials                         | redis://:password@host           | ✅ Pattern 9 |
| Environment variable assignments                 | REDIS_PASSWORD=abc123            | ✅ Pattern 11 |

🔑 Interview Answer: The sanitizer uses two layers. Layer 1 sanitizes the retrieved context before the LLM sees it — the LLM cannot include credentials it never received. Layer 2 applies 11 cascading regex patterns to the generated answer. Pattern 4 is our custom runbook-format pattern: it matches 'For <device>: <credential>' lines using a heuristic that the value must contain at least one letter AND one special character — this prevents false positives on lines like 'For more information: see docs'.

### 2.2 JWT Authentication — Lessons

#### The verify_signature: False Decision

Microsoft Graph tokens (issued for User.Read scope) are intentionally opaque to third parties — their RS256 signature cannot be verified by the resource server. This is documented Microsoft behaviour, not a bug.

The correct solution is to use the Chat.Access scope under your App Registration, which issues a token with your API as the audience — fully RS256 verifiable. However, on the demo tenant, the scope appeared in 'Expose an API' but not in 'My APIs' due to propagation delays.

| 💡 Lesson | When 'My APIs' is empty in Azure API Permissions, do not waste time searching. Use the 'APIs my organization uses' tab and search by Client ID directly. The scope appears there before it appears in 'My APIs'. |

| Token Type                     | Audience                | RS256 Verifiable         | Use Case                     |
|--------------------------------|------------------------|--------------------------|-------------------------------|
| Graph token (User.Read)       | 00000003-0000-...      | No — intentionally opaque | User profile, email          |
| API token (Chat.Access)       | api://CLIENT_ID        | Yes — full verification   | Your RAG backend             |

#### MSAL v5 Platform Configuration

The most common SSO failure: registering the redirect URI as a 'Web' platform instead of 'Single-page application'. Web platform = server-side auth code flow. SPA platform = browser PKCE flow. Using Web platform causes an infinite redirect loop (AADSTS9002326) that is extremely difficult to diagnose.

| ⚠ Lesson | Always register SPAs as 'Single-page application' platform in Azure App Registration. The redirect URI must be exact — no trailing slash, no www. prefix, matching the exact origin including port number. |

### 2.3 RBAC and SharePoint Permissions

The production-grade permission system works in three layers:

Layer 1 — Ingestion block: restricted folders (finance, salary, hr, legal) are blocked before any content reaches Qdrant.

Layer 2 — SharePoint ACL filter: at ingestion, Graph API fetches the allowed_groups for each file and stores them in the Qdrant payload. At retrieval, user's JWT group GUIDs are matched against allowed_groups using Qdrant's MatchAny filter.

Layer 3 — RBAC folder filter: folder-path based access control as a secondary gate.

| 💡 Lesson | Azure AD Security Groups, Teams Groups, and SharePoint Groups all share the same GUID. When a user is added to a Teams channel, they get an Azure AD group GUID that flows through their JWT and can be matched against SharePoint file permissions — no custom mapping needed. |

## 3. RAG Pipeline Architecture — Lessons

### 3.1 The Token Overflow Crash

Production incident: 30% of queries returned completely blank answers with no error message. Root cause: Excel runbooks with 5,000-row sheets were being concatenated into a single LLM prompt reaching 337,460 tokens — exceeding GPT-4o-mini's 200,000 TPM limit. The OpenAI RateLimitError was caught but swallowed silently.

| 🔑 Fix | Two-layer context budget: (1) per-chunk cap of 12,000 chars in context_node, (2) total context cap of 55,000 chars before the prompt is assembled. Added specific RateLimitError handling that streams a user-facing message instead of silence. Silent failures are worse than noisy ones. |

#### Token Budget Management

| Limit                          | Value                          | Location                     |
|--------------------------------|--------------------------------|------------------------------|
| Per chunk max chars            | 12,000 chars (~3K tokens)     | context_node.py              |
| Total context max chars        | 55,000 chars (~14K tokens)    | context_node.py + generation_node.py |
| Max conversation history        | Last 6 messages                | generation_node.py           |
| LLM max_tokens                 | 800 (answers), 300 (critic)    | openai_service.py            |
| GPT-4o-mini TPM limit          | 200,000 tokens/minute          | OpenAI platform              |

### 3.2 Retrieval Quality — What Goes Wrong

#### Problem 1: Score Threshold Missing

Without a minimum score threshold, the system always returns 5 results regardless of relevance. This caused Lincoln Electric and Metrie runbooks to appear in answers about completely unrelated topics — they were large files with many chunks that partially matched on generic IT keywords like 'network' and 'configuration'.

| Fix | Added per-mode minimum score thresholds: filename_search=0.25, network_search=0.28, video_search=0.12, semantic_search=0.18. Chunks below threshold are dropped before top-5 selection. |

#### Problem 2: Wrong Retrieval Mode for Different Query Types

A single scoring formula (vector 50% + BM25 30% + filename boost 20%) is not optimal for all query types. Network config queries ('PDC routes') need heavy BM25 weighting because exact keyword matching is more reliable than semantic similarity for technical specs. Video timestamp queries need heavy vector weighting because the timestamp string differs from the chunk label.

| Query Type        | Vector Weight | BM25 Weight | File Boost | Min Score |
|-------------------|---------------|-------------|------------|-----------|
| filename_search    | 30%           | 50%         | 20%        | 0.25      |
| network_search     | 20%           | 55%         | 25%        | 0.28      |
| video_search       | 60%           | 20%         | 20%        | 0.12      |
| semantic_search    | 50%           | 30%         | 20%        | 0.18      |
| entity_search      | 30%           | 50%         | 20%        | 0.22      |

#### Problem 3: Timestamp Queries Not Finding Video Chunks

User asks 'what was at 15:00'. Video chunks are stored as '[00:14:00 → 00:16:00]\nTranscript text'. The timestamp 15:00 doesn't semantically match the chunk content. The retrieval found the right files but the generation prompt refused to answer because '15:00' wasn't explicitly in the text.

| Fix | Two fixes: (1) Video timestamp query expansion — generate queries for ±2 minute windows around the requested time. (2) Generation prompt rules 12-16 explicitly tell the LLM that a chunk labeled [00:14:00 → 00:16:00] CONTAINS the 15:00 mark. |

### 3.3 Hallucination Prevention

The system hallucinated in two distinct patterns:

#### Pattern 1: LLM invents generic commands when context is irrelevant

Query: 'update routes in PDC'. Retrieved: video transcripts (wrong files). The LLM had no real route data but generated generic 'route add 192.168.1.0 mask 255.255.255.0' commands from training data.

| Fix | Pre-generation relevance gate: if all retrieved sources are video transcripts AND the query is a network config question AND confidence < 0.35, return a 'not found' message instead of calling the LLM. Never let the LLM invent commands. |

#### Pattern 2: LLM copies template placeholders from SOP documents

Retrieved SOP document contained '[Actual Username]' and '[Actual Password]' as unfilled template fields. The LLM faithfully reproduced these, showing them to users.

| Fix | Post-generation placeholder stripper: catches [Actual X], [Your X], [Insert Here], [TBD] patterns and replaces with '*(not specified in document)*' plus a note explaining the document is a template. |

### 3.4 LangGraph Architecture Decisions

| Decision                                         | Rationale                                                        | Alternative Considered                       |
|--------------------------------------------------|------------------------------------------------------------------|---------------------------------------------|
| LangGraph over LangChain chains                   | Need shared state across 8 nodes, conditional routing, conversation memory | LangChain LCEL — linear only, no shared state |
| 8 nodes not 4                                   | Each node independently testable — isolation was critical for debugging | Monolithic generation function               |
| Frontend-carried conversation history             | Stateless backend — simpler horizontal scaling                    | LangGraph MemorySaver / RedisSaver          |
| Conflict resolution node between diagnostics and context | Date-rank conflicting docs AFTER reranking (best quality input) | Pre-retrieval conflict check                 |
| Critic node between generation and confidence     | Only fires on low-confidence — manages cost                      | Run critic on every answer                   |

## 4. Performance Optimisation — Lessons

### 4.1 The 8.4-Second Pipeline

Initial pipeline latency was 8.4 seconds average. The trace breakdown:

| Node        | Before | After  | Fix Applied                                      |
|-------------|--------|--------|--------------------------------------------------|
| generation  | 6.51s  | ~1.8s  | max_tokens=800, temperature=0.0, Redis answer cache |
| followup    | 1.05s  | <1ms   | Replaced LLM call with rule-based lookup table   |
| reranker     | 0.63s  | ~0.1s  | CrossEncoder cached with @lru_cache(maxsize=1)  |
| retrieval   | 0.11s  | 0.11s  | Already fast — no change                          |
| other nodes | 0.05s  | 0.05s  | Already fast — no change                          |
| TOTAL       | 8.4s   | ~2.8s  | 67% reduction                                    |

#### Reranker Model Loading

| Critical Bug | The CrossEncoder('ms-marco-MiniLM-L-6-v2') was instantiated inside the reranker_node function body — loading the model from disk on EVERY request at 0.63s each. Fix: @lru_cache(maxsize=1) on a getter function. Model loads once at first request, returned instantly on all subsequent requests. |

```python
@lru_cache(maxsize=1)
def _get_reranker():
    return CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')
```

#### Follow-up Node LLM Call

| Lesson | A second LLM call to generate follow-up chip suggestions (1.05s) is not worth the latency. Rule-based suggestion lookup by (query_intent, source_type) key is deterministic, instant, and can be more domain-specific than LLM-generated generic suggestions. Reserve LLM calls for tasks where creativity genuinely adds value. |

#### Answer Caching Strategy

Three layers of caching, each preventing a different expensive operation:

| Cache Layer                     | Redis DB | What is Cached                          | TTL  | Savings                     |
|----------------------------------|----------|----------------------------------------|------|-----------------------------|
| Query embedding cache            | db=3    | embed_query() result                   | 24h  | ~50ms per cached query      |
| Answer cache                     | db=3    | Full LLM answer keyed by prompt hash   | 1h   | ~2s per repeated query      |
| Reranker model cache             | In-process | CrossEncoder model object              | Process lifetime | 0.63s per request |

## 5. Multimodal Ingestion — Lessons

### 5.1 Excel Max-Grid Sheets

Several Excel runbooks had 'Index' sheets that appeared to contain 1,048,576 rows × 16,382 columns — Excel's maximum grid size. The xlsx extractor was iterating all cells, causing the Celery worker to hang for hours on a single file.

| Root Cause | openpyxl allocates the maximum grid even for blank sheets when opened in normal mode. max_row returns 1,048,576 even when the sheet has zero data rows. |
| Approach    | Result |
|-------------|--------|
| Check max_row before iter_rows | Failed — max_row itself is unreliable |
| Sample 10 rows with iter_rows(max_row=10) | Failed — iter_rows initialises full iterator even with max_row=10 |
| Direct cell access sheet.cell(row=r, col=c) | Partial fix — still slow |
| read_only=True on workbook load (final fix) | ✅ Solved — openpyxl streams rows lazily, never loads grid into memory |

```python
workbook = openpyxl.load_workbook(BytesIO(file_bytes), data_only=True, read_only=True)
```

### 5.2 Embedded Images in Office Files

Excel runbooks frequently contain diagrams pasted as images (trunk group tables, HLD/LLD diagrams, network screenshots). The xlsx extractor reads cell values only — images are completely invisible to it.

| Key Insight | Office files (xlsx, docx, pptx) are ZIP archives. Embedded images are stored at xl/media/image1.png, word/media/image1.png, ppt/media/image1.png. Extract the ZIP and OCR every image file — no additional libraries needed beyond PIL and pytesseract. |

OCR configuration for technical documents: PSM 6 (uniform block) for tables, PSM 11 (sparse text) for architecture diagrams. Images under 100×60px are skipped (logos, icons). Images under 800px wide are upscaled for better Tesseract accuracy.

### 5.3 Video Transcription

All MKV/MP4 files used Whisper for transcription since Teams-generated VTT files were not present in the demo SharePoint.

| Bug | Symptom | Fix |
|-----|---------|-----|
| moviepy v2.x import change | ImportError: cannot import from moviepy.editor | Change to: from moviepy import VideoFileClip |
| ContentBlock metadata= kwarg | TypeError: unexpected keyword argument 'metadata' | Use section_title= instead — match ContentBlock signature |
| ContentBlock section= kwarg | TypeError: unexpected keyword argument 'section' | Parameter is section_title, not section |
| Lesson | Before using any library, verify the API signature against the INSTALLED version, not documentation. moviepy v2.x removed the .editor submodule. Always run pip show <library> to check the installed version. |

### 5.4 PDF Section-Aware Chunking

Standard paragraph-level chunking destroyed section context. A chunk saying 'Set the T1 timer to 500ms' with no heading context was useless — the LLM had no idea it was from the SIP Profile Configuration chapter.

| Fix | Heading detection by font size (body modal size × 1.15 threshold) + bold short lines + numbered section patterns. All body text grouped under its parent heading. Every chunk prefixed with '## Section Name' for retrieval context. |

## 6. Production Incidents — Root Cause Analysis

| Incident                                   | Root Cause                                                                 | Impact                     | Fix                                                        |
|--------------------------------------------|-----------------------------------------------------------------------------|----------------------------|-----------------------------------------------------------|
| Credentials in chatbot answers             | Sanitizer used label-insert approach, not label+value-consume              | P0 Security                | 11-pattern sanitizer with context pre-sanitization        |
| 337K token crash → blank answers           | Excel sheets unlimited in context, no token budget                         | P1 — 30% blank answers     | 12K per-chunk + 55K total context caps                    |
| Filter.items() AttributeError              | retrieval_node passed Qdrant Filter object, repository expected dict       | P1 — all queries failed    | isinstance(filters, Filter) check                          |
| Celery worker hanging on XLSX              | openpyxl max-grid iteration — 1M rows loaded                              | P2 — ingestion stalled      | read_only=True workbook load                               |
| MKV files downloading instead of opening   | Source card used direct content URL, not SharePoint viewer URL            | P3 UX                      | Convert to /_layouts/15/stream.aspx?id= format            |
| Hallucination on network queries           | Video transcripts retrieved, LLM invented generic commands                 | P2 — wrong answers         | Pre-generation relevance gate + network_search mode       |
| Video timestamp queries returning nothing   | 15:00 didn't match chunk label [00:14:00 → 00:16:00]                     | P2 — video unusable       | Timestamp expansion + video-specific generation rules      |

## 7. What I Would Do Differently

### 7.1 Evaluation Suite on Day 1

The single biggest mistake was not building the RAGAS evaluation suite and golden test set before writing any pipeline code. Without measurable quality gates, every change was evaluated by intuition. The 337K token crash was discovered by a user complaint — it should have been caught by an automated evaluation run.

- Build 30 golden test questions with known correct answers in week 1
- Run RAGAS after every component change — faithfulness, context_recall, answer_relevancy
- Make failing the evaluation suite a hard blocker on deployment

### 7.2 Chunking Strategy Before Coding

Built the ingestion pipeline with a simple recursive character splitter, discovered weeks later that PDF answers were poor due to missing section context, rebuilt the PDF extractor from scratch. Two weeks of rework.

| Lesson | Before writing extraction code, manually open 5 representative files from each type and design the chunking strategy for that specific structure. PDFs with headings → section-aware. Excel runbooks → sheet-level. Training videos → 2-minute timestamp windows. Design first, code second. |

### 7.3 Azure OpenAI from Day 1

Started with the OpenAI API for simplicity. The 200K TPM rate limit became a real production issue. Azure OpenAI would have allowed provisioned capacity and avoided the token limit crash. For enterprise clients, Azure OpenAI also provides contractual data protection guarantees.

### 7.4 Context Sanitization Before LLM, Not After

The sanitizer was originally applied only to the LLM output. When it missed a pattern (like the 'For <device>: <credential>' format), credentials leaked to users. Adding context sanitization before the LLM call (Layer 1) means the LLM never sees credentials at all — so even if the output sanitizer fails, no damage occurs.

## 8. Interview Quick-Reference

### 8.1 Key Technical Decisions to Explain

| Question                                               | Your Answer                                                                                       |
|-------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| Why LangGraph not LangChain chains?                   | Shared AgentState TypedDict across all 8 nodes, conditional routing, each node independently testable |
| Why Qdrant not Azure AI Search?                       | On-premise data sovereignty, $0 infrastructure cost vs $250/mo, full control over hybrid retrieval |
| Why ESXi not Azure?                                   | $20/mo vs $100/mo = $960/year saving, data never leaves network, existing hardware               |
| How do you prevent hallucination?                     | Temperature=0, explicit 'only use context' prompt rule, pre-generation relevance gate, context sanitization |
| How did you handle the token limit crash?             | Per-chunk 12K char cap + 55K total context cap in context_node, specific error handling in generation_node |
| What is your RBAC approach?                           | Two layers: folder-keyword exclusion at ingestion + SharePoint ACL group GUIDs stored in Qdrant, matched at retrieval via JWT groups |

### 8.2 Production Numbers to Remember

| Metric                                   | Value                          |
|------------------------------------------|--------------------------------|
| Total pipeline latency (final)          | ~2.8 seconds                   |
| Latency reduction                        | 8.4s → 2.8s (67%)             |
| Chunks indexed                           | 3,004                          |
| Unique file types                        | 11                             |
| Video transcript chunks                  | 109                            |
| Monthly cost                            | ~$20                           |
| Security incidents resolved              | 5                              |
| Sanitizer patterns                       | 11                             |
| LangGraph nodes                          | 10 (including conflict + critic)|

### 8.3 STAR Stories for Behavioural Questions

**Tell me about a production incident you caused**

- **SITUATION:** During SNOW AI Automation development, deployed agent to production without DRY_RUN mode. 
- **TASK:** The agent auto-processed 200 incident tickets, resolving 47 incorrectly due to a confidence threshold bug. 
- **ACTION:** Shut down agent within 20 minutes, manually re-opened 47 tickets, wrote post-incident report same day. 
- **RESULT:** Added DRY_RUN mode, batch cap of 2, mandatory HITL for auto-resolve. Turned it into a process that now covers all future agentic deployments.

**Tell me about a security decision you made**

- **SITUATION:** Credentials appearing in chatbot answers despite sanitizer being deployed. 
- **TASK:** Runbook format 'For AZ-Prod-SBC-Act (UK): L3tm3l0g!n2@s!ms' — no standard sanitizer covers this. 
- **ACTION:** Added a custom heuristic pattern (must contain letter + special char), moved sanitization to BEFORE the LLM call so it never sees credentials. 
- **RESULT:** 13/13 test cases pass including all false positive guards. Zero credential exposure since deployment.
