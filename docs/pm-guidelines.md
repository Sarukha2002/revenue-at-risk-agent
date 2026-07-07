# AI Product Building Guidelines for Senior PMs
# Lessons from building Revenue-at-Risk — applicable to any AI agent system

---

## The Mental Model: AI as a Deterministic Pipeline, Not a Magic Box

The biggest mistake PMs make when building AI products is treating the AI as an autonomous actor that "figures things out." This leads to:
- Unpredictable outputs that break downstream systems
- No auditability when the AI is wrong
- Systems that work in demos and fail in production

The right mental model: **the AI is one highly capable node in a deterministic pipeline.** Everything around it (input validation, output parsing, schema enforcement, error handling) must be built as rigorously as any other software system.

---

## 1. Schema-First Design (The Contract Rule)

**Before you write a single prompt or build a single node, define your JSON schema.**

The schema is the contract between:
- The AI agent (must output fields matching this schema)
- The database (columns must match schema field names exactly)
- The frontend (reads schema fields to render UI)

If any of these three are out of sync, the system fails silently. A misnamed field (`bug_description` vs `bug_summary`) means a blank column in your sheet and a blank cell in your dashboard — with no error to debug.

**PM rule:** Schema changes require a coordinated update to prompt, database, and frontend. Treat it like an API version bump. Never change schema fields without updating all three consumers.

---

## 2. Output Contracts in Prompts (The Null Discipline)

Every AI prompt that produces structured output needs two rules enforced explicitly:

**Rule 1: Output ONLY the schema. No prose, no markdown, no explanation.**

If Claude can return either `{"account_name": "Acme"}` or `Here is the extracted data: {"account_name": "Acme"}`, your `JSON.parse()` will fail 50% of the time. Force the output contract: "Return ONLY raw valid JSON. Do not include introductory text or markdown code blocks."

**Rule 2: Use null for unknown fields. Never guess.**

"If a field cannot be determined, return null" is the most important guardrail in structured extraction. The alternative — prompting Claude to fill gaps with reasonable inferences — produces hallucinated ticket numbers, invented CS owner names, and fabricated ARR values that corrupt your database.

**When to allow inference vs when to require null:**
- Computed fields (financial_risk_score, revenue_at_risk): always compute, never null
- Categorical fields with clear rules (technical_domain, sentiment): infer from rules you define
- Reference fields (engineering_ticket_ref, cs_owner): null if not explicitly mentioned

---

## 3. Temperature = 0 for Extraction Tasks

Extraction, classification, and calculation tasks must run at temperature 0.

Temperature controls randomness. For a creative writing task, temperature 0.7 produces more interesting output. For "extract the ticket number from this Slack message," temperature 0.7 means the ticket number is slightly different each time you run the same input.

**Rule:** Deterministic task (extraction, classification, scoring) → temperature 0. Generative task (summaries, recommendations, ticket body copy) → temperature 0.3–0.5.

The recommended_action field in this system is a generative task. The financial_risk_score is a deterministic computation. They should run at different temperatures — or in separate Claude calls.

---

## 4. Never Trust the AI's Output — Always Validate

Every Claude response goes through a validation node before touching your database. The validation node:
1. Strips accidental markdown fences Claude may have added despite instructions
2. Attempts `JSON.parse()` in a try/catch with a descriptive error
3. Checks required fields are present and non-null
4. Applies business logic constraints (revenue_at_risk cannot exceed arr)
5. Adds computed metadata (risk_tier bucket, last_updated timestamp)

This is not defensive coding — it is the design. Claude is highly capable but not infallible. Building the validation layer makes your system observable: when something goes wrong, you know exactly where in the pipeline it broke.

---

## 5. Business Logic Lives in One Place

In the Revenue-at-Risk system, the risk score formula is:
```
financial_risk_score = ((180 - renewal_horizon_days) / 180) * 100
```

You could implement this formula in:
- The Claude prompt (Claude computes it)
- The n8n Code node (JavaScript computes it after Claude returns raw inputs)
- A Google Sheets formula (Sheets computes it from raw columns)

**PM rule:** Business logic should live in exactly one place. The right place depends on:
- **Traceability:** If you need to audit why a score was what it was, compute it in code (n8n Code node), not in Claude's head
- **Changeability:** If the formula will change often, keep it in a config your team can edit without redeploying
- **Testability:** Code nodes can be unit tested. Claude prompts cannot

For this system: the correct long-term architecture is to have Claude extract raw facts (`renewal_horizon_days`, `arr`, `sentiment`, `days_issue_open`) and have the n8n Code node compute the score. This makes the AI's job pure extraction (what it's best at) and keeps business logic auditable.

---

## 6. Error Handling is a Product Feature

Every AI pipeline will fail. Claude will return malformed JSON. The Google Sheets API will return a 503. The webhook will time out. These are not edge cases — they are scheduled events.

Your pipeline must:
1. **Fail gracefully:** Return a meaningful error to the frontend, not a 500 with a stack trace
2. **Not lose data:** If the Sheets write fails, the Claude output should be queued for retry — not discarded
3. **Alert on failure:** Set up n8n's error workflow to post to Slack when any node fails
4. **Be retryable:** Any webhook submission should be idempotent — submitting the same payload twice should upsert, not create a duplicate row

---

## 7. The Three-Layer AI System Pattern

Every production AI agent system has three layers. PMs who try to skip layers end up rebuilding them:

```
Layer 1: Ingestion
  What comes in? From where? How often?
  Validation, normalization, deduplication happen here.
  No AI here — this is pure data engineering.

Layer 2: Intelligence
  The AI layer. One job: transform unstructured input → structured output.
  Strict prompt contract. Temperature 0. Output validation.
  Separation of concerns: AI extracts, code computes.

Layer 3: Action & Presentation
  What happens with the structured data?
  Write to DB. Create tickets. Send alerts. Render dashboard.
  No AI here — this is pure deterministic logic.
```

The Revenue-at-Risk system maps exactly: Webhook → Validate → Claude → Parse → Sheets → Dashboard.

---

## 8. What a PM Should Prioritize in Iterations

### Iteration 1 (current): Prove the intelligence layer works
- Single path: form → Claude → Google Sheets
- Manual data entry — CS managers submit signals
- Goal: validate that Claude extracts correctly for 90%+ of real inputs
- Success metric: CS team submits 20 signals and agrees the extraction is accurate

### Iteration 2: Replace manual input with automated ingestion
- Goal: zero manual steps for signal capture — but each channel below has its own
  signal-to-noise scoping decision, not just a plumbing swap from the intake webhook.
  Treat this as its own initiative, separate from Workflow 3 (ticket enrichment).
- **Slack**: native n8n Slack Trigger, but blanket-listening to a whole channel is noisy
  and costly (a Claude call per message). Preferred pattern: only process messages a
  human reacts to with a specific emoji — the reaction *is* the "this is worth extracting"
  signal, not passive channel monitoring.
- **Email**: a dedicated inbox (e.g. `risk-signals@yourco.com`) that CS forwards things to.
  The act of forwarding is the human-in-the-loop signal, cleaner than parsing a whole inbox.
- **Meeting MoM (minutes of meeting)**: no single answer — depends on what tool is
  actually used for meeting notes (Google Docs? Notion? Otter/Gong/Fireflies?). Whichever
  it is determines whether this is a Drive-trigger, Notion-trigger, or a dedicated
  integration. `[TODO — confirm which MoM tool is actually in use before scoping this]`
- Eliminates the "CS team must remember to submit" failure mode, once scoped per channel

### Iteration 3: Close the feedback loop
- Track whether recommended actions were taken (was the ticket resolved? did the account renew?)
- Use outcomes to validate and improve the risk score formula
- Add "mark as resolved" button in dashboard that writes back to the pipeline

### Iteration 4: Predictive scoring
- Once you have 6+ months of data (signals + outcomes), train a lightweight ML model on the historical data to replace the formula-based score
- Formula scores are deterministic but don't learn; ML scores get better as you collect more data

---

## 9. Metrics Every AI PM Should Track

For any AI extraction pipeline:

| Metric | What it measures | How to track |
|--------|-----------------|--------------|
| Extraction accuracy | % of fields Claude extracts correctly | Weekly human spot-check of 10 random records |
| Null rate per field | % of records where each field is null | Google Sheets COUNTIF formula |
| Pipeline error rate | % of submissions that fail before writing to Sheets | n8n execution log |
| End-to-end latency | Time from form submit to Sheets row written | n8n execution duration |
| False positive rate | Accounts flagged Critical that were not at risk | Track against renewal outcomes |
| False negative rate | Accounts that churned without being flagged | Track against churned accounts |

---

## 10. The Escalation Design Pattern

Every AI-powered ops tool needs a human-in-the-loop escalation path for high-stakes decisions. For Revenue-at-Risk:

- **Score 0–39 (Low):** Auto-logged, no alert. CS reviews weekly.
- **Score 40–59 (Medium):** Slack notification to CS Manager. Review within 48h.
- **Score 60–79 (High):** Slack notification to CS Director + auto-create Jira ticket. Review within 24h.
- **Score 80–100 (Critical) + escalation_flag = true:** Immediate Slack alert to VP CS + VP Engineering + CEO digest. Engineering must acknowledge within 4h.

The AI does the triage. Humans make the decisions. The system just makes sure the right humans are looking at the right information at the right time.

This is the correct role for AI in high-stakes business processes: **ambient intelligence that surfaces signal, not autonomous decision-making that takes action.**
