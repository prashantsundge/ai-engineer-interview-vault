
### AI-driven ServiceNow ticket classification, SSH diagnostics, and remediation pipeline
Built with LangGraph · LangChain · GPT-4o · ChromaDB · Redis · PostgreSQL · Netmiko

1. Project Overview
2. Architecture Diagram
3. Technology Stack
4. Full Ticket Lifecycle
5. Classification Strategy
6. Agent Reference
7. File-by-File Reference
8. Configuration Reference
9. SNOW API Integration
10. SSH Diagnostic & Remediation Flow
11. HITL — Human in the Loop
12. Verification & Auto-Closure
13. Cost Controls
14. Database Schema
15. Setup & Installation
16. Environment Variables
17. Adding a New Issue Type
18. Phase 1 Scope & Roadmap

### Project Overview

The NOC team receives ~200 ServiceNow tickets per day from LogicMonitor and Zabbix monitoring tools, plus user-submitted tickets. Every ticket arrives with:
- Category → empty
- Subcategory → empty
- Issue Type → empty
- Contact Type → empty
- Issue Trigger Source → empty

Engineers manually read each description, identify the issue, and fill these fields — then begin troubleshooting. This is repetitive, slow, and error-prone.

#### Issue Type Classification

| Issue Type                     | Classification | SSH Diagnostic | Remediation               |
|--------------------------------|----------------|----------------|---------------------------|
| TLS Handshake Failure          | ✅              | ✅              | ✅ (HITL required)        |
| IP Peer Path Down/Flap        | ✅              | ✅              | ✅ (HITL required)        |
| Ping Loss                      | ✅              | ✅              | ✅ (HITL required)        |
| All other issue types          | ✅              | ❌              | ❌                        |

All 25 issue types in the SNOW taxonomy are classified. Only the 3 above get SSH diagnostics and remediation in Phase 1.

### Architecture Diagram

```text
        SNOW In-Progress Ticket
                │
                ▼
        AI reads description
                │
                ├─ Layer 1: Eventsource fingerprint match (85% of tickets, zero LLM cost)
                │
                └─ Layer 2: GPT-4o + ChromaDB RAG (15% ambiguous/user tickets)
                        │
                        ▼
        Fills 5 SNOW fields (empty fields only — no overwrite)
        Writes structured WorkNote
                │
                ▼
        SSH into device — run read-only diagnostic commands
        Parse output → Issue confirmed? YES / NO
                │
                ├─ NO  → "No active issue" WorkNote → Close ticket
                │
                └─ YES → Human approval required (HITL)
                                │
                                ├─ Approved → Run remediation commands
                                │
                                └─ Rejected → Escalate to engineer
                                        │
                                        ▼
                        Verification: T+15 → T+45
                        All GREEN → Auto-close
                        Any RED   → Escalate to engineer
```

### Technology Stack

| Component        | Technology                | Purpose                                      |
|------------------|--------------------------|----------------------------------------------|
| Orchestration     | LangGraph (StateGraph)   | Workflow-first agentic pipeline              |
| AGENT 1: Poller Agent | -                    | -                                            |
| -                  | GET /api/now/table/incident (state=2, your team's group) | - |
| -                  | Redis dedup check (sys_id TTL 24h) | - |
| -                  | Skip if all 5 fields already populated | - |
| -                  | Returns batch of TicketState objects | - |
| LangGraph StateGraph — per-ticket pipeline | - | - |
| AGENT 2: Classifier Agent | - | - |
| -                  | Layer 1: fingerprints.yaml exact match (conf=1.0) | - |
| -                  | Layer 2: ChromaDB → GPT-4o (conf=0-1) | - |
| -                  | Extract entities (host, IP, port, peer, zone...) | - |
| -                  | Derive Contact Type + Issue Trigger Source | - |
| -                  | Validate against category_map.yaml taxonomy | - |
| AGENT 3: SNOW Updater Agent | - | - |
| -                  | Build PATCH payload (empty fields only) | - |
| -                  | PATCH /api/now/table/incident/{sys_id} | - |
| -                  | Write classification WorkNote | - |
| -                  | Write PostgreSQL audit record | - |
| AGENT 4: Remediation Router + Sub-Agents | - | - |
| -                  | Phase 1 issue types: | - |
| -                  | TLS Agent | tls_agent.py |
| -                  | Ping/IP Peer Agent | ping_loss_agent.py |
| -                  | All other types → WorkNote "Manual review" → SKIP | - |
| -                  | SSH Flow (via ssh_tools.py + Netmiko): | - |
| -                  | Layer 1: Read-only show/get commands | - |
| -                  | → No issue → Close ticket | - |
| -                  | → Issue confirmed → HITL approval gate | - |
| -                  | Layer 2: Write commands (approved only) | - |
| AGENT 5: Verification + Closure Agent | - | - |
| -                  | T+15 min: lightweight SSH check | - |
| -                  | T+45 min: final SSH check | - |
| -                  | All GREEN → auto-close SNOW ticket | - |
| -                  | Any RED   → escalate to engineer | - |

### Full Ticket Lifecycle

#### Entry Condition
- Tickets must be In Progress (state=2) in SNOW, assigned to your team's group, with at least one of: Category, Subcategory, or Issue Type empty.

#### Step-by-Step Flow
1. **POLL (every 3 minutes)**
   - APScheduler triggers poller_agent.run_poll_cycle()
   - GET /api/now/table/incident with exact query filters:
     - active=true, state=2, your assignment group,
     - excluding test companies and specific users
   - Returns up to 50 tickets per cycle

2. **DEDUP**
   - For each ticket:
     - Redis GET snow_ai:processed:{sys_id}
     - Key exists? SKIP (already processed within 24h)
     - Key missing? Continue
     - Redis SETEX snow_ai:processed:{sys_id} 86400

3. **FIELD CHECK**
   - If ALL of (category, subcategory, u_issue_type) are non-empty → SKIP
   - (Nothing for us to do — engineer already filled them)

4. **CLASSIFY**
   - Extract Eventsource from description
   - Detect vendor/subcategory from device name + keywords
   - Parse all entities (host, IP, port, peer, zone, alert_id...)
   - Layer 1: fingerprints.yaml lookup
     - match? → issue_type, confidence=1.0, method=deterministic
     - no match? → Layer 2
   - Layer 2: ChromaDB + GPT-4o
     - Redis LLM cache check (same eventsource seen before?)
     - ChromaDB query → top-5 similar historical tickets
     - GPT-4o prompt: description + RAG context + valid taxonomy
     - Parse JSON response → validate against taxonomy
     - Cache result in Redis (1h TTL)

Coverage: ~85% of tickets
Every LogicMonitor alert contains an Eventsource field that uniquely identifies the alert type. This is matched against config/fingerprints.yaml.

5. **DERIVE CONTACT FIELDS**
   - opened_by → integration_accounts.yaml lookup
     - LogicMonitor/Zabbix/Hepic → Monitoring Tool / Internal
     - Maestro → Email / Internal
     - Engineer name → detect channel → Internal Portal / Internal
     - Unknown → Monitoring Tool / Internal (default)

6. **SNOW UPDATE (empty fields only)**
   - Build PATCH payload: only fields currently empty in SNOW
   - PATCH /api/now/table/incident/{sys_id}
   - Write WorkNote: [AI AutoClassified] with all classification details
   - Write PostgreSQL ticket_audit row

7. **REMEDIATION ROUTING**
   - issue_type in Phase 1 set?
     - NO  → WorkNote "[AI AutoClassified] Remediation pending — manual review"
           remediation_status = skipped
           END for this ticket
     - YES → route to correct sub-agent

8. **SSH DIAGNOSTIC (Layer 1 — read-only)**
   - Load device from devices.yaml by hostname/IP
   - Netmiko SSH connect (with retry)
   - Write WorkNote: "Connecting to {hostname} for diagnostic..."
   - Execute read-only show/get commands from runbook YAML
   - Write each command output to WorkNote immediately
   - Parse output for green/red indicators
   - Issue confirmed = NO?
     - → Write "No active issue" WorkNote
     - → SNOW state → Resolved
     - → END
   - Issue confirmed = YES?
     - → Write diagnostic findings WorkNote
     - → Route to HITL gate

9. **HITL APPROVAL GATE**
   - Write SNOW WorkNote with full diagnostic findings
   - Request engineer approval for Layer 2 commands
   - Wait up to HITL_APPROVAL_TIMEOUT_SECONDS (default 1h)
   - APPROVED → Execute Layer 2 remediation commands
   - REJECTED → Escalate to engineer with findings WorkNote
   - TIMEOUT  → Escalate to engineer with timeout WorkNote

10. **REMEDIATION (Layer 2 — write commands, HITL approved only)**
    - Execute remediation commands from runbook YAML
    - Write each command + output to WorkNote
    - Write to ssh_command_log (DB)

11. **VERIFICATION (T+15 and T+45)**
    - APScheduler fires at T+15 and T+45 minutes post-remediation
    - Lightweight SSH check (single command)
    - Write verification WorkNote
    - T+15 GREEN + T+45 GREEN → auto-close ticket
    - Any RED at T+45 → escalate to engineer with full findings

### Classification Strategy

#### Layer 1 — Deterministic (Zero LLM Cost)
- Ticket Description → extract "Eventsource: SIPTLSClientHandshakeFailure"
- fingerprints.yaml lookup

##### Fingerprint table (subset):

| Eventsource                             | Issue Type                     | Remediation       |
|-----------------------------------------|--------------------------------|-------------------|
| SIPTLSClientHandshakeFailure            | TLS Handshake Failure          | ✅ Phase 1        |
| SIPTLSHandshakeNegotiationStartFailure  | TLS Handshake Failure          | ✅ Phase 1        |
| sonusSbxPathCheckPingStateDownNotification2 | IP Peer Path Down/Flap        | ✅ Phase 1        |
| PathChkPingState - Down                | Ping Loss                      | ✅ Phase 1        |
| NodeReboot                              | Reboot                         | ❌ Classify only   |
| SystemUpAfterUnplannedRestart          | Reboot                         | ❌ Classify only   |
| sonusCpRedundGroupSwitchOverNotification | Failover                      | ❌ Classify only   |
| sonusSbxCertificateExpireSoon          | Certificate                    | ❌ Classify only   |
| SessionAgentDown                        | Session Agent Down/Flap       | ❌ Classify only   |
| acProxyConnectionLost                   | Gateway                        | ❌ Classify only   |

Full list: config/fingerprints.yaml — 25+ entries.

For ambiguous Eventsources or user-submitted tickets with no Eventsource, every classification output — from either layer — is validated against config/category_map.yaml before being written to SNOW. This prevents any invalid Category/Subcategory/Issue Type combination from ever reaching ServiceNow.

### File: agents/poller_agent.py
#### Trigger: APScheduler, every POLL_INTERVAL_SECONDS (default: 180s)

| Function                     | Purpose                                      |
|-----------------------------|----------------------------------------------|
| run_poll_cycle()            | Main entry point — full poll cycle          |
| is_already_processed(sys_id) | Redis dedup check                           |
| mark_as_processed(sys_id)   | Redis SETEX with TTL                        |
| remove_from_processed(sys_id)| For testing/reprocessing                     |

### Agent Reference

#### Agent 1 — Poller Agent

| Function                     | Purpose                                      |
|-----------------------------|----------------------------------------------|
| check_redis_connection()     | Health check                                 |
| _build_ticket_state(raw)     | Raw SNOW dict → TicketState                 |

Failure behaviour:
- SNOW API down → log error + return empty batch (scheduler retries next cycle)
- Redis down → log warning + process ticket anyway
- Individual ticket parse fails → skip + count in stats log

#### Agent 2 — Classifier Agent

| Function                     | Purpose                                      |
|-----------------------------|----------------------------------------------|
| run_classifier(state)       | LangGraph node entry point                   |
| layer1_classify(eventsource, subcategory) | Deterministic fingerprint match |
| layer2_classify(description, ...) | LLM + RAG classification                |
| derive_contact_and_trigger(opened_by) | Derive Contact Type + Trigger Source |
| validate_classification(cat, sub, type) | Taxonomy cross-check               |
| _get_rag_context(description) | ChromaDB similarity query                   |
| _call_llm(...)              | GPT-4o API call                             |

Failure behaviour:
- Layer 1 miss → route to Layer 2 (expected path for ~15%)
- Layer 2 LLM fails → default to Others + confidence=0.0
- Invalid taxonomy combo → default to Others
- Any unhandled exception → safe defaults set, error logged, pipeline continues

#### Agent 3 — SNOW Updater Agent

| Function                     | Purpose                                      |
|-----------------------------|----------------------------------------------|
| run_updater(state)          | LangGraph node entry point                   |
| build_patch_payload(...)     | No-overwrite field payload builder           |
| write_classification_to_snow(...) | PATCH + WorkNote                       |
| write_audit_record(...)      | PostgreSQL audit write                       |

No-overwrite rule: Every field is individually checked. If category is already "SBC" in SNOW, we skip writing it even if our classification agrees. Only truly empty fields are written.

#### Agent 4 — Remediation Router + Sub-Agents

| Sub-Agent                   | Issue Types                                 | Runbook                                   |
|-----------------------------|---------------------------------------------|-------------------------------------------|
| tls_agent.py                | TLS Handshake Failure                       | config/runbooks/tls_handshake.yaml       |
| ping_loss_agent.py          | IP Peer Path Down/Flap, Ping Loss         | config/runbooks/ping_loss.yaml           |

### Agent 5 — Verification + Closure Agent

| Function                     | Purpose                                      |
|-----------------------------|----------------------------------------------|
| run_verification(state)     | LangGraph node entry point                   |
| build_verification_worknote(...) | WorkNote for verification results       |

### File-by-File Reference

#### observability/exceptions.py

- SnowAIBaseError
  - ConfigurationError — missing/invalid .env or YAML config
  - ServiceNowError — base for all SNOW API errors
    - SnowAPIError — non-2xx response (not auth/rate-limit)
    - SnowAuthError — HTTP 401/403 invalid credentials
    - SnowRateLimitError — HTTP 429 too many requests
    - SnowTicketNotFoundError — HTTP 404 ticket deleted
    - SnowFieldValidationError — invalid taxonomy value rejected
  - ClassificationError — base for classification failures
    - FingerprintMatchError — fingerprints.yaml infrastructure error
    - LLMClassificationError — GPT-4o call failed or bad JSON
    - RAGRetrievalError — ChromaDB query failed
    - InvalidClassificationError — taxonomy validation failed
  - EntityExtractionError — description parser failure
  - SSHError — base for all SSH errors
    - SSHConnectionError — TCP connection failed (device unreachable)
    - SSHAuthenticationError — wrong SSH credentials
    - SSHCommandError — command returned error output
    - SSHTimeoutError — connection or command timed out
    - SSHDeviceNotFoundError — hostname not in devices.yaml
  - RemediationError — base for remediation failures
    - RunbookLoadError — YAML file missing or invalid
    - RunbookExecutionError — unrecoverable runbook step failure
    - RemediationTimeoutError — exceeded REMEDIATION_TIMEOUT_SECONDS
  - VerificationError — T+15/T+45 checkpoint infrastructure failure
  - DeduplicationError — Redis write failure
  - DatabaseError — base for PostgreSQL errors
    - DBConnectionError — cannot connect to PostgreSQL
    - DBAuditWriteError — audit record write failed
  - VectorStoreError — ChromaDB unavailable
  - WorkNoteError — WorkNote write failed after all retries

#### observability/logger.py

Key functions:
| Function                     | Use case                                      |
|-----------------------------|----------------------------------------------|
| configure_logging()         | Call once at startup. JSON (prod) or console (dev). |
| get_logger(__name__)       | Get module logger — use in every file.      |
| bind_ticket_context(ticket_id, sys_id) | Bind ticket_id to all logs in current async context. |
| clear_ticket_context()      | Clear at end of each node (prevents context leak). |
| log_worknote(...)           | Standard WorkNote write log entry.          |
| log_ssh_command(...)        | Standard SSH command execution log entry.   |
| log_snow_api(...)           | Standard SNOW API call log entry.           |

Log output (JSON mode):
Sensitive fields auto-scrubbed: password, api_key, token, secret, authorization → ***REDACTED***

#### config/settings.py

```python
from config.settings import get_settings
settings = get_settings()  # cached, reads .env once
```

##### Properties
- settings.snow_incident_url        # full SNOW API URL
- settings.postgres_dsn             # SQLAlchemy connection string
- settings.fingerprints_yaml        # Path to fingerprints.yaml
- settings.verification_t1_minutes  # 15
- settings.verification_t2_minutes  # 45

#### config/fingerprints.yaml

```yaml
fingerprints:
  SIPTLSClientHandshakeFailure:
    issue_type: "TLS Handshake Failure"
```

### Adding a new fingerprint: Add one block here. Zero code changes required.

#### config/category_map.yaml

Purpose: SNOW predefined taxonomy. Ground truth for validation.
Contains:
- All valid Category → Subcategory → Issue Type combinations
- contact_type_values list
- issue_trigger_source_values list
- subcategory_to_vendor map for Netmiko driver selection

#### config/integration_accounts.yaml

```yaml
"Logic Monitor" → Contact Type: Monitoring Tool, Trigger Source: Internal
"Zabbix Monitoring" → Contact Type: Monitoring Tool, Trigger Source: Internal
"Hepic Alert" → Contact Type: Monitoring Tool, Trigger Source: Internal
"Maestro Alert" → Contact Type: Email, Trigger Source: Internal
```

#### config/devices.yaml

```yaml
# Step 1: Generate encryption key (once)
python tools/encrypt_password.py --generate-key
# Add key to .env as DEVICE_PASSWORD_ENCRYPTION_KEY
# Step 2: Encrypt password
python tools/encrypt_password.py --password "mypassword"
# Step 3: Add to devices.yaml
- hostname: "JPYOK-SBC1"
  ip: "10.0.0.1"
  vendor: "sonus"
  device_type: "linux"
  username: "admin"
  password_enc: "gAAAAAB..."
  port: 22
  enabled: true
```

#### config/runbooks/tls_handshake.yaml

Variable substitution in commands: {host}, {ip}, {port}, {peer}, {zone}, {ticket_id}, {approver}

Purpose: Commands for IP Peer Path Down / Ping Loss across all 3 vendors.

Vendor sections:
- sonus — commands TBD (fill in)
- audiocodes — show sip peer status, show interface
- oracle_acme — show sip-interface, show peer-status

### Key components

| Class                       | Purpose                                      |
|-----------------------------|----------------------------------------------|
| TicketState                 | TypedDict — the LangGraph pipeline state    |
| ExtractedEntities           | 11 structured fields from description        |
| ClassificationOutput        | Classifier agent output (validated)         |
| SNOWUpdatePayload           | PATCH body builder with no-overwrite logic   |
| DiagnosticResult            | SSH Layer 1 output                          |
| VerificationCheck           | Single T+15/T+45 checkpoint result          |
| HITLRequest                 | Human approval request + decision            |
| ClassificationMethod        | Enum: deterministic / llm / rag / manual    |
| RemediationStatus           | Enum: pending / running / clean / remediated / escalated / skipped / failed |

### Helper functions

Purpose: Device model with Fernet password decryption and Netmiko connection dict.

### Usage

Purpose: SQLAlchemy ORM — 6 audit tables.

| Table                      | Rows written by         | Purpose                                      |
|---------------------------|-------------------------|----------------------------------------------|
| ticket_audit              | Updater Agent           | One row per processed ticket — full classification + outcome |
| worknote_log              | Every agent             | Every WorkNote attempt with success flag     |
| ssh_command_log           | SSH Agent               | Every SSH command + full output              |
| verification_log          | Verification Agent      | T+15 + T+45 results                          |
| hitl_log                  | Remediation Agent       | HITL request + engineer decision             |
| error_log                 | Any agent               | All unhandled exceptions with stack trace    |

### Purpose

ServiceNow REST API client — all SNOW interactions go through here.

| Function                     | HTTP | Purpose                                      |
|-----------------------------|------|----------------------------------------------|
| poll_inprogress_tickets()   | GET  | Fetch batch of In Progress tickets          |
| get_ticket(sys_id)          | GET  | Fetch single ticket by sys_id               |
| build_patch_payload(current, classification) | — | No-overwrite payload builder                 |
| patch_ticket(sys_id, payload)| PATCH| Update SNOW fields                          |
| write_worknote(sys_id, note) | PATCH| Append WorkNote                             |
| update_ticket_classification(...) | PATCH×2 | Fields + WorkNote in one call            |
| close_ticket(sys_id, ...)    | PATCH| Set state=6 + closure note                  |
| escalate_ticket(sys_id, ...)  | PATCH| Write escalation WorkNote                    |

### Retry policy

Tenacity with exponential backoff. Retries on SnowAPIError and SnowRateLimitError. Does NOT retry SnowAuthError (credentials won't fix themselves).

### Purpose

Extract structured data from raw ticket descriptions.

| Function                     | Output                                      |
|-----------------------------|----------------------------------------------|
| extract_eventsource(description) | Eventsource string                       |
| parse_description(description) | ExtractedEntities (all 11 fields)      |
| detect_subcategory(description, ci, short_desc) | "Sonus" / "AudioCodes" / "Oracle ACME" / "Kandy" |
| extract_display_value(field) | Clean string from SNOW display_value dict |
| parse_snow_ticket(raw)      | All SNOW fields as clean string dict      |
| build_command_context(entities, ...) | Substitution dict for runbook commands |

### Purpose

CLI to generate Fernet keys and encrypt device passwords.

### Purpose

Agent 1 — SNOW poll + Redis dedup + batch TicketState creation.

Key function: run_poll_cycle() → List[TicketState]

Poll cycle stats logged:
- total_fetched — tickets returned by SNOW
- dedup_skipped — already processed this cycle
- all_fields_populated — classification already complete
- parse_failed — malformed tickets
- queued_for_processing — sent to classifier

### Purpose

Agent 2 — two-layer classification engine.

Key function: run_classifier(state: TicketState) → TicketState

Layer 1 path: extract_eventsource → fingerprints.yaml → ClassificationOutput(confidence=1.0)

Layer 2 path: Redis cache check → ChromaDB → GPT-4o → taxonomy validate → ClassificationOutput

### Configuration Reference

#### Key YAML Files

| File                          | Editable at runtime? | Requires restart? |
|-------------------------------|----------------------|--------------------|
| fingerprints.yaml             | Yes                  | Yes (cached at startup) |
| category_map.yaml             | Yes                  | Yes                |
| integration_accounts.yaml      | Yes                  | Yes                |
| devices.yaml                  | Yes                  | Yes                |
| runbooks/tls_handshake.yaml   | Yes                  | Yes                |
| runbooks/ping_loss.yaml       | Yes                  | Yes                |

No code changes. Restart the service.

### Adding a New Eventsource Fingerprint

```yaml
# config/fingerprints.yaml
NewEventSource:
  issue_type: "Reboot"          # must match category_map.yaml exactly
  remediation: false            # true only for Phase 1 types
  description: "What this is"
```

### SNOW API Integration

#### Instance

GET — Poll In Progress Tickets

```http
GET /api/now/table/incident
  ?sysparm_query=active=true^state=2
    ^company!=<excluded>^company!=<excluded>
    ^opened_by!=<excluded>
    ^assignment_group!=<excluded>
    ^assignment_group=<your_team_group>
    ^sys_created_by!=sairam.shapur
  &sysparm_fields=number,description,short_description,sys_id,state,
    u_issue_trigger_source,u_rpt_created_on,subcategory,assignment_group,
    company,cmdb_ci,u_issue_category,category,u_issue_type,contact_type,
    opened_by,work_notes
  &sysparm_limit=50
  &sysparm_display_value=true
```

PATCH — Update Classification Fields

```http
PATCH /api/now/table/incident/{sys_id}
Content-Type: application/json
{
  "category": "SBC",
  "subcategory": "Sonus",
  "u_issue_type": "TLS Handshake Failure",
  "contact_type": "Monitoring Tool",
  "u_issue_trigger_source": "Internal",
  "work_notes": "[AI AutoClassified]\nIssue Type: TLS Handshake Failure\n..."
}
```

### Custom Field Names

| Display Name                | API Field                     |
|-----------------------------|-------------------------------|
| Issue Trigger Source        | u_issue_trigger_source        |
| Issue Category              | u_issue_category (read-only, always "Others") |
| Contact Type                | contact_type                  |

All SSH connections via Netmiko. Connection parameters from devices.yaml.

| Vendor                      | Netmiko device_type          |
|-----------------------------|-------------------------------|
| Sonus SBC 5000             | linux                         |
| AudioCodes                  | audiocodes_shell              |
| Oracle ACME Packet          | acme_packet                   |
| Kandy                       | linux                         |

Every SSH action — connection attempt, command execution, result — is immediately written as a WorkNote to the SNOW ticket. Engineers can see real-time diagnostic progress in SNOW.

### Device Connection

#### WorkNote on Every SSH Action

```text
[AI SSH Diagnostic — TLS Handshake Failure]
Device: JPYOK-SBC2 (10.0.0.1)
Connected at: 2024-01-15 10:30:05 UTC
Step 1 — TLS session status check:
<command output>
Step 2 — Certificate validity check:
<command output>
Findings: Issue confirmed — TLS certificate expired.
```

### No Issue Found Flow

```text
[AI Diagnostic — No Active Issue]
Investigated: TLS Handshake Failure
Device: JPYOK-SBC2
Checked at: 2024-01-15 10:30:05 UTC
Findings: All indicators normal — no active issue confirmed.
Recommendation: Continue monitoring. Ticket eligible for closure.
```

### HITL — Human in the Loop

#### HITL WorkNote (written to SNOW)

```text
[AI HITL — Approval Required]
Ticket: INC0012345
Device: JPYOK-SBC2
Issue: TLS certificate expired (confirmed by diagnostic)
Proposed action: Restart TLS service / clear session
```

If no response within HITL_APPROVAL_TIMEOUT_SECONDS (default: 1 hour), the ticket is escalated to an engineer with full diagnostic findings.

### Checkpoint

| Minutes after remediation | Purpose                                      |
|---------------------------|----------------------------------------------|
| T+15                      | Confirm fix held, no regression              |
| T+45                      | Final gate before auto-close                 |

Both checkpoints GREEN → ticket automatically closed with summary WorkNote. Any checkpoint RED at T+45 → WorkNote with full findings written, ticket remains open for engineer.

### Component Cost Risk

| Component                  | Cost Risk                                   | Control Implemented                          |
|----------------------------|---------------------------------------------|----------------------------------------------|
| GPT-4o                     | HIGH                                       | Layer 1 covers ~85% of tickets (zero LLM cost) |
| GPT-4o                     | HIGH                                       | Redis caches LLM results per Eventsource (1h TTL) |
| GPT-4o                     | MEDIUM                                     | max_tokens=1000 hard cap per call          |
| GPT-4o                     | MEDIUM                                     | temperature=0.0 (no sampling randomness)   |
| ChromaDB                   | LOW                                        | Local deployment, no API cost               |
| LangSmith                  | LOW                                        | Configurable LANGSMITH_TRACE_SAMPLE_RATE (0.0–1.0) |
| Redis                      | NEGLIGIBLE                                  | Local deployment                             |
| SNOW API                   | LOW                                        | Batch polling every 3 min (not real-time)  |
| DeepEval                   | DEV ONLY                                   | Runs in CI tests, never in production pipeline |

### Command

```text
<exact command to be run>
To APPROVE: Reply to this WorkNote with "APPROVE"
To REJECT:  Reply to this WorkNote with "REJECT"
Timeout:    1 hour from this message
```

### Approval Timeout

### Verification & Auto-Closure

#### Checkpoints

| Auto-Close Criteria        | Escalation Criteria                         | Closure WorkNote                             |
|----------------------------|---------------------------------------------|----------------------------------------------|
| [AI Auto-Close — All Verifications Passed] | Ticket: INC0012345 | Issue Type: TLS Handshake Failure |
| Device: JPYOK-SBC2        | Remediation Timeline:                       | 10:30 — Diagnostic confirmed TLS cert expired |
| 10:35 — HITL approved by Engineer A | 10:36 — TLS service restarted |
| 10:51 — T+15 check: GREEN (TLS sessions active) | 11:21 — T+45 check: GREEN (sustained recovery) |
| All verification checks passed. Ticket auto-closed. |

### Cost Controls

| Column                     | Type      | Description                               |
|---------------------------|-----------|-------------------------------------------|
| ticket_id                 | VARCHAR(20) | INC number                               |
| sys_id                    | VARCHAR(64) | SNOW sys_id (unique)                    |
| issue_type                | VARCHAR(128) | Classified issue type                    |
| category                  | VARCHAR(64) | Classified category                       |
| subcategory               | VARCHAR(64) | Classified subcategory                    |
| confidence_score          | FLOAT     | 0.0–1.0                                  |
| classification_method      | VARCHAR(32) | deterministic/llm/rag/manual             |
| snow_updated              | BOOLEAN   | PATCH succeeded                           |
| remediation_status        | VARCHAR(32) | pending/clean/remediated/escalated/skipped |
| t1_result                 | VARCHAR(16) | T+15: green/red/skipped                  |
| t2_result                 | VARCHAR(16) | T+45: green/red/skipped                  |
| closure_eligible          | BOOLEAN   | All checks passed                        |

Plus: worknote_log, ssh_command_log, verification_log, hitl_log, error_log.

### Python 3.11+

- Docker + Docker Compose
- Access to ServiceNow instance
- OpenAI API key

### Database Schema

- ticket_audit — Primary audit table

### Setup & Installation

#### Prerequisites

#### Quick Start

```bash
# 1. Clone and enter project
git clone <repo-url>
cd snow-ai-automation
# 2. Create virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
# 3. Install dependencies
pip install -r requirements.txt
# 4. Set up environment
cp .env.example .env
# Edit .env with your credentials
# 5. Generate device password encryption key
python tools/encrypt_password.py --generate-key
# Add output to .env as DEVICE_PASSWORD_ENCRYPTION_KEY
# 6. Add devices to config/devices.yaml
# (encrypt each password first)
# 7. Start supporting services
docker-compose up -d redis postgres chromadb
```

### Variable

| Required                  | Default | Description                               |
|---------------------------|---------|-------------------------------------------|
| SNOW_INSTANCE_URL         | ✅      | —                                         |
| SNOW_USERNAME              | ✅      | SNOW API service account                  |
| SNOW_PASSWORD              | ✅      | SNOW API password                         |
| OPENAI_API_KEY            | ✅      | OpenAI API key                           |
| DEVICE_PASSWORD_ENCRYPTION_KEY | ✅ | Fernet key for devices.yaml               |
| POSTGRES_USER             | ✅      | PostgreSQL username                       |
| POSTGRES_PASSWORD          | ✅      | PostgreSQL password                       |
| LANGCHAIN_API_KEY         | ⚪      | —                                         |
| REDIS_HOST                | ⚪      | localhost                                 |
| REDIS_DEDUP_TTL_SECONDS   | ⚪      | 86400                                    |
| POLL_INTERVAL_SECONDS      | ⚪      | 180                                      |
| VERIFICATION_T1_MINUTES    | ⚪      | 15                                       |
| VERIFICATION_T2_MINUTES    | ⚪      | 45                                       |
| LOG_LEVEL                 | ⚪      | INFO                                     |
| LOG_FORMAT                | ⚪      | json                                     |

Full list: see .env.example

This system is designed for zero-code expansion. Adding a new issue type requires only file edits:

### Step 1 — Add fingerprint (if LogicMonitor alert)

```yaml
# config/fingerprints.yaml
NewEventsourceName:
  issue_type: "Your New Issue Type"
  remediation: false  # set true when ready for Phase 2
  description: "What this alert means"
```

### Step 2 — Add to taxonomy (if not already there)

```yaml
# config/category_map.yaml
category_map:
  SBC:
    Sonus:
      issue_types:
        - "Your New Issue Type"   # add here
```

### Step 3 — Add remediation agent (when ready)

No changes to LangGraph, Poller, Classifier, or Updater agents.

- ✅ Auto-classification of all 25 issue types
- ✅ Two-layer classification (deterministic + LLM/RAG)
- ✅ Full SSH diagnostics for TLS Handshake Failure
- ✅ Full SSH diagnostics for IP Peer Path Down / Ping Loss
- ✅ HITL approval gate for all write commands
- ✅ Progressive verification (T+15, T+45)
- ✅ Auto-closure and engineer escalation
- Alert storm detection and noise suppression
- Parent-child ticket linking
- Confidence-based human-in-the-loop routing
- Full Runbook Automation Engine (RBA) for more issue types

### Releases

No releases published

### Packages

No packages published

### Contributors

1. prashantsundge Prashant Sundge

### File: agents/remediation/new_issue_agent.py

```python
from agents.remediation.base_remediation_agent import BaseRemediationAgent

class NewIssueAgent(BaseRemediationAgent):
    ...
```

### Step 4 — Add runbook YAML

```yaml
# config/runbooks/new_issue.yaml
runbook:
  name: "New Issue Diagnostic"
  issue_type: "Your New Issue Type"
  ...
```

### Step 5 — Register in router

```python
# agents/remediation/base_remediation_agent.py
ROUTER_MAP = {
    "TLS Handshake Failure": TLSAgent,
    "IP Peer Path Down/Flap": PingLossAgent,
    "Your New Issue Type": NewIssueAgent,  # add here
}
```

### Phase 1 Scope & Roadmap

- Phase 1 (Current)
- Tier 1 (Next after Phase 1 stable)
- Tier 2 (Intelligence layer)
