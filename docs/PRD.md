# PRD — Revenue-at-Risk Containment System

**Status:** Pilot / Iteration 1 in progress
**Owner:** Solo build (PM + implementer), self-hosted pilot
**Last updated:** 2026-07-01

---

## 1. Problem Statement

Enterprise customer success signal — a furious Slack message, an escalation on a call, a
comment buried in a spreadsheet — routinely fails to reach the people who can act on it,
attached to the number that would make them act: dollars at risk.

A $500K account with a 23-day-old P0 outage and a 77-day renewal window is not the same
priority as a $50K account with a minor cosmetic bug, but without a system to compute and
surface that difference, both look like "a ticket in the backlog." CS teams escalate
informally; engineering triages by gut feel or ticket age; nobody has a single ranked view
of which at-risk accounts represent the most revenue exposure, right now.

## 2. Goals

- Convert unstructured CS signal (Slack messages, spreadsheet rows) into structured,
  scored risk records without manual re-entry
- Surface a single dashboard ranked by `revenue_at_risk` (ARR × churn risk), not by ticket
  age or squeaky-wheel escalation volume
- Route the right signal to the right human at the right urgency (CS Manager → CS Director
  → VP CS/Eng → CEO digest) based on score, not on who shouts loudest
- Keep the AI's role narrow and auditable: extraction and categorization, not autonomous
  decision-making

## 3. Non-Goals (for this pilot)

- No autonomous customer-facing actions (no auto-sent emails, no auto-resolved tickets)
- No predictive ML scoring yet — Iteration 1–3 use a deterministic formula, by design (see
  §6). ML scoring is explicitly deferred to Iteration 4, once outcome data exists to train on
- No multi-tenant / multi-org support — this is a single-org pilot

## 4. Users & Stakeholders

| Role | What they need from this system |
|---|---|
| CS Manager | A fast way to log a signal (form/Slack) and get an immediate risk score back |
| CS Director | Notified within 24–48h for High/Medium risk accounts to prioritize review |
| VP CS / VP Engineering | Notified immediately for Critical accounts; owns the escalation dashboard |
| CEO | Daily digest of Critical, escalation-flagged accounts only — signal, not noise |
| Engineering (ticket assignee) | A Jira ticket with dollar impact and urgency already attached, not a bare bug report |

## 5. Success Metrics

Pulled from `docs/pm-guidelines.md` §9 — these are the metrics that matter more than "does
the demo look good":

| Metric | Target for Iteration 1 |
|---|---|
| Extraction accuracy (human spot-check) | 90%+ across 20 real submitted signals |
| Null rate per field | Tracked, not zero — nulls are correct behavior when a field truly isn't mentioned |
| Pipeline error rate | Logged via n8n execution history |
| False positive rate (flagged Critical, not actually at risk) | Tracked against renewal outcomes once available |
| False negative rate (churned without being flagged) | Tracked against churned accounts once available |

**Iteration 1 exit criterion (from `build-guide.md`):** CS team submits 20 real signals and
agrees the extraction is accurate.

## 6. System Overview

Full architecture: `docs/architecture.md`. Summary:

```
Slack / Sheets / Jira → n8n workflow engine → Claude (extraction + scoring)
   → validation/parse layer → Google Sheets (system of record) → Lovable dashboard
```

Three-layer pattern (from `pm-guidelines.md` §7):
- **Ingestion** — pure data engineering, no AI. Validates, normalizes, deduplicates.
- **Intelligence** — the only AI layer. One job: unstructured text → structured JSON.
  Strict output contract, temperature 0 for extraction.
- **Action & Presentation** — pure deterministic logic. Writes to sheet, creates tickets,
  renders dashboard. No AI.

This separation is a deliberate constraint, not an implementation detail: it's what makes
the system auditable when something goes wrong, and it's the core thesis of
`pm-guidelines.md` — "AI is one node in a deterministic pipeline, not a magic box."

## 7. AI System Specification

### Model & inference settings
| Setting | Value | Why |
|---|---|---|
| Model | `claude-sonnet-4-6` | Current production model at time of build |
| Temperature — extraction/scoring | `0.0` | Deterministic task (formula application, categorical rules) — any variance is a bug |
| Temperature — generative fields (`recommended_action`, Jira ticket body) | `0.3–0.5` (recommended; current single-call implementation runs everything at 0 — see Known Gaps) | Generative tasks benefit from some variance; forcing temp 0 everywhere is a simplification made for the v1 single-call pipeline |
| Max tokens | 1024 | Output rarely exceeds ~600 tokens; headroom without cost bleed |
| Timeout | 30s | Typical response 3–8s; wide safety margin |

### Output contract (the schema is the product)
Two schema files define the contract, per `pm-guidelines.md` §1 (Schema-First Design):
- `schema/account-risk-baseline.json` — the extraction-layer contract (what Claude outputs
  per raw signal): `account_name`, `arr`, `renewal_horizon_days`, `financial_risk_score`,
  `revenue_at_risk`, `technical_domain`, `bug_severity`, `sentiment`, `escalation_flag`,
  `bug_summary`, `cs_owner`, `recommended_action`, `engineering_ticket_ref`, `source_channel`
- `schema/account-risk.json` — the unified per-account record (aggregates multiple issues
  per account, adds `churn_risk_score`, `churn_risk_tier`, `executive_summary`)

**Contract rule:** every field name must match exactly across the Claude prompt, the
Google Sheet column headers, and the Lovable dashboard. A mismatch fails silently — a blank
column, not an error. Schema changes are versioned like an API — prompt, sheet, and
frontend update together or not at all.

### Prompt inventory
Four distinct prompts, each scoped to one job (`prompts/claude-prompts.md`,
`prompts/claude-system-prompt-v2.md`):

1. **Slack extraction** — raw message → partial `AccountRisk` fields, nulls for anything
   not explicitly stated
2. **Risk scoring** — structured inputs → `churn_risk_score` (weighted: days open, severity,
   days-to-renewal, escalation count, sentiment)
3. **Sheets normalization** — messy spreadsheet row → clean JSON (handles `"$500K"` →
   `500000`, `"Sept 15"` → `2026-09-15`, etc.)
4. **Jira ticket body generator** — risk data → plain-text ticket body (the one prompt that
   is intentionally generative, not extractive — output is prose, not JSON)

### Guardrails baked into the prompts (not left to hope)
- **Null discipline:** "never guess or hallucinate" is stated explicitly in every extraction
  prompt. This is called out in `pm-guidelines.md` as *the* most important guardrail — a
  hallucinated ticket ID routes an engineer to a ticket that doesn't exist.
- **Output-only-JSON contract:** every structured prompt states "no markdown, no
  explanation, no code fences" — because `JSON.parse()` fails hard the moment Claude adds
  a sentence of preamble.
- **Formula-in-prompt vs. formula-in-code tension (explicitly named, not resolved):**
  `financial_risk_score` is currently computed *by Claude* inside the prompt. The prompt's
  own PM notes (`claude-system-prompt-v2.md`) flag this as a conscious trade-off: keeping
  the formula in one place (the prompt) is simpler, but for an audit-critical system the
  better long-term architecture is Claude extracts raw facts only, and an n8n Code node
  computes the score — because code is testable and traceable, a prompt's internal
  reasoning is not. This is tracked as a known architectural debt, not an oversight.

### Known gaps (intentional, tracked)
- Single Claude call currently handles both extraction (should be temp 0) and generation
  of `recommended_action` (would ideally run at temp 0.3–0.5 in a separate call) — v1
  simplification, not yet split.
- No inbound webhook authentication (see `docs/pm-decision-log.md`, Step 2) — required
  before this pipeline is reachable outside `localhost`.
- No context/iteration budget defined anywhere in this system, because nothing here is a
  multi-step agent loop yet — see `docs/loop-engineering-worksheet.md` for where that will
  first become relevant (Workflow 3, and any future Iteration 3/4 feedback loop).

## 8. Scope by Iteration

(Full detail in `docs/build-guide.md`)

| Iteration | Scope | Status |
|---|---|---|
| 1 | Prove intelligence layer: manual signal submission → Claude → Sheets → dashboard | In progress (this pilot) |
| 2 | Automated ingestion: Slack bot auto-submits from `#cs-enterprise` | Not started |
| 3 | Feedback loop: track outcomes, "mark as resolved," validate the score formula against real churn | Not started — first genuine candidate for loop-engineering treatment (autonomous, ongoing, needs a stopping condition and stakes-classified actions) |
| 4 | Predictive scoring: replace formula with a trained model once 6+ months of outcome data exists | Not started |

## 9. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Schema drift between prompt/sheet/frontend | Treat schema changes as API version bumps — update all three consumers together (`pm-guidelines.md` §1) |
| Silent pipeline failures | Validation node before every DB write; n8n error workflow posts to Slack on any node failure |
| Unbounded autonomous loop (Workflow 3 and beyond) | `docs/loop-engineering-worksheet.md` — stopping conditions, tool audits, stakes-classified actions, kill-switch |
| Un-authenticated public webhook | Flagged, not yet fixed — must close before tunneling beyond localhost |
| AI computing business logic invisibly | Formula-in-prompt vs. formula-in-code tension tracked explicitly above, not hidden |

## 10. Open Questions

- Formula placement: move `financial_risk_score` computation from prompt to n8n Code node
  before or after Iteration 1 sign-off?
- Data store: graduate from Google Sheets to Airtable/Supabase at what trigger point (row
  count? concurrent-write conflicts? both are mentioned as valid next steps in
  `docs/architecture.md`)?
- Who validates webhook authenticity once this leaves `localhost` — see
  `docs/pm-decision-log.md` Step 2?
