# Detection Duet
### Agentic Threat Detection Pipelines

> **Conference Notes — Detection Engineering Talk · Verkada Security**
> *Source: Live talk slides · AWS architecture · Built in-house by Verkada*

> *"Developing threat detections is slow, repetitive work. Learn how we cut threat detection development and maintenance time by over 80%."*

---

| Metric | Value |
|--------|-------|
| Reduction in dev + maintenance time | **80%** |
| Agents in the system | **2** (Creation + Maintenance) |
| Human review still required | **Always** |

---

## 01 // The Problem

Threat detection engineering is structurally slow — not because analysts lack skill, but because every single rule requires holding three things in your head at once: the adversary behavior, the log schema, and the detection platform syntax. Then you validate it, ship it, and maintain it when schemas change. Multiply this by hundreds of rules and one analyst cannot keep up.

**Why detection work is slow by design:**

- ✗ Context-heavy — requires simultaneous knowledge of the threat, log schema, and SIEM syntax
- ✗ Repetitive — similar rule patterns rewritten from scratch every time, no reuse
- ✗ Fragile — log source schema changes silently break existing rules
- ✗ Unscalable — one analyst writing rules manually cannot pace a growing attack surface
- ✗ Tribal knowledge — detection intent rarely lives in the rule itself, it lives in people

Verkada's answer: two agents. One for **creating new detections** from unstructured ideas. One for **maintaining existing rules** when log sources drift. Together — the Detection Duet.

---

## 02 // The Built-In-House Stack

Before understanding the agentic creation workflow, it helps to see the full security data platform these detections run on. Verkada built the entire pipeline themselves — from log collection through SOAR response.

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌─────────────┐     ┌──────────────────┐
│   Log Gathering  │     │   Preprocessing  │     │     Analysis     │     │  Alerting   │     │      SOAR        │
│                  │     │                  │     │                  │     │             │     │                  │
│  Log Pull (Grove)│ ──► │ Log Normalization│ ──► │Streaming Analysis│ ──► │    Slack    │ ──► │ Alerts Enrichment│
│  Log Push        │     │ Log Enrichment   │     │ Batch Analysis   │     │   Opsgenie  │     │ Tracecat         │
│                  │     │ (Substation)     │     │ (StreamAlert)    │     │   Linear    │     │ AI Response      │
└─────────────────┘     └──────────────────┘     └──────────────────┘     └─────────────┘     └──────────────────┘
```

**Key tools:**
- **Grove** — handles log pull
- **Substation** — normalizes and enriches logs
- **StreamAlert** — the detection engine (streaming and batch)
- **Linear** — appears in both alerting and as a trigger in the creation workflow
- **Tracecat** — handles SOAR orchestration

---

## 03 // Idea Generation — Where It All Starts

Detection ideas come from three distinct channels, each feeding a common **Idea** node that hands off to the creation pipeline. The AI Research channel is the differentiator — it lets the system proactively identify coverage gaps rather than only reacting to external research.

| Source | Inputs |
|--------|--------|
| **Existing Research** | Blog posts, GitHub repositories, Public research papers |
| **Verkada's Research** | Internal security reviews, Threat research findings |
| **AI Research** | Risk Registry, Log Inventory, MITRE ATT&CK detection strategies |

> 💡 **Why the AI Research channel matters:** By feeding the Risk Registry, Log Inventory, and MITRE ATT&CK strategies into an agent, Verkada can generate detection ideas proactively — not just react to external research. The agent identifies what behaviors you should be detecting that you currently aren't.

---

## 04 // The Creation Workflow — Full Pipeline

An unstructured idea enters. A pull request against the detection repository exits the other end. Every step in between is agentic except for the final human review gate.

---

### Step 01 — Idea Arrives (Unstructured)

An unstructured idea enters the pipeline from any of the three idea sources. At this point it may be nothing more than a sentence: *"we should detect root console logins from anonymizing infrastructure."* No code, no schema, no ATT&CK mapping — just intent.

---

### Step 02 — Context Gathering (Bedrock + DataCatalog + S3 + Notion)

Three inputs are gathered in parallel and processed by **Amazon Bedrock** to produce structured metadata:

- `Notion` → **Log Source Overview** — human-written docs about which log sources exist
- `DataCatalog` → **Log Schema** — actual field names, types, and formats
- `S3` → **Log Samples** — real example events for grounding the generation

Bedrock combines all three into a structured metadata object the rest of the pipeline uses.

---

### Step 03 — Create MITRE ATT&CK Mapping

Using the enriched metadata, the agent maps the detection idea to the correct ATT&CK technique(s).

**Output:** confirmed `technique_id`, tactic, and sub-technique. This anchors the detection in a standard framework and determines which code examples get pulled in the next step.

---

### Step 04 — Decide Type (Streaming or Scheduled)

A decision node routes to one of two branches based on the nature of the threat behavior:

**Streaming Detection** — real-time, event-driven (e.g., single login from TOR exit node). Runs against live log streams in StreamAlert.

**Scheduled Detection** — aggregate or time-window behavior (e.g., 10 failed logins in 5 minutes). Runs as a batch query on a schedule.

Each branch pulls its own type-matched **Code Examples** from the repo and builds an appropriate **Prompt** before passing to the Lambda.

---

### Step 05 — Code Generation Lambda *(The Terminal Execution Step)*

Both branches converge here. The Lambda wakes up with **no memory of prior runs** — everything it needs arrived in the event payload. It calls an LLM with the fully-assembled prompt (intent + schema + ATT&CK mapping + code examples) and generates detection rule code.

The Coding Agent inside this Lambda has two tools available:

- `Our Repository` — read/write access to the detection rule codebase
- `Athena Skill` — run SQL queries against real log data in Amazon Athena to validate rule logic before the PR is opened

The few-shot code examples force style conformance — the generated rule will match your team's naming conventions, severity constants, return format, and metadata structure.

---

### Step 06 — Open PR (Mandatory Human Gate)

The Lambda's final output is a pull request opened against the detection repository. No rule is deployed until a human approves the PR.

The analyst reviews: does the logic match the intent? Is the scope correct? Does it hold up against real log data?

> **The agent proposes. The analyst disposes.**

---

## 05 // Prompt Engineering with Internal Context

The quality of generated code is entirely determined by prompt quality. Generic prompts produce generic rules. The power here is **internal context injection** — schema, log samples, and existing rules that no LLM could have from training data alone.

### 4-Layer Prompt Structure

```
## Layer 1 — Role + Detection Type
You are a detection engineer. Generate a {detection_type}
detection rule for StreamAlert using the schema and examples below.

## Layer 2 — Threat Context (from Bedrock enrichment)
Technique:  {technique_id} — {technique_name}
Tactic:     {tactic}
Behavior:   {condition_description}
Log source: {log_source}

## Layer 3 — Schema + Log Sample (from DataCatalog + S3)
Available fields:
  eventName          (string)  e.g. ConsoleLogin
  userIdentity.type  (string)  Root | IAMUser | AssumedRole
  sourceIPAddress    (string)  Caller IP
  {...all fields from DataCatalog}

Sample log event (real, from S3):
  {real_log_sample}

## Layer 4 — Code Examples (type-matched, from repository)
# Match the style, structure, field naming, and return format exactly.
{example_rule_1}
{example_rule_2}

## Instruction
Generate a detection rule for: {condition_description}
Return ONLY the rule function. No explanation. No markdown.
```

> ⚠️ **Layer 4 is what forces style conformance.** Without the few-shot examples, the LLM invents its own structure that won't match your SIEM's expected schema. With them, the generated rule looks like it was written by a senior engineer on your team — same field names, same severity constants, same return format.

---

## 06 // Sync vs. Async Invocation

The Lambda can run in two modes depending on urgency. Both produce the same output (a PR) — they differ in whether the caller waits or submits to a queue.

| Dimension | Synchronous | Asynchronous |
|-----------|-------------|--------------|
| **Trigger** | Direct API call, analyst tool | Linear ticket, SQS queue, EventBridge schedule |
| **Caller behavior** | Waits. Lambda returns result directly. | Fires and forgets. Lambda notifies via SNS/Slack when done. |
| **Best for** | On-demand interactive rule drafts, analyst experimenting in real time | Batch backlog, overnight maintenance runs, Linear ticket queue |
| **Linear ticket role** | Not the primary trigger in sync mode | Ticket creation fires webhook → API Gateway → Lambda. Lambda comments back with generated rule + PR link. |
| **Risk** | LLM calls can hit 30s+ — API Gateway timeout boundary | Context can go stale if threat intel shifts while rule sits in queue |

---

## 07 // Validation — Agent + Human Split

The Coding Agent doesn't just write code and stop. It runs automated validation using the **Athena Skill** — querying real log data to test the rule before the PR opens. But automated validation has hard limits. Manual review covers what the agent structurally cannot verify.

### Agent Validates — Automated

- → Syntax correctness — does the code parse without errors?
- → Schema compliance — do all referenced fields exist in DataCatalog?
- → Athena test against known true positive log events
- → False positive check against baseline benign event samples
- → ATT&CK technique ID is valid and resolves correctly
- → Severity and metadata conform to repository standards

### Human Validates — Always Required

- → Does the logic reflect the actual intended threat behavior?
- → Is the detection scope too broad or too narrow?
- → Are there edge cases in the condition logic the agent missed?
- → Is the alert title useful to a responder at 2am?
- → Does this duplicate an existing rule?
- → Does the severity match operational risk tolerance?

> 🔒 **The trust boundary:** The Code Generation Lambda is where agent reasoning becomes executable artifact. Everything after that Lambda is in the human domain until the PR is approved. No merge = no deploy. The agent proposes. The analyst disposes. This is not a convenience — it is the architecture.

---

## 08 // Full Pipeline Reference

| Stage | Tool / Service | What It Does |
|-------|---------------|--------------|
| **Idea Source** | Blog posts, GitHub, Notion, Risk Registry, MITRE ATT&CK | Generates unstructured detection intent |
| **Context Gathering** | Amazon Bedrock, DataCatalog, S3, Notion | Produces structured metadata: schema, log samples, log source overview |
| **ATT&CK Mapping** | Amazon Bedrock + MITRE ATT&CK | Maps intent to confirmed technique ID, tactic, sub-technique |
| **Type Decision** | Agent decision node | Routes to streaming or scheduled branch with type-matched code examples |
| **Prompt Assembly** | Code Examples (repo) + enriched metadata | Builds the 4-layer prompt: role, threat context, schema, few-shot examples |
| **Code Generation Lambda** | AWS Lambda + Bedrock LLM + Coding Agent | Generates detection rule code from assembled prompt |
| **Validation** | Athena Skill | Tests rule against real log data; checks true/false positive behavior |
| **Human Review** | GitHub PR → analyst approval | Mandatory gate — no rule deploys without human sign-off |
| **Deployment** | Repository merge → StreamAlert | Rule goes live in streaming or batch analysis pipeline |

---

*Detection Duet — Conference Notes*
*Compiled by **Threatcraft** · THOR Collective*
*Based on live talk slides, Verkada Security Engineering*

*Stack: Grove · Substation · StreamAlert · Tracecat · AWS Bedrock · Lambda · Athena · S3 · DataCatalog · Linear · GitHub · MITRE ATT&CK*