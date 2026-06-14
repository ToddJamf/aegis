# Mentor — Capability: 1:1 Prep
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/mentor/capability-1on1.md
#
# Three direction variants in one file — the SINGLE dispatch path for all 1:1 prep.
# Direction resolved via detect_direction() in shared/identity.md BEFORE this file loads.
# Data routing → shared/source-routing.md · recovery/escalation → shared/error-handling.md ·
# footer/voice → shared/output-discipline.md · field selects → references/field-registry.md
#
# Triggers (via protocol): "1:1 with [name]", "prep me for [name]", "1:1 prep [name]",
#                          "prep for my 1:1 with [name]"
# Direction → section:
#   DOWN + target CSM-tier   → § DIRECTION DOWN — CSM TARGET (FLL requester)
#   DOWN + target FLL-tier   → § DIRECTION DOWN — FLL TARGET (Segment Leader / SR Leadership / Dept Head)
#   UP                       → § DIRECTION UP — ALL TIERS

---

## Permission

| Direction | Required tier | Additional check |
|-----------|--------------|-----------------|
| DOWN / CSM target | FLL | `verify_downward(requester, target_csm)` — hard fail if not in hierarchy |
| DOWN / FLL target | Segment Leader, SR Leadership, Dept Head | `verify_downward(requester, target_fll)` — hard fail if not in hierarchy |
| UP | Any (CSM and above) | None — prepping for own manager |

`verify_downward` is a relationship check from `shared/identity.md`, not a security boundary — queries
run as the connected seat. No access / unresolved identity → `error-handling.md` §0 escalate.

---

## Target resolution (all directions)

1. Extract name from trigger phrase.
2. `run_query` on `gsuser`, last-name CONTAINS filter (field-registry name-lookup pattern), `IsActiveUser=true`. Do NOT use `resolve_user` — truncates on common first names (error-handling A4).
3. Multiple matches → disambiguate: *"Found [N] people with that name — which one? [list names + team]"*
4. `verify_downward` (DOWN only). Hard fail if FALSE.
5. Cache target (Gsid, Name, Email, CS_Territory_Team__gc, CS_Territory_Region__gc).

---
---

# DIRECTION DOWN — CSM TARGET

*FLL prepping for a 1:1 with a named CSM on their team.*

## Sequence (~6–7 calls, parallel batch once `target_csm.Gsid` is confirmed)

GSID constants (Account_Status, Renewal_Status, CSM_Sentiment) come from `references/field-registry.md`.
**Compliance/health rollups use `group_by` + render-layer math — never `SUM(CASE WHEN ...)` (P_5026 confirmed, Jamf tenant 2026-05-11). `OverdueTaskCount` is a dead field — P_5005; use `TotalTaskCount` + DueDate sort.**

**Call 1 — Book compliance summary:** `run_query` on `company`, select `Customer_Category__gc`, `Engaged_This_Period_by_CSM__gc`, `COUNT(Gsid) AS AccountCount`, `SUM(Arr) AS TotalArr`; where `Csm EQ target_csm.Gsid AND Account_Status__gc EQ <CurrentCustomer GSID>`; `group_by` category + engaged flag. Render computes engaged/total per category.

**Call 2 — Escalations on their book:** `company`, select Name, Arr, Renewal_Status__gc, RenewalDate, Churn_Risk_Level__gc, Open_CSE_Risk_Mitigation__gc, Overall_Score__gc, CSM_Sentiment__gc; where `Csm EQ target AND Account_Status EQ <CC> AND Renewal_Status__gc IN [Escalated, Late Escalated]`; sort Arr desc, limit 5.

**Call 3 — Open/overdue CTAs:** `call_to_action`, select CompanyId__gr.Name, Name, PriorityId__gr.Name, DueDate, TotalTaskCount, StatusId__gr.Name; where `OwnerId EQ target AND IsClosed EQ false`; sort DueDate asc, limit 10.

**Call 4 — Staircase wins + watch (Pattern B, multi-account):** take up to 5 targets (escalations first, then activity gaps); `staircase_account_lookup` each; `staircase_analyze_account` on high-confidence only (query: "Current risk, concerns, or positive signals this period?"). Per source-routing B2: discount expansion readiness 50% when risk ≥4; use evidence date as recency proxy. Merge into wins/watch. None or not connected → label ⚠ Staircase unavailable, fall back to Gainsight signals.

**Call 5 — Activity gaps (top 5 by ARR):** `company`, select Name, Arr, Customer_Category__gc, Last_Timeline_Entry_Engagement__gc, Last_Engagement_Timeframe__gc, CSM_Sentiment__gc, RenewalDate; where `Csm EQ target AND Account_Status EQ <CC> AND Engaged_This_Period_by_CSM__gc EQ false`; sort Arr desc, limit 5.

**Call 6 — Team compliance benchmark (2-step, parallel with 1–5):**
- 6a: `gsuser`, select Gsid; where `CS_Territory_Team__gc EQ target.CS_Territory_Team__gc AND IsActiveUser EQ true`, limit 50 → `team_gsids`.
- 6b (after 6a): `company`, select `Engaged_This_Period_by_CSM__gc`, `COUNT(Gsid) AS AccountCount`; where `Csm IN team_gsids AND Account_Status EQ <CC>`; `group_by` engaged flag. Render computes team %.
- Empty team or <2 CSMs → skip 6b, mark benchmark unavailable.

**Wins pull (Gainsight-native, parallel):** `company` where `Csm EQ target AND Account_Status EQ <CC> AND CSM_Sentiment__gc IN [A, B]`; sort Arr desc, limit 5.

## Render — DOWN / CSM Target (chat)

```
📰 1:1 Prep — [CSM Name]  ·  [date]
[N] accounts  ·  [compliance]%  (team avg [X]% · [N] CSMs)  ·  [territory / team]
[⚠ Team benchmark unavailable — CS_Territory_Team__gc not set]  ← if applicable

📋 Book at a glance
  Spotlight: N · Retain: N · Grow: N · React: N
  ARR on escalated accounts: $X  (or "None escalated")
  Compliance vs team: [CSM]% / [team avg]%  ·  [above/below/on par]  ← omit if benchmark unavailable

🚨 Escalations ([N])  ← omit if none
  [Account] · $ARR · [renewal date] · [status chip] · [🚨 if CSE mitigation active]

🏆 Wins this period  ← omit if no signal
  [Account] — $ARR — [one sentence: what changed]
  Staircase: [1–2 sentence narrative]  ← omit if unavailable

⚠️ Watch list  ← declining but not formally escalated
  [Account] · $ARR · [last touch / gap signal]

📅 CTAs
  [N] open · [N] overdue
  Top overdue: [Account] — [CTA name] — due [date] ([N] days overdue)  ← top 2

💬 Conversation starters
  [exactly 3 data-driven talking points]
```

**Conversation starters — synthesis rules.** Exactly 3, each grounded in a specific signal, actionable, connected to the why. Priority: (1) top escalation; (2) compliance gap if below team avg or <50% — give the %, team avg, and a hypothesis (book composition? React-heavy?); (3) a win as a recognition/coaching moment; (4) watch-list early warning; (5) coaching pattern (stale sentiment, CTA backlog). **Tone:** coaching register — talking points an FLL would actually use, not a data readout.

```
Example (illustrative — fictional account):
💬 Conversation starters
  1. Acme is the biggest escalation at $X — CSE mitigation is active. Ask where it stands and what you can unblock.
  2. Compliance is 34% vs team avg 51%. Her React accounts are ~45% of the book — likely structural, not behavioral. Walk through how she's logging React touches.
  3. Acme renewed cleanly and sentiment moved to A. Ask her to reflect on what worked — worth extracting for the team.
```

## Edge cases — DOWN / CSM Target
- CSM not found → *"Couldn't find [name] in Gainsight. Check spelling or try last name only."*
- Not on team → verify_downward fails → refuse, don't proceed.
- Zero-book CSM (Call 1 = 0 rows) → *"No active accounts for [CSM]. Check assignment or confirm they're a CSM, not an FLL."*
- No escalations → omit section, render "None escalated."
- No wins → omit, don't manufacture.
- Staircase not connected → skip, label ⚠ unavailable.
- Book <5 → "Small book — compliance % may not be meaningful."
- Benchmark unavailable → omit team avg, use 50% floor for the compliance starter.

---
---

# DIRECTION DOWN — FLL TARGET

*Segment Leader / SR Leadership / Dept Head prepping for a 1:1 with a named FLL direct report. Target manages a team, carries no accounts directly — output is team-health based.*

## Sequence (~5 calls). Call 1 first (its result feeds 2–4); fire 2–5 parallel after.

**Call 1 — FLL's team CSM list:** `gsuser`, select Gsid, Name, CS_Territory_Team__gc; where `Manager EQ target_fll.Gsid AND IsActiveUser EQ true` → cache `fll_team_gsids`.

**Call 2 — Team health snapshot:** `company`, select Overall_Score__gc, Engaged_This_Period_by_CSM__gc, `COUNT(Gsid) AS AccountCount`, `SUM(Arr) AS TotalArr`; where `Csm IN fll_team_gsids AND Account_Status EQ <CC>`; `group_by` score + engaged flag. (No SUM(CASE WHEN) — P_5026.)

**Call 3 — Escalations across the team:** `company`, select Name, Arr, Csm__gr.Name AS CSM, RenewalDate, Renewal_Status__gc, Churn_Risk_Level__gc, Open_CSE_Risk_Mitigation__gc, Overall_Score__gc; where `Csm IN fll_team_gsids AND Account_Status EQ <CC> AND Renewal_Status__gc IN [Escalated, Late Escalated]`; sort Arr desc, limit 10.

**Call 4 — Engagement compliance per CSM:** `company`, select Csm__gr.Name AS CSM, Engaged_This_Period_by_CSM__gc, `COUNT(Gsid) AS AccountCount`; where `Csm IN fll_team_gsids AND Account_Status EQ <CC>`; `group_by` CSM + engaged flag, limit 50. Render computes per-CSM %.

**Call 5 — Staircase (Pattern B, top escalations):** lookup + analyze high-confidence only, as Call 4 in CSM-target section.

## Render — DOWN / FLL Target (chat)

```
📰 1:1 Prep — [FLL Name]  ·  [date]
[N] CSMs · [N] accounts · Team compliance: [X]% · [territory / region]

📋 Team health
  A: N · B: N · C: N · D: N · F: N
  Total ARR: $X · Engaged this period: N / total ([X]%)
  Compliance range: [lowest CSM]% – [highest CSM]%

🚨 Escalations ([N])  ← omit if none
  [Account] · $ARR · CSM: [Name] · [status chip] · [🚨 CSE mitigation if active]
  Staircase: [1–2 sentence signal]  ← if available

📊 Compliance by CSM  ← omit if all CSMs above 50%
  [CSM Name]  [compliance]%  [N accounts]

💬 Conversation starters
  [exactly 3 — Segment Leader → FLL register]
```

**Conversation starters:** same synthesis rules, register adjusted (Segment Leader coaching an FLL). Focus: team-level patterns, escalation ownership, compliance outliers, pipeline risk.

## Edge cases — DOWN / FLL Target
- FLL not found → *"Couldn't find [name]. Check spelling or try last name only."*
- Not in hierarchy → verify_downward fails → refuse.
- No team (Call 1 = 0) → *"No active CSMs reporting to [FLL]. Check org structure or contact CS Ops."*
- Staircase not connected → skip, label ⚠ unavailable.

---
---

# DIRECTION UP — ALL TIERS

*Any tier prepping for a 1:1 with their own manager. Output is an upward narrative — tell your own story clearly. Not coaching — summarizing your segment/team health for the conversation up.*

## Sequence (~4 calls, parallel)

Scope = requester's full downward org (`identity.teamGsids`). CSM tier (no teamGsids) → scope = own book (self-scoped reads may use `literal: CURRENT_USER` per source-routing).

**Call 1 — Health distribution:** `company`, select Overall_Score__gc, `COUNT(Gsid)`, `SUM(Arr)`; where `Csm IN identity.teamGsids AND Account_Status EQ <CC>`; `group_by` score.

**Call 2 — Top escalations:** `company`, select Name, Arr, Csm__gr.Name AS CSM, RenewalDate, Renewal_Status__gc, Churn_Risk_Level__gc, Open_CSE_Risk_Mitigation__gc, Overall_Score__gc; where `Csm IN teamGsids AND Account_Status EQ <CC> AND Renewal_Status__gc IN [Escalated, Late Escalated]`; sort Arr desc, limit 10.

**Call 3 — Wins (high sentiment + healthy):** `company` where `Csm IN teamGsids AND Account_Status EQ <CC> AND CSM_Sentiment__gc IN [A, B]`; sort Arr desc, limit 5.

**Call 4 — Staircase (Pattern B, top escalations):** scoped to `identity.teamGsids`.

## Render — UP (chat — shorter, narrative-forward)

```
📰 1:1 Prep (upward) — [Requester] → [Manager]  ·  [date]
[N] accounts · $X total ARR · [territory / segment]

📊 Segment health
  A: N · B: N · C: N · D: N · F: N  ($X ARR)
  Compliance: [X]% engaged this period

🚨 Escalations ([N])  ← omit if none
  [Account] · $ARR · CSM: [Name] · [status chip]
  What I need: [1 sentence on the support needed from the manager]

🏆 Wins worth mentioning
  [Account] — $ARR — [one sentence]

💬 What to surface
  [exactly 3 — things to proactively raise]
```

**"What to surface" rules:** exactly 3, framed as things the requester should own and raise proactively (not surprises). Priority: (1) escalation needing manager visibility/help, with a specific ask; (2) a risk trend; (3) a win worth naming. **Tone:** first-person narrative — sounds like a leader speaking about their segment. *"I have 3 escalations — the biggest is [Account] at $Xk. I'm working with CSE and need [X] from you."*

## Edge cases — UP
- No teamGsids (CSM, no reports) → scope to own book; output is account-level, adjust headers.
- Manager not found / not in chain → ask once: *"I couldn't confirm [name] is your manager in Gainsight. Prepping for your skip-level?"*
- No escalations → "No escalations — clean portfolio."
- No wins → omit.
- Staircase not connected → skip (no label needed upward).

---

*2026.06.13 — Ported from oracle/capability-1on1.md (itself merged from Sentinel v3.7 1on1 / 1on1-fll / 1on1-up). Now the SINGLE dispatch path for all 1:1 prep — direction selects the section. Example conversation-starters genericized (real customer names + ARR removed per repo confidentiality rule). Shared layer wired: identity (detect_direction/verify_downward), source-routing (B2 recency on Staircase), error-handling (§0, P_5026/P_5005 preserved), output-discipline (footer/voice). GSID constants referenced from field-registry. detect_direction/verify_downward live in shared/identity.md, not redefined here.*
