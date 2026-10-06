# AI Solution Architect Interview Prep — RAG & Agents

Oct 6, 2026 ·

Answers are written in your voice, anchored to KnowledgeAI (SharePoint RAG), the Autonomous NOC platform, and the Azure stack, so every answer ends with a real example. Read the two design sections first; the Q&A sections reuse their vocabulary.

## How to answer any design question

Architect interviews are not testing whether you know LangChain. They test whether you can turn a vague business ask into a system with explicit trade-offs. Use the same five steps every time, out loud:

1. **Clarify the problem and the numbers.** Users, documents, queries per day, latency target, accuracy bar, data sensitivity, budget. Ask 3 to 4 questions before drawing anything. Interviewers mark this step heavily.
2. **Decide the simplest thing that could work.** Prompt-only, then RAG, then agents, then fine-tuning. Say why you stop where you stop.
3. **Draw the pipeline left to right.** Ingestion, retrieval or orchestration, generation, delivery. Name the component at each box and the alternative you rejected.
4. **Add the non-functional layer.** Security (identity, RBAC, data residency), evaluation, observability, cost, failure modes and fallbacks. This is what separates an engineer from an architect.
5. **Close with an example and a metric.** One sentence from a real project with a number: "On KnowledgeAI this design saved 8,750 hours a year for 100 plus users."

Two habits that score: say "it depends on" and then name the thing it depends on, and give a number whenever you can (chunk size, top-k, latency, cost per query).

## Designing a RAG system

RAG is an information retrieval problem first and an LLM problem second. Most production failures are retrieval failures. Design it as two pipelines that meet at the prompt.

**Pipeline 1: Ingestion (offline, batch or event-driven)**

| Stage | What you decide | Default choice and why |
| --- | --- | --- |
| Source connectors | Which systems, how to detect change, permissions capture | SharePoint via Microsoft Graph API; delta queries for change; store ACLs with every document |
| Parsing | PDF, DOCX, PPTX, HTML, tables, scanned images | Layout-aware parser (Azure Document Intelligence or Unstructured); OCR only when needed; tables kept as markdown |
| Chunking | Size, overlap, boundary strategy | 300 to 800 tokens, 10 to 15 percent overlap, split on headings and paragraphs, never mid-table; parent-child chunks for long docs |
| Enrichment | Metadata to filter on later | Title, section, author, date, department, ACL groups, doc type, source URL |
| Embedding | Model, dimension, cost, language | text-embedding-3-large (Azure OpenAI) or bge for on-prem; same model at query time, always |
| Index | Vector DB, hybrid support, filtering | Qdrant or Azure AI Search with vector plus BM25 and payload filters; one collection per tenant or ACL domain |

**Pipeline 2: Query (online, latency-bound)**

| Stage | What you decide | Default choice and why |
| --- | --- | --- |
| Query understanding | Rewrite, expand, route | LLM rewrite of follow-up questions using chat history; classify intent to pick the index |
| Security filter | Enforce who can see what | Resolve user's groups from Entra ID; filter at retrieval, never after generation |
| Hybrid retrieval | Semantic plus keyword | Vector top-20 plus BM25 top-20, fuse with Reciprocal Rank Fusion; keyword catches IDs, codes, names that embeddings miss |
| Reranking | Precision on the final set | Cross-encoder reranker (Cohere, bge-reranker) down to top-5; biggest single accuracy gain after hybrid |
| Prompt assembly | Context budget, citations | Numbered sources, instruction to answer only from context and say when it cannot; cap at 4 to 8 k tokens |
| Generation | Model, temperature, streaming | GPT-4o class for quality, mini models for simple intents; temperature 0 to 0.2; stream to the UI |
| Post-processing | Citations, guardrails, caching | Validate citations exist in context; PII and safety checks; semantic cache for repeated questions |

**The non-functional layer (say this unprompted)**

- **Evaluation:** a golden set of 100 to 300 question-answer pairs from real users; measure retrieval recall at k, answer faithfulness and relevance (RAGAS or custom LLM-as-judge); run on every change to chunking, embedding or prompt.
- **Observability:** log query, retrieved chunk IDs, scores, prompt, answer, latency, tokens per request; trace with LangSmith, Langfuse or Azure AI Foundry tracing. Without this you cannot debug a bad answer.
- **Freshness:** delta sync schedule by source; re-embed only changed chunks; version the index so a rollback is possible.
- **Cost:** cache embeddings and answers; route simple queries to small models; measure cost per query, not per month.
- **Failure modes:** no relevant chunks (say so, do not guess), stale index, permission leak, prompt injection inside documents, oversized tables chunked badly.

**Your example to close with:** KnowledgeAI. SharePoint via Graph API, hybrid retrieval with BM25 plus semantic plus reranking, Entra ID SSO and RBAC enforced at retrieval, citations with clickable SharePoint URLs, LangGraph orchestration, Qdrant, FastAPI. Result: 100 plus users, 8,750 plus hours a year saved.

## Designing an agentic system

Use an agent only when the path to the answer is not known in advance. If you can write the steps as a fixed pipeline, write the pipeline; it is cheaper, faster and testable. Agents earn their cost when the system must choose tools, loop until a condition is met, or recover from partial failure.

**Decision ladder (say it in this order)**

1. Single prompt with structured output. Enough for classification, extraction, summarisation.
2. Fixed workflow (chain). Steps known, order known; LLM calls inside deterministic code.
3. Single agent with tools. One LLM loop deciding which tool to call next; bounded by max steps.
4. Multi-agent. Specialised agents (planner, researcher, executor, reviewer) with a supervisor; use only when one context window or one role cannot hold the job.

**Reference architecture**

| Layer | What it does | Design decisions |
| --- | --- | --- |
| Orchestrator | Holds state, decides next step, enforces limits | LangGraph state graph (explicit nodes and edges, checkpointing, resumable) over free-form ReAct loops; max iterations, timeouts, budget per run |
| Tools | The agent's hands: APIs, databases, search, code | Expose through Model Context Protocol (MCP) servers so tools are reusable across agents and clients; strict JSON schemas; idempotent where possible |
| Memory | Short-term (the run), long-term (across runs) | Run state in the graph checkpoint (Redis or Postgres); long-term facts in a vector or key-value store with explicit write policy; never let the agent write unbounded memory |
| Planning | How work is decomposed | Plan-and-execute for long tasks (plan once, execute steps, re-plan on failure); ReAct for short tool loops |
| Guardrails | What the agent may not do | Allow-list of tools per agent; read-only by default; Human-in-the-Loop approval node before any write or production change; input and output content filters |
| Evaluation | Did it do the job | Task success rate on a scenario set, tool-call accuracy, steps per task, cost per task; replay traces from production |
| Observability | Why it did that | Every step logged with inputs, tool outputs, decisions; trace per run (LangSmith, Langfuse, Azure AI Foundry); alert on loops and budget breaches |
| Delivery | How it reaches users | API (FastAPI), Teams or Slack bot, scheduled trigger; async with status callbacks for long runs |

**Control points interviewers probe**

- **Loops and runaway cost:** max steps, token budget per run, a kill switch, and a "give up and escalate" node.
- **Non-determinism:** temperature 0, structured outputs, deterministic tools, replayable traces; test with scenario suites, not single prompts.
- **Safety of actions:** separate read tools from write tools; write tools go through an approval node; log who approved.
- **Prompt injection via tool results:** treat every tool output and document as untrusted data; never let it change the system prompt or tool permissions.
- **State and recovery:** checkpoint after each node so a crashed run resumes rather than restarts; idempotency keys on external calls.
- **Multi-agent coordination:** supervisor routes, workers do not talk to each other directly; shared state is explicit, not implicit.

**Your example to close with:** Autonomous NOC. An incident-validation agent scans the ServiceNow queue every 3 minutes, classifies, pulls device health through tools, proposes a remediation (tunnel bounce, interface reset), and routes it through a Human-in-the-Loop approval node before execution. LangGraph state graph with MCP tool servers, every run traced. Result: 50 to 70 percent less human effort, 30 to 40 percent lower MTTR, 90 percent fewer proactive tickets.

## RAG questions and answers

**1. Walk me through how you would design a RAG system for 50,000 internal documents and 2,000 users.** Start with questions: document types, update frequency, permission model, latency target, accuracy bar. Then two pipelines. Ingestion: connectors with change detection, layout-aware parsing, 300 to 800 token chunks with overlap, metadata including ACLs, embeddings into a hybrid index. Query: rewrite the question with history, resolve the user's groups and filter at retrieval, hybrid search with RRF, rerank to top-5, generate with citations, validate citations, cache. Then evaluation on a golden set, tracing, delta sync, cost per query. Close: "This is the KnowledgeAI design; 100 plus users, 8,750 hours a year saved."

**2. Why hybrid retrieval instead of pure vector search?** Embeddings capture meaning but lose exact tokens: product codes, ticket numbers, names, acronyms. BM25 catches those; vectors catch paraphrase. Fusing both with Reciprocal Rank Fusion and then reranking gave us the largest accuracy jump on KnowledgeAI, especially for SOP lookups where users type IDs.

**3. How do you choose chunk size?** It depends on the question type and the model's context budget. Short factual questions want small chunks (200 to 400 tokens) for precision; explanatory questions want larger ones (600 to 1,000) for coherence. I start around 500 with 10 to 15 percent overlap, split on document structure, keep tables whole, and tune on the golden set by measuring recall at k. Parent-child chunking gives both: retrieve on small, return the parent.

**4. How do you evaluate a RAG system?** Separate retrieval from generation. Retrieval: recall at k and MRR against labelled relevant chunks. Generation: faithfulness (is the answer supported by context), answer relevance, and citation accuracy, scored by an LLM judge calibrated against a human-labelled sample. A golden set of 100 to 300 real questions, run in CI on every change. In production: thumbs up and down, "no answer" rate, and sampled human review.

**5. How do you handle permissions so a user never sees a document they should not?** Capture ACLs at ingestion, store them as metadata on every chunk, resolve the user's identity and groups through Entra ID at query time, and apply the filter inside the retrieval query. Never filter after generation; the model has already seen the text. Re-sync ACLs on a schedule and on change events. Test with a permission test suite of users and documents.

**6. The model hallucinates even with RAG. What do you do?** First check retrieval: in most cases the right chunk was not retrieved or was ranked low. Fix with hybrid search, reranking, better chunking. Then the prompt: instruct the model to answer only from context and to say "not found" otherwise; lower temperature; require citations and validate them programmatically. Then the model: a stronger model for low-confidence cases. Measure faithfulness before and after each change.

**7. How do you keep the index fresh?** Change detection per source (Graph API delta queries, database change data capture, file hashes), re-embed only changed chunks, soft-delete removed documents, and version the index so a bad ingestion can be rolled back. Freshness SLA defined per source: policies hourly, archives weekly.

**8. When would you fine-tune instead of using RAG?** RAG for knowledge that changes or must be cited. Fine-tuning for style, format, domain vocabulary, or latency when the same behaviour repeats and knowledge is stable. Often both: a fine-tuned small model as the generator inside a RAG pipeline. Fine-tuning does not fix hallucination about facts; retrieval does.

**9. How would you reduce cost per query?** Semantic cache for repeated questions, route simple intents to a small model, cap context tokens through reranking rather than stuffing top-20, batch embeddings, use cheaper embedding models where recall allows, and measure cost per query on a dashboard so the team sees it.

**10. What are the biggest mistakes you have seen in RAG projects?** Skipping evaluation and tuning on vibes, chunking by fixed characters through tables, embedding with one model and querying with another, applying permissions after generation, no tracing so bad answers cannot be debugged, and treating documents as trusted input to the prompt.

## Agentic AI questions and answers

**1. What is an AI agent, and when is it the wrong choice?** An LLM in a loop that observes state, chooses an action from a set of tools, executes it, and repeats until a goal or a limit is reached. It is the wrong choice when the steps are known in advance: a fixed chain is cheaper, faster and testable. I use the ladder: prompt, workflow, single agent, multi-agent, and stop at the lowest rung that solves the problem.

**2. How do you design a multi-agent system? When does it beat a single agent?** A supervisor agent routes work to specialised workers (planner, retriever, executor, reviewer) that share explicit state. It beats a single agent when one role's tools or context would overload a single prompt, or when roles need different permissions or models. Workers never talk to each other directly; the supervisor owns the flow so it can be traced and bounded.

**3. What is Model Context Protocol and why does it matter architecturally?** MCP is an open standard for exposing tools, resources and prompts to LLM applications through a server with typed schemas. Architecturally it decouples tools from agents: one ServiceNow MCP server serves every agent and every client (LangGraph, Claude, Copilot) without rewriting integrations. It also gives a single place to enforce permissions and logging on tool calls.

**4. How do you stop an agent from looping or running up cost?** Hard limits in the orchestrator: max steps, token budget per run, wall-clock timeout, and a terminal "escalate to human" node. Detect repeated identical tool calls and break. Alert on budget breaches. In LangGraph these are graph edges and a recursion limit, not hopes in a prompt.

**5. How do you make agents safe to act on production systems?** Separate read tools from write tools. Read by default. Every write passes through a Human-in-the-Loop approval node that shows the proposed action, evidence and blast radius; the approver's identity is logged. Allow-list tools per agent; idempotency keys on external calls; a dry-run mode. On the NOC platform, remediation (tunnel bounce, interface reset) only executes after approval.

**6. How do you test and evaluate agents?** Scenario suites: 50 to 200 realistic tasks with expected outcomes, run on every change. Metrics: task success rate, tool-call accuracy, steps per task, cost per task, latency, escalation rate. Replay production traces as regression tests. Temperature 0 and structured outputs for repeatability. Red-team scenarios for prompt injection and unsafe actions.

**7. How do you handle memory in agents?** Three kinds. Run state: the graph checkpoint, persisted (Redis or Postgres) so a run resumes after a crash. Conversation memory: summarised history within a token budget. Long-term memory: explicit facts written by a controlled step into a vector or key-value store, with a retention and write policy; never let the agent append freely.

**8. LangGraph versus CrewAI versus AutoGen versus Azure AI Agent Service: how do you pick?** LangGraph when I need explicit control flow, checkpointing and production reliability; it is my default. CrewAI for quick role-based prototypes. AutoGen for conversational multi-agent research patterns. Azure AI Agent Service or Semantic Kernel when the enterprise is standardised on Azure and wants managed identity, tracing and governance built in. The decision is about control, observability and the team's platform, not features.

**9. How do you defend against prompt injection in an agent?** Treat every tool result, document and web page as untrusted data. Keep system instructions and tool permissions outside the model's reach. Validate tool arguments against schemas. Use content filters on inputs and outputs. Require approval for writes. Log and alert on instruction-like content in tool outputs.

**10. Describe an agent you built in production and what went wrong the first time.** The NOC incident-validation agent. First version used a free-form ReAct loop; it occasionally re-ran the same health check ten times and once proposed a remediation on the wrong device because a tool result was ambiguous. Fix: moved to a LangGraph state graph with explicit nodes, a step limit, structured tool outputs with device IDs validated, and an approval node before any action. Success rate and trust went up; proactive tickets fell 90 percent.

## Architecture, security, cost and operations

**1. How do you decide build versus buy for an enterprise AI capability?** Buy (Copilot, Azure AI Search, managed agent services) when the use case is generic, data is already in that platform, and differentiation is low. Build when the workflow is core to the business, needs custom tools or permissions, or the vendor's data handling fails compliance. Usually a hybrid: managed models and identity, custom orchestration and tools. State the total cost over three years, not the licence price.

**2. How do you choose between Azure OpenAI, OpenAI direct, Anthropic and open-source models?** By data residency and compliance first (Azure OpenAI keeps data in-tenant with enterprise agreements), then by task quality on our own eval set, then by cost and latency. Design the orchestration to be model-agnostic with an abstraction layer so a model swap is a config change. Open-source (Llama, Mistral, Qwen) when data cannot leave the network or volume makes per-token pricing unviable.

**3. What does a reference architecture for enterprise GenAI look like on Azure?** Entra ID for identity; API Management as the gateway with rate limits and key management; FastAPI orchestration on AKS or Container Apps; Azure OpenAI for models; Azure AI Search or Qdrant for retrieval; Azure AI Foundry for agent service, evaluation and tracing; Key Vault for secrets; Application Insights and Log Analytics for observability; GitHub Actions for CI/CD with evaluation gates. Private endpoints and no public model access.

**4. How do you secure an LLM application?** Identity on every request, least-privilege tool access, retrieval-time permission filters, input and output content filtering, prompt-injection defences, secrets outside prompts, PII redaction where required, audit logs of prompts and outputs, data retention policy, and a red-team exercise before go-live. Map it to the OWASP Top 10 for LLM applications when the interviewer wants a framework.

**5. How do you take a GenAI prototype to production?** Define the success metric and the eval set first. Harden the pipeline: structured outputs, retries, timeouts, fallbacks to a smaller model or a canned response. Add tracing and cost tracking. Containerise, CI/CD with an evaluation gate that blocks regressions. Pilot with a user group, collect feedback, then scale. Most prototypes fail at evaluation and permissions, not at the model.

**6. How do you control and forecast cost?** Measure cost per request by component (embedding, retrieval, generation). Levers: caching, model routing by intent, context trimming via reranking, batching offline work, provisioned throughput for steady load, and budgets with alerts. Forecast from requests per day times average tokens; show the finance team a dashboard, not an estimate.

**7. What is LLMOps and what does it include?** The operational discipline for LLM applications: prompt and model versioning, evaluation datasets and automated evals in CI, tracing and monitoring in production, feedback loops from users, cost and latency dashboards, drift detection when documents or models change, and incident runbooks for bad outputs.

**8. How do you handle latency for a chat application?** Stream tokens to the UI, parallelise retrieval and query rewriting, keep reranking to a small candidate set, cache embeddings and frequent answers, use smaller models for simple intents, and set a latency budget per stage (for example 300 ms retrieval, 2 s first token). Measure p95, not average.

**9. How do you explain AI risk to a CIO?** Four risks in plain words: wrong answers (hallucination), data leakage (permissions and vendors), runaway cost, and unsafe actions by agents. For each, the control we have and the metric we watch. Then the business number the system delivers so the trade-off is clear.

**10. What would you do in your first 90 days as AI Solution Architect here?** Days 1 to 30: inventory use cases, data, platforms, security posture; meet stakeholders; pick one high-value, low-risk use case. Days 31 to 60: reference architecture, evaluation framework, governance guardrails; deliver the first pilot. Days 61 to 90: production path for the pilot, a reusable platform (identity, gateway, retrieval, tracing), and a roadmap of the next three use cases with cost and value.

## Behavioural and leadership questions

Answer with Situation, Action, Result in under 90 seconds, and end on a number. Keep these four stories ready; they cover most questions.

| Question they ask | Story to use | The number to end on |
| --- | --- | --- |
| Tell me about a complex system you architected | Autonomous NOC: ServiceNow data, agentic validation, HITL remediation, LangGraph and MCP | 50 to 70 percent less effort, 30 to 40 percent lower MTTR |
| A project where the first design was wrong | NOC agent v1 free-form loop; re-architected to a state graph with limits and approval | Proactive tickets down 90 percent |
| How you drove adoption of AI with non-technical users | KnowledgeAI: citations with clickable links, SSO so no new login, workshops for business teams | 100 plus users, 8,750 hours a year |
| A time you pushed back on a stakeholder | A request to skip permission filtering to launch faster; showed a leak scenario, delivered RBAC at retrieval in the same sprint | Zero permission incidents |

**How do you convince a team to adopt a design they did not choose?** Show the evaluation numbers side by side, run a short pilot on their use case, and give them ownership of one component. Data and a shared win beat authority.

**How do you keep up with a field that changes monthly?** A fixed reading routine, a personal lab where I rebuild one new technique a month on a real dataset, certifications when they force depth (the Azure AI Apps and Agents cert this year), and teaching: explaining it to others is the fastest test of whether I understand it.

**How do you handle a leadership ask for "an AI strategy" with no clear problem?** Turn it into three candidate use cases ranked by value, data readiness and risk; propose one pilot with a success metric and a 60-day timeline. A strategy without a first delivery is a slide deck.

**What do you do when a model's output cannot be fully trusted?** Design the human into the loop where the cost of error is high, measure the error rate, and expand automation only as the measured rate earns it. That is exactly how the NOC remediation went from approval-only to selective auto-execution.

## Whiteboard drills

Practise each aloud in 15 minutes: 3 minutes of questions, 7 minutes of drawing, 5 minutes on non-functionals. Record yourself once.

**Drill 1: Customer support assistant for a bank (RAG).** 20,000 policy and product documents, 5,000 agents, answers must cite policy, no customer PII in the model, 2 second first token. Expect to be probed on permission filtering, citation validation, PII redaction before embedding, and what happens when no policy matches.

**Drill 2: IT operations auto-remediation (agent).** Incoming alerts from monitoring, runbooks in Confluence, actions through ServiceNow and network APIs. Expect to be probed on read versus write tools, approval thresholds by blast radius, loop limits, replayable traces, and how you prove it is safer than the human process. This is your NOC story; make the diagram crisp.

**Drill 3: Multi-agent research assistant for analysts.** Pulls from internal reports, the web and a data warehouse, produces a cited brief. Expect to be probed on supervisor and worker roles, shared state, cost per brief, how you evaluate a long-form output, and prompt injection from web pages.

**Checklist before you stop talking**

- [ ] Asked about users, volume, latency, accuracy bar, sensitivity, budget
- [ ] Named the simplest option and why it is or is not enough
- [ ] Drew ingestion or orchestration, retrieval or tools, generation, delivery
- [ ] Covered identity and permissions, evaluation, observability, cost, failure modes
- [ ] Closed with a real project and a number


**Are langchain can create agents ?**

Yes — and the interviewer is probably testing whether you know the *nuance*, because the answer changed over the last two years.

**The short answer to give:**

"Yes. LangChain has always been able to create agents, but how has changed. Originally it was `initialize_agent` / `AgentExecutor` with patterns like ReAct and OpenAI-functions agents — a loop where the LLM picks a tool, runs it, and repeats. Those legacy agent classes are now deprecated. Since LangChain 1.0 (late 2025) the standard way is `create_agent()`, which gives you a tool-calling agent in a few lines — and under the hood it's built on LangGraph. So today: LangChain for a quick, standard tool-calling agent; LangGraph directly when I need custom control flow — branching, checkpointing, human-in-the-loop nodes, multi-agent supervisors. In production I use LangGraph because I want the explicit state graph and limits."

**The three things that answer signals:**
1. You know the history (AgentExecutor → deprecated).
2. You know the current API (`create_agent`, LangGraph underneath).
3. You know *when to use which* — that's the architect part.

**If they follow up with "then why do you need LangGraph at all?":** LangChain's `create_agent` is a pre-built loop — one agent, one set of tools, LLM decides next step. LangGraph is the engine beneath it: you draw the graph yourself, so you can add an approval node before a write, a loop limit, a checkpoint to resume a crashed run, or route between multiple agents. Same ecosystem, different level of control.

One caution: the exact `create_agent` naming is from LangChain 1.0 — verify the current signature on the docs before an interview, since the API moved fast in 2025. If the interviewer is on an older codebase they may still say `AgentExecutor`; just acknowledge both.
