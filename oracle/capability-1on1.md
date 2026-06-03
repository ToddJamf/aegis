# Oracle — Capability: 1:1 Prep
# Aegis v1.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/oracle/capability-1on1.md
#
# Three direction variants in one file. Protocol routes here for all 1:1 prep triggers.
# Direction resolved via detect_direction() in shared/identity.md before this file loads.
#
# Triggers: "1:1 with [name]", "prep me for [name]", "1:1 prep [name]", "prep for my 1:1 with [name]"
#
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

---

## Target resolution (all directions)

1. Extract name from trigger phrase.
2. Run `run_query` on `gsuser` with last-name CONTAINS filter (`references/field-registry.md` name lookup pattern). Do NOT use `resolve_user` — truncates on fuzzy match.
3. Filter to `IsActiveUser=true`.
4. If multiple matches → disambiguate: *"Found [N] people with that name — which one? [list names + team]"*
5. Run `verify_downward` (DOWN directions only). Hard fail if FALSE.
6. Cache target (Gsid, Name, Email, CS_Territory_Team__gc, CS_Territory_Region__gc).

---

---

# DIRECTION DOWN — CSM TARGET

*FLL prepping for a 1:1 with a named CSM on their team.*

---

## Sequence (~6–7 calls)

Fire as parallel batch once `target_csm.Gsid` is confirmed.

**Call 1 — Book compliance summary:**
```python
# Groups by category AND engaged flag. Render layer computes compliance %:
# engaged_count / total_count per category.
# Do NOT use SUM(CASE WHEN ...) — P_5026 confirmed Jamf tenant (2026-05-11).
run_query(
    object_name="company",
    select=[
        "Customer_Category__gc",
        "Engaged_This_Period_by_CSM__gc",
        "COUNT(Gsid) AS AccountCount",
        "SUM(Arr) AS TotalArr"
    ],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [target_csm.Gsid], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    group_by=["Customer_Category__gc", "Engaged_This_Period_by_CSM__gc"],
    limit=20
)
```

**Call 2 — Escalations on their book:**
```python
run_query(
    object_name="company",
    select=["Name", "Arr", "Renewal_Status__gc", "RenewalDate",
            "Churn_Risk_Level__gc", "Open_CSE_Risk_Mitigation__gc",
            "Overall_Score__gc", "CSM_Sentiment__gc"],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [target_csm.Gsid], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "Renewal_Status__gc", "operator": "IN",
         "value": ["1I00VM2QFXVQWI47QSFAZ1SAZJY69GIITKHG",   # Escalated
                   "1I00S7LFYQ5DJNFTY2ZBXMOD710F5XYR7RWC"],  # Late Escalated
         "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=5
)
```

**Call 3 — Open/overdue CTAs on their book:**
```python
# OverdueTaskCount removed — P_5005 dead field confirmed Jamf tenant (2026-05-11).
# Use TotalTaskCount as workload signal; DueDate sort surfaces overdue first.
run_query(
    object_name="call_to_action",
    select=["CompanyId__gr.Name AS CompanyName", "Name",
            "PriorityId__gr.Name AS Priority",
            "DueDate", "TotalTaskCount", "StatusId__gr.Name AS Status"],
    where=[
        {"name": "OwnerId", "operator": "EQ", "value": [target_csm.Gsid], "alias": "A"},
        {"name": "IsClosed", "operator": "EQ", "value": [False], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    sort_by=[{"sortField": "DueDate", "sortOrder": "asc"}],
    limit=10
)
```

**Call 4 — Staircase wins + watch list (Pattern B — multi-account):**
```python
# Collect up to 5 target accounts — escalations first, then top activity gaps
staircase_targets = (escalations or [])[:5] or (activity_gaps or [])[:5]

# Parallel lookup
lookup_results = [staircase_account_lookup(name=acct["Name"]) for acct in staircase_targets]

# Parallel analyze — high-confidence matches only
staircase_signals = [
    staircase_analyze_account(
        account_id=match[0]["account_id"],
        query="Current risk, concerns, or positive signals for this account this period?"
    )
    for match in lookup_results
    if match and match[0].get("confidence") == "high"
]
```
Merge into wins (positive) and watch list (risk). If all lookups miss or Staircase not connected → label ⚠ Staircase unavailable, fall back to Gainsight signals.

**Call 5 — Activity gaps (top 5 by ARR):**
```python
run_query(
    object_name="company",
    select=["Name", "Arr", "Customer_Category__gc",
            "Last_Timeline_Entry_Engagement__gc",
            "Last_Engagement_Timeframe__gc",
            "CSM_Sentiment__gc", "RenewalDate"],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [target_csm.Gsid], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "Engaged_This_Period_by_CSM__gc", "operator": "EQ", "value": [False], "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=5
)
```

**Call 6 — Team compliance benchmark (2-step, parallel with Calls 1–5):**

Step 6a — get all active CSMs on the same team:
```python
team_csms = run_query(
    object_name="gsuser",
    select=["Gsid"],
    where=[
        {"name": "CS_Territory_Team__gc", "operator": "EQ",
         "value": [target_csm.CS_Territory_Team__gc], "alias": "A"},
        {"name": "IsActiveUser", "operator": "EQ", "value": [True], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    limit=50
)
team_gsids = [row["Gsid"] for row in team_csms]
```

Step 6b — team compliance (fires after 6a):
```python
# Do NOT use SUM(CASE WHEN ...) — P_5026.
team_compliance_raw = run_query(
    object_name="company",
    select=[
        "Engaged_This_Period_by_CSM__gc",
        "COUNT(Gsid) AS AccountCount"
    ],
    where=[
        {"name": "Csm", "operator": "IN", "value": team_gsids, "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    group_by=["Engaged_This_Period_by_CSM__gc"],
    limit=5
)
# team_compliance_pct = engaged_row["AccountCount"] / total
```

If `CS_Territory_Team__gc` is empty or Step 6a returns < 2 CSMs → skip Step 6b, mark team benchmark unavailable.

**Wins pull (Gainsight-native, parallel with Calls 1–5):**
```python
run_query(
    object_name="company",
    select=["Name", "Arr", "RenewalDate", "Renewal_Status__gc",
            "CSM_Sentiment__gc", "Overall_Score__gc"],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [target_csm.Gsid], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "CSM_Sentiment__gc", "operator": "IN",
         "value": ["1I005ZWPK9ZBV2E3EMWRPESWDAWVK4R60KIZ",   # A
                   "1I005ZWPK9ZBV2E3EMWIB8BE7T1KHE98E7X1"],  # B
         "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=5
)
```

---

## Render — DOWN / CSM Target

Chat output.

```
📰 1:1 Prep — [CSM Name]  ·  [date]
[N] accounts  ·  [compliance]%  (team avg [X]% · [N] CSMs)  ·  [territory / team]
[⚙ CS Ops — viewing as [target] ([email])]  ← run-as mode only
[⚠ Team benchmark unavailable — CS_Territory_Team__gc not set]  ← if applicable

📋 Book at a glance
  Spotlight: N  ·  Retain: N  ·  Grow: N  ·  React: N
  ARR on escalated accounts: $X  (or "None escalated" if clean)
  Compliance vs team: [CSM]% / [team avg]%  ·  [above avg / below avg / on par]
    ← omit if team benchmark unavailable

🚨 Escalations ([N])  ← omit if none
  [Account]  ·  $ARR  ·  [renewal date]  ·  [status chip]  ·  [🚨 if CSE mitigation active]

🏆 Wins this period  ← omit if no signal
  [Account] — $ARR — [one sentence: what changed or happened]
  Staircase: [1–2 sentence narrative]  ← omit if Staircase unavailable

⚠️ Watch list  ← accounts declining but not formally escalated
  [Account]  ·  $ARR  ·  [last touch / gap signal]

📅 CTAs
  [N] open  ·  [N] overdue
  Top overdue: [Account] — [CTA name] — due [date] ([N] days overdue)  ← top 2

💬 Conversation starters
  [3 data-driven talking points]
```

**Conversation starters — synthesis rules:**

Generate exactly 3. Each grounded in a specific signal, actionable, connected to the why.

Priority order:
1. Escalations — if any exist, at least one starter addresses the top escalation
2. Compliance gap — if below team average OR below 50%: specific %, team avg, hypothesis about why (book composition? React-heavy? methodology?)
3. Wins — recognition/coaching moment
4. Watch list — early warning surfaced
5. Coaching pattern — stale sentiment, CTA backlog, React accounts not logging touches

**Tone:** coaching register. Talking points an FLL would actually use — not a data readout.

```
Example:
💬 Conversation starters
  1. AbleNet is the biggest escalation at $1.36M — CSE mitigation is active. Ask where it stands and what you can unblock.
  2. Compliance is 34% vs team avg 51%. Her React accounts are 45% of the book — likely structural, not behavioral. Walk through how she's logging React touches.
  3. Toast renewed cleanly and sentiment moved to A. Ask her to reflect on what worked — worth extracting for the team.
```

---

## Edge cases — DOWN / CSM Target

- CSM not found → *"Couldn't find [name] in Gainsight. Check spelling or try last name only."*
- CSM not on team → verify_downward fails → refuse. Do not proceed.
- Zero-book CSM → Call 1 returns 0 rows → hard stop: *"No active accounts found for [CSM Name]. Check account assignment or confirm they're a CSM (not an FLL)."*
- No escalations → omit section, render "None escalated" in Book at a glance.
- No wins signal → omit wins section. Don't manufacture content.
- Staircase not connected → skip all Staircase calls, label ⚠ unavailable.
- Book < 5 accounts → note "Small book — compliance % may not be statistically meaningful."
- Team benchmark unavailable → omit team avg, fall back to 50% floor for compliance starter.

---

---

# DIRECTION DOWN — FLL TARGET

*Segment Leader / SR Leadership / Dept Head prepping for a 1:1 with a named FLL direct report.*

Target is an FLL — they manage a team, do not carry accounts directly. Output is team-health based.

---

## Sequence (~5 calls)

**Call 1 fires first** — its result (`fll_team_gsids`) is required by Calls 2–4. Fire Calls 2–5 in parallel after Call 1 returns.

**Call 1 — FLL's team CSM list:**
```python
run_query(
    object_name="gsuser",
    select=["Gsid", "Name", "CS_Territory_Team__gc"],
    where=[
        {"name": "Manager", "operator": "EQ", "value": [target_fll.Gsid], "alias": "A"},
        {"name": "IsActiveUser", "operator": "EQ", "value": [True], "alias": "B"}
    ],
    where_filter_expression="A AND B"
)
```
Cache as `fll_team_gsids`.

**Call 2 — Team health snapshot:**
```python
# Do NOT use SUM(CASE WHEN ...) — P_5026.
run_query(
    object_name="company",
    select=[
        "Overall_Score__gc",
        "Engaged_This_Period_by_CSM__gc",
        "COUNT(Gsid) AS AccountCount",
        "SUM(Arr) AS TotalArr"
    ],
    where=[
        {"name": "Csm", "operator": "IN", "value": fll_team_gsids, "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    group_by=["Overall_Score__gc", "Engaged_This_Period_by_CSM__gc"]
)
```

**Call 3 — Escalations across FLL's team:**
```python
run_query(
    object_name="company",
    select=["Name", "Arr", "Csm__gr.Name AS CSM", "RenewalDate",
            "Renewal_Status__gc", "Churn_Risk_Level__gc",
            "Open_CSE_Risk_Mitigation__gc", "Overall_Score__gc"],
    where=[
        {"name": "Csm", "operator": "IN", "value": fll_team_gsids, "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "Renewal_Status__gc", "operator": "IN",
         "value": ["1I00VM2QFXVQWI47QSFAZ1SAZJY69GIITKHG",
                   "1I00S7LFYQ5DJNFTY2ZBXMOD710F5XYR7RWC"],
         "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=10
)
```

**Call 4 — Engagement compliance per CSM:**
```python
# Groups by CSM AND engaged flag. Render layer computes per-CSM compliance %.
# Do NOT use SUM(CASE WHEN ...) — P_5026.
run_query(
    object_name="company",
    select=[
        "Csm__gr.Name AS CSM",
        "Engaged_This_Period_by_CSM__gc",
        "COUNT(Gsid) AS AccountCount"
    ],
    where=[
        {"name": "Csm", "operator": "IN", "value": fll_team_gsids, "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    group_by=["Csm__gr.Name", "Engaged_This_Period_by_CSM__gc"],
    limit=50
)
```

**Call 5 — Staircase (Pattern B — top escalations):**
```python
staircase_targets = (escalations or [])[:5]
lookup_results = [staircase_account_lookup(name=acct["Name"]) for acct in staircase_targets]
staircase_signals = [
    staircase_analyze_account(
        account_id=match[0]["account_id"],
        query="Current risk, concerns, or positive signals for this account this period?"
    )
    for match in lookup_results
    if match and match[0].get("confidence") == "high"
]
```

---

## Render — DOWN / FLL Target

Chat output.

```
📰 1:1 Prep — [FLL Name]  ·  [date]
[N] CSMs  ·  [N] accounts  ·  Team compliance: [X]%  ·  [territory / region]

📋 Team health
  A: N · B: N · C: N · D: N · F: N
  Total ARR: $X  ·  Engaged this period: N / total ([X]%)
  Compliance range: [lowest CSM]% – [highest CSM]%

🚨 Escalations ([N])  ← omit if none
  [Account]  ·  $ARR  ·  CSM: [Name]  ·  [status chip]  ·  [🚨 CSE mitigation if active]
  Staircase: [1–2 sentence signal]  ← if available

📊 Compliance by CSM  ← omit if all CSMs are above 50%
  [CSM Name]  [compliance]%  [N accounts]
  ...

💬 Conversation starters
  [3 data-driven talking points for the Segment Leader → FLL conversation]
```

**Conversation starters:** Same synthesis rules as DOWN/CSM Target. Adjusted register — Segment Leader coaching an FLL, not an FLL coaching a CSM. Focus: team-level patterns, escalation ownership, compliance outliers, pipeline risk.

---

## Edge cases — DOWN / FLL Target

- FLL not found → *"Couldn't find [name] in Gainsight. Check spelling or try last name only."*
- FLL not in hierarchy → verify_downward fails → refuse.
- FLL has no team (Call 1 returns 0) → *"No active CSMs found reporting to [FLL Name]. Check org structure or contact CS Ops."*
- Staircase not connected → skip, label ⚠ unavailable.

---

---

# DIRECTION UP — ALL TIERS

*Any tier prepping for a 1:1 with their own manager. Output is an upward narrative — tell your own story clearly.*

Goal: not coaching — summarizing your segment/team health for a conversation with your manager. Same underlying data as heat-map, different synthesis register.

---

## Sequence (~4 calls, parallel)

Scope = requester's full downward org (`identity.teamGsids`). If CSM tier (no teamGsids), scope = own book.

**Call 1 — Health distribution:**
```python
run_query(
    object_name="company",
    select=[
        "Overall_Score__gc",
        "COUNT(Gsid) AS AccountCount",
        "SUM(Arr) AS TotalArr"
    ],
    where=[
        {"name": "Csm", "operator": "IN", "value": identity.teamGsids, "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    group_by=["Overall_Score__gc"]
)
```

**Call 2 — Top escalations:**
```python
run_query(
    object_name="company",
    select=["Name", "Arr", "Csm__gr.Name AS CSM", "RenewalDate",
            "Renewal_Status__gc", "Churn_Risk_Level__gc",
            "Open_CSE_Risk_Mitigation__gc", "Overall_Score__gc"],
    where=[
        {"name": "Csm", "operator": "IN", "value": identity.teamGsids, "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "Renewal_Status__gc", "operator": "IN",
         "value": ["1I00VM2QFXVQWI47QSFAZ1SAZJY69GIITKHG",
                   "1I00S7LFYQ5DJNFTY2ZBXMOD710F5XYR7RWC"],
         "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=10
)
```

**Call 3 — Wins (high sentiment + healthy):**
```python
run_query(
    object_name="company",
    select=["Name", "Arr", "Csm__gr.Name AS CSM", "RenewalDate",
            "Renewal_Status__gc", "CSM_Sentiment__gc", "Overall_Score__gc"],
    where=[
        {"name": "Csm", "operator": "IN", "value": identity.teamGsids, "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "CSM_Sentiment__gc", "operator": "IN",
         "value": ["1I005ZWPK9ZBV2E3EMWRPESWDAWVK4R60KIZ",
                   "1I005ZWPK9ZBV2E3EMWIB8BE7T1KHE98E7X1"],
         "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=5
)
```

**Call 4 — Staircase (Pattern B — top escalations):**

Same as DOWN/FLL Call 5 above, scoped to `identity.teamGsids`.

---

## Render — UP

Chat output. Shorter and narrative-forward — this is a summary the requester can speak from, not a coaching brief.

```
📰 1:1 Prep (upward) — [Requester Name] → [Manager Name]  ·  [date]
[N] accounts  ·  $X total ARR  ·  [territory / segment]

📊 Segment health
  A: N · B: N · C: N · D: N · F: N  ($X ARR)
  Compliance: [X]% engaged this period

🚨 Escalations ([N])  ← omit if none
  [Account]  ·  $ARR  ·  CSM: [Name]  ·  [status chip]
  What I need: [1 sentence on what support is needed from the manager]

🏆 Wins worth mentioning
  [Account] — $ARR — [one sentence: what's going well]

💬 What to surface
  [3 items the requester should proactively raise with their manager]
```

**"What to surface" synthesis rules:**

Generate exactly 3. Framed as things the requester should own and surface proactively — not things that will surprise their manager.

Priority order:
1. Escalation needing manager visibility or help — with a specific ask
2. A risk trend (compliance declining, health cluster moving to D/F, renewal cluster concentrating)
3. A win or positive momentum worth naming

**Tone:** narrative, first-person framing. Sounds like a manager speaking about their segment, not reading a report. *"I have 3 escalations — the biggest is [Account] at $Xk. I'm working with CSE on it and need [X] from you."*

---

## Edge cases — UP

- No teamGsids (CSM tier with no direct reports) → scope to own book. Output is account-level, not team-level. Adjust headers accordingly.
- Manager not found in Gainsight (target not in manager chain) → ask once: *"I couldn't confirm [name] is your manager in Gainsight. Are you prepping for your skip-level?"*
- No escalations → render "No escalations — clean portfolio."
- No wins signal → omit wins section.
- Staircase not connected → skip, no label needed in upward output (less critical than coaching outputs).

---

*Oracle capability-1on1.md v1.0 (2026-06-03) — Merged from Gainsight Sentinel v3.7 references/1on1.md, references/1on1-fll.md, references/1on1-up.md. All three direction variants in one file. P_5026 and P_5005 notes preserved from v3.7.*
