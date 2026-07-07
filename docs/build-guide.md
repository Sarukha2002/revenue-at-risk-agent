# Step-by-Step Build Guide — Revenue-at-Risk Containment System
# Synthesized from the framework steps + PM-layer enhancements

---

## Overview: What You're Building

A 4-node agentic pipeline that transforms raw CS Slack messages into a prioritized BI dashboard showing enterprise accounts sorted by dollar revenue at risk.

**Input:** Unstructured CS feedback ("Acme is furious, SSO broken for 3 weeks, $500K account")  
**Output:** Structured risk record → Google Sheets row → Lovable dashboard card

**Time to build:** 2–4 hours  
**Cost to run:** ~$0.002 per submission (Claude Sonnet pricing)

---

## Step 1: Define the JSON Schema (The Contract)

**File:** `/schema/account-risk-baseline.json`

This is the invisible contract between Claude, your database, and your UI. Every field name in this schema must match exactly across all three layers.

**Key fields for v1:**
```json
{
  "account_name": "string",
  "arr": "number (USD integer)",
  "renewal_horizon_days": "integer",
  "financial_risk_score": "number (0–100)",
  "revenue_at_risk": "number (USD)",
  "technical_domain": "Frontend | API | Data Pipeline | Authentication | Database | Infrastructure | Integrations | Unknown",
  "bug_severity": "P0 | P1 | P2 | P3 | null",
  "days_issue_open": "integer | null",
  "sentiment": "Critical | Frustrated | Concerned | Neutral | Satisfied",
  "escalation_flag": "boolean",
  "bug_summary": "string (max 200 chars)",
  "cs_owner": "string | null",
  "recommended_action": "string",
  "engineering_ticket_ref": "string | null",
  "source_channel": "slack | google_sheets | manual_form | jira"
}
```

**PM note:** Set these as your Google Sheet column headers (Row 1) before building anything else. The order matters — it determines column position in your dashboard.

---

## Step 2: Set Up the AI Engine (Claude System Prompt)

**File:** `/prompts/claude-system-prompt-v2.md`

**In n8n, configure the Anthropic node as follows:**

**Model:** `claude-sonnet-4-6`  
*(Not claude-3-5-sonnet — that's an older model. Use the current production model.)*

**Temperature:** `0`  
*(Deterministic extraction — no creativity needed here.)*

**Max tokens:** `1024`

**System prompt:** Paste the full prompt from `/prompts/claude-system-prompt-v2.md`

The prompt instructs Claude to:
1. Calculate `financial_risk_score` using the formula:
   ```
   score = ((180 - renewal_horizon_days) / 180) * 100
   If renewal_horizon_days > 180 → use multiplier 0.2 → score = 20
   ```
2. Categorize `technical_domain` from symptom keywords
3. Infer `sentiment` and `bug_severity` from CS language
4. Output ONLY raw JSON — no markdown, no explanation

**PM enhancement over the base guide:** Add temperature: 0 explicitly. The base guide omits this — without it, n8n uses Claude's default which introduces non-determinism in structured extraction.

---

## Step 3: Build the Agentic Pipeline in n8n

**File:** `/n8n/workflow-revenue-at-risk.json` — import this directly into n8n

The workflow is 6 nodes (base guide has 3 — the extra 3 make it production-ready):

```
[Webhook] → [Validate & Normalize] → [Claude] → [Parse & Validate Output] → [Google Sheets] → [Respond to Frontend]
                                                          ↓
                                                   [Error Response]
```

### Node 1: Intake Webhook
- Type: Webhook (POST)
- Path: `/revenue-at-risk`
- Response mode: Response Node (allows custom response after pipeline completes)
- This URL goes into your Lovable frontend's fetch() call

### Node 2: Validate & Normalize (PM addition — not in base guide)
- Type: Code (JavaScript)
- Validates required fields are present
- Normalizes ARR strings: `"$500K"` → `500000`
- **Why this matters:** If Claude receives `"arr": "$500K"` as a string instead of `500000` as a number, it may compute `financial_risk_score` incorrectly or return null. Normalize before the AI sees it.

### Node 3: Claude via HTTP Request
- Type: HTTP Request (POST)
- URL: `https://api.anthropic.com/v1/messages`
- Auth: HTTP Header Auth (Name: `x-api-key`, Value: your Anthropic API key)
- Headers: `anthropic-version: 2023-06-01`, `content-type: application/json`
- Body: JSON with model, system prompt, temperature: 0, max_tokens: 1024, and user message template

**n8n user message field (map these from previous node):**
```
Account Name: {{ $json.account_name }}
Annual Contract Value (ARR): {{ $json.arr }}
Days to Renewal: {{ $json.renewal_horizon_days }}
Source: {{ $json.source_channel }}

Raw CS Feedback:
{{ $json.raw_feedback }}
```

### Node 4: Parse & Validate Claude Output (PM addition)
- Type: Code (JavaScript)
- Strips accidental markdown fences from Claude's response
- `JSON.parse()` in try/catch with descriptive error
- Adds `risk_tier` bucket (Critical/High/Medium/Low)
- Adds `last_updated` timestamp
- Caps `revenue_at_risk` at `arr` (safety check)

### Node 5: Google Sheets
- Type: Google Sheets (Append row)
- Map each `{{ $json.field_name }}` to its column header
- Sheet must have column headers matching schema field names exactly

### Node 6: Respond to Webhook
- Returns `{ success: true, financial_risk_score, revenue_at_risk, risk_tier }` to Lovable
- Lovable uses this to show an immediate score to the CS manager after submission

---

## Step 4: Build the Frontend (Lovable.dev)

**File:** `/lovable-prompt.md` — paste this entire prompt into Lovable

The Lovable prompt generates two views:

**Intake Portal:**
- Account Name, ARR, Days to Renewal, Source dropdown, CS Feedback textarea
- "Submit to AI Agent" button → POST to n8n webhook URL
- Loading state: "AI is analyzing account risk..."
- Success: shows returned risk score

**Risk Dashboard:**
- 4 KPI cards: Total Revenue at Risk, Critical Accounts, High Risk Accounts, Avg Days Open
- Sortable table: Account, ARR, Risk Score (colored badge), Technical Domain, Sentiment, Days Open, Revenue at Risk, Escalation flag, Recommended Action
- Row colors: Critical = red left border, escalation_flag = 🚨
- Expandable rows: show full bug_summary and recommended_action
- Filter bar: by Technical Domain, Risk Tier, Account Name search
- Auto-refreshes every 60 seconds from n8n dashboard data webhook
- "Download CSV" button

**PM enhancement over the base guide:** The base guide's Lovable prompt is minimal (2 views, basic table). The enhanced version adds: KPI cards, expandable rows, filter bar, CSV export, risk score color coding, escalation flag indicators, loading/error/empty states, and mobile responsiveness. This is the difference between a prototype and a tool CS managers will actually use daily.

---

## Step 5: Wire Dashboard Data (Second n8n Workflow)

The dashboard needs to read all rows from Google Sheets and return them as JSON. Build this simple 3-node workflow in n8n:

```
[Webhook GET /dashboard-data] → [Google Sheets Read All] → [Respond with JSON Array]
```

- Webhook: GET, path `/dashboard-data`  
- Google Sheets: Read all rows from RiskData sheet  
- Respond: `{{ JSON.stringify($json) }}`

Set this webhook URL as `VITE_DASHBOARD_WEBHOOK_URL` in Lovable's environment.

---

## Google Sheets Setup

Create a new Google Sheet with one tab named `RiskData`. Row 1 headers (copy exactly):

```
account_name | arr | renewal_horizon_days | financial_risk_score | revenue_at_risk | risk_tier | technical_domain | bug_severity | days_issue_open | sentiment | escalation_flag | bug_summary | cs_owner | recommended_action | engineering_ticket_ref | source_channel | last_updated
```

---

## Testing the Pipeline

**Test payload for the webhook:**
```json
{
  "account_name": "Acme Corp",
  "arr": 500000,
  "renewal_horizon_days": 77,
  "raw_feedback": "Just got off the phone with Acme — they are FURIOUS. SSO has been broken for 3 weeks now and their entire team can't log in. This is a $500K account and they're seriously considering leaving. We need engineering on this TODAY. ENG-4421 is just sitting there unassigned. Sarah here — 4th time escalating.",
  "source_channel": "slack"
}
```

**Expected output:**
- `financial_risk_score`: ~57.2
- `revenue_at_risk`: ~286000
- `technical_domain`: "Authentication"
- `bug_severity`: "P0"
- `sentiment`: "Critical"
- `escalation_flag`: true
- `risk_tier`: "High"

---

## What to Build Next (PM Roadmap)

| Priority | Feature | Why |
|----------|---------|-----|
| P0 | Slack bot auto-ingestion | CS managers won't manually submit — remove that step |
| P0 | Jira ticket auto-creation for Critical accounts | Closes the loop from signal to action |
| P1 | Dedup logic (same account submitted twice) | Prevents dashboard inflation |
| P1 | "Mark as Resolved" in dashboard | Track outcomes for eval |
| P2 | Email digest to VP Engineering (daily, sorted by revenue_at_risk) | Gets the right eyes on the data without login |
| P2 | Slack alert for new Critical records | Real-time ops response |
| P3 | Replace formula score with ML model | More accurate after 6 months of outcome data |
