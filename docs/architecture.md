# Revenue-at-Risk Containment System — Architecture

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                                  │
│                                                                       │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────┐              │
│  │  Slack      │   │ Google Sheets│   │ Jira/Linear │              │
│  │ #cs-team    │   │ (ARR + accts)│   │ (eng tickets│              │
│  │ #enterprise │   │              │   │              │              │
│  └──────┬──────┘   └──────┬───────┘   └──────┬──────┘              │
└─────────┼────────────────┼──────────────────┼────────────────────────┘
          │                │                  │
          ▼                ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      n8n WORKFLOW ENGINE                             │
│                                                                       │
│  Workflow 1: Slack Ingestion     Workflow 3: Ticket Enrichment       │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐    │
│  │ Slack Trigger           │    │ Schedule: every 15 min        │    │
│  │ → Filter CS channels    │    │ → Read master JSON            │    │
│  │ → Claude: Extract JSON  │    │ → Claude: Write ticket body   │    │
│  │ → Sheets: Lookup ARR    │    │ → Create/update Jira ticket   │    │
│  │ → Claude: Score risk    │    │ → Add $ impact comment        │    │
│  │ → Write to master DB    │    └──────────────────────────────┘    │
│  └─────────────────────────┘                                         │
│                                  Workflow 4: Dashboard API           │
│  Workflow 2: Sheets Sync         ┌──────────────────────────────┐    │
│  ┌─────────────────────────┐    │ Webhook: GET /dashboard       │    │
│  │ Schedule: hourly        │    │ → Read master JSON            │    │
│  │ → Read Google Sheets    │    │ → Aggregate by risk tier      │    │
│  │ → Claude: Normalize     │    │ → Sort by revenue_at_risk     │    │
│  │ → Merge into master DB  │    │ → Return JSON to Lovable      │    │
│  └─────────────────────────┘    └──────────────────────────────┘    │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼ Master JSON Store
                         ┌────────────────┐
                         │  n8n Static    │
                         │  Data / Airtable│
                         │  / Supabase    │
                         └───────┬────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      LOVABLE DASHBOARD                               │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Revenue at Risk: $4.2M        High Risk Accounts: 7         │   │
│  │                                                               │   │
│  │  ┌──────────┬──────────┬────────┬────────┬────────────────┐  │   │
│  │  │ Account  │ ARR      │ Risk   │ Days   │ Ticket         │  │   │
│  │  │          │          │ Score  │ Open   │                │  │   │
│  │  ├──────────┼──────────┼────────┼────────┼────────────────┤  │   │
│  │  │ Acme     │ $500K    │ 🔴 0.91│ 23     │ ENG-4421 [P0]  │  │   │
│  │  │ Globex   │ $320K    │ 🔴 0.87│ 18     │ ENG-4398 [P0]  │  │   │
│  │  │ Initech  │ $240K    │ 🟡 0.62│ 9      │ ENG-4412 [P1]  │  │   │
│  │  └──────────┴──────────┴────────┴────────┴────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Detail

### Step 1 — Raw Signal Collection
Slack messages from CS team channels are ingested in real time. No structure is assumed — messages may be: "Acme is escalating again, their SSO is broken for 3 weeks, they pay $500K", or "just got off a call with Globex — they're really unhappy with the API latency."

Google Sheets are polled hourly and contain the ground truth for: account names, ARR, contract renewal dates, CS owner names, and any manually logged issues.

### Step 2 — Claude Extraction
Every raw Slack message passes through Claude with a strict extraction prompt. Output is a partial `AccountRisk` JSON object. Missing fields are left null — not hallucinated.

### Step 3 — ARR Enrichment
The extracted `account` name is fuzzy-matched against the Google Sheet's account list to pull in: `arr`, `renewal_date`, `cs_owner`, `contract_tier`. This is deterministic — no AI involved.

### Step 4 — Risk Scoring
A second Claude call scores `churn_risk` (0.0–1.0) based on: days open, severity signals in the message, number of escalations, days until renewal, and ARR. This produces `revenue_at_risk = arr * churn_risk`.

### Step 5 — Ticket Enrichment
For each high-risk account (score > 0.6), n8n checks if a Jira ticket exists. If not, it creates one. Either way, it posts a comment: "Revenue at Risk: $320K — Renewal in 47 days — Churn Risk: 0.87."

### Step 6 — Dashboard Serve
The Lovable dashboard polls a Webhook URL every 60 seconds. n8n returns aggregated JSON sorted by `revenue_at_risk` descending.

---

## Tech Stack Decisions

| Layer | Tool | Why |
|-------|------|-----|
| Orchestration | n8n (self-hosted or cloud) | Visual workflow builder, native Slack/Sheets/Jira integrations, webhook support |
| AI | Claude Sonnet via API | Best-in-class unstructured text extraction with strict output contracts |
| Frontend | Lovable | Generates production-quality React dashboard from a single prompt |
| Data store | n8n Static Data → Airtable or Supabase | Start with n8n static, graduate to Airtable (no-code) or Supabase (SQL) |
| Ticket system | Jira (or Linear) | Native n8n nodes available for both |

---

## n8n Setup Requirements

1. n8n instance (cloud at n8n.cloud or self-hosted via Docker)
2. Credentials configured in n8n:
   - Slack OAuth app with `channels:history`, `channels:read` scopes
   - Google Sheets OAuth service account
   - Anthropic API key (HTTP Header Auth node)
   - Jira API token (or Linear API key)
3. Four workflows imported from `/n8n/` directory in this repo
