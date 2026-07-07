# Loop Engineering Worksheet — Workflow 3 (Ticket Enrichment)

Applying Marily's "loop engineering" framework to the one part of Revenue-at-Risk that
actually behaves like a loop: a schedule-triggered process that runs forever, unattended,
making decisions (create ticket / update ticket / comment) without a human in the room.

The intake pipeline (Workflow 1) is a single Claude call per request — it doesn't need this
worksheet. Workflow 3 does, because it runs every 15 minutes indefinitely and nobody is
watching each cycle.

Items marked `[TODO — verify]` are things this worksheet deliberately does NOT assume an
answer for. Per the "audit your tools" rule: test them standalone before trusting them.

---

## 1. Stopping condition

A recurring workflow doesn't have a final "done" — but each *run* of it does. Define
success per-run, not for the workflow's lifetime.

**Per-run success (binary, measurable, checked by the workflow itself):**
- Every account present in the master data store at run start was evaluated (`accounts_processed == accounts_in_store`)
- Every account with `financial_risk_score > 60` ends the run with exactly one open Jira ticket
- Every such ticket carries a comment updated within this run (`$ impact`, renewal days, churn risk — not stale from a prior run)
- Zero unhandled parse errors from the Claude ticket-body generation step

Add a final Code node in Workflow 3 that asserts these counts and routes any mismatch to
the error workflow / Slack alert — not just a log line. Silent count mismatches are exactly
the "mostly working" failure Marily describes.

**Workflow-level kill condition (not in the current design — add this):**
If 3 consecutive scheduled runs fail the per-run assertion above, disable the workflow
(`n8n workflow deactivate` via API, or a manual flag checked at the top of the run) and
send a distinct "Workflow 3 auto-disabled" Slack alert. Otherwise a broken run state
just keeps re-firing every 15 minutes, silently producing bad tickets or bad comments
for hours before anyone notices — the fintech "zero flags for two weeks" story from the
post, just running on a cron instead of a single long execution.

---

## 2. Tool inventory

| Tool / call | Used for | Rate limit | Failure behavior | Idempotent? |
|---|---|---|---|---|
| Master data read (Airtable/Supabase/n8n static data) | Load current accounts + scores | `[TODO — verify against actual store chosen]` | `[TODO — test: what does a timed-out read return in n8n? Empty array or error?]` | Read-only, N/A |
| Claude API — ticket body generation | Draft ticket description text (generative, not extraction — should run at temp 0.3–0.5 per `pm-guidelines.md` rule 3, not temp 0) | Standard Anthropic API limits for your tier | `[TODO — test: does the Code node distinguish a 429 from a malformed response? Does it retry or fail the whole run?]` | N/A (drafting text) |
| Jira — check ticket exists | Dedup check before create | `[TODO — verify your Jira instance's API rate limit]` | `[TODO — test: what happens if this call 503s — does the workflow then create a duplicate ticket?]` | Read-only, N/A |
| Jira — create ticket | New ticket for newly-qualifying accounts | same as above | `[TODO — test standalone: retry-safe, or does a retry after a false-negative "exists" check create a dupe?]` | **Not idempotent as designed** — see §3 |
| Jira — add/update comment | Post `$ impact` line | same as above | `[TODO — test: comment on a since-deleted or closed ticket — does it error or silently no-op?]` | Should be — same comment content posted twice is harmless, but confirm it doesn't append duplicates each 15-min cycle |

Test each of these **standalone in n8n** (not as part of a full run) before trusting the
workflow: kill the network mid-call, feed it a malformed response, hit it twice fast.
That's the "boring work" the post says most teams skip.

---

## 3. Stakes map

| Action | Reversible? | Customer-facing? | Stakes | Current design | Recommendation |
|---|---|---|---|---|---|
| Create Jira ticket | Yes (can be closed) | No | Low–Medium | Autonomous (score > 0.6) | Fine as-is — this is the intended "ambient intelligence surfaces signal" pattern from `pm-guidelines.md` §10 |
| Post `$ impact` comment | Yes | No | Low | Autonomous | Fine as-is |
| Update/re-open an existing ticket | Partially — can override a human's manual triage | No | **Medium** | Not explicitly designed — architecture doc says "either way, it posts a comment," implying it may touch tickets a human already closed or re-prioritized | Add a guard: skip auto-update if the ticket was manually modified (priority, status) more recently than the bot's last write. Otherwise the bot can silently undo an engineer's judgment call. |
| (Roadmap, not yet built) Auto-resolve/close a ticket when score drops | No — closes visibility into a case that may reopen | No | **High** | Not built | When you build the "mark as resolved" feedback loop (Iteration 3), this should NOT be autonomous — require explicit human confirmation, same logic as the post's "no autonomous trades, ever" example |

---

## 4. Success + guardrail pairing

Write both halves — a workflow that hits the goal by breaking the guardrail is a silent failure, not a success.

```
Success = (every score>60 account has a Jira ticket, comment updated within this run cycle)
          AND (zero duplicate tickets per account)
          AND (no ticket whose priority/status was manually set since the bot's
               last write is overwritten by this run)
          AND (workflow auto-disables + alerts after 3 consecutive failed runs,
               rather than continuing to fire every 15 minutes against a broken state)
```

---

## What to actually do with this

- [ ] Fill in every `[TODO — verify]` cell by testing that tool standalone in n8n
- [ ] Add the per-run assertion Code node + kill-switch described in §1
- [ ] Add the "don't overwrite manual edits" guard described in §3 before this goes to production
- [ ] When Iteration 3 (feedback loop / "mark as resolved") is built, re-run this same worksheet for that workflow — it's the next real loop candidate, and per §3 its auto-resolve action should require a human click, not run autonomously

---

## 5. Implementation design (as actually built and deployed)

Everything above this line is the framework applied on paper before Workflow 3 existed. This
section describes what's actually running: n8n workflow id `DC7KMPDGHq6aa569`
("Revenue-at-Risk — Workflow 3: Ticket Enrichment"), built once a real Jira credential
("Jira SW Cloud account", Jira Cloud site `sarukha2.atlassian.net`, project `KAN`) existed to
test against, per §2's own "test each tool standalone before trusting it" rule. Verified live on
2026-07-06: a real execution created real tickets in project `KAN` for qualifying accounts.

Two design decisions changed from the original paper design below, both deliberately, both
recorded here rather than left as silent drift between doc and code (the same mistake the
Workflow 2 naming collision was, worth not repeating):

1. **The loop uses n8n's native AI Agent node, not a hand-rolled Code-node ReAct loop.**
   n8n ships a real LangChain-based `AI Agent` node (`@n8n/n8n-nodes-langchain.agent`) with a
   built-in bounded tool-calling loop (`options.maxIterations`, set to **5** here) and native
   `ai_tool` connections to sub-nodes. This is more battle-tested than hand-rolling the same
   ReAct loop in a Code node calling the raw Anthropic HTTP API. The trade-off: its stopping
   condition is "the model stops calling tools and returns text" (bounded by `maxIterations` as
   the hard cap), not the originally-designed explicit `finish_ticket_enrichment` tool as a
   positive exit signal. In practice this has been fine — the system prompt instructs the agent
   to respond with a one-sentence summary and stop once its single action is done, and
   `maxIterations` is the deterministic, code-enforced backstop if it doesn't.
2. **The manual-edit guardrail is content-restricted, not status-gated.** The original design
   skipped commenting entirely once a ticket left "To Do" (any human touch silenced the bot).
   In practice this meant "did we make progress" tickets — the exact case the bot should keep
   informed about — went dark. The current design (decided 2026-07-06) instead lets the agent
   comment on any **non-closed** ticket (`To Do` or `In Progress`), but the comment content is
   constrained to be **strictly informational** — current $ revenue at risk, renewal horizon,
   risk level, nothing else. It must never suggest an action, never reference or react to the
   ticket's status/priority/assignee, and never imply agreement or disagreement with how a human
   is handling it. Only `Done` (closed) tickets are skipped outright. This is the same underlying
   principle as before — the bot doesn't get to make judgment calls that belong to a human — just
   drawn at the *content* boundary instead of the *status* boundary, because silence on an
   in-progress ticket was worse than a data-only comment.
3. **The trigger is an on-demand webhook fired from the dashboard, not a Schedule Trigger.**
   (Decided 2026-07-06.) The original design ran on a 15-minute schedule against every
   score>60 account in bulk — genuinely unattended, which is exactly why §1's per-run assertion
   and 3-strikes kill switch existed as safety nets. The revised design instead exposes a
   webhook that fires once per click of a "Create Jira Ticket" button next to a specific issue
   in the Lovable dashboard: a human looks at one issue and explicitly decides to escalate it,
   one at a time. This is arguably a *better* fit for `pm-guidelines.md` §10's "AI does triage,
   humans make decisions" than the original bulk design was — there's no unattended loop left to
   put a kill switch on. The trade-off: nothing currently *forces* every qualifying account to
   get a ticket the way the scheduled version would have — if nobody clicks the button, nothing
   happens. That's an intentional product choice for now (surfacing candidates is the
   dashboard's job; escalating them is a human's), not an oversight.

### Outer workflow shape (all native n8n nodes)

1. **Create Jira Ticket Webhook** (Webhook Trigger, POST `/webhook/create-jira-ticket`, Header
   Auth `x-webhook-secret`) — fired by the dashboard's per-issue button, body `{ account_name }`
2. **Read RiskData** (Google Sheets, same `RiskData` tab Workflow 1 writes to)
3. **Collapse & Filter Accounts** (Code node) — despite the name, no longer filters by score;
   scopes the read down to the one `account_name` given in the webhook body (throws if not
   found), collapses duplicate rows for that account (max `financial_risk_score` wins — inline
   stand-in for the descoped `schema/account-risk.json` per-account rollup, see README), and
   precomputes deterministic `account_label` (`acct-<slug>`) and `severity_label`
   (`sev-<bug_severity>`) that the agent is instructed to use exactly as given, never invent.
   The `> 60` threshold that used to gate this step is gone — a human clicking the button for a
   specific issue *is* the escalation decision now, not a score.
4. **Enrichment Agent** (`@n8n/n8n-nodes-langchain.agent`) — the reasoner, detailed below.
   `options.returnIntermediateSteps: true` so the real tool outputs (not just the agent's
   free-text summary) are available downstream.
5. **Extract Ticket Key** (Code node) — walks `intermediateSteps` to pull the actual Jira issue
   key out of whichever tool ran (`Create Jira Ticket`'s response `.key`, or the first match from
   `Search Jira Ticket`'s response), rather than trusting the agent's prose to contain it
   correctly.
6. **Update RiskData Row** (Google Sheets, `appendOrUpdate` matched on `account_name`) — writes
   `jira_ticket_key` back to that account's row, so Workflow 4's dashboard read surfaces it with
   no changes needed to Workflow 4 itself (it's a raw passthrough of every column).
7. **Respond to Dashboard** — returns `{ success, account_name, jira_ticket_key, action_taken,
   summary }` so the button's click handler can update that row's UI immediately.

No per-run assertion or kill switch in this shape — see "Known gaps" below for why that's a
lower-priority gap now than it was under the scheduled-bulk design.

### Tools exposed to the Enrichment Agent

All three are **native n8n Jira Software Tool nodes** (`n8n-nodes-base.jiraTool`), not raw HTTP
calls — this matters operationally: an earlier attempt used the generic LangChain
`toolHttpRequest` node and it failed every invocation with `has a "supplyData" method but no
"execute" method"`, even after enabling `N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true` in
`docker-compose.yml` (kept enabled regardless — harmless, and the Enrichment Agent node itself
still surfaces a related lint warning). Switching to the first-party `jiraTool` node (same
`n8n-nodes-base` package as the Google Sheets node, which never had this problem) resolved it
immediately. Lesson for future tool nodes: prefer a dedicated first-party Tool-variant node over
a generic HTTP tool node when one exists for the target service.

| Tool node | Operation | Purpose |
|---|---|---|
| `Search Jira Ticket` | `issue` / `getAll`, JQL `project = KAN AND labels = "<account_label>"` | Dedup check — always called first |
| `Create Jira Ticket` | `issue` / `create`, project `KAN`, issue type `Task` | New ticket when none exists |
| `Add Jira Comment` | `issueComment` / `add` | Informational-only update on a non-closed existing ticket |

`account_label` and `severity_label` are passed into each tool via n8n's `$fromAI()` expression
binding (the model fills them in as tool-call arguments), sourced from the deterministic values
step 3 precomputed — the agent is instructed never to invent its own label.

Temperature: **0.3** on the Anthropic Chat Model node, not 0 — per §2's original note, this
loop's generative work (ticket description, comment text) is drafting, not extraction, so a
small amount of variability is correct here even though it would be a bug in Workflow 1's
extraction call.

### Enrichment Agent system prompt (as deployed)

```
You are the Ticket Enrichment agent for the Revenue-at-Risk system. For the ONE account
described in the user message, ensure exactly one Jira ticket in project KAN correctly
reflects its current risk. Follow these steps in order:

1. ALWAYS call search_jira_ticket_by_label first, using the exact account_label given.
   Never skip this step and never guess whether a ticket exists.
2. If no ticket is found: call create_jira_ticket with a clear summary (include the account
   name and technical domain), a full description covering the $ revenue at risk, renewal
   horizon, technical domain, bug summary, and recommended action, and both the account_label
   and severity_label given, exactly as provided.
3. If a ticket already exists: check its status from the search result.
   - If status is 'Done' (or otherwise closed/resolved), do NOT comment — the issue is
     resolved from a human standpoint. Just report that you're skipping it because it's
     already closed.
   - Otherwise (status is 'To Do', 'In Progress', or anything not closed): call
     add_jira_comment with a STRICTLY INFORMATIONAL update — current $ revenue at risk,
     renewal horizon, and risk level, and nothing else. This comment must never suggest an
     action, never tell anyone what to do next, never reference or react to the ticket's
     current status/priority/assignee, and never imply agreement or disagreement with how a
     human is handling it. It is a data refresh, not a directive — a human already owns the
     next step once a ticket exists.
4. NEVER attempt to resolve, close, delete, reopen, or change the status/priority of any
   ticket. No tool exists here for that on purpose — do not try to work around it via
   comments or any other means.
5. NEVER create more than one ticket for the same account_label. If
   search_jira_ticket_by_label returns any match, you MUST NOT call create_jira_ticket.
6. Once you've completed the single appropriate action (create, comment, or skip) for this
   account, respond with a one-sentence plain-text summary of what you did and why, and stop
   — do not call further tools.
```

### What is explicitly NOT in this loop

Per §3's High-stakes row: there is no `resolve_ticket` or `close_ticket` tool among the three
above, on purpose, and step 4 of the system prompt reinforces it. Auto-closing a ticket when a
score drops is Iteration 3 (not yet built) and per `pm-guidelines.md` §10 must require an
explicit human click — giving the agent a tool that can do it autonomously here would be
building the exact thing the project's HITL guardrail argues against, just with better tooling
ergonomics. When Iteration 3 is scoped, that action gets its own human-approval step in the
Lovable dashboard, not a 4th tool in this list.

### Known gaps (tracked, not hidden)

- No 3-strikes kill switch or Slack failure alert (§1) — this was written for the original
  scheduled-bulk design's unattended-loop risk profile, which no longer applies now that the
  trigger is a human clicking a button once per issue (decision 3 above). Lower priority now,
  worth revisiting only if a bulk/scheduled mode is reintroduced. No Slack credential exists in
  this n8n instance regardless.
- No per-run assertion checking "every score>60 account ends with exactly one ticket" — same
  reasoning; there's no "run" to assert over anymore, just individual on-demand invocations.
- Workflow is **active**, listening on `/webhook/create-jira-ticket`. The dashboard's
  per-issue button and `jira_ticket_key` column (`lovable-prompt.md`) still need to be deployed
  against it — see README's status table for what's done vs. pending on the frontend side.
- `Extract Ticket Key`'s parsing of the Agent's `intermediateSteps` is best-effort (tool-name
  substring matching, defensive JSON parsing) and hasn't been verified against every possible
  shape LangChain might emit — per §2's own "test standalone before trusting" rule, watch the
  first several real runs to confirm `jira_ticket_key` actually lands correctly before relying
  on it.
