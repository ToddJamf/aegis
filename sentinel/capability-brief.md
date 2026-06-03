# Sentinel — Capability 3: Pre-call Brief
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-brief.md

Load on demand when user requests a brief on a specific account.

**Triggers:** "brief me on [customer]", "[customer] overview", "what's going on with [customer]", customer name + any context-pulling phrase.

---

## Optional inputs

- **`call_context`** — passed by the automated scheduler. Contains `{ time_et, meeting_title, attendees: [{name, company}] }`. When present, render the Calendar context block at top. On-demand calls omit this param — skip the block.

---

## Scope (tier-aware)

| Tier | Allowed accounts | ARR |
|------|-----------------|-----|
| CSE | Own + same-region (no prompt) · Cross-region → soft flag, then render | Full — unrestricted |
| CSM | Own book + same-region · Cross-region → soft flag, then render | Full — unrestricted |
| Leader | Any account — no gate | Full — unrestricted |
| Not-CS | Any account — no gate | Full — unrestricted |

**ARR policy:** Cap 3 is unrestricted for all tiers. Single-account lookups don't expose portfolio data. No ownership check, no band fallback.

**Cross-region flag (CSE/CSM only):**
```
[Account] is in [REGION] — your territory is [USER_REGION]. Pulling the brief anyway.
```

---

## Sequence — three-batch approach

### Batch 1 — Gainsight (fire immediately, render on return, ~2s)

1. `resolve_customer` — default to Company unless user says "relationship." See `error-handling.md` Section 5.

2. Seven parallel calls (fire as soon as GSID is known):
   - `run_query` on `company`: category, renewal date/status, ARR, ARR band, sentiment, CSM, PreviousCsm, engagement fields
   - `ask_scorecard(company_ids=[gsid])` — health score
   - `fetch_cta_list` where `IsClosed=false AND CompanyId=<gsid>` — include `CSE_Request__gc` and `Original_Request_Notes__gc`
   - `fetch_success_plan_list` where `CompanyId=<gsid> AND IsClosed=false`
   - `fetch_timeline_activity_list` where `GsCompanyId=<gsid>`, `ActivityDate >= 90 days ago`, limit 10 (90d window keeps response inline)
   - `run_query` on `relationship`: `Cloud_Family__gc`, `Paid_For_Quantity__gc`, `Deployed__gc`, `Total_Enrolled_Devices__gc`, `Last_Login__gc`, `Environment_Type__gc` — **required on every brief**
   - *(CSE only)* Ask Jamf lookup — fires after CTA data lands (Batch 3)

   **IsClosed filter:** pass as boolean `false`, not string `"false"`. String comparison silently returns all CTAs.
   **Relationship query:** do NOT include `StatusId__gr.Name` or `RelationshipTypeId__gr.Name` — PICKLIST fields, throws P_5068.

3. Render full brief immediately. Include placeholders:
   ```
   Staircase signals: ⏳ loading...
   ```
   (CSE only)
   ```
   📄 Docs: ⏳ loading...
   ```

### Batch 2 — Staircase (fire in parallel with Batch 1, append on return)

Fire `staircase_query("Current risk and recent concerns for {customer_name}?")` alongside Batch 1. Replace placeholder when it returns.

**Render rules:**
- Numeric scores: `Staircase signals: Health {score} · Sentiment {score} ({grade}). {1–2 sentence narrative.}`
- Qualitative only (no numeric scores): drop score prefix entirely, go straight to narrative. Do NOT render `Health — · Sentiment —`.
- Staircase not connected: omit placeholder entirely.

### Batch 3 — Ask Jamf docs (CSE tier only)

**Only for CSE.** CSM and Leader get the renewal context block instead.

1. Wait for Batch 1 CTA data
2. Scan CTA names, `CSE_Request__gc`, `Original_Request_Notes__gc`, SP names, Staircase signals for topic signals
3. Formulate a specific contextual question — not generic
4. Product filter:
   | Topic | inclusions |
   |---|---|
   | Jamf Pro | `{"metadata": {"product": ["pro"]}}` |
   | K-12 / education | `{"metadata": {"product": ["pro", "school"]}}` |
   | Security | `{"metadata": {"product": ["pro", "protect"]}}` |
   | Mixed / general | omit inclusions |

5. Call:
   ```python
   retrieve_documents(
       okta_user_id="{requester.email}",
       query="{contextual question}",
       inclusions={"metadata": {"product": ["{product list}"]}}
   )
   ```

6. **URL filtering:** keep only results on `learn.jamf.com`, `support.jamf.com`, or `community.jamf.com`. Exclude `internal_articles/` paths.

7. If < 3 public results: supplement with web search scoped to `site:learn.jamf.com OR site:support.jamf.com OR site:community.jamf.com {query}`.

8. If Ask Jamf not connected: omit placeholder and block entirely. No error message.

---

## CTA field enrichment

### CSE CTAs

Include in `fetch_cta_list` select: `CSE_Request__gc`, `Original_Request_Notes__gc`.

Render under each CTA (CSE only):
```
  • {Name} — {Priority}, due {date} ({age} days), {overdue task count} overdue
    CSE Request: "{text}"        ← only if populated
    Request Details: "{text}"    ← only if populated
```

### CSM CTAs

Base fields only (name, priority, due date, overdue task count) until `Reason`, `Success Criteria`, `Executive Sponsor`, `Playbook` fields are confirmed against field-registry.md.

---

## Render template

### Calendar context block (conditional — only when `call_context` passed)

```
{time ET} — {meeting title} · {company name}
Attendees: {Name} ({Company}), {Name} ({Company})
────────────────────────────────────────────────
```

### Account header

```
{Customer Name}  ·  {Category}  ·  {ARR}  ·  Renewal {date} ({status})
CSM: {Csm Name}  ·  Sentiment: {grade} ({last updated})
[⚠️ New to account {N} days — transitioned from {PreviousCsmName}]  ← if CSM transition in last 90 days

Health: {score}/{label} on {scorecard_name}
```

ARR: render full figure for all tiers. No band fallback.

### Open CTAs

```
Open CTAs ({count}):
  • {Name} — {Priority}, due {date} ({age} days), {overdue task count} overdue
    CSE Request: "{text}"        ← CSE tier only; only if populated
    Request Details: "{text}"    ← CSE tier only; only if populated
  ... (top 3)
```

### Active Success Plans

```
Active Success Plans ({count}):
  • {Name} — {percent}% complete, due {date}, type {Type}
  ... (top 2; if >3, flag "abnormal SP volume — may include stale auto-created plans")
```

### Products

```
Products ({count} relationships):
  • {Cloud_Family__gc_PicklistLabel} ({Product_Type__gc}, {Environment_Type__gc_PicklistLabel}) — {Total_Enrolled_Devices__gc}/{Paid_For_Quantity__gc} enrolled, {Deployed__gc}% deployed, last login {Last_Login__gc}
```

Rules:
- Product name: `Cloud_Family__gc_PicklistLabel` (Pro / Protect / Security Cloud / Connect / etc.). Fallback: `Product_Type__gc`.
- Null enrollment data → "no enrollment data" — do not leave blank.
- Flag thresholds: ⚠️ Deployed% < 50% · 🚨 Deployed% < 10% or Last Login stale >60d.
- Exclude Sandbox/Demo/Trial from flag thresholds.

### Recent Activity

```
Recent Activity (last 5):
  • {date} — {TypeName}: {Subject} ({AuthorName})
```

Notes truncated to 200 chars. Strip HTML, signatures, forwarded headers before truncating.

### Staircase signals

Replaced inline after Batch 2. See Batch 2 render rules.

### Renewal context block (CSM and Leader only — conditional)

Only renders when: `RenewalDate` ≤180 days OR `RenewalStatus` is Late/Escalated/Notice of Churn.

```
🔄 Renewal context:
  Renewal: {date} ({N} days out) · Status: {status} · ARR: {amount}
  {1 sentence on pricing friction from Staircase if present}
  {1 sentence on exec sponsor engagement if in timeline}
  Playbook: {link}  ← only if a specific URL exists
```

Urgency: 🚨 ≤30 days / Late/Escalated/Notice of Churn · ⚠️ 31–90 days · (no flag) 91–180 days.

### Docs block (CSE only)

```
📄 Docs:
  • [{Title}]({learn.jamf.com or community URL})
```

3–5 results. Hyperlinks only. No internal_articles/ paths.

---

## Lean brief — Not CS tier

Same Batch 1 + Batch 2 (Staircase). No Batch 3 (Ask Jamf). No Docs block.

**Omit:** Renewal context block, Success plan detail (count only), CSE Request fields, meeting type detection in Prep.

**Staircase:** render as labeled block, not inline narrative.

**Prep label:** `🎯 Before your call:` (not `🎯 Prep:`)

---

## Prep synthesis

### Prep header

Always: `🎯 Prep:` — never `## Prep`, never any other heading.

With meeting type: `🎯 Prep (QBR):`, `🎯 Prep (Renewal):`, `🎯 Prep (Cadence):`, `🎯 Prep (Onboarding):`, `🎯 Prep (Escalation):`

Meeting type detection (from `call_context.meeting_title` or CTA names):
| Detected | Label |
|---|---|
| QBR / EBR / Business Review | `Prep (QBR)` |
| Renewal / Contract Review | `Prep (Renewal)` |
| Cadence / Check-in / Weekly Sync | `Prep (Cadence)` |
| Onboarding / Kickoff / Launch | `Prep (Onboarding)` |
| Escalation / Incident | `Prep (Escalation)` |

### CSE Prep

Priority order: CSE Request text → Technical friction → Open CTAs → Adoption gap → Stalled SP → Staircase risk → Renewal heat (only if ≤60d).

Rules: verb-driven, tied to data. 1–3 actions max. Tone: technical brief, senior engineer prepping a colleague.

### CSM Prep

Priority order: Renewal heat (≤30d is item 1, no exceptions) → Meeting type framing → Exec sponsor signal → Staircase relationship signal → Adoption gap → Success plan progress → Cadence gap.

Rules: same verb-driven structure. Tone: coaching register, manager talking to rep before a high-stakes call.

### Leader Prep

CSM prep logic by default. CSE-style docs available on explicit request.

### Not CS Prep

Priority: Live issues → Adoption gaps → Staircase risk. Label `🎯 Before your call:`. 1–2 items max. Neutral briefing register. Omit if account is healthy.

---

## Edge cases

- Customer not found → `error-handling.md` Section 3
- Empty Staircase → omit Staircase block entirely
- Staircase no numerics → skip score prefix, go to narrative
- Empty scorecard measures → render overall score only
- Mostly-null record → render `—` for nulls
- >3 active SPs → flag stale-volume
- Null enrollment data → "no enrollment data"
- Ask Jamf < 3 public results → supplement with web search fallback
- CSE CTA fields not populated → omit labels silently
- `call_context` not passed → skip calendar block, skip meeting type detection
