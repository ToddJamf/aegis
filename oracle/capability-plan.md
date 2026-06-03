# Oracle — Capability: Draft Success Plan
# Aegis v1.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/oracle/capability-plan.md
#
# Load when: user asks to create or draft a success plan for a named account.
# Triggers: "create a success plan for [customer]", "draft a plan for [customer]",
#           "success plan for [customer]", "I need a plan for [customer]".
# Permission: CSM, FLL, Segment Leader, SR Leadership, Department Head. Not-CS → refused.

---

## What a Success Plan is

A Gainsight Success Plan is a structured, goal-oriented engagement plan tied to a customer account:

- **Plan header** — name, company, type, due date, status, owner
- **Plan Info** — Description, Company Highlights (stakeholders), Action Plan
- **CTAs** — 3 default action items with names, dates, objectives, comments, success criteria, owners
- **Tasks** — optional sub-items (offered, not auto-generated)

Plan types:

| Type | Use when |
|------|----------|
| Onboarding | New account, low adoption, getting started |
| Adoption | Existing account with usage gaps or declining health |
| Customer Project | Specific initiative named in timeline (migration, rollout, integration) |
| Consultancy | PS engagement signals present |

---

## Signal pull — run in parallel

| Source | What to fetch |
|--------|--------------|
| Gainsight `company` | `ask_scorecard`, open CTAs, last 5 timeline entries (90d filter), active SPs, renewal date, ARR band, category, CSM, Engagement_Model__gc |
| Gainsight `relationship` | Product relationships — Cloud_Family, Paid_For_Quantity, Deployed, Total_Enrolled_Devices, Last_Login |
| Staircase (Pattern A) | Sentiment, recent concerns, engagement recency — `staircase_account_lookup` → `staircase_analyze_account` on high-confidence match only. Do NOT use `staircase_query`. |

**Timeline call:** `ActivityDate >= 90 days ago`, limit 5.

---

## Plan type inference

| Signal pattern | Recommended type |
|----------------|-----------------|
| New account, low adoption, onboarding CTAs in timeline | Onboarding |
| Existing account + Deployed% < 50% or declining health | Adoption |
| Specific project named in timeline (migration, rollout, SSO, integration) | Customer Project |
| PS engagement signals (consulting CTAs, SOW references) | Consultancy |

If signals match two types equally, surface both with reasoning and ask. Do not guess.

---

## Synthesis

Pull signals first. Present the full draft immediately for confirmation — no pre-draft interview. User edits after seeing the draft.

### 1. Plan header

| Field | Value |
|-------|-------|
| Name | [Company] + [Type] + goal phrase — e.g., "Target — Adoption Plan: Protect Deployment" |
| Company | [Company name] |
| Type | Inferred from signals |
| Due Date | Anchored to renewal date if known; default 90 days from today if unknown |
| Status | Active |
| Owner | Plan requester (default) |

### 2. Plan Info sections

- **Description** — 1–2 sentences: customer situation → plan purpose.
- **Company Highlights** — key stakeholders from Gainsight + Staircase: name, title, engagement role. Max 5.
- **Action Plan** — 3–5 sentences grounded in signals. No generic filler.

### 3. CTAs (3 defaults)

Default CTA names by type:
- **Onboarding** → Information Gathering & Planning · Testing & Implementation · Follow-up
- **Adoption** → Adoption Assessment · Deployment Push · Adoption Review
- **Customer Project** → Project Kickoff · Testing & Implementation · Project Close
- **Consultancy** → Project Scoping · Delivery · Sign-off & Review

Adjust names if timeline signals suggest something more specific.

**CTA dates:** Divide plan duration into equal thirds. Land on weekdays — move to nearest Friday if weekend.

**Objective category by type:** Onboarding → Full Implementation / Information Gathering · Adoption → Product Adoption / Popular Workflow Implementation · Customer Project → Project Outline / Popular Workflow Implementation · Consultancy → Project Outline / Information Gathering.

Per CTA: Comments (2–3 sentences from signals) · Success criteria (2–4 specific bullets) · Status: New · Owner: plan owner.

### 4. Tasks

Do not generate unprompted. If asked after reviewing the draft: suggest 3–4 task names per CTA, confirm which to keep, write 1-sentence descriptions.

---

## Paste-ready output format

```
── PLAN HEADER ──────────────────────────────────────────
Name:       [Plan name]
Company:    [Company name]
Type:       [Plan type]
Due Date:   [YYYY-MM-DD]
Status:     Active
Owner:      [Name]

── DESCRIPTION ──────────────────────────────────────────
[1–2 sentences]

── COMPANY HIGHLIGHTS ───────────────────────────────────
• [Name] — [Title] — [Role]
...

── ACTION PLAN ──────────────────────────────────────────
[3–5 sentence narrative]

── CTA 1: [Name] ────────────────────────────────────────
Due Date:           [YYYY-MM-DD]
Objective Category: [Category]
Owner:              [Name]
Status:             New

Comments:
[2–3 sentences]

Success Criteria:
• [Criterion]
• [Criterion]
• [Criterion]

── CTA 2: [Name] ────────────────────────────────────────
[same structure]

── CTA 3: [Name] ────────────────────────────────────────
[same structure]
```

Phase 1: paste-ready output only. No direct Gainsight writes — direct writes are Phase 2.

---

## Graceful degradation

- **Thin Gainsight data** → lighter synthesis, flag missing fields inline
- **Staircase unavailable** → proceed with warning: *"⚠ Staircase not connected — stakeholder and risk signals unavailable"*
- **Snowflake unavailable** → proceed without license utilization; note only if Adoption plan type

---

## Edge cases

- Customer not found → `shared/error-handling.md` Section 3
- Existing active SP of same type → flag: *"[Company] already has an active [Type] plan ([name], [%] complete). Create new or update existing?"*
- >3 active SPs → flag stale-volume before proceeding
- Renewal date in the past → prompt: *"Renewal date is in the past — using 90-day default. Confirm?"*

---

*Oracle capability-plan.md v1.0 (2026-06-03) — Ported from Gainsight Sentinel v3.7 references/plan.md. Tier updated: CSM/FLL/Segment Leader/SR Leadership/Dept Head. CSE tier removed pending Oracle CSE capability decision.*
