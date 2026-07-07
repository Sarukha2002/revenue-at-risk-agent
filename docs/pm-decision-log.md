# Principal PM Decision Log — Revenue-at-Risk

A running log of every non-trivial decision made while building this pilot, captured at
Principal PM altitude: the questions raised, the trade-offs weighed, the decision made and
why, the stakeholder questions this would raise on a real team, and interview questions
this experience could plausibly generate.

This file is appended to after every build step — it is the actual portfolio artifact,
not just the working pipeline.

---

## Step 1 — Install Docker Desktop

**What we built:** Docker Desktop installed locally as the runtime for self-hosting n8n.

**Principal PM questions raised:**
- Why containerize a tool for a solo pilot instead of installing it directly?
- What does this choice cost, and is that cost worth it for a pilot?

**Trade-offs:**
- `npm install n8n -g` → faster to get running, zero extra software, but "works on my laptop" doesn't transfer anywhere else.
- Docker → adds Docker Desktop's resource footprint and a small learning curve, but the container is portable to any real host.

**Decision & rationale:** Chose Docker. Almost every real self-hosted n8n deployment runs containerized (ECS, Kubernetes, Railway, Fly.io, a bare VM). Building the pilot this way means the skill and the artifact both transfer directly to a "real" deployment later — the alternative optimizes for pilot speed at the cost of everything downstream.

**Stakeholder questions (if built with a team):** Who owns this box once it's more than a laptop pilot? Is it Engineering-owned infra (their deploy process, their on-call) or explicitly a PM-owned prototype that is *not* production — and who decides when that changes?

**Interview questions this could generate:**
- *"Walk me through a technical trade-off you made building this, and why."* → Docker vs. bare install is a clean, concrete answer: optimized for portability and production-realism over the fastest possible local setup.
- *"How do you think about the line between a prototype and production infrastructure?"* → Answer with the ownership question above — most teams get burned when a scrappy pilot quietly becomes load-bearing with no assigned owner.

---

## Step 2 — Run n8n via `docker-compose.yml`

**What we built:** `docker-compose.yml` at repo root, running the official n8n image with a named volume for persistence, `N8N_SECURE_COOKIE=false` for local http access, and a configurable timezone. Owner account created via the n8n web UI at `localhost:5678`.

**Principal PM questions raised:**
- What's the backup/disaster-recovery story if the volume is ever lost?
- Is it safe to weaken a security default (`N8N_SECURE_COOKIE=false`) to unblock local dev, and when does that debt have to be paid down?
- What, exactly, does the n8n login protect — and what does it *not* protect?

**Trade-offs:**
- Named Docker volume → zero-config persistence, but lives on one laptop with no redundancy. A managed Postgres backend or scheduled volume backup would be more durable but is overkill for a pilot.
- `N8N_SECURE_COOKIE=false` → unblocks local http development immediately, but is a security default being deliberately turned off, not just a neutral config choice.

**Decision & rationale:**
- Accepted the single-laptop volume for now, but noted the mitigant already in place: `n8n/workflow-revenue-at-risk.json` is a version-controlled export of the workflow definition in this repo — a legitimate, low-effort backup strategy for the workflow logic itself (though not for execution history or credentials, which only live in the volume).
- Accepted the cookie flag as a known, temporary trade-off with an explicit trigger to revisit: **the moment this instance is reachable by anything other than `localhost`** (i.e., the tunnel step, coming next).
- Flagged that n8n's owner login protects the *editor UI* only. Webhook endpoints (e.g. the intake URL Lovable will POST to) are intentionally public by design — login does not gate them. The open question this creates: nothing in Workflow 1 currently authenticates that a webhook call actually came from Lovable rather than a stranger who found the URL. Noted as a gap to close before tunneling, not after.

**Stakeholder questions (if built with a team):**
- "What's our recovery story if this box dies — and who is responsible for testing that recovery actually works, not just that a backup exists?"
- "Who is responsible for validating the authenticity of an inbound webhook call once this is public — is that a security team review item, or does the PM/eng building the pilot own that decision alone?"

**Interview questions this could generate:**
- *"Tell me about a time you had to make a security trade-off to move fast. How did you manage the risk?"* → `N8N_SECURE_COOKIE=false`: named the trade-off explicitly, scoped it to localhost-only, and set an explicit trigger condition for when it must be revisited — rather than either blocking on perfect security or silently shipping the weakened default.
- *"How do you think about backup and disaster recovery for a system you're prototyping?"* → Distinguish between "a backup exists" (workflow JSON in git) and "recovery is proven to work" (untested) — most teams only find out the difference during an actual incident.
- *"What's a gap you found in your own system before anyone else did?"* → The webhook-authentication gap: found by asking "what does this login actually protect?" rather than assuming login = secure.

---

## Template for future steps

```
## Step N — <name>

**What we built:**

**Principal PM questions raised:**

**Trade-offs:**

**Decision & rationale:**

**Stakeholder questions (if built with a team):**

**Interview questions this could generate:**
```
