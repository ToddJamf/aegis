# Sentinel — Capability 5: Activity Gap Scan (CSM Book)
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-activity-gap-csm.md

Load on demand when a CSM asks about stale accounts or engagement compliance.

**Triggers:** "activity gap" / "haven't I touched" / "stale accounts" / "engagement compliance" — CSM tier only.

If a Leader asks → do NOT load this file. Route to Cap 6 (`capability-leader-activity-gap.md`).

---

## Scope

Own book only.

---

## Query

```python
run_query(
    object_name="company",
    select=[
        "Name", "Csm", "Customer_Category__gc", "Arr", "ARR_Band__gc",
        "CSM_Sentiment__gc", "Engaged_This_Period_by_CSM__gc",
        "Last_Timeline_Entry_Engagement__gc",
        "Last_Timeline_Entry_CSM_Email__gc",
        "RenewalDate", "Status"
    ],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [<user.Gsid>], "alias": "A"},
        {"name": "Engaged_This_Period_by_CSM__gc", "operator": "EQ", "value": [False], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    sort_by=[{"sortField": "Last_Timeline_Entry_Engagement__gc", "sortOrder": "asc"}],
    limit=100
)
```

---

## Output per account

Name · category · last engagement date · days since · sentiment · ARR band.

---

## React accounts — email composite

For React accounts: composite last touch = MAX(`Last_Timeline_Entry_Engagement__gc`, `Last_Timeline_Entry_CSM_Email__gc`). Label as "Last touch (call or email)."

An account with a recent email is NOT stale even if no call was logged. Don't flag it.

---

## Cadence reference

Render as a footer reminder on every Cap 5 output:

*"Spotlight = bi-weekly · Retain = 3 weeks · Grow = monthly · React = ~6mo CTA-driven (email counts)"*

---

## Output format

Chat (inline) for ≤10 stale accounts. HTML artifact for >10 accounts. Include cadence reminder footer in both formats.
