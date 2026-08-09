## Q7. How do you enforce agent autonomy boundaries?

What they're testing: Do you enforce safety in code or just in prompts? Can you classify actions by risk level?

Your Answer:

"I enforce autonomy boundaries through four explicit layers in code — nothing is controlled by prompting alone, because the LLM can always be convinced to override a prompt instruction.

Layer 1 — Action classification by risk level. Every action in the pipeline is pre-classified before the agent runs:

| Action                          | Risk Level        | Gate                                         |
|---------------------------------|-------------------|---------------------------------------------|
| Poll SNOW tickets               | Read-only         | Always allowed                              |
| RESTCONF query to device        | Read-only         | Always allowed                              |
| Write WorkNote                  | Reversible write   | Requires confidence ≥80% + content validation |
| PATCH category/subcategory       | Reversible write   | Same gates + empty-field-only rule         |
| Change ticket state             | Irreversible write | Blocked entirely — HITL required            |
| Resolve ticket                  | Irreversible write | Blocked entirely in current phase           |
| Device SSH login                | External action    | Disabled until classification proven stable  |

Layer 2 — DRY_RUN mode as a hard switch. Default is DRY_RUN=true. The pipeline runs fully — polls, classifies, validates — but the final write step checks this flag before touching SNOW. It cannot be bypassed by the LLM. It's an OS environment variable read at runtime, not a prompt instruction.

Layer 3 — Content validation gate. Before any PATCH, `validate_description_match()` runs. It checks that the description text actually contains keywords matching the classification. If you classify as 'Ping Loss' but the description has no 'ping' or 'icmp' keyword — blocked. This is what stopped Network tickets from being classified as SBC after the safety incident. The LLM cannot override this because it runs after the LLM has already finished.

Layer 4 — Confidence threshold. Below 80% confidence, no write happens regardless of what the LLM returned. `issue_type=Others` is always blocked — it means the system couldn't classify with certainty.

I learned these layers the hard way. In the first production run without DRY_RUN, all four were missing. The pipeline modified 50+ tickets incorrectly in one cycle. That incident drove the entire safety architecture rebuild."

## Q9. What is your observability strategy?

What they're testing: Can you debug a production AI system? Do you know what to log beyond just errors?

Your Answer:

"Observability for an agentic AI system is more complex than a standard API because you need to trace decisions, not just errors. I built observability at four levels:

Level 1 — Structured logging with `structlog`. Every log entry is JSON with consistent fields: `ticket_id`, `sys_id`, `event`, `timestamp`, `level`. This means I can grep any ticket's complete journey across all agents using just its INC number. Every classification decision logs: what layer matched, what event source was found, what confidence was returned, why it was skipped. Example:

```json
{"ticket_id": "INC0716077", "eventsource": "PathChkPingState - Down",
 "issue_type": "Ping Loss", "confidence": 1.0, "method": "layer1_fingerprint",
 "event": "classifier.layer1_match"}
```

Level 2 — LangSmith for LLM call tracing. Every GPT-4o call is traced end-to-end: the full prompt sent, the raw response received, token count, latency. When a ticket is misclassified, I open LangSmith and see exactly what context the LLM had and what it returned. Without this, debugging a Layer 2 classification failure is impossible — you're guessing at what the model saw.

Level 3 — Log files with rotation. `LOG_FILE=logs/snow_ai.log` configured in `.env`. `RotatingFileHandler` at 10MB per file, 5 backups retained. Console output goes to the terminal during development. File output captures the full session for post-run review. This was critical after the production incident — I had no logs to review because the session was terminated and console output was lost. Log files fixed that permanently.

Level 4 — Skip reason tracking. Every skipped ticket logs a specific reason: `job.skip_others`, `job.skip_low_confidence`, `job.skip_content_mismatch`. At the end of every cycle, the summary log shows: total fetched, processed, skipped by reason. This tells me at a glance whether the classifier is working or whether the safety gates are over-filtering.

What I'm building next is Prometheus metrics for: tickets classified per hour, confidence score distribution, Layer 1 vs Layer 2 hit rate, and skip reason breakdown. Combined with a Grafana dashboard, an on-call engineer can see system health without reading logs."

## Q12. How do you evaluate your RAG system's quality?

What they're testing: Do you know the difference between offline and online evaluation? Have you actually measured anything?

Your Answer:

"I evaluate at three levels — offline accuracy, online skip rate monitoring, and engineer override tracking.

Offline evaluation — golden dataset. I built a golden dataset of 50 tickets with known correct classifications from our historical Zabbix and LogicMonitor data. For each ticket, I know the correct: category, subcategory, issue_type, and whether it should be classified at all. I run the full pipeline against this dataset in dry-run mode and measure:

- Classification accuracy — how many issue_types match the ground truth
- False positive rate — tickets the system classified but shouldn't have (non-SBC content)
- False negative rate — SBC tickets incorrectly skipped due to low confidence
- Layer 1 hit rate — what percentage matched fingerprints (target: 85%+)

For RAG specifically — using DeepEval, I measure faithfulness (does the classification match the retrieved similar tickets?) and context relevance (did the retriever return actually similar historical tickets?).

Online monitoring — skip reason distribution. In production, every cycle logs a summary: 2 fetched, 0 processed, 2 skipped — with specific reasons. If I see 100% skipped with reason low_confidence, the RAG retriever is failing — it's not finding similar enough historical tickets to boost confidence. If I see content_mismatch skips increasing, the fingerprints may be drifting from actual descriptions.

Engineer override tracking. When an engineer looks at a WorkNote written by the AI and changes the classification, that's a ground truth signal. I track how often engineers override AI classifications by comparing SNOW field values before and after AI processing. High override rate on a specific issue_type means that fingerprint or LLM prompt needs tuning.

The honest answer on current state: My RAG is partially operational — ChromaDB vector store is built but the fingerprint loader hasn't been run yet to populate it. Layer 1 deterministic matching is what's carrying 100% of current classifications. Layer 2 LLM is being called for Zabbix tickets without Eventsource, but until ChromaDB is populated with historical fingerprints, the RAG retrieval step is a no-op and the LLM classifies from the description alone. That's why I see `subcategory=Others` coming back — the LLM doesn't have enough domain context without the RAG backing it."

## Q13. A pipeline incident modified 50 production tickets wrongly. How did you handle it?

What they're testing: Crisis management, root cause analysis discipline, what you learned, and whether you take ownership.

Your Answer — use STAR format throughout:

**Situation:** "During the first production run of the SNOW AI Automation pipeline, I terminated the session after discovering it had modified 50+ SNOW tickets incorrectly — including Network category tickets being reclassified as SBC, and some tickets being auto-resolved with 10% confidence. This was on the development SNOW instance, not production, but it was still a serious incident with real tickets being corrupted."

**Task:** "My immediate priority was: stop further damage, understand every ticket that was touched, identify all root causes, and rebuild with proper controls before any further run."

**Action** — 6 root causes identified and fixed:

"I did a structured root cause analysis. I found 6 independent failures, each of which alone would have caused the incident:

- **RC1** — No dry-run mode. The pipeline wrote to SNOW immediately on first run with no preview capability. Fix: `DRY_RUN=true` as the default. Cannot write to SNOW without explicitly setting it to false.
- **RC2** — Batch size setting ignored. I had set `SNOW_POLL_BATCH_SIZE=2` in `.env` but Pydantic Settings uses `@lru_cache`. The settings object was already loaded with the old value of 50. Fix: clear the `lru_cache` after `.env` changes, and verify at startup by printing the loaded value.
- **RC3** — Redis dedup fail-open. Redis was not running. `is_already_processed()` returned False for every ticket because it couldn't connect. All 50 tickets were processed every cycle. Fix: dedup failure now logs a warning but uses in-memory set tracking within the cycle as fallback.
- **RC4** — No category whitelist. Network tickets had empty `u_issue_type` field so they passed the 'needs classification' check. No validation that the ticket's content was actually SBC-related. Fix: `validate_description_match()` function — description must contain SBC/SIP/TLS/ping keywords, and non-SBC hostname patterns (CSW, switch, Linux server) are explicitly rejected.
- **RC5** — No confidence threshold. 10% confidence classifications were being written to SNOW. Fix: gate at 80% minimum. Anything below is logged as skipped with reason.
- **RC6** — Auto-resolve enabled. For tickets where RESTCONF returned `pingState=UP`, `resolve_ticket_auto()` was called immediately. Fix: state field is never touched in the current phase. Resolve is completely disabled until classification is fully validated over multiple weeks.

**Result:** "The rebuild took two days. Every subsequent run has been `DRY_RUN=true` until I manually review the logs and confirm the classification is correct. No ticket has been incorrectly modified since. The incident was actually valuable — it forced me to build a safety architecture that I should have built first. The six safety gates are now the foundation of the system, not an afterthought."

**Key message to close with:** "I document incidents with a post-mortem. The rule I follow: never add a safety gate after an incident without understanding exactly which root cause it closes. Generic 'be more careful' fixes don't prevent recurrence. Specific code-level gates do."

## Q14. How would you scale this system to handle 200 tickets simultaneously?

What they're testing: Production systems thinking — distributed architecture, concurrency, cost, failure modes.

Your Answer:

"Current architecture: single APScheduler process, sequential ticket processing, one SNOW API call at a time. That works for 200 tickets per day with a 3-minute poll interval. For 200 tickets simultaneously — meaning burst processing — I'd evolve in three stages.

**Stage 1** — Parallelise within the current process. Replace sequential `for ticket in ticket_states` with `ThreadPoolExecutor`. SNOW API calls are I/O-bound — 5 parallel threads would reduce a 10-ticket batch from 50 seconds to ~10 seconds. Redis dedup still works with thread-safe operations. This costs nothing and handles 10x throughput.

```python
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = {executor.submit(process_ticket, state): state 
               for state in ticket_states}
```

**Stage 2** — Celery distributed workers. Replace APScheduler with a Celery task queue backed by Redis. Each ticket becomes an independent task. Worker pool scales horizontally — add more workers on more machines without changing the core processing logic. The SNOW poll job enqueues one Celery task per ticket. Workers process in parallel with no shared state.

Key design decision: the LangGraph state per ticket is stateless between runs — it's created fresh from the SNOW ticket data each time. So workers share nothing — no concurrency bugs.

**Stage 3** — LLM cost and rate limit management at scale. At 200 simultaneous tickets hitting GPT-4o Layer 2, you hit OpenAI rate limits immediately. Three mitigations:

- Redis cache: cache LLM classification result by event source string with 1-hour TTL. Same `SIPTLSClientHandshakeFailure` in 20 tickets → 1 LLM call, 19 cache hits.
- Batch LLM calls: group similar ticket descriptions and classify in one prompt using OpenAI batch API — 50% cost reduction.
- Model tiering: Layer 1 fingerprints cover 85% of tickets with zero LLM cost. Only 15% hit GPT-4o. At 200 simultaneous tickets, that's 30 LLM calls — well within rate limits.

Failure handling at scale: At scale, SNOW API throttles at ~60 requests per minute. Use exponential backoff with jitter on all SNOW calls. Implement a circuit breaker — if SNOW returns 429 three times in a row, pause the worker pool for 60 seconds rather than hammering the API.

The number I'd give them: Current architecture: ~10 tickets/minute. With Stage 1 threading: ~50 tickets/minute. With Stage 2 Celery (10 workers): ~200 tickets/minute. For a real-world NOC with 200 alerts/day spread across 24 hours, even Stage 1 is more than sufficient. Stage 2 becomes relevant if you're doing burst processing — end-of-shift backlogs or multi-customer environments."

One tip for the interview: For every answer, anchor to something that actually happened in your system. The incident story, the Zabbix parsing discovery, the two Sonus subcategory problem — these real details are what make your answers stand out from candidates who only know textbook architecture. Interviewers at TCS senior level know immediately when someone is describing theory versus something they actually debugged at 2am.