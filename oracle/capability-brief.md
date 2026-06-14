# Oracle — Capability: Pre-Call Brief
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/oracle/capability-brief.md
#
# Load when: user requests a brief on a named account.
# Triggers: "brief me on [customer]", "[customer] overview", "what's going on with [customer]",
#           "I have a call with [customer]", customer name + any context-pulling phrase.
#
# Data routing → shared/source-routing.md (which tool per read — don't re-decide here)
# Recovery/escalation → shared/error-handling.md (A recover · B degrade · §0 escalate)
# Signal synthesis → shared/signal-synthesis.md (DO/HEAR/SAY — reasoning governance, not display)
# Footer/voice → shared/output-discipline.md
# Field selects → references/field-registry.md

---

## Scope

**One brief. Full, for every authenticated seat.** No lean/Not-CS variant — server-side
permissions are the data floor, so a seat renders exactly what Gainsight authorizes it to see.
No access at all → don't render a thin brief, escalate via `shared/error-handling.md` §0.

| Tier | Allowed accounts | ARR |
|------|-----------------|-----|
| CSM | Own book + same region (no prompt) · cross-region → soft flag, then render | Unrestricted |
| FLL → Dept Head | Any account — no gate | Unrestricted |

Tier + book scope resolve from `shared/identity.md`. ARR is unrestricted for `brief` (single-account
lookups don't expose portfolio data — no ownership check, no band fallback).

**Cross-region flag (CSM only):**
> *"[Account] is in [REGION] — your territory is [USER_REGION]. Pulling the brief anyway."*

---

## Optional inputs

- **`call_context`** — passed by the scheduler when firing `brief` for a calendar event:
  `{ time_et, meeting_title, attendees: [{name, company}] }`. When present, render the Calendar
  block at the top and use `meeting_title` to seed the timeline `contextual_user_query` and the
  prep meeting-type variant. On-demand calls omit it.

---

## IA Account handling

IA and non-IA accounts are two sides of the same coin — the same parent company represented twice
with different products, separate CS teams, and separate ownership. Always show both sides together.

| Field | IA Account | Non-IA Account |
|-------|-----------|----------------|
| CSM | Real CSM (IA CSM) | Often "Jamf Digital Team" |
| ARR | Populated (often larger) | Populated |
| Customer_Category | **Always null** | Populated |
| ARR_Band | **Always null** | Populated |
| Renewal_Status | **Always null** | Populated |
| Scorecard | Null (unscored) | Populated |
| Relationships/Products | **None** | Products live here |

**When both exist — always combined brief.** Don't ask "which one?" Pull both in parallel, render
together. This applies regardless of who is asking — regular CSM, IA CSM, or leader. No exceptions.

IA CSM book scope: `Csm EQ [ia_csm_gsid]` returns IA accounts correctly — the Csm field is
populated on IA company records the same way as non-IA. IA CSMs see "Products: None" on their side;
the combined brief is how they get the full product picture from the non-IA sibling.

**Combined render — two-panel:**
```
── [Company Name] — Combined Brief ──────────────────────────
IA Account  ·  CSM: [Name]  ·  ARR: $[amount]  ·  Renewal: [date]
Non-IA Account  ·  CSM: [Name]  ·  ARR: $[amount]  ·  Renewal: [date]
Combined ARR: $[total]
── [Company Name] (IA) ──────────────────────────────────────
[IA data — null fields render as —, no error]
── [Company Name] ───────────────────────────────────────────
[Non-IA data — full brief]
```
Staircase: query once on the parent name (no "(IA)" suffix); results span both relationships.

---

## Data sequence

Routing lives in `source-routing.md`. This section says only what the brief needs and how it batches.
Fire everything that has no dependency in parallel; render Batch 1 the moment it returns, don't wait on Staircase.

### Batch 1 — Gainsight (fire on GSID, ~2s)

1. **Resolve + attributes** — `resolve_customer` attributes mode → ARR, CSM, Stage, Renewal Date,
   Segment in one call (source-routing: account attributes). Apply IA handling if matches include an
   (IA) variant. Run in parallel with identity resolution.
2. **Supplement select** — one `run_query` on `company` for brief-specific fields not in attributes
   mode (Customer_Category, ARR band, sentiment, PreviousCsm, engagement fields,
   Engaged_This_Period_by_CSM__gc, Churn_Risk_Identified__gc, Number_of_Escalated_Renewals__gc)
   per `references/field-registry.md`. Dead field removed: `Engagement_Model__gc` (P_5005).
   *(If a probe shows attributes mode already returns a field, drop it here — don't double-fetch.)*
3. **Health + trend** — `ask_scorecard(company_ids=[gsid])`. Read `overall_score` AND
   `overall_score_history` — the history drives the trend arrow (see Render).
4. **Open CTAs** — `fetch_cta_list` where `IsClosed=false AND CompanyId=<gsid>`, with linked objects
   (source-routing: CTAs). Include `CSE_Request__gc`, `Original_Request_Notes__gc`.
   *IsClosed: boolean `false`, not string — string returns all (error-handling A4-adjacent).*
5. **Active SPs** — `fetch_success_plan_list` where `CompanyId=<gsid> AND IsClosed=false`.
6. **Products** — `run_query` on `relationship`: Cloud_Family__gc, Paid_For_Quantity__gc,
   Deployed__gc, Total_Enrolled_Devices__gc, Last_Login__gc, Environment_Type__gc. **Required on
   every brief.**
7. **Recent relevant activity** — `fetch_timeline_activity_list` with
   `contextual_user_query = <meeting purpose>` (from `call_context.meeting_title`, or "recent risk,
   engagement, and open items" on-demand). Semantic-ranked + auto-RAG per source-routing — **this
   replaces the old 90-day run_query window.** Pass explicit select (error-handling A3: `GsCompanyId`,
   `GsCreatedByUserId__gr.Name AS AuthorName`, `NotesPlainText AS Notes`). Limit ≤10.

Render full brief on Batch 1 return with `Staircase signals: ⏳ loading...`.

### Batch 2 — Staircase (fire in parallel with Batch 1, Pattern A)

`staircase_account_lookup(name=customer_name)` immediately — don't wait.
- `confidence == "high"` → `staircase_analyze_account(account_id, query="Current risk, concerns, and recent engagement signals?")` (source-routing B7: single account = narrative drill).
- Low/no confidence → mark unavailable, skip analyze.

**Never `staircase_query` for briefs** — unscoped, misattributed. Pattern A only. Replace the
`⏳` placeholder when analyze returns.

---

## Render — locked skeleton

These sections render **in this order, every brief**. A section with no data renders its empty
state — it never disappears. Only the Renewal context block is conditional (stated trigger below).
No model discretion on structure — this is the format-consistency contract.

### Calendar block (only when `call_context` passed)
```
{time ET} — {meeting title} · {company name}
Attendees: {Name} ({Company}), {Name} ({Company})
────────────────────────────────────────────────
```

### Account header (canonical — UAT-locked 2026-05-11, + trend arrow 2026-06-13)
```
── {Customer Name} ──────────────────────────────────────
CSM: {Csm Name}  ·  {(prev: {PreviousCsmName} — account transfer) if applicable}
Category: {Category}  ·  ARR: {ARR}
{renewal status chip}  Renewal: {date} ({N} days)  ·  Status: {Renewal_Status}
Health: {label} ({score}) {trend}  ·  Sentiment: {grade} (updated {date})
Engaged this period: ✅  ← or ❌ Not engaged this period
```

**Health trend** — from `ask_scorecard.overall_score_history`, latest vs. immediately prior:
- `↑` improved · `↓` declined · `→` no change since last score
- **Insufficient data:** one score only, or empty history → drop the arrow, keep the line, append
  *"(no prior score to compare)"*. Never blank the Health line. *(This is the brief's invariant-section
  rule; it intentionally overrides error-handling B1's "skip trend" for this surface.)*
- Not scored at all → `Health: not scored yet`.

**Engaged chip** — `Engaged_This_Period_by_CSM__gc`: `true`→✅ · `false`→❌ · `null`→omit chip.
**Renewal status chip:** Escalated/Late Escalated/Notice of Churn/Churn → 🚨 [Status] · Late → ⚠️ Late · Open → none.

### Health (covered in header line above — always present)

### Products
```
Products ({count} relationships):
  • {Cloud_Family__gc_PicklistLabel} ({Product_Type__gc}, {env}) — {enrolled}/{paid} enrolled, {deployed}% deployed, last login {date}
  • ⚠️ {Product} — Deployed% < 50%
  • 🚨 {Product} — Deployed% < 10% or last login stale >60d
```
Name = `Cloud_Family__gc_PicklistLabel` (fallback `Product_Type__gc`). Null enrollment → "no
enrollment data", never blank. Exclude Sandbox/Demo/Trial from flag thresholds. **No products →
"No active product relationships."**

### Open CTAs
```
Open CTAs ({count}):
  • {Name} — {Priority}, due {date} ({age} days), {overdue task count} overdue
    CSE Request: "{text}"      ← only if populated
    Request Details: "{text}"  ← only if populated
```
Top 3. **None → "No open CTAs."**

### Active Success Plans
```
Active Success Plans ({count}):
  • {Name} — {percent}% complete, due {date}, type {Type}
```
Top 2. **None → "No active success plans."** >3 → flag "abnormal SP volume — may include stale auto-created plans."

### Recent Activity
```
Recent Activity ({count}, ranked to this call):
  • {date} — {TypeName}: {Subject} ({AuthorName})
```
Semantic-ranked (Batch 1 #7), not chronological dump. Notes → 200 chars, strip HTML/signatures/
forwarded headers first. **None → "No recent activity surfaced."**

### Staircase signals
Replace `⏳` after Batch 2.
- With numerics: `Staircase signals: Health {score} · Sentiment {score} ({grade}). {narrative.}`
- Prose only: drop the score prefix, lead with narrative.
- **Not connected: render "Staircase: not connected"** — don't silently omit (invariant-section rule).

### Renewal context block — CONDITIONAL
Renders only when `RenewalDate` ≤180 days OR `RenewalStatus` is Late/Escalated/Notice of Churn.
Omit entirely if >180 days out AND on track — this is the one section allowed to disappear.
```
🔄 Renewal context:
  Renewal: {date} ({N} days out) · Status: {status} · ARR: {amount}
  {1 sentence on pricing friction / risk from Staircase if present}
  {1 sentence on exec sponsor engagement if in timeline}
  Playbook: {link}  ← only if URL exists
```
Urgency: 🚨 ≤30d or escalated · ⚠️ 31–90d · none 91–180d.

---

## Prep synthesis (always renders)

### Header
`🎯 Prep:` — meeting-type variant from `call_context.meeting_title` and CTA names:

| Keyword | Label |
|---------|-------|
| QBR / EBR / Business Review | `🎯 Prep (QBR):` |
| Renewal / Contract Review | `🎯 Prep (Renewal):` |
| Cadence / Check-in | `🎯 Prep (Cadence):` |
| Onboarding / Kickoff | `🎯 Prep (Onboarding):` |
| Escalation / Incident | `🎯 Prep (Escalation):` |
| None | `🎯 Prep:` |

### Signal synthesis

Apply `shared/signal-synthesis.md` — DO/HEAR/SAY triangulation governs how prep is reasoned.
Labels never appear in output. The verdict surfaces as prose.

**Synthesis inputs for this brief:**
- SAY → scorecard score/trend, CSM sentiment, open CTAs, timeline notes
- HEAR → Staircase analyze result (Batch 2)
- DO → product deployment%, last login, activity frequency

**Output format — prose verdict:**
```
{Plain-language summary of what the signals show, 1–2 sentences}
{If two witnesses disagree: explicit call-out of the gap and what it means for this call}
{1 suggested opening line or concrete action, grounded in the verdict}
```

All three agree + healthy → "Account is in good shape — next cadence touch on [date by category]."
Degradation (Staircase unavailable) → note it, run SAY/DO two-witness check. See signal-synthesis.md.

### Priority within SAY
1. Renewal ≤30 days or escalated → item 1, no exceptions. "Lock the renewal."
2. Meeting-type framing (QBR → open with adoption metrics; escalation → don't open with agenda)
3. Exec sponsor signal — exec-visible CTA or exec named in recent timeline

### Priority within HEAR
4. Staircase sentiment trend, pricing sensitivity, engagement trajectory

### Priority within DO
5. Product adoption gap — Deployed% < 50% at renewal = relationship conversation, not just technical
6. Stalled SP past due
7. Cadence gap — last engagement > cadence window, acknowledge if >30 days

**Voice: CSM coaching register — the intentional default.** Prose, not a data dump. One prep output,
no role branching. Verdict + opening: 1–3 lines max, verb-driven, data-grounded.

---

## Edge cases

- Customer not found / ambiguous → `shared/error-handling.md` A4 (account team check first → context narrowing → ask plainly)
- No access / permission denied → `shared/error-handling.md` §0 (escalate, don't render thin)
- IA variant → always combined brief, no exceptions — two sides of the same coin
- Staircase no numerics → prose-only; not connected → "Staircase: not connected"
- Empty scorecard → "Health: not scored yet"; one score → no arrow + "(no prior score to compare)"
- >3 active SPs → stale-volume flag
- Null enrollment → "no enrollment data"
- `call_context` absent → skip calendar block; timeline query defaults to "recent risk, engagement, open items"
- `PreviousCsm` populated → inline transfer note after CSM name

---

*2026.06.13c — IA handling updated: combined brief is now unconditional (no CSM single-ownership exception). IA and non-IA are always shown together — two sides of the same coin. IA CSM book scope confirmed correct via Csm EQ [gsid] — no identity.md change required.*
*2026.06.13b — DO/HEAR/SAY synthesis framework added to Prep section. SAY (Gainsight record) / HEAR (Staircase comms) / DO (product behavior) triangulation replaces priority-order bullets. Two-witness divergence = the insight. Dead field `Engagement_Model__gc` removed from account header render template (confirmed P_5005, UAT 2026-06-13).*
*2026.06.13 — Rewritten for the lean architecture. ONE full brief (lean/Not-CS variant + Not-CS prep deleted — §0 handles no-access). Prep kept CSM coaching voice as the intentional CS Ops default (one prep, no role branching); role-tailored prep deferred (D3). Health + trend arrow from ask_scorecard history. Timeline → semantic fetch_timeline_activity_list (contextual_user_query), replacing the 90d run_query window. Attributes via resolve_customer attributes mode. Invariant render skeleton (sections render empty states, never vanish; only renewal context conditional). Data routing delegated to source-routing.md, recovery/escalation to error-handling.md, footer/voice to output-discipline.md. Scope simplified to CSM-own-book vs leader-any (resolves from identity.md). Tenant canon kept: IA handling, UAT-locked header, prep priority order.*
*2026.06 (v1.0) — Ported from Sentinel v3.7 references/brief.md; tier model updated.*
