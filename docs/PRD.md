# 1-Pager: Luna AI (V1)

**Author / Date:** — / Oct 2026 · **Status:** Draft for shaping review

## 1. Description
Luna AI is an embeddable chat widget businesses drop on their website. Visitors chat with an AI agent grounded in the business's own knowledge; a human on the business team takes over when the AI can't handle it.

## 2. Problem
**Website visitors leave without answers because small teams can't staff live chat around the clock — so questions go unanswered, and support only happens during business hours.**

*Not solving:* email/ticketing, help centers, product tours, mobile apps, billing/pricing.

## 3. Why this is a real problem
*Fill in — 3 to 5 strong data points, quantitative and qualitative. Quality over quantity.*
- [ ] % of visitor messages arriving outside staffed hours
- [ ] Current first-response times / abandonment rate on unanswered questions
- [ ] Cost of staffing live chat for a small team
- [ ] Churn or lost conversion tied to unanswered pre-sales questions
- [ ] Why now: [e.g. grounded answering is finally cheap and accurate enough for small teams]

## 4. Success
*Two metrics. More than that and none of them filter decisions.*

1. **AI resolution rate:** ≥ 60% of on-topic conversations resolved with zero human replies, within 30 days of a site going live.
2. **Handoff quality:** human takeover occurs in ≤ 10% of conversations, and every escalated conversation is claimed by a human within 5 minutes during staffed hours.

*Guardrail metric (not a goal):* 0 off-topic questions answered by the agent.

## 5. Audience
Small businesses (1–5 support people) with a marketing/support website and a docs-heavy knowledge base.

**Not:** enterprise support orgs, marketplaces with two-sided chat.

## 6. What (directional — for shaping, not a build order)

**Flow A — Owner uploads knowledge (ingestion):**
Dashboard (asks for upload URL) → Object Storage (direct upload via presigned URL) → Function (parse + chunk) → OpenAI (embed chunks) → Postgres/pgvector (save chunks + vectors).

**Flow B — Visitor asks a question (answering):**
Widget → Next.js Server (verify visitor + limits) → AI Gateway (classify) → OpenAI (embed question) → Postgres (best chunks) → AI Gateway (answer from chunks) → PartyKit (stream to widget).

Off-topic messages ("what's 5×10", "write me a song") **terminate at the classify step** — this is the enforcement point for the §4 guardrail metric. Every off-topic answer is burned tokens and new attack surface.

**Flow C — Handoff to a human (state machine):**
AI answering → Waiting (unsure, or visitor asks) → Human (AI stops replying) → Closed.

**Shaping test (≤10 moving pieces):** counted end-to-end, V1 is currently ~15 — snippet · widget · PartyKit · server API · dashboard · auth · presign · ingest function · embed · pgvector search · classifier · answerer · stream · state machine · takeover. **V1 must cut.** Proposed cuts: Markdown-only ingestion, no doc update/delete/re-embed, no roles beyond owner.

## 7. Non-Goals (V1)
Email/ticketing · help center · product tours · mobile apps · billing and pricing plans · multi-agent routing/teams · analytics beyond the two success metrics.

## 8. Key Risks & Open Questions
- **Missing confidence step.** Flow B has no confidence computation, yet Flow C's "unsure → Waiting" requires one. *Fix:* the answer step returns a groundedness score; the score drives the Waiting transition, alongside an explicit visitor request.
- **Confidence threshold:** who sets it, and what happens at the boundary? This is the product's entire trust curve.
- **Latency budget:** classify + retrieve + generate vs. "instant chat" expectations.
- **Ingestion reality:** PDFs are messy — is V1 Markdown-only?
- **Embed constraints:** cross-origin, CSP, mobile web, performance on the host site.
- **Lifecycle holes:** no re-embed on doc edit, no delete/retract, no ingestion failure surface — a stale knowledge base silently produces wrong answers.
- **No citations in Flow B.** The answer is generated from chunks with no source surfaced to the visitor or the human — hurts trust and degrades handoff.
- **"Waiting" lacks an owner and a timeout.** What happens if nobody claims it?
- **No abuse ceiling.** Server "checks visitor + limits" is unspecified — per-visitor rate limits and workspace spend caps need numbers.
- Have engineering and design been in the room for shaping yet? If not, no build starts.

## 9. Timeline (proposed)
Shaping session (design + eng) → 1 week review/align → build → exit criterion: **widget live on a real site, AI answering from uploaded docs, human able to take over.**

## 10. First Principles
- **Tenant isolation is non-negotiable.** Every query, chunk, conversation and file is scoped to a workspace (SQL per workspace). A cross-workspace leak is a company-ending bug, not a ticket.
- **Truthfulness beats fluency.** An unanswered question is recoverable; a confident wrong answer is not.
- **Conversation cost is a product metric, not an infra detail.** Off-topic answers and unbounded retries are the two ways V1 bills itself to death.

## 11. Technical Architecture & Constraints
- **Hub:** Next.js server — API + AI orchestration.
- **Realtime:** PartyKit WebSockets, streaming to widget and dashboard.
- **Data:** Neon — Better-auth (owner/team sessions), AI Gateway (classify + answer), Postgres + pgvector (app data + search).
- **Files:** Object Storage, presigned uploads.
- **Deploy:** Vercel, `production` + `development` branches. Migrations run on `development` first — required gate for pgvector schema changes.

## Appendix
- **System diagram:** visitor chat widget (iframe, visitor token) on the customer's site ↔ PartyKit realtime ↔ Dashboard (owner + team) ↔ Next.js server hub ↔ Neon (Auth, AI Gateway, Postgres + pgvector, Functions, Object Storage) + OpenAI embeddings; deployed on Vercel.
- **Build against:** `docs/` for specs and this PRD.
