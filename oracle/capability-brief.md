# Oracle — Capability: Pre-Call Brief
# Aegis v1.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/oracle/capability-brief.md
#
# Load when: user requests a brief on a named account.
# Triggers: "brief me on [customer]", "[customer] overview", "what's going on with [customer]",
#           "I have a call with [customer]", customer name + any context-pulling phrase.

---

## Scope (tier-aware)

| Tier | Allowed accounts | Brief variant | ARR |
|------|-----------------|---------------|-----|
| CSM | Own book + same region (no prompt) · Cross-region → soft flag, then render | Full | Unrestricted |
| FLL | Any account — no gate | Full | Unrestricted |
| Segment Leader | Any account — no gate | Full | Unrestricted |
| SR Leadership | Any account — no gate | Full | Unrestricted |
| Department Head | Any account — no gate | Full | Unrestricted |
| Not-CS | Any account — no gate | Lean (see below) | Unrestricted |

**ARR is unrestricted for `brief` — all tiers.** Single-account lookups don't expose portfolio data. No ownership check, no band fallback.

**Cross-region flag (CSM only):**
> *"[Account] is in [REGION] — your territory is [USER_REGION]. Pulling the brief anyway."*

---

## Optional inputs

- **`call_context`** — passed by automated scheduler when firing `brief` for a calendar event. Contains `{ time_et, meeting_title, attendees: [{name, company}] }`. When present, render the Calendar context block at the top. On-demand calls omit this — skip the block.

---

## IA Account handling

IA accounts and non-IA accounts are the same parent company represented twice — different product, separate CS team, separate ownership. They are complementary, not duplicate.

| Field | IA Account | Non-IA Account |
|-------|-----------|----------------|
| CSM | Real CSM | Often "Jamf Digital Team" |
| ARR | Populated (often larger) | Populated |
| Customer_Category | **Always null** | Populated |
| ARR_Band | **Always null** | Populated |
| Renewal_Status | **Always null** | Populated |
| Scorecard | Null (unscored) | Populated |
| Relationships/Products | **None** | Products live here |

**When both accounts exist — generate a combined brief.** Do not ask "which one?" Pull both in parallel.

**Exception — CSM tier, unambiguous ownership:** If the requester owns exactly one of the two accounts, render their account as primary with a sibling note. Offer: *"There's also a sibling account — [Name] (CSM: [name]). Want me to pull that too?"*

**Combined brief render — two-panel format:**
```
── [Company Name] — Combined Brief ──────────────────────────

IA Account  ·  CSM: [Name]  ·  ARR: $[amount]  ·  Renewal: [date]
Non-IA Account  ·  CSM: [Name]  ·  ARR: $[amount]  ·  Renewal: [date]
Combined ARR: $[total]

── [Company Name] (IA) ──────────────────────────────────────
[IA account data — null fields render as —, no error]

── [Company Name] ───────────────────────────────────────────
[Non-IA account data — full brief]
```

Staircase: query once using the parent company name (no "(IA)" suffix). Results likely span both relationships.

---

## Sequence — three-batch approach

### Batch 1 — Gainsight (fire immediately, render on return, ~2s)

1. **Resolve customer** via `resolve_customer`. Apply IA Account handling if multiple matches include an (IA) variant. See `shared/error-handling.md` Section 5. Run in parallel with identity resolution.

2. **Seven parallel calls** (fire as soon as GSID is known):
   - `run_query` on `company` — lean overview select per `references/field-registry.md`: category, renewal date/status, ARR ($), ARR band, sentiment, CSM, PreviousCsm, Engagement_Model__gc, engagement fields, Engaged_This_Period_by_CSM__gc
   - `ask_scorecard(company_ids=[gsid])` — health score
   - `fetch_cta_list` where `IsClosed=false AND CompanyId=<gsid>` — open CTAs (include `CSE_Request__gc` and `Original_Request_Notes__gc` in select)
   - `fetch_success_plan_list` where `CompanyId=<gsid> AND IsClosed=false` — active SPs
   - `fetch_timeline_activity_list` where `GsCompanyId=<gsid>`, `ActivityDate >= 90 days ago`, limit 10 — **90d filter is critical; omitting it dumps years of history to file**
   - `run_query` on `relationship` — full product select per field-registry: Cloud_Family__gc, Paid_For_Quantity__gc, Deployed__gc, Total_Enrolled_Devices__gc, Last_Login__gc, Environment_Type__gc — **required on every brief, do not skip**

   **IsClosed filter:** Pass as boolean `false` in value array, not string `"false"`. String silently returns all CTAs.

3. **Render full brief** as soon as Batch 1 returns. Do not wait for Staircase. Include placeholders:
   ```
   Staircase signals: ⏳ loading...
   ```

### Batch 2 — Staircase (Pattern A — fire in parallel with Batch 1)

Fire `staircase_account_lookup(name=customer_name)` at the same time as Batch 1 — do not wait.

- `confidence == "high"` → fire `staircase_analyze_account(account_id=result[0]["account_id"], query="Current risk, concerns, and recent engagement signals for this account?")`
- Low confidence or no match → mark as unavailable, skip analyze call

Replace `⏳ loading...` with Staircase narrative when analyze returns.

**Do NOT use `staircase_query`** for briefs — it is unscoped and returns misattributed results. Pattern A only.

**Staircase render:**
- With numeric scores: `Staircase signals: Health {score} · Sentiment {score} ({grade}). {narrative.}`
- Without numerics (prose only): drop score prefix, start with narrative directly. No `Health — · Sentiment —` placeholders.
- Not connected: omit placeholder entirely.

---

## Render

### Calendar context block (only when `call_context` is passed)

```
{time ET} — {meeting title} · {company name}
Attendees: {Name} ({Company}), {Name} ({Company})
────────────────────────────────────────────────
```

### Account header (canonical — locked UAT 2026-05-11)

```
── {Customer Name} ──────────────────────────────────────
CSM: {Csm Name}  ·  {(prev: {PreviousCsmName} — account transfer) if applicable}
Category: {Category}  ·  ARR: {ARR}  ·  Engagement Model: {Engagement_Model__gc_PicklistLabel}
{renewal status chip}  Renewal: {date} ({N} days)  ·  Status: {Renewal_Status}
Health: {score}  ·  Sentiment: {grade} (updated {date})
Engaged this period: ✅  ← or ❌ Not engaged this period
```

**Engaged this period chip** — source: `Engaged_This_Period_by_CSM__gc`. `true` → ✅. `false` → ❌ (red register). `null` → omit.

**Renewal status chip:**

| Status | Chip |
|--------|------|
| Escalated / Late Escalated | 🚨 [Status] |
| Notice of Churn / Churn | 🚨 [Status] |
| Late | ⚠️ Late |
| Open | no chip |

### Open CTAs

```
Open CTAs ({count}):
  • {Name} — {Priority}, due {date} ({age} days), {overdue task count} overdue
    CSE Request: "{text}"      ← only if populated
    Request Details: "{text}"  ← only if populated
```

Top 3 shown. CSE Request / Request Details for all tiers (field-registry confirms availability).

### Active Success Plans

```
Active Success Plans ({count}):
  • {Name} — {percent}% complete, due {date}, type {Type}
```

Top 2 shown. If >3 plans, flag: "abnormal SP volume — may include stale auto-created plans."

### Products

```
Products ({count} relationships):
  • {Cloud_Family__gc_PicklistLabel} ({Product_Type__gc}, {env}) — {enrolled}/{paid} enrolled, {deployed}% deployed, last login {date}
  • ⚠️ {Product} — Deployed% < 50%
  • 🚨 {Product} — Deployed% < 10% or last login stale >60d
```

- Use `Cloud_Family__gc_PicklistLabel` as product name. Fallback: `Product_Type__gc`.
- If enrollment data null: render "no enrollment data" — do not silently omit.
- Exclude Sandbox/Demo/Trial from flag thresholds.

### Recent Activity

```
Recent Activity (last 5):
  • {date} — {TypeName}: {Subject} ({AuthorName})
```

Notes truncated to 200 chars. Strip HTML, signatures, and forwarded headers before truncating.

### Staircase signals

Replace inline after Batch 2. See Batch 2 render rules.

### Renewal context block (CSM and Leader tiers only — conditional)

Only renders when `RenewalDate` is within 180 days OR `RenewalStatus` is Late/Escalated/Notice of Churn. Omit entirely if > 180 days out AND status is on track.

```
🔄 Renewal context:
  Renewal: {date} ({N} days out) · Status: {status} · ARR: {amount}
  {1 sentence on pricing friction or risk from Staircase if present}
  {1 sentence on exec sponsor engagement if in timeline}
  Playbook: {link}  ← only if URL exists; omit otherwise
```

Urgency: 🚨 ≤30 days or escalated status · ⚠️ 31–90 days · no flag 91–180 days.

---

## Prep synthesis

### Prep header

```
🎯 Prep:
```

Meeting type variants (detect from `call_context.meeting_title` and CTA names):

| Keyword | Label |
|---------|-------|
| QBR / EBR / Business Review | `🎯 Prep (QBR):` |
| Renewal / Contract Review | `🎯 Prep (Renewal):` |
| Cadence / Check-in | `🎯 Prep (Cadence):` |
| Onboarding / Kickoff | `🎯 Prep (Onboarding):` |
| Escalation / Incident | `🎯 Prep (Escalation):` |
| None detected | `🎯 Prep:` |

### CSM Prep — priority order

1. Renewal ≤30 days or escalated → item 1, no exceptions. "Lock the renewal."
2. Meeting type framing (QBR → open with adoption metrics; escalation → don't open with agenda)
3. Exec sponsor signal — if exec-visible CTA open or exec named in recent timeline
4. Staircase relationship signal — sentiment trend, pricing sensitivity, engagement gap
5. Product adoption gap — Deployed% < 50% at renewal = relationship conversation, not just technical
6. Stalled SP past due — relationship signal for CSMs
7. Cadence gap — last engagement > cadence window → acknowledge if >30 days

Rules: 1–3 actions max. Verb-driven, data-grounded. If healthy: "Account is in good shape — next cadence touch on [date based on category]." **Tone:** coaching register.

### FLL / Leader Prep — same logic as CSM Prep

Leader gets CSM prep logic by default.

### Not-CS Prep — priority order

1. Live issues — open overdue CTAs or health score D/F
2. Adoption gaps — Deployed% < 50%
3. Staircase risk signal

Label: `🎯 Before your call:` (not `🎯 Prep:`). 1–2 items max. No renewal coaching, no cadence guidance. If healthy and no open issues: omit section entirely. **Tone:** neutral briefing register.

---

## Lean brief — Not CS tier

Same Batch 1 + Batch 2 data calls as full brief.

**Include:** account header, open CTAs, products, Staircase signals, `🎯 Before your call:`

**Omit:** renewal context block, success plan detail (surface count only), meeting type coaching

---

## Edge cases

- Customer not found → ambiguous resolution (`shared/error-handling.md` Section 3)
- IA account variant → combined brief; CSM single-ownership exception applies
- Staircase no numerics → prose-only render (no score prefix)
- Staircase unavailable → omit block entirely
- Empty scorecard → render overall score only
- >3 active SPs → flag stale-volume
- Null enrollment data in products → "no enrollment data" — do not leave blank
- `call_context` not passed → skip calendar block and meeting type detection
- `PreviousCsm` populated → inline transfer note after CSM name

---

*Oracle capability-brief.md v1.0 (2026-06-03) — Ported from Gainsight Sentinel v3.7 references/brief.md. Tier model updated: CSE removed (CSM model now covers this), Leader → FLL/Segment Leader/SR Leadership/Dept Head. CSE-specific Batch 3 (Ask Jamf docs) removed pending Oracle CSE capability decision.*
