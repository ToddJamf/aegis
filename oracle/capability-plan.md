# Oracle — Capability: Draft Success Plan
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/oracle/capability-plan.md
#
# Load when: user asks to create or draft a success plan for a named account.
# Triggers: "create a success plan for [customer]", "draft a plan for [customer]",
#           "success plan for [customer]", "I need a plan for [customer]".
#
# Data routing → shared/source-routing.md   Recovery/escalation → shared/error-handling.md
# Footer/voice → shared/output-discipline.md   Field selects → references/field-registry.md
# Writes (Phase 2) → references/write-gateway.md

---

## Scope — CS-tier capability

Plan drafting is the **one intentional tier gate** left in Oracle (briefs are open to every seat;
plan is not). Resolve CS role from `shared/identity.md`.

- **CS role (CSM → Dept Head)** → proceed.
- **Authenticated, no CS role** → don't refuse with a dead-end; capability-scope note:
  *"Success plan drafting is a CS capability. You can still pull a full brief on [account] — want that?"*
- **No access / permission denied / no record** → `shared/error-handling.md` §0 escalate.

---

## What a Success Plan is

A structured, goal-oriented engagement plan tied to an account:
- **Plan header** — name, company, type, due date, status, owner
- **Plan Info** — Description, Company Highlights (stakeholders), Action Plan
- **CTAs** — action items with dates, objectives, comments, success criteria, owners
- **Tasks** — the working steps under each CTA, each carrying its accelerator (see CTA discipline)

| Type | Use when |
|------|----------|
| Onboarding | New account, low adoption, getting started |
| Adoption | Existing account with usage gaps or declining health |
| Customer Project | Specific initiative named in timeline (migration, rollout, integration) |
| Consultancy | PS engagement signals present |

---

## Cleanup before create (Bluhm B6)

Before drafting anything new, sweep for hygiene and offer closure first — don't pile a new plan on
top of rot:
- Active SPs **past due >90 days**
- SPs **not updated >90 days**
- Plans/CTAs **owned by someone who's left** (owner inactive in `gsuser`)

Surface them:
> *"Before I draft: [Company] has 2 success plans worth a look first — [Name] (past due 140d), [Name] (owner left Jamf). Close or reassign those, or draft the new one anyway?"*

If clean, skip silently — no empty "nothing to clean up" line.

---

## Signal pull — parallel

Routing per `source-routing.md`. Fire together:
- **Attributes** — `resolve_customer` attributes mode → ARR, CSM, Stage, Renewal, Segment
- **Supplement** — `run_query` on `company` for ARR band, category, Engagement_Model__gc (field-registry)
- **Health** — `ask_scorecard` (score + trend + driving measures)
- **Open CTAs / active SPs** — `fetch_cta_list` (IsClosed=false, linked), `fetch_success_plan_list`
- **Products** — `run_query` on `relationship` (Cloud_Family, Paid_For_Quantity, Deployed, Total_Enrolled_Devices, Last_Login)
- **Recent context** — `fetch_timeline_activity_list` with `contextual_user_query="initiatives, projects, risks, and goals for a success plan"` (semantic, not a blunt window)
- **Stakeholders** — `fetch_customer_contacts` for Company Highlights
- **Staircase** — Pattern A: `staircase_account_lookup` → `staircase_analyze_account` on high-confidence only. Never `staircase_query`.

---

## Plan type inference

| Signal pattern | Type |
|----------------|------|
| New account, low adoption, onboarding CTAs in timeline | Onboarding |
| Existing account + Deployed% < 50% or declining health | Adoption |
| Specific project named in timeline (migration, rollout, SSO, integration) | Customer Project |
| PS engagement signals (consulting CTAs, SOW references) | Consultancy |

Two types match equally → surface both with reasoning, ask. Don't guess.

---

## Synthesis

Pull signals, then present the full draft for confirmation — no pre-draft interview. User edits after seeing it.

### Plan header
| Field | Value |
|-------|-------|
| Name | [Company] + [Type] + goal phrase — "Target — Adoption Plan: Protect Deployment" |
| Company | [Company name] |
| Type | Inferred |
| Due Date | Anchored to renewal if known; else 90 days out |
| Status | Active |
| Owner | Requester (default) |

### Plan Info
- **Description** — TLDR only, 1–3 sentences: customer situation → plan purpose. Not a wall of text.
- **Company Highlights** — stakeholders from `fetch_customer_contacts` + Staircase: name, title, role. Max 5.
- **Action Plan** — 3–5 sentences grounded in signals. No generic filler.

### CTA content discipline (Bluhm B4) — what "good" looks like
Applies in Phase 1 paste-ready output, not just on write.
- **CTA description = TLDR** (1–3 sentences). The detail lives in the tasks.
- **≥2 tasks per CTA.** A CTA with no tasks is a header, not a plan.
- **Tasks carry the accelerator, not the instruction.** A task description holds the *actual* pre-drafted
  material — the email body, the agenda, the discovery-question list, the checklist — never "draft an
  email to the sponsor." Write the email. That's the value over a raw plan.
- **HTML, not markdown,** in rich-text fields (Gainsight renders HTML; markdown shows as literal `**`).
- **No internal labels in customer-facing fields.** Never leak risk tiers, churn scores, or internal
  classifications into a field a customer could see.

### Email-type tasks — Verify Before Sending (Bluhm B5)
Any task that drafts customer-facing email ships with this checklist appended, so the human catches
AI overcommitment before it goes out:
```
✅ Verify before sending:
  □ Commitments here are ones we're authorized to make
  □ Dates are achievable
  □ Tone fits this customer
  □ Recipients confirmed
```

### Default CTA names by type
- **Onboarding** → Information Gathering & Planning · Testing & Implementation · Follow-up
- **Adoption** → Adoption Assessment · Deployment Push · Adoption Review
- **Customer Project** → Project Kickoff · Testing & Implementation · Project Close
- **Consultancy** → Project Scoping · Delivery · Sign-off & Review

Adjust if timeline signals suggest something more specific. **Dates:** divide plan duration in equal
thirds, land on weekdays (nearest Friday if weekend). **Objective category by type:** Onboarding →
Full Implementation / Information Gathering · Adoption → Product Adoption / Popular Workflow
Implementation · Customer Project → Project Outline / Popular Workflow Implementation · Consultancy →
Project Outline / Information Gathering. Per CTA: comments (2–3 sentences from signals) · success
criteria (2–4 specific bullets) · status New · owner = plan owner.

---

## Paste-ready output

```
── PLAN HEADER ──────────────────────────────────────────
Name:       [Plan name]
Company:    [Company name]
Type:       [Plan type]
Due Date:   [YYYY-MM-DD]
Status:     Active
Owner:      [Name]

── DESCRIPTION ──────────────────────────────────────────
[TLDR, 1–3 sentences]

── COMPANY HIGHLIGHTS ───────────────────────────────────
• [Name] — [Title] — [Role]

── ACTION PLAN ──────────────────────────────────────────
[3–5 sentence narrative]

── CTA 1: [Name] ────────────────────────────────────────
Due Date:           [YYYY-MM-DD]
Objective Category: [Category]
Owner:              [Name]
Status:             New
Description:        [TLDR, 1–3 sentences]
Success Criteria:
  • [Criterion]  • [Criterion]
Tasks:
  • [Task name] — [accelerator: the actual drafted email / agenda / script]
  • [Task name] — [accelerator]
    ✅ Verify before sending: [checklist — email tasks only]

── CTA 2 / CTA 3 ────────────────────────────────────────
[same structure]
```

Phase 1: paste-ready output only. **Phase 2 (direct Gainsight writes):** route through
`references/write-gateway.md` — note that writes execute as the connected seat, not as the resolved
tier (verify_downward is courtesy, not control). HTML in rich-text fields, two-step prepare→create
recipe per write-gateway.

---

## Graceful degradation (per error-handling §B)

- Thin Gainsight data → lighter synthesis, flag missing fields inline
- Staircase unavailable → *"⚠ Staircase not connected — stakeholder and risk signals unavailable"*, proceed
- Snowflake unavailable → proceed without license utilization; note only if Adoption type

---

## Edge cases

- Customer not found / ambiguous → `error-handling.md` A4
- No access / permission denied → `error-handling.md` §0
- Cleanup items present → B6 gate above, offer closure first
- Existing active SP of same type → *"[Company] already has an active [Type] plan ([name], [%] complete). New or update existing?"*
- >3 active SPs → stale-volume flag before proceeding
- Renewal date in the past → *"Renewal date is in the past — using 90-day default. Confirm?"*

---

*2026.06.13 — Rewritten for the lean architecture. CS-tier gate kept (one intentional gate); no-access → §0, non-CS → capability-scope note (not a dead-end refusal). Data pulls delegated to source-routing (semantic timeline, ask_scorecard trend, resolve_customer attributes, fetch_customer_contacts stakeholders). Bluhm steals baked in: B6 cleanup-before-create gate, B4 CTA content discipline (TLDR descriptions, ≥2 tasks/CTA with accelerator content not instructions, HTML not markdown, no internal labels), B5 Verify-Before-Sending on email tasks. Tasks now ship with the draft (B4) — supersedes the old "don't generate tasks unprompted." Phase 2 writes flagged: run as connected seat, route via write-gateway.*
*2026.06 (v1.0) — Ported from Sentinel v3.7 references/plan.md; tier model updated.*
