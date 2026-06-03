# Radar — Capability: Activity Gap / Compliance
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/radar/capability-gaps.md
#
# Two variants in one file — routed by tier.
# CSM: "activity gap", "stale accounts", "haven't I touched" → My Gaps
# FLL: "team activity gap", "team compliance", "who needs coaching" → Team Compliance

---

## Routing

| Tier | Trigger | Variant |
|------|---------|---------|
| CSM | "activity gap" / "haven't I touched" / "stale accounts" / "engagement compliance" | § MY GAPS |
| FLL | "team activity gap" / "team compliance" / "team activity report" / "who needs coaching" / "coaching signals" | § TEAM COMPLIANCE |

---

---

# MY GAPS — CSM Activity Gap Scan

**Single query:** own book + `Engaged_This_Period_by_CSM__gc=false`. Sort by `Last_Timeline_Entry_Engagement__gc` ascending (longest gap first).

```python
run_query(
    object_name="company",
    select=[
        "Name", "Gsid", "Customer_Category__gc",
        "Arr", "ARR_Band__gc",
        "RenewalDate", "Renewal_Status__gc",
        "Last_Timeline_Entry_Engagement__gc",
        "Last_Timeline_Entry_CSM_Email__gc",
        "Last_Engagement_Timeframe__gc",
        "CSM_Sentiment__gc", "Overall_Score__gc",
        "Engagement_Model__gc",
    ],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [<requester.Gsid>], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "Engaged_This_Period_by_CSM__gc", "operator": "EQ", "value": [False], "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Last_Timeline_Entry_Engagement__gc", "sortOrder": "asc"}],
    limit=100
)
```

### Cadence map

| Category | Expected cadence | Overdue threshold |
|----------|-----------------|-------------------|
| Spotlight | Bi-weekly | 14 days |
| Retain | 3 weeks | 21 days |
| Grow | Monthly | 30 days |
| React | ~6 months (CTA-driven) | 180 days |

### Per-account row format

```
[Account] · [Category] · Last: [date] · Overdue ~[X] days ([N]-day cadence) → [Action]
```

Action line: derive from signals — escalation status, sentiment, renewal proximity. Start with sharpest available signal. No generic filler.

**React accounts:** Last touch = MAX(`Last_Timeline_Entry_Engagement__gc`, `Last_Timeline_Entry_CSM_Email__gc`). Note: *"React cadence is CTA-driven — email counts as a valid touch."*

**Output:** Chat response.

---

---

# TEAM COMPLIANCE — FLL Activity Gap & Coaching

**Triggers:** "team activity gap" / "team compliance" / "who needs coaching" / "coaching signals" / "team performance patterns" / "[region] compliance".

**Permission:** FLL tier. Own team by default.

---

## Scope resolution

| User says | Scope |
|-----------|-------|
| "My team" / default | `Csm IN [identity.teamGsids]` |
| Region name | `CS_Territory_Region__gc = <region>` ⚠ May return P_5005 on company object — fall back to named-team scope if filter fails |
| "All" / "global" | No Csm filter |

---

## Sequence (~4 calls)

**Call 1 — CSM list (from identity.teamGsids or scope re-query)**

**Call 2 — Compliance roll-up:**
```python
# Groups by CSM AND engaged flag. Render layer computes per-CSM compliance %.
# Do NOT use SUM(CASE WHEN ...) — P_5026 confirmed Jamf tenant (2026-05-11).
run_query(
    object_name="company",
    select=[
        "Csm__gr.Name AS CSM",
        "Engaged_This_Period_by_CSM__gc",
        "COUNT(Gsid) AS AccountCount",
        "SUM(Arr) AS TotalArr"
    ],
    where=[
        {"name": "Csm", "operator": "IN", "value": [<scope_csm_gsids>], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    group_by=["Csm__gr.Name", "Engaged_This_Period_by_CSM__gc"],
    limit=50
)
```

Per-CSM compute: compliance % = engaged_count / total_count. Team avg across all CSMs.
Flag: 🔴 if >1σ below team avg. Flag: 🚨 if 0% compliance.

**Call 3 — Stale accounts ($50k+, last engagement 60–90d or >90d):**
```python
run_query(
    object_name="company",
    select=["Name", "Arr", "Csm__gr.Name AS CSM",
            "Customer_Category__gc",
            "Last_Timeline_Entry_Engagement__gc",
            "CSM_Sentiment__gc", "RenewalDate"],
    where=[
        {"name": "Csm", "operator": "IN", "value": [<scope_csm_gsids>], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "Arr", "operator": "GTE", "value": [50000], "alias": "C"},
        {"name": "Engaged_This_Period_by_CSM__gc", "operator": "EQ", "value": [False], "alias": "D"}
    ],
    where_filter_expression="A AND B AND C AND D",
    sort_by=[{"sortField": "Last_Timeline_Entry_Engagement__gc", "sortOrder": "asc"}],
    limit=20
)
```

**Call 4 — Top timeline themes (coaching signals):**
```python
fetch_timeline_activity_list(
    where={"conditions": [
        {"fieldName": "GsCreatedByUserId", "operator": "IN",
         "value": [<scope_csm_gsids>], "alias": "A"},
        {"fieldName": "ActivityDate", "operator": "GTE",
         "value": ["<60d_ago>T00:00:00Z"], "alias": "B"}
    ], "expression": "A AND B"},
    contextual_user_query="recurring themes, blockers, customer initiatives, deployment topics, expansion conversations, escalation drivers, product feedback, renewal discussions",
    select=["Subject", "ActivityDate", "TypeName",
            "GsCompanyId__gr.Name AS CompanyName",
            "GsCreatedByUserId__gr.Name AS AuthorName",
            "NotesPlainText AS Notes"],
    limit=40
)
```

Extract top 3 themes via keyword clustering on Subject + Notes.

---

## Render — Team Compliance

HTML artifact.

```
📊 Team Compliance — [FLL Name]  ·  [date]
Team avg: [X]%  ·  [N] CSMs  ·  [N] accounts  ·  $X total ARR

Compliance by CSM:
  [CSM Name]  [compliance]%  [N accounts]  🔴 (if flagged)
  ...

⚠️ Stale $50k+ accounts (not engaged this period):
  [Account]  ·  [CSM]  ·  $ARR  ·  Last: [date] ([N] days)  ·  [renewal date if <180d]
  ...

💡 Coaching signals (top 3 themes from timeline):
  [Theme 1] — [supporting evidence from timeline entries]
  [Theme 2]
  [Theme 3]
```

---

*Radar capability-gaps.md v2.0 (2026-06-03) — Ported from Gainsight Sentinel v3.7 references/my-book.md (my-gaps) and references/compliance.md. P_5026 note preserved.*
