# Lovable.dev Prompt — Revenue-at-Risk Dashboard
# Paste this entire prompt into Lovable.dev to generate the frontend

---

## Prompt to paste into Lovable

```
Build a modern, enterprise-grade "Revenue-at-Risk Containment Dashboard" for a B2B SaaS Customer Success and Engineering leadership team. Use a dark professional color scheme (deep navy/slate background, white text, with red/amber/green for risk indicators). Use Tailwind CSS and React.

The app has two views navigable via a top tab bar: "Intake Portal" and "Risk Dashboard".

---

VIEW 1: INTAKE PORTAL

A clean form for CS Managers to submit account risk signals. Include:

Header: "Submit Account Risk Signal" with subtitle "AI will extract risk data and score this account automatically."

Form fields:
- Account Name: text input with placeholder "e.g. Acme Corp"
- Annual Contract Value (ARR): number input with $ prefix, placeholder "e.g. 500000"
- Days to Renewal: number input, placeholder "e.g. 77"
- Source: a single-select dropdown with these exact separated options, each its own distinct clickable item (selecting one must not affect or visually merge with any other option): "Slack Message" (value "slack"), "Call Notes" (value "manual_form"), "Email Forward" (value "manual_form"), "Google Sheets Row" (value "google_sheets"), "Manual Entry" (value "manual_form")
- CS Client Feedback: large textarea (minimum 8 rows), placeholder "Paste the raw Slack message, email, or call notes here. The AI will extract the structured data automatically. Include account name, issue description, urgency signals, and any ticket references."
- Submit button: full-width, dark blue, label "Submit to AI Agent →"

On submit:
- Show a loading spinner with text "AI is analyzing account risk..."
- POST to the n8n webhook URL (store as VITE_N8N_WEBHOOK_URL env variable)
- Include header: "x-webhook-secret": VITE_WEBHOOK_SECRET (store as its own env variable — this is required or the webhook rejects the request with a 401)
- Request body: { account_name, arr: parseInt(arr), renewal_horizon_days: parseInt(days), raw_feedback, source_channel }
- On success: show a green success banner with the returned financial_risk_score and risk_tier, then clear the form
- On error: show a red error banner with the error message

---

VIEW 2: RISK DASHBOARD

Data is fetched from the Google Sheets webhook endpoint (store as VITE_DASHBOARD_WEBHOOK_URL, auto-refresh every 60 seconds). Include header "x-webhook-secret": VITE_WEBHOOK_SECRET on this fetch too — same requirement as the intake form, this endpoint rejects unauthenticated requests.

The response is an object with two arrays, already server-side aggregated — no client-side grouping needed:
```
{
  "accounts": [ { account_id, account_name, arr, contract_tier, renewal_date, days_to_renewal, cs_owner, issues_json, escalation_count, sentiment, churn_risk_score, churn_risk_tier, revenue_at_risk, recommended_action, executive_summary, jira_ticket_key, primary_technical_domain, top_issue_summary, last_synced }, ... ],
  "signals": [ { account_name, arr, renewal_horizon_days, financial_risk_score, revenue_at_risk, risk_tier, technical_domain, bug_severity, days_issue_open, sentiment, escalation_flag, bug_summary, cs_owner, recommended_action, engineering_ticket_ref, source_channel, last_updated, jira_ticket_key }, ... ]
}
```
`accounts` is one entry per account (already deduplicated and aggregated — churn_risk_score is 0–1, churn_risk_tier is lowercase critical/high/medium/low). `signals` is one entry per raw submitted complaint (same shape as before, risk_tier is Title Case, escalation_flag is a boolean).

This view has two sub-tabs: "Executive Summary" (default) and "Operational Detail". Executives land on the summary; CS/ops teams click into the operational table to actually work the queue. Use Recharts for any charts.

Use the `accounts` array for EVERYTHING in the Executive Summary sub-tab and for the 4 KPI cards at the top of Operational Detail. Use the `signals` array ONLY for the Operational Detail TABLE itself (the sortable row list further down) — that one intentionally shows every individual complaint, not one row per account.

--- SUB-TAB A: EXECUTIVE SUMMARY (default) ---

Designed for a Customer Success VP or exec skimming this for 30 seconds, not analyzing rows.

All data in this sub-tab comes from the `accounts` array (already one entry per account — do not group or dedupe anything yourself).

Hero section (top, full width, visually dominant — larger and bolder than anything else on the page):
- "Total Revenue at Risk" as a large number ($X.XM format) — sum revenue_at_risk across the accounts array, with a one-line subtext: "across N accounts"
- Do NOT show a week-over-week trend arrow or percentage change — we don't have historical snapshots yet, so don't fabricate a trend. Just the current total.

Below the hero — 3 smaller secondary metric cards in a row (visually subordinate to the hero):
1. "Critical Accounts" — count where churn_risk_tier = "critical", red accent
2. "Needs Escalation" — count where escalation_count > 0, amber accent
3. "Renewing in 30 Days" — count where days_to_renewal <= 30, blue accent

"Accounts Needing Attention" — a card list (NOT a table), showing the top 5 accounts by revenue_at_risk descending:
- Each card: Account Name (bold, large), Revenue at Risk ($ formatted, prominent), executive_summary as the one-line plain-English summary (this is a purpose-written 2-sentence leadership summary, not a truncated complaint — show it in full or truncate only if it overflows the card), a churn_risk_tier badge, and a small "View Details" link that switches to the Operational Detail tab filtered to that account
- In the corner of each card, the same Jira Ticket tag-or-button described in the Operational Detail table below (compact size) — same data source (jira_ticket_key), same "Create Jira Ticket" webhook call, same success/error behavior. Executives skimming this view can escalate a top-5 account directly from here without switching tabs.
- Small, low-emphasis text at the bottom of each card: "Updated {relative time}" computed from that account's last_synced field (e.g. "Updated 4m ago") — this just signals the data is kept fresh, don't use any internal jargon like "workflow" or "rollup" anywhere near it
- No other fields, no raw field names like "technical_domain" — keep this card visually clean and scannable

Two small breakdown charts side by side:
1. A horizontal bar chart: count of accounts per churn_risk_tier (critical/high/medium/low), color-coded to match badge colors elsewhere
2. A horizontal bar chart: revenue_at_risk summed per primary_technical_domain — label this chart with a business-friendly title like "Where Is Risk Concentrated?" rather than the raw field name. (primary_technical_domain is each account's dominant issue domain, not a full breakdown of every complaint — that's an intentional simplification, not a bug.)

Keep this entire sub-tab to one screen's worth of scrolling on desktop — this is a glance view, not a report.

--- SUB-TAB B: OPERATIONAL DETAIL ---

A real-time BI table showing all individual complaints sorted by revenue_at_risk descending. This is the detailed working view for CS/ops teams.

Top of dashboard — 4 KPI metric cards in a row. Cards 1–3 come from the `accounts` array (same as Executive Summary — an account with 2 complaint rows must not be double-counted here either). Card 4 is genuinely about individual issues, not accounts, so it uses `signals`:
1. "Total Revenue at Risk" — sum of accounts[].revenue_at_risk, formatted as $X.XM or $XXXk
2. "Critical Accounts" — count where churn_risk_tier = "critical", red background
3. "High Risk Accounts" — count where churn_risk_tier = "high", amber background
4. "Avg Days to Resolution" — average of signals[].days_issue_open across all rows, blue background

Below the KPIs — a sortable data table with these columns:
- Account Name (bold)
- ARR (formatted as $XK or $XM)
- Risk Score (0–100, shown as a colored badge: ≥80 red, 60–79 amber, 40–59 yellow, <40 green)
- Risk Tier (pill badge: Critical=red, High=amber, Medium=yellow, Low=green)
- Technical Domain (pill badge with icon: Authentication=🔐, API=⚡, Data Pipeline=📊, Frontend=🖥, Database=🗄, Infrastructure=⚙, Integrations=🔗)
- Sentiment (Critical=🔴, Frustrated=🟠, Concerned=🟡, Neutral=⚪, Satisfied=🟢). Add a small ⓘ icon next to this column's header — click or hover shows a short tooltip: "Color legend: 🔴 Critical (furious/threatening to leave) · 🟠 Frustrated (very unhappy) · 🟡 Concerned (worried, asking for updates) · ⚪ Neutral (no strong emotion) · 🟢 Satisfied (happy/positive)."
- Days Open (red if >14, amber if 7-14, green if <7). Add a small ⓘ icon next to this column's header — tooltip: "Number of days since this issue was first reported and is still unresolved."
- Revenue at Risk ($formatted, bold red if >$100k). Add a small ⓘ icon next to this column's header — tooltip: "Revenue at Risk = Annual Contract Value × Risk Score. Risk Score is driven primarily by how soon the account renews — the closer the renewal date, the higher the score, since time-to-renewal is the dominant churn signal."
- Escalation (🚨 icon if escalation_flag = true, empty if false)
- Recommended Action (truncated to 80 chars with tooltip on hover showing full text)
- Jira Ticket — the ONLY ticket column (do not add a separate "Eng Ticket" column). Look up this row's account_name in the `accounts` array and use THAT entry's jira_ticket_key, not anything on the signal row itself (signals also carries its own jira_ticket_key field, but the accounts array is the authoritative one now — use it for consistency). Every signal row for the same account must show the same state.
  - If that account's jira_ticket_key IS present: show a clickable tag/pill with the ticket key (e.g. "KAN-123") linking to https://sarukha2.atlassian.net/browse/{jira_ticket_key}, opening in a new tab. No button.
  - If NOT present: show a small "Create Jira Ticket" button instead. On click:
    - Show a small inline loading spinner on the button (disable it, don't lock the whole row)
    - POST to the webhook URL stored as VITE_CREATE_TICKET_WEBHOOK_URL, with header "x-webhook-secret": VITE_WEBHOOK_SECRET, body { account_name }
    - On success (response has jira_ticket_key): replace the button with the same clickable tag on EVERY row for that account_name (not just the one clicked), using the key from the response — no full dashboard refetch needed, just update local state
    - On error: show a small red inline message next to the button ("Failed to create ticket — retry") and keep the button clickable again
- Last Updated (relative time: "2h ago", "Just now")

If a row's engineering_ticket_ref is not null, do NOT give it its own column — show it as small secondary text in the row's expanded detail view instead (it's just an informational mention extracted from the raw complaint text, not an actionable ticket).

Row behavior:
- Rows where risk_tier = "Critical" have a left red border accent and subtle red row background
- Rows where escalation_flag = true have a 🚨 in the leftmost position
- Clicking a row expands it inline to show: full bug_summary, full recommended_action, cs_owner, source_channel
- Table is sortable by clicking column headers (default sort: revenue_at_risk descending)
- Add a filter bar above the table: filter by Technical Domain (multi-select), Risk Tier (multi-select), and a text search on Account Name. The search input needs a small "×" clear button that appears at its right edge once text is typed, which clears the input and refocuses it on click.

---

ADDITIONAL REQUIREMENTS:

- Empty states: if no data, show "No accounts at risk — great news! Submit a signal from the Intake Portal." with a checkmark illustration
- Loading states: skeleton loader rows while data fetches
- Error states: "Dashboard data unavailable" with a retry button
- Mobile responsive: stack the KPI cards 2x2 on mobile, make table horizontally scrollable
- Add a "Download CSV" button above the table that exports the current filtered view
- Navigation header: show "Risk Guardian" as the logo text (with subtitle "Revenue Containment Console" beneath it, smaller/muted) on the left, current user avatar placeholder on the right
- No authentication required for this MVP — anyone with the URL can access

---

TECH NOTES:
- Use fetch() for all API calls, not axios
- Store webhook URLs as environment variables: VITE_N8N_WEBHOOK_URL, VITE_DASHBOARD_WEBHOOK_URL, and VITE_CREATE_TICKET_WEBHOOK_URL
- Use date-fns for relative time formatting
- Tailwind for all styling — no CSS files
- No backend needed — all data comes from n8n webhooks
```

---

## After Lovable generates the app

### Connect the n8n webhook URLs
In Lovable's environment settings, add:
```
VITE_N8N_WEBHOOK_URL=https://your-n8n-instance.com/webhook/revenue-at-risk
VITE_DASHBOARD_WEBHOOK_URL=https://your-n8n-instance.com/webhook/dashboard-data
VITE_CREATE_TICKET_WEBHOOK_URL=https://your-n8n-instance.com/webhook/create-jira-ticket
VITE_WEBHOOK_SECRET=your-webhook-secret-value
```
All three webhooks are protected with Header Auth (`x-webhook-secret`) — every request from
Lovable must include this header or n8n returns a 401 "Authorization data is wrong!" error.

Note: the current tunnel URL is ephemeral (Cloudflare Quick Tunnel) and will change if the tunnel
restarts — update all three `VITE_*_WEBHOOK_URL` values in Lovable whenever that happens, until a
stable domain is set up.

### Dashboard data webhook (n8n Workflow 4)
Triggers on GET to `/webhook/dashboard-data`, reads both the `RiskData` and `AccountRollup`
sheets, and returns `{ accounts: [...], signals: [...] }` (see the exact shape documented above
in the prompt). Already built — see `n8n/workflow-dashboard-data.json`.

### Create Jira ticket webhook (n8n Workflow 3)
Triggered by the per-issue "Create Jira Ticket" button described above. Triggers on POST to
`/webhook/create-jira-ticket`, body `{ account_name }`. Looks up that account in `RiskData`, runs
an AI agent that checks Jira for an existing ticket and creates one (or posts an informational
comment on an existing one) in project `KAN`, writes the resulting `jira_ticket_key` back to the
matching `RiskData` row, triggers Workflow 5 to refresh that account's rollup, and responds with
`{ success, account_name, jira_ticket_key, action_taken, summary }`. Already built and live — see
`docs/loop-engineering-worksheet.md` §5.

### Account rollup (n8n Workflow 5 — new, not exposed to the frontend directly)
Not called by Lovable at all — triggered internally by Workflows 1 and 3 whenever a signal is
submitted or a ticket is created. Reads every `RiskData` row for one account, aggregates them
into the `schema/account-risk.json` shape (one Claude call just for the `executive_summary`
field, everything else computed deterministically), and writes the result to the `AccountRollup`
sheet, which is what Workflow 4 reads for the `accounts` array above. Mentioned here only so it's
clear where `accounts[]`'s data actually comes from.

### Google Sheet column headers (Row 1)
Create these exact headers in your RiskData sheet:
```
account_name | arr | renewal_horizon_days | financial_risk_score | revenue_at_risk | risk_tier | technical_domain | bug_severity | days_issue_open | sentiment | escalation_flag | bug_summary | cs_owner | recommended_action | engineering_ticket_ref | source_channel | last_updated | jira_ticket_key
```
`jira_ticket_key` starts empty for every row — Workflow 3 fills it in the first time someone
clicks "Create Jira Ticket" for that account.

Create a second tab named `AccountRollup` with these headers — Workflow 5 populates it
automatically, you don't fill it in manually:
```
account_id | account_name | arr | contract_tier | renewal_date | days_to_renewal | cs_owner | issues_json | escalation_count | sentiment | churn_risk_score | churn_risk_tier | revenue_at_risk | recommended_action | executive_summary | jira_ticket_key | primary_technical_domain | top_issue_summary | last_synced
```
